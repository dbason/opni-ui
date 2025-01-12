<script>
import SortableTable from '@/components/SortableTable';
import { getClusters } from '@/product/opni/utils/requests/management';
import { getClusterStats } from '@/product/opni/utils/requests';
import CapabilityButton from '@/product/opni/components/CapabilityButton';
import Loading from '@/components/Loading';
import { getClusterStatus as getMonitoringBackendStatus } from '@/product/opni/utils/requests/monitoring';
import { getLoggingCluster } from '@/product/opni/utils/requests/logging';
import { isEmpty } from 'lodash';
import EditClusterDialog from './dialogs/EditClusterDialog';
import UninstallCapabilitiesDialog from './dialogs/UninstallCapabilitiesDialog';
import CancelUninstallCapabilitiesDialog from './dialogs/CancelUninstallCapabilitiesDialog';
import CantDeleteClusterDialog from './dialogs/CantDeleteClusterDialog';

export default {
  components: {
    CapabilityButton,
    CancelUninstallCapabilitiesDialog,
    CantDeleteClusterDialog,
    EditClusterDialog,
    Loading,
    SortableTable,
    UninstallCapabilitiesDialog,
  },
  async fetch() {
    await this.load();
    await this.loadStats();
  },

  data() {
    return {
      loading:                      false,
      statsInterval:                null,
      isMonitoringBackendInstalled: false,
      isLoggingBackendInstalled:    false,
      clusters:                     [],
      headers:                      [
        {
          name:          'status',
          labelKey:      'tableHeaders.status',
          sort:          ['status.message'],
          value:         'status',
          formatter:     'StatusBadge'
        },
        {
          name:          'nameDisplay',
          labelKey:      'tableHeaders.name',
          sort:          ['nameDisplay'],
          value:         'nameDisplay',
          width:         250,
          formatter:     'TextWithClass',
          formatterOpts: {
            getClass(row, value) {
              // Value could either be a cluster ID in a UUID format or a
              // friendly name set by the user, if available. If the value is
              // a cluster ID, display it in a monospace font.
              // This regex will match UUID versions 1-5.
              const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

              return uuidRegex.test(value) ? 'monospace' : '';
            }
          }
        },
        {
          name:          'id',
          labelKey:      'tableHeaders.id',
          sort:          ['id'],
          value:         'id',
        },
        {
          name:          'capabilities',
          labelKey:      'tableHeaders.capabilities',
          sort:          ['capabilities'],
          value:         'capabilities',
          formatter:     'ListBubbles'
        },
      ]
    };
  },

  created() {
    this.$on('remove', this.onClusterDelete);
    this.$on('edit', this.openEditDialog);
    this.$on('copy', this.copyClusterID);
    this.$on('uninstallCapabilities', this.openUninstallCapabilitiesDialog);
    this.$on('cancelUninstallCapabilities', this.openCancelUninstallCapabilitiesDialog);
    this.$on('cantDeleteCluster', this.openCantDeleteClusterDialog);
    this.statsInterval = setInterval(this.loadStats, 10000);
  },

  beforeDestroy() {
    this.$off('remove');
    this.$off('edit');
    this.$off('copy');
    this.$off('uninstallCapabilities');
    this.$off('cancelUninstallCapabilities');
    this.$off('cantDeleteCluster');
    if (this.statsInterval) {
      clearInterval(this.statsInterval);
    }
  },

  methods: {
    onClusterDelete() {
      this.load();
    },

    openEditDialog(cluster) {
      this.$refs.dialog.open(cluster);
    },

    openUninstallCapabilitiesDialog(cluster, capabilities) {
      this.$refs.capabilitiesDialog.open(cluster, capabilities);
    },

    openCancelUninstallCapabilitiesDialog(cluster, capabilities) {
      this.$refs.cancelCapabilitiesDialog.open(cluster, capabilities);
    },

    openCantDeleteClusterDialog(cluster) {
      this.$refs.cantDeleteClusterDialog.open(cluster);
    },

    copyClusterID(cluster) {
      this.$copyText(cluster.id);
    },

    cancelCapabilityUninstall(cluster, capabilities) {
      cluster.clearCapabilityStatus(capabilities);
    },

    async load() {
      try {
        this.loading = true;
        this.$set(this, 'clusters', await getClusters(this));
      } finally {
        this.loading = false;
      }
    },
    async loadStats() {
      await Promise.all(this.clusters.map(c => c.updateCabilityLogs()));

      try {
        const [monitoringStatus, loggingStatus] = await Promise.all([getMonitoringBackendStatus(), getLoggingCluster()]);

        this.$set(this, 'isMonitoringBackendInstalled', monitoringStatus.state !== 'NotInstalled');
        this.$set(this, 'isLoggingBackendInstalled', !isEmpty(loggingStatus));
      } catch (ex) {
        this.$set(this, 'isMonitoringBackendInstalled', false);
        this.$set(this, 'isLoggingBackendInstalled', false);
      }

      try {
        if (this.isMonitoringBackendInstalled) {
          const details = await getClusterStats(this);

          this.clusters.forEach((cluster) => {
            this.$set(cluster, 'stats', details.find(d => d.userID === cluster.id));
          });
        }
      } catch (ex) {}
    },

    async onDialogSave() {
      this.$set(this, 'clusters', await getClusters(this));
      await this.loadStats();

      this.$refs.capabilitiesDialog.close(false);
      this.$refs.cancelCapabilitiesDialog.close(false);
    }
  },
};
</script>
<template>
  <Loading v-if="loading || $fetchState.pending" />
  <div v-else>
    <header>
      <div class="title">
        <h1>Agents</h1>
      </div>
      <div class="actions-container">
        <n-link class="btn role-primary" :to="{ name: 'agent-create' }">
          Add
        </n-link>
      </div>
    </header>
    <SortableTable
      :rows="clusters"
      :headers="headers"
      :search="false"
      default-sort-by="expirationDate"
      key-field="id"
      :sub-rows="true"
    >
      <template #col:capabilities="{row}">
        <td>
          <CapabilityButton label="Monitoring" type="metrics" :cluster="row" :is-backend-installed="isMonitoringBackendInstalled" />
          <CapabilityButton label="Logging" type="logs" :cluster="row" :is-backend-installed="isLoggingBackendInstalled" />
        </td>
      </template>
      <template #sub-row="{row, fullColspan}">
        <tr v-if="row.displayLabels.length > 0" class="sub-row">
          <td :colspan="fullColspan" class="cluster-status">
            Labels:
            <span v-for="label in row.displayLabels" :key="label" class="bubble ml-5">
              {{ label }}
            </span>
          </td>
        </tr>
      </template>
    </SortableTable>
    <EditClusterDialog ref="dialog" @save="load" />
    <UninstallCapabilitiesDialog ref="capabilitiesDialog" @save="onDialogSave" @cancel="cancelCapabilityUninstall" />
    <CancelUninstallCapabilitiesDialog ref="cancelCapabilitiesDialog" @save="onDialogSave" />
    <CantDeleteClusterDialog ref="cantDeleteClusterDialog" />
  </div>
</template>

<style lang="scss" scoped>
::v-deep {
  .main-row {
    border-top: 1px solid var(--sortable-table-top-divider);
  }

  .sub-row {
    &, td {
      border-bottom: none;
    }
  }

  .nowrap {
    white-space: nowrap;
  }

  .monospace {
    font-family: $mono-font;
  }

  .cluster-status {
    padding-left: 40px;
  }

  .capability-status {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: flex-start;
  }
}
</style>
