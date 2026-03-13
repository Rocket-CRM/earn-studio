<template>
  <div class="ec-group-card" :class="{ 'ec-group-card--expanded': isExpanded }">
    <div class="ec-group-card__header" @click="toggleExpand">
      <div class="ec-group-card__connect-dot" :ref="el => $emit('dot-ref', { groupId: group?.id, el })"></div>
      <div class="ec-group-card__header-left">
        <div class="ec-group-card__icon" :style="iconStyle">
          <svg width="18" height="18" viewBox="0 0 20 20" fill="none">
            <rect x="3" y="3" width="5" height="14" rx="1.5" fill="currentColor" opacity="0.7"/>
            <rect x="10" y="6" width="5" height="11" rx="1.5" fill="currentColor" opacity="0.5"/>
          </svg>
        </div>
        <span class="ec-group-card__name">{{ group?.name || 'Untitled Group' }}</span>
        <span class="ec-group-card__badge" v-if="conditionCount > 0">
          <svg width="12" height="12" viewBox="0 0 20 20" fill="none">
            <circle cx="10" cy="10" r="3" fill="currentColor"/>
            <path d="M10 3v4M10 13v4M3 10h4M13 10h4" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
          </svg>
          {{ conditionCount }}
        </span>
        <span class="ec-group-card__link-badge" v-if="linkedFactorCount > 0" :title="`${linkedFactorCount} earn factor(s) linked`">
          <svg width="12" height="12" viewBox="0 0 20 20" fill="none">
            <path d="M8.5 11.5a4 4 0 005.657 0l2.828-2.828a4 4 0 00-5.657-5.657L10 4.343" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
            <path d="M11.5 8.5a4 4 0 00-5.657 0L3.015 11.328a4 4 0 005.657 5.657L10 15.657" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
          </svg>
          {{ linkedFactorCount }}
        </span>
      </div>
      <div class="ec-group-card__header-right">
        <button class="ec-group-card__action-link" @click.stop="$emit('add-condition', group)">
          + Add Condition
        </button>
        <button class="ec-group-card__icon-btn" @click.stop="$emit('edit-group', group)" title="Edit group">
          <svg width="16" height="16" viewBox="0 0 20 20" fill="none">
            <path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/>
          </svg>
        </button>
        <button class="ec-group-card__icon-btn ec-group-card__chevron" :class="{ 'ec-group-card__chevron--up': isExpanded }">
          <svg width="16" height="16" viewBox="0 0 20 20" fill="none">
            <path d="M6 8l4 4 4-4" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </button>
      </div>
    </div>

    <div v-if="isExpanded && conditions?.length" class="ec-group-card__body">
      <table class="ec-group-card__table">
        <thead>
          <tr>
            <th>Type</th>
            <th>Items</th>
            <th>Logic</th>
            <th>Threshold type</th>
            <th>Excess</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="cond in conditions" :key="cond?.id">
            <td>{{ formatEntityType(cond?.entity || cond?.filter_type) }}</td>
            <td>
              <span class="ec-group-card__items-badge">
                {{ getItemCount(cond) }}
                <svg width="10" height="10" viewBox="0 0 20 20" fill="none">
                  <circle cx="10" cy="10" r="7" stroke="currentColor" stroke-width="2" fill="none"/>
                </svg>
              </span>
            </td>
            <td>{{ cond?.operator || 'OR' }}</td>
            <td>
              {{ formatThresholdType(cond?.threshold_unit) }}
              <button
                v-if="cond?.threshold_unit"
                class="ec-group-card__threshold-icon"
                :title="`Min: ${cond?.min_threshold || '—'}, Max: ${cond?.max_threshold || '—'}`"
              >
                <svg width="12" height="12" viewBox="0 0 20 20" fill="none">
                  <rect x="3" y="3" width="14" height="14" rx="2" stroke="currentColor" stroke-width="1.5" fill="none"/>
                  <path d="M7 10h6" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                </svg>
              </button>
            </td>
            <td>{{ cond?.apply_to_excess_only ? 'Yes' : 'No' }}</td>
          </tr>
        </tbody>
      </table>
    </div>

    <div v-if="isExpanded && !conditions?.length" class="ec-group-card__empty">
      No conditions defined yet.
    </div>
  </div>
</template>

<script>
import { computed } from 'vue';

const ENTITY_TYPE_LABELS = {
  product_product: 'Product',
  product_sku: 'SKU',
  product_brand: 'Brand',
  product_category: 'Category',
  store: 'Store',
  store_attribute_set: 'Store Set',
  tier: 'Tier',
  persona: 'Persona',
};

const THRESHOLD_LABELS = {
  amount: 'Purchase amount',
  quantity_primary: 'Qty (primary)',
  quantity_secondary: 'Qty (secondary)',
};

