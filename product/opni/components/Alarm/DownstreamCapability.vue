<script>
import ArrayListSelect from '@/components/form/ArrayListSelect';
import LabeledSelect from '@/components/form/LabeledSelect';
import UnitInput from '@/components/form/UnitInput';
import Loading from '@/components/Loading';
import { loadClusters, loadChoices } from './shared';
import { AlertType } from '~/product/opni/models/alerting/Condition';

const TYPE = 'downstreamCapability';
const CONSTS = {
  TYPE,
  ENUM:        AlertType.DOWNSTREAM_CAPABILTIY,
  TYPE_OPTION: {
    label: 'Downstream Capabiltiy',
    value: TYPE
  },
  DEFAULT_CONFIG: {
    downstreamCapability: {
      clusterId: { id: '' }, capabilityState: [], for: '30s'
    }
  },
};

export default {
  ...CONSTS,

  components: {
    ArrayListSelect,
    LabeledSelect,
    Loading,
    UnitInput,
  },

  props: {
    value: {
      type:     Object,
      required: true
    }
  },

  async fetch() {
    await Promise.all([this.loadChoices(), this.loadClusters()]);
  },

  data() {
    return {
      ...CONSTS,
      clusterOptions: [],
      choices:        { clusters: [] },
      error:          '',
    };
  },

  methods: {
    async loadChoices() {
      await loadChoices(this, this.TYPE, this.ENUM);
    },

    async loadClusters() {
      await loadClusters(this);
    },
  },

  computed: {
    downstreamCapabilityClusterOptions() {
      const options = this.clusterOptions;

      if (!options.find(o => o.value === this.value.clusterId.id)) {
        this.$set(this.value.clusterId, 'id', options[0]?.value || '');
      }

      return options;
    },
    downstreamCapabilityFor: {
      get() {
        return Number.parseInt(this.value.for || '0');
      },

      set(value) {
        this.$set(this.value, 'for', `${ (value || 0) }s`);
      }
    },
    downstreamCapabilityStateOptions() {
      if (!this.value.clusterId) {
        return [];
      }

      const options = this.choices.clusters[this.value.clusterId.id]?.states || [];

      if ((!this.value.capabilityState || this.value.capabilityState.length === 0) && !options.find(o => o === this.value.state)) {
        this.$set(this.value, 'capabilityState', [options[0]] || []);
      }

      return options;
    },
  },
};
</script>
<template>
  <Loading v-if="$fetchState.pending" />
  <div v-else>
    <h4>
      Target Metric
    </h4>
    <div class="row mt-10">
      <div class="col span-6">
        <LabeledSelect v-model="value.clusterId.id" label="Cluster" :options="downstreamCapabilityClusterOptions" :required="true" />
      </div>
      <div class="col span-6">
        <UnitInput v-model="downstreamCapabilityFor" label="Duration" suffix="s" :required="true" />
      </div>
    </div>
    <div class="row mt-10">
      <div class="col span-12">
        <ArrayListSelect
          v-model="value.capabilityState"
          label="State"
          :disabled="downstreamCapabilityStateOptions.length === 0"
          :options="downstreamCapabilityStateOptions"
          :required="true"
          add-label="Add State"
        />
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
</style>
