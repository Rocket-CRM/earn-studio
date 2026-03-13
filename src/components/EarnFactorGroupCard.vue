<template>
  <div class="ef-group-card" :class="{ 'ef-group-card--expanded': isExpanded }">
    <div class="ef-group-card__header" @click="toggleExpand">
      <div class="ef-group-card__header-left">
        <div class="ef-group-card__icon">
          <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
            <rect x="2" y="4" width="7" height="5" rx="1" fill="currentColor" opacity="0.8"/>
            <rect x="2" y="11" width="7" height="5" rx="1" fill="currentColor" opacity="0.6"/>
            <rect x="11" y="4" width="7" height="5" rx="1" fill="currentColor" opacity="0.4"/>
            <rect x="11" y="11" width="7" height="5" rx="1" fill="currentColor" opacity="0.3"/>
          </svg>
        </div>
        <span class="ef-group-card__name">{{ group?.name || 'Untitled Group' }}</span>
        <span class="ef-group-card__badge" v-if="factorCount > 0">
          <svg width="12" height="12" viewBox="0 0 20 20" fill="none">
            <path d="M10 2C10.5523 2 11 2.44772 11 3V8H16C16.5523 8 17 8.44772 17 9V11C17 11.5523 16.5523 12 16 12H11V17C11 17.5523 10.5523 18 10 18C9.44772 18 9 17.5523 9 17V12H4C3.44772 12 3 11.5523 3 11V9C3 8.44772 3.44772 8 4 8H9V3C9 2.44772 9.44772 2 10 2Z" fill="currentColor"/>
          </svg>
          {{ factorCount }}
        </span>
      </div>
      <div class="ef-group-card__header-right">
        <button class="ef-group-card__action-link" @click.stop="$emit('add-factor', group)">
          + Add earn factor
        </button>
        <button class="ef-group-card__icon-btn" @click.stop="$emit('edit-group', group)" title="Edit group">
          <svg width="16" height="16" viewBox="0 0 20 20" fill="none">
            <path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/>
          </svg>
        </button>
        <button class="ef-group-card__icon-btn ef-group-card__chevron" :class="{ 'ef-group-card__chevron--up': isExpanded }">
          <svg width="16" height="16" viewBox="0 0 20 20" fill="none">
            <path d="M6 8l4 4 4-4" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </button>
      </div>
    </div>

    <div v-if="isExpanded" class="ef-group-card__body">
      <div
        v-for="factor in factors"
        :key="factor?.id"
        class="ef-factor-row"
        :ref="el => setFactorRef(factor?.id, el)"
        @mouseenter="hoveredFactorId = factor?.id"
        @mouseleave="hoveredFactorId = null"
      >
        <div class="ef-factor-row__left">
          <div class="ef-factor-row__type-icon" :class="factor?.earn_factor_type === 'rate' ? 'ef-factor-row__type-icon--rate' : 'ef-factor-row__type-icon--multiplier'">
            <svg v-if="factor?.earn_factor_type === 'rate'" width="14" height="14" viewBox="0 0 20 20" fill="none">
              <path d="M10 2l2.39 4.84L17.27 7.6l-3.64 3.54.86 5.01L10 13.77l-4.49 2.36.86-5L2.73 7.6l4.88-.76L10 2z" fill="currentColor"/>
            </svg>
            <svg v-else width="14" height="14" viewBox="0 0 20 20" fill="none">
              <path d="M10 3L12.5 8H17L13.5 11.5L15 17L10 14L5 17L6.5 11.5L3 8H7.5L10 3Z" fill="currentColor"/>
            </svg>
          </div>
          <div class="ef-factor-row__info">
            <div class="ef-factor-row__name">{{ factor?.earn_factor_amount ? getFactorDisplayName(factor) : 'Unnamed Factor' }}</div>
            <div class="ef-factor-row__meta">
              <span class="ef-factor-row__currency">{{ getCurrencyLabel(factor) }}</span>
              <span class="ef-factor-row__type-label">({{ factor?.earn_factor_type === 'rate' ? 'Base rate' : 'Multiplier' }})</span>
              <template v-if="factor?.earn_factor_type === 'rate' && factor?.earn_factor_amount">
                <span class="ef-factor-row__separator">|</span>
                <span>Rate : ฿{{ factor.earn_factor_amount }} = 1 point</span>
              </template>
              <template v-if="factor?.window_start || factor?.window_end">
                <span class="ef-factor-row__separator">|</span>
                <span>{{ formatDateRange(factor?.window_start, factor?.window_end) }}</span>
              </template>
            </div>
          </div>
        </div>
        <div class="ef-factor-row__right">
          <span v-if="factor?.earn_factor_type === 'multiplier' && factor?.earn_factor_amount" class="ef-factor-row__multiplier">
            Total Multiplier : <strong>{{ factor.earn_factor_amount }}x</strong>
          </span>
          <button class="ef-group-card__icon-btn" @click="$emit('edit-factor', factor)" title="Edit factor">
            <svg width="16" height="16" viewBox="0 0 20 20" fill="none">
              <path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/>
            </svg>
          </button>
          <button class="ef-group-card__icon-btn" @click.stop="toggleFactorExpand(factor?.id)">
            <svg width="16" height="16" viewBox="0 0 20 20" fill="none">
              <path d="M6 8l4 4 4-4" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
            </svg>
          </button>
        </div>

        <button
          v-if="hoveredFactorId === factor?.id"
          class="ef-factor-row__connect-btn"
          @click.stop="$emit('connect-factor', factor)"
          title="Link to condition group"
        >
          <svg width="14" height="14" viewBox="0 0 20 20" fill="none">
            <path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/>
          </svg>
        </button>
      </div>

      <div v-if="!factors?.length" class="ef-group-card__empty">
        No earn factors yet. Click "+ Add earn factor" to create one.
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue';