export default {
  props: {
    group: { type: Object, default: () => ({}) },
    conditions: { type: Array, default: () => [] },
    linkedFactorCount: { type: Number, default: 0 },
    expanded: { type: Boolean, default: false },
    iconColor: { type: String, default: null },
  },
  emits: ['add-condition', 'edit-group', 'toggle-expand', 'dot-ref'],
  setup(props, { emit }) {
    const isExpanded = computed(() => props.expanded);
    const conditionCount = computed(() => props.conditions?.length || 0);

    const iconColors = ['#2C6ECB', '#D82C0D', '#8A6116', '#008060', '#6D28D9', '#0D9488'];
    const iconStyle = computed(() => {
      const color = props.iconColor || iconColors[Math.abs(hashCode(props.group?.id || '')) % iconColors.length];
      return { background: `${color}15`, color };
    });

    function hashCode(str) {
      let hash = 0;
      for (let i = 0; i < str.length; i++) {
        hash = ((hash << 5) - hash) + str.charCodeAt(i);
        hash |= 0;
      }
      return hash;
    }

    function toggleExpand() {
      emit('toggle-expand', props.group?.id);
    }

    function formatEntityType(entity) {
      return ENTITY_TYPE_LABELS[entity] || entity || '—';
    }

    function getItemCount(cond) {
      const ids = cond?.entity_ids || cond?.filter_ids || [];
      return Array.isArray(ids) ? ids.length : 0;
    }

    function formatThresholdType(unit) {
      return THRESHOLD_LABELS[unit] || '—';
    }

    return {
      isExpanded,
      conditionCount,
      iconStyle,
      toggleExpand,
      formatEntityType,
      getItemCount,
      formatThresholdType,
    };
  },
};
</script>

<style scoped lang="scss">
@import 'polaris-weweb-styles';

.ec-group-card {
  @include polaris-card;
  position: relative;
  overflow: visible;

  &--expanded {
    .ec-group-card__header {
      border-bottom: 1px solid var(--p-color-border);
    }
  }

  &__connect-dot {
    position: absolute;
    left: -8px;
    top: 50%;
    transform: translateY(-50%);
    width: 8px;
    height: 8px;
    border-radius: var(--p-border-radius-full);
    background: var(--p-color-border);
    border: 2px solid var(--p-color-bg-surface);
    z-index: 5;
  }

  &__header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: var(--p-space-300) var(--p-space-400);
    cursor: pointer;
    transition: background var(--p-motion-duration-100) var(--p-motion-ease);

    &:hover {
      background: var(--p-color-bg-surface-hover);
    }
  }

  &__header-left {
    display: flex;
    align-items: center;
    gap: var(--p-space-300);
    min-width: 0;
  }

  &__header-right {
    display: flex;
    align-items: center;
    gap: var(--p-space-200);
    flex-shrink: 0;
  }

  &__icon {
    width: 32px;
    height: 32px;
    min-width: 32px;
    border-radius: var(--p-border-radius-200);
    display: flex;
    align-items: center;
    justify-content: center;
  }

  &__name {
    @include polaris-text-body;
    font-weight: var(--p-font-weight-semibold);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__badge {
    @include polaris-badge-info;
    font-size: 11px;
    padding: 2px 6px;
    gap: 2px;
  }

  &__link-badge {
    @include polaris-badge-default;
    font-size: 11px;
    padding: 2px 6px;
    gap: 2px;
    background: var(--p-color-bg-fill-info-secondary);
    color: var(--p-color-text-info);
  }

  &__action-link {
    @include polaris-button-plain;
    font-size: var(--p-font-size-300);
    white-space: nowrap;
  }

  &__icon-btn {
    width: 28px;
    height: 28px;
    min-width: 28px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: transparent;
    border: none;
    border-radius: var(--p-border-radius-100);
    color: var(--p-color-icon);
    cursor: pointer;
    transition: background var(--p-motion-duration-100) var(--p-motion-ease);

    &:hover {
      background: var(--p-color-bg-surface-hover);
      color: var(--p-color-icon-hover);
    }
  }

  &__chevron {
    transition: transform var(--p-motion-duration-200) var(--p-motion-ease);

    &--up {
      transform: rotate(180deg);
    }
  }

  &__body {
    padding: var(--p-space-200) var(--p-space-300);
  }

  &__table {
    @include polaris-table;
    font-size: var(--p-font-size-300);

    th {
      @include polaris-table-header;
      padding: var(--p-space-200) var(--p-space-300);
      font-size: var(--p-font-size-275);
      text-transform: uppercase;
      letter-spacing: 0.5px;
    }

    td {
      @include polaris-table-cell;
      padding: var(--p-space-200) var(--p-space-300);
      font-size: var(--p-font-size-300);
    }

    tr:last-child td {
      border-bottom: none;
    }
  }

  &__items-badge {
    display: inline-flex;
    align-items: center;
    gap: 3px;
    background: var(--p-color-bg-fill-critical-secondary);
    color: var(--p-color-text-critical);
    padding: 1px 6px;
    border-radius: var(--p-border-radius-full);
    font-size: 11px;
    font-weight: var(--p-font-weight-medium);
  }

  &__threshold-icon {
    display: inline-flex;
    align-items: center;
    background: transparent;
    border: none;
    color: var(--p-color-icon);
    cursor: pointer;
    padding: 0;
    margin-left: 2px;
  }

  &__empty {
    @include polaris-text-body-subdued;
    text-align: center;
    padding: var(--p-space-400);
  }
}
</style>
