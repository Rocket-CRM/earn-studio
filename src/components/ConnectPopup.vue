<template>
  <div class="connect-popup-overlay" @click.self="$emit('close')">
    <div class="connect-popup" :style="popupStyle">
      <div class="connect-popup__header">
        <h4 class="connect-popup__title">Link to Earn Condition Group</h4>
        <button class="connect-popup__close" @click="$emit('close')">×</button>
      </div>
      <div class="connect-popup__search">
        <input class="connect-popup__search-input" v-model="search" placeholder="Search groups..." />
      </div>
      <div class="connect-popup__list">
        <div
          v-for="cg in filteredGroups"
          :key="cg?.id"
          class="connect-popup__item"
          @click="$emit('select', cg)"
        >
          <div class="connect-popup__item-info">
            <span class="connect-popup__item-name">{{ cg?.name || 'Untitled' }}</span>
            <span class="connect-popup__item-meta">
              {{ cg?.conditions_count || cg?.conditions?.length || 0 }} condition(s)
            </span>
          </div>
          <svg width="16" height="16" viewBox="0 0 20 20" fill="none" class="connect-popup__item-arrow">
            <path d="M8 6l4 4-4 4" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </div>
        <div v-if="!filteredGroups?.length" class="connect-popup__empty">
          No condition groups available
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue';

export default {
  props: {
    conditionGroups: { type: Array, default: () => [] },
    position: { type: Object, default: null },
  },
  emits: ['close', 'select'],
  setup(props) {
    const search = ref('');

    const filteredGroups = computed(() => {
      const q = search.value?.toLowerCase()?.trim();
      if (!q) return props.conditionGroups || [];
      return (props.conditionGroups || []).filter(g =>
        g?.name?.toLowerCase()?.includes(q)
      );
    });

    const popupStyle = computed(() => {
      if (!props.position) return {};
      return {
        top: `${props.position.y || 0}px`,
        left: `${props.position.x || 0}px`,
      };
    });

    return { search, filteredGroups, popupStyle };
  },
};
</script>

<style scoped lang="scss">
@import 'polaris-weweb-styles';

.connect-popup-overlay {
  position: fixed;
  inset: 0;
  z-index: var(--p-z-index-4);
}

.connect-popup {
  @include polaris-popover;
  position: fixed;
  min-width: 280px;
  max-width: 360px;
  max-height: 400px;
  display: flex;
  flex-direction: column;

  &__header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: var(--p-space-300) var(--p-space-300);
    border-bottom: 1px solid var(--p-color-border);
  }

  &__title {
    @include polaris-text-body;
    font-weight: var(--p-font-weight-semibold);
    margin: 0;
  }

  &__close {
    width: var(--p-space-600); height: var(--p-space-600);
    display: flex; align-items: center; justify-content: center;
    background: transparent; border: none; color: var(--p-color-icon); cursor: pointer;
    font-size: var(--p-font-size-350);
    border-radius: var(--p-border-radius-100);
    &:hover { background: var(--p-color-bg-surface-hover); }
  }

  &__search {
    padding: var(--p-space-200) var(--p-space-300);
    border-bottom: 1px solid var(--p-color-border);
  }

  &__search-input {
    @include polaris-search-field;
    font-size: var(--p-font-size-300);
  }

  &__list {
    flex: 1;
    overflow-y: auto;
    padding: var(--p-space-100) 0;
  }

  &__item {
    @include polaris-action-list-item;
    justify-content: space-between;
  }

  &__item-info {
    display: flex;
    flex-direction: column;
    gap: var(--p-space-050);
    min-width: 0;
  }

  &__item-name {
    @include polaris-text-body;
    font-weight: var(--p-font-weight-medium);
  }

  &__item-meta {
    font-size: var(--p-font-size-275);
    color: var(--p-color-text-secondary);
  }

  &__item-arrow {
    color: var(--p-color-icon);
    flex-shrink: 0;
  }

  &__empty {
    @include polaris-text-body-subdued;
    text-align: center;
    padding: var(--p-space-400);
  }
}
</style>
