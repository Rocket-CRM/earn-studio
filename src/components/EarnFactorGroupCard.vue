<template>
  <div class="efg">
    <!-- Group label row (not a container — just a section header) -->
    <div class="efg__label">
      <div class="efg__label-icon">
        <svg width="13" height="13" viewBox="0 0 20 20" fill="none">
          <rect x="2" y="4" width="7" height="5" rx="1" fill="currentColor" opacity="0.8"/>
          <rect x="2" y="11" width="7" height="5" rx="1" fill="currentColor" opacity="0.6"/>
          <rect x="11" y="4" width="7" height="5" rx="1" fill="currentColor" opacity="0.4"/>
          <rect x="11" y="11" width="7" height="5" rx="1" fill="currentColor" opacity="0.3"/>
        </svg>
      </div>
      <span class="efg__label-name">{{ group?.name || 'Untitled Group' }}</span>
      <button class="efg__label-add" @click.stop="$emit('add-factor', group)">+ Add earn factor</button>
      <button class="efg__label-btn" @click.stop="$emit('edit-group', group)" title="Edit group">
        <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
      </button>
    </div>

    <!-- Factor rows (flat, each its own block — not nested in a container) -->
    <div
      v-for="factor in factors"
      :key="factor?.id"
      class="ef-row"
      :ref="el => registerRef(factor?.id, el)"
    >
      <div class="ef-row__indicator" :class="factor?.target_currency === 'ticket' ? 'ef-row__indicator--credit' : 'ef-row__indicator--points'"></div>
      <div class="ef-row__body">
        <div class="ef-row__icon" :class="factor?.target_currency === 'ticket' ? 'ef-row__icon--credit' : 'ef-row__icon--points'">
          <svg v-if="factor?.earn_factor_type === 'rate'" width="14" height="14" viewBox="0 0 20 20" fill="none">
            <path d="M10 2l2.39 4.84L17.27 7.6l-3.64 3.54.86 5.01L10 13.77l-4.49 2.36.86-5L2.73 7.6l4.88-.76L10 2z" fill="currentColor"/>
          </svg>
          <svg v-else width="14" height="14" viewBox="0 0 20 20" fill="none">
            <path d="M10 3L12.5 8H17L13.5 11.5L15 17L10 14L5 17L6.5 11.5L3 8H7.5L10 3Z" fill="currentColor"/>
          </svg>
        </div>
        <div class="ef-row__info">
          <div class="ef-row__title">{{ getTitle(factor) }}</div>
          <div class="ef-row__meta">
            <span>{{ getTypeLabel(factor) }}</span>
            <span class="ef-row__sep"></span>
            <span>{{ getRateLabel(factor) }}</span>
            <template v-if="getDateLabel(factor)">
              <span class="ef-row__sep"></span>
              <span>{{ getDateLabel(factor) }}</span>
            </template>
          </div>
        </div>
        <div class="ef-row__right">
          <div v-if="factor?.earn_factor_type === 'multiplier'" class="ef-row__mult">
            Total Multiplier : <strong>{{ factor?.earn_factor_amount || 0 }}x</strong>
          </div>
          <button class="ef-row__btn" @click.stop="$emit('edit-factor', factor)" title="Edit">
            <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
          </button>
          <button class="ef-row__btn ef-row__btn--chevron" title="Expand">
            <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M6 7l4 4 4-4" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/></svg>
          </button>
        </div>
      </div>
      <button class="ef-row__connect" @click.stop="$emit('connect-factor', factor, $event)" title="Link to condition group">
        <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/></svg>
      </button>
    </div>

    <div v-if="!factors?.length" class="efg__empty">No factors</div>
  </div>
</template>

<script>
export default {
  props: {
    group: { type: Object, default: () => ({}) },
    factors: { type: Array, default: () => [] },
  },
  emits: ['add-factor', 'edit-group', 'edit-factor', 'connect-factor', 'factor-ref'],
  setup(props, { emit }) {
    function registerRef(factorId, el) {
      if (el && factorId) emit('factor-ref', { factorId, el });
    }

    function getTitle(f) {
      if (!f) return 'Unnamed';
      if (f.name || f.target_entity_name) return f.name || f.target_entity_name;
      const c = f.target_currency === 'ticket' ? 'Credit' : 'Points';
      return f.earn_factor_type === 'rate' ? `${c} Starter Rules` : `${c} Power Boost`;
    }

    function getTypeLabel(f) {
      if (!f) return '';
      const c = f.target_currency === 'ticket' ? 'Store credit' : 'Points';
      return `${c} (${f.earn_factor_type === 'rate' ? 'Base rate' : 'Multiplier'})`;
    }

    function getRateLabel(f) {
      if (!f) return '';
      if (f.earn_factor_type === 'rate') return `Rate : ฿${f.earn_factor_amount || 0} = 1 point`;
      return `${f.earn_factor_amount || 0}x multiplier`;
    }

    function getDateLabel(f) {
      if (!f?.window_start && !f?.window_end) return '';
      const fmt = d => { if (!d) return '—'; try { const dt = new Date(d); return `${String(dt.getDate()).padStart(2,'0')}.${String(dt.getMonth()+1).padStart(2,'0')}.${String(dt.getFullYear()).slice(2)}`; } catch { return '—'; } };
      return `${fmt(f.window_start)} - ${fmt(f.window_end)}`;
    }

    return { registerRef, getTitle, getTypeLabel, getRateLabel, getDateLabel };
  },
};
</script>

