<script>

export default {
  components: { },
  props:      {
    item: {
      type:     Object,
      required: true
    },
    selected: {
      type:    Boolean,
      default: false
    }
  },
  methods: {
    removeFavorite() {
      this.$store.dispatch('type-map/removeFavorite', this.type.name);
    },
  },
  computed: {
    iconClass() {
      if (!this.item.icon) {
        return null;
      }

      return `icon-${ this.item.icon }`;
    },

    hasChildren() {
      return this.item.children && this.item.children.some(c => c.display);
    }
  }
};
</script>

<template>
  <n-link
    :to="item.route"
    tag="li"
    class="child nav-type"
    :class="{[`depth-${item.depth}`]: true, selected}"
  >
    <a>
      <span class="label" :class="{'no-icon': !item.icon}">
        <i v-if="iconClass" class="icon icon-fw" :class="iconClass" />
        {{ item.label }}
      </span>
      <i v-if="hasChildren" class="icon toggle icon-chevron-down"></i>
    </a>
  </n-link>
</template>
<style lang="scss" scoped>
li {
  list-style-type: none;
}

A I {
  position: relative;
  color: var(--muted);
}

A {
  display: flex;
  flex-direction: row;

  justify-content: space-between;
  align-items: center;
}

.selected {
  padding: 0;

  A, A I {
    color: var(--body-text);
  }

  A {
    background-color: var(--nav-active);
  }
}

.child {
    margin: 0 var(--outline) 0 0;

    .label {
      align-items: center;
      grid-area: label;
      display: flex;
      overflow: hidden;
      text-overflow: ellipsis;

      &.no-icon {
        padding-left: 3px;
      }

      ::v-deep .highlight {
        background: var(--diff-ins-bg);
        color: var(--body-text);
        padding: 2px;
      }
    }

    A {
      display: grid;
      grid-template-areas: "label count";
      grid-template-columns: auto auto;
      grid-column-gap: 5px;
      font-size: 14px;
      line-height: 22px;
      padding: 7.5px 7px 7.5px 10px;
      margin: 0 0 0 -3px;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
      color: var(--body-text);

      &:hover {
        background: var(--nav-hover);
        text-decoration: none;

        ::v-deep .icon {
          color: var(--body-text);
        }
      }
    }

    .favorite {
      grid-area: favorite;
      font-size: 12px;
      position: relative;
    }

    .count {
      grid-area: count;
      font-size: 12px;
      text-align: right;
      justify-items: center;
      padding-right: 4px;
    }

    $leftMultiplier: 8px;

    &.depth-2 .label {
      padding-left: $leftMultiplier * 2;
    }

    &.depth-3 .label {
      padding-left: $leftMultiplier * 3;
    }

    &.depth-4 .label {
      padding-left: $leftMultiplier * 4;
    }

    & A {
      padding: 3px 7px 3px 10px;
    }

    &.nav-type:not(.depth-1) {
      A {
        font-size: 13px;
        padding: 2px 7px 2px 10px;
      }

      ::v-deep .label I {
        padding-right: 2px;
      }
    }
  }
</style>
