<script>
import SortableTable from '@/components/SortableTable';
import Loading from '@/components/Loading';
import { getClusterStatus, getAlertConditions } from '@/product/opni/utils/requests/alerts';
import CloneToClustersDialog from './dialogs/CloneToClustersDialog';
import { getClusters } from '~/product/opni/utils/requests/management';

export default {
  components: {
    CloneToClustersDialog, Loading, SortableTable
  },
  async fetch() {
    await this.load();
    await this.updateStatuses();
  },

  data() {
    return {
      clusters:          [],
      loading:           false,
      statsInterval:     null,
      conditions:        [],
      isAlertingEnabled: false,
      headers:           [
        {
          name:          'status',
          labelKey:      'tableHeaders.status',
          value:         'status',
          formatter:     'StatusBadge',
          width:     100
        },
        {
          name:          'nameDisplay',
          labelKey:      'tableHeaders.name',
          value:         'nameDisplay',
          width:    250
        },
        {
          name:          'description',
          labelKey:      'tableHeaders.description',
          value:         'description',
          width:    250
        },
        {
          name:      'type',
          labelKey:  'tableHeaders.type',
          value:     'typeDisplay'
        },
      ]
    };
  },

  created() {
    this.$on('remove', this.onRemove);
    this.$on('clone', this.onClone);
    this.statsInterval = setInterval(this.updateStatuses, 10000);
  },

  beforeDestroy() {
    this.$off('remove');
    this.$off('clone');
    if (this.statsInterval) {
      clearInterval(this.statsInterval);
    }
  },

  methods: {
    onRemove() {
      this.load();
    },

    onClone(alarm) {
      this.$refs.dialog.open(alarm, alarm.clusterId);
    },

    async load() {
      try {
        this.loading = true;

        const status = (await getClusterStatus()).state;
        const isAlertingEnabled = status === 'Installed';

        this.$set(this, 'isAlertingEnabled', isAlertingEnabled);

        if (!isAlertingEnabled) {
          return;
        }

        const clusters = await getClusters(this);

        this.$set(this, 'clusters', clusters);

        this.$set(this, 'conditions', await getAlertConditions(this, clusters));
        await this.updateStatuses();
      } finally {
        this.loading = false;
      }
    },
    async updateStatuses() {
      const promises = this.conditions.map(c => c.updateStatus());

      try {
        await Promise.all(promises);
      } catch (ex) {}
    }
  },
};
</script>
<template>
  <Loading v-if="loading || $fetchState.pending" />
  <div v-else>
    <header>
      <div class="title">
        <h1>Alarms</h1>
      </div>
      <div v-if="isAlertingEnabled" class="actions-container">
        <n-link class="btn role-primary" :to="{name: 'alarm-create'}">
          Create
        </n-link>
      </div>
    </header>
    <SortableTable
      v-if="isAlertingEnabled"
      :rows="conditions"
      :headers="headers"
      :search="false"
      default-sort-by="expirationDate"
      key-field="id"
      group-by="clusterDisplay"
    >
      <template #group-by="{group: thisGroup}">
        <div v-trim-whitespace class="group-tab">
          Cluster: {{ thisGroup.ref }}
        </div>
      </template>
    </SortableTable>
    <div v-else class="not-enabled">
      <h4>
        Alerting must be enabled to use Alarms. <n-link :to="{name: 'alerting-backend'}">
          Click here
        </n-link> to enable alerting.
      </h4>
    </div>
    <CloneToClustersDialog ref="dialog" :clusters="clusters" @save="load" />
  </div>
</template>

<style lang="scss" scoped>
::v-deep {
  .nowrap {
    white-space: nowrap;
  }

  .monospace {
    font-family: $mono-font;
  }
}

.not-enabled {
    text-align: center;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100%;
  }
</style>