<style scoped lang="scss">
@import 'polaris-weweb-styles';

.efg {
  &__label {
    display: flex;
    align-items: center;
    gap: var(--p-space-200);
    padding: var(--p-space-050) 0;
    margin-bottom: var(--p-space-200);
  }

  &__label-icon {
    width: 22px;
    height: 22px;
    min-width: 22px;
    border-radius: var(--p-border-radius-100);
    background: #F54239;
    color: #fff;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  &__label-name {
    @include polaris-text-subtitle-sm;
    flex: 1;
    min-width: 0;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__label-add {
    @include polaris-button-plain;
    font-size: var(--p-font-size-300);
    white-space: nowrap;
    padding: 2px 6px;
    min-height: auto;
  }

  &__label-btn {
    width: 22px;
    height: 22px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: none;
    border: none;
    border-radius: var(--p-border-radius-100);
    color: var(--p-color-icon);
    cursor: pointer;
    &:hover { background: var(--p-color-bg-surface-hover); }
  }

  &__empty {
    @include polaris-text-description;
    color: var(--p-color-text-disabled);
    text-align: center;
    padding: var(--p-space-200) 0;
  }
}

.ef-row {
  position: relative;
  display: flex;
  align-items: stretch;
  background: var(--p-color-bg-surface);
  border: 1px solid var(--p-color-border-info);
  border-radius: var(--p-border-radius-200);
  box-shadow: var(--p-shadow-card-sm);
  margin-bottom: var(--p-space-200);
  overflow: visible;

  &:last-child { margin-bottom: 0; }

  &:hover {
    box-shadow: var(--p-shadow-card-hover);
    .ef-row__btn { opacity: 1; }
    .ef-row__connect { opacity: 1; }
  }

  &__indicator {
    width: 3px;
    min-width: 3px;
    border-radius: var(--p-border-radius-200) 0 0 var(--p-border-radius-200);
    &--points { background: var(--p-color-text-info); }
    &--credit { background: var(--p-color-text-success); }
  }

  &__body {
    flex: 1;
    display: flex;
    align-items: center;
    gap: var(--p-space-200);
    padding: var(--p-space-300) var(--p-space-300);
    min-width: 0;
  }

  &__icon {
    width: 30px;
    height: 30px;
    min-width: 30px;
    border-radius: 5px;
    border: 0.7px solid var(--p-color-border);
    display: flex;
    align-items: center;
    justify-content: center;
    &--points { color: var(--p-color-text-info); }
    &--credit { color: var(--p-color-text); }
  }

  &__info {
    flex: 1;
    min-width: 0;
  }

  &__title {
    @include polaris-text-subtitle;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__meta {
    display: flex;
    align-items: center;
    gap: var(--p-space-200);
    @include polaris-text-description;
    white-space: nowrap;
  }

  &__sep {
    @include polaris-separator-dot;
  }

  &__right {
    display: flex;
    align-items: center;
    gap: var(--p-space-200);
    flex-shrink: 0;
  }

  &__mult {
    font-size: var(--p-font-size-300);
    color: var(--p-color-text-success);
    white-space: nowrap;
    strong { font-weight: var(--p-font-weight-bold); }
  }

  &__btn {
    width: 22px;
    height: 22px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: none;
    border: none;
    border-radius: var(--p-border-radius-100);
    color: var(--p-color-icon);
    cursor: pointer;
    opacity: 0;
    transition: opacity 0.1s, background 0.1s;
    &:hover { background: var(--p-color-bg-fill-transparent-hover); }
    &--chevron {
      opacity: 1;
      background: var(--p-color-border);
      border-radius: var(--p-border-radius-150);
      &:hover { background: var(--p-color-border-hover); }
    }
  }

  &__connect {
    position: absolute;
    right: -15px;
    top: 50%;
    transform: translateY(-50%);
    width: 28px;
    height: 28px;
    border-radius: var(--p-border-radius-full);
    background: var(--p-color-bg-fill-brand);
    color: var(--p-color-text-on-color);
    border: 3px solid var(--p-color-bg-surface);
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    opacity: 0;
    z-index: 5;
    transition: opacity 0.1s, background 0.1s, transform 0.1s;
    box-shadow: var(--p-shadow-button);
    &:hover {
      background: var(--p-color-bg-fill-brand-hover);
      transform: translateY(-50%) scale(1.1);
    }
  }
}
</style>