export default {
  props: {
    group: { type: Object, default: () => ({}) },
    factors: { type: Array, default: () => [] },
    expanded: { type: Boolean, default: false },
  },
  emits: ['add-factor', 'edit-group', 'edit-factor', 'connect-factor', 'toggle-expand', 'factor-ref'],
  setup(props, { emit }) {
    const isExpanded = computed(() => props.expanded);
    const hoveredFactorId = ref(null);
    const expandedFactors = ref({});

    const factorCount = computed(() => props.factors?.length || 0);

    function toggleExpand() {
      emit('toggle-expand', props.group?.id);
    }

    function setFactorRef(factorId, el) {
      if (el && factorId) {
        emit('factor-ref', { factorId, el });
      }
    }

    function toggleFactorExpand(id) {
      expandedFactors.value[id] = !expandedFactors.value[id];
    }

    function getFactorDisplayName(factor) {
      if (!factor) return 'Unnamed';
      const type = factor.earn_factor_type === 'rate' ? 'Rate' : 'Multiplier';
      const currency = factor.target_currency === 'ticket' ? 'Store credit' : 'Points';
      return `${currency} ${type}`;
    }

    function getCurrencyLabel(factor) {
      if (factor?.target_currency === 'ticket') return 'Store credit';
      return 'Points';
    }

    function formatDateRange(start, end) {
      const fmt = (d) => {
        if (!d) return '';
        const dt = new Date(d);
        return `${String(dt.getDate()).padStart(2, '0')}.${String(dt.getMonth() + 1).padStart(2, '0')}.${String(dt.getFullYear()).slice(2)}`;
      };
      const s = fmt(start);
      const e = fmt(end);
      if (s && e) return `${s} - ${e}`;
      if (s) return `From ${s}`;
      if (e) return `Until ${e}`;
      return '';
    }

    return {
      isExpanded,
      hoveredFactorId,
      expandedFactors,
      factorCount,
      toggleExpand,
      setFactorRef,
      toggleFactorExpand,
      getFactorDisplayName,
      getCurrencyLabel,
      formatDateRange,
    };
  },
};
</script>

<style scoped lang="scss">
@import 'polaris-weweb-styles';

.ef-group-card {
  @include polaris-card;
  overflow: hidden;

  &--expanded {
    .ef-group-card__header {
      border-bottom: 1px solid var(--p-color-border);
    }
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
    background: var(--p-color-bg-fill-critical-secondary);
    color: var(--p-color-text-critical);
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
    padding: var(--p-space-200);
  }

  &__empty {
    @include polaris-text-body-subdued;
    text-align: center;
    padding: var(--p-space-400);
  }
}

.ef-factor-row {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--p-space-300) var(--p-space-300);
  margin-bottom: var(--p-space-100);
  border: 1px solid var(--p-color-border);
  border-radius: var(--p-border-radius-200);
  background: var(--p-color-bg-surface);
  transition: box-shadow var(--p-motion-duration-150) var(--p-motion-ease);

  &:hover {
    box-shadow: var(--p-shadow-card-hover);
  }

  &:last-child {
    margin-bottom: 0;
  }

  &__left {
    display: flex;
    align-items: center;
    gap: var(--p-space-200);
    min-width: 0;
    flex: 1;
  }

  &__type-icon {
    width: 24px;
    height: 24px;
    min-width: 24px;
    border-radius: var(--p-border-radius-100);
    display: flex;
    align-items: center;
    justify-content: center;

    &--rate {
      color: var(--p-color-text);
      border-left: 3px solid var(--p-color-text-info);
    }

    &--multiplier {
      color: var(--p-color-text);
      border-left: 3px solid var(--p-color-text-success);
    }
  }

  &__info {
    min-width: 0;
    flex: 1;
  }

  &__name {
    @include polaris-text-body;
    font-weight: var(--p-font-weight-semibold);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__meta {
    font-size: var(--p-font-size-300);
    color: var(--p-color-text-secondary);
    display: flex;
    align-items: center;
    gap: var(--p-space-100);
    flex-wrap: wrap;
  }

  &__separator {
    color: var(--p-color-border);
  }

  &__right {
    display: flex;
    align-items: center;
    gap: var(--p-space-100);
    flex-shrink: 0;
  }

  &__multiplier {
    font-size: var(--p-font-size-300);
    color: var(--p-color-text-critical);

    strong {
      font-weight: var(--p-font-weight-bold);
    }
  }

  &__connect-btn {
    position: absolute;
    right: -14px;
    top: 50%;
    transform: translateY(-50%);
    width: 28px;
    height: 28px;
    border-radius: var(--p-border-radius-full);
    background: var(--p-color-bg-fill-brand);
    color: var(--p-color-text-on-color);
    border: 2px solid var(--p-color-bg-surface);
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    z-index: 10;
    box-shadow: var(--p-shadow-button);
    transition: transform var(--p-motion-duration-100) var(--p-motion-ease);

    &:hover {
      transform: translateY(-50%) scale(1.1);
    }
  }
}
</style>
