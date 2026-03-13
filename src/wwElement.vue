<template>
  <div class="earn-studio" :style="rootStyles" ref="rootRef">
    <div class="earn-studio__columns" ref="columnsRef">
      <!-- Left Column: Earn Factor Groups -->
      <div class="earn-studio__column earn-studio__column--left" :style="{ width: content?.leftColumnWidth || '420px' }">
        <div class="earn-studio__column-header">
          <h2 class="earn-studio__column-title">Earn factor group</h2>
          <button class="earn-studio__create-btn" @click="openCreateFactorGroup">Create</button>
        </div>
        <div class="earn-studio__column-body">
          <div v-if="loadingFactorGroups" class="earn-studio__loading">
            <div class="earn-studio__spinner"></div>
          </div>
          <EarnFactorGroupCard
            v-for="group in factorGroups"
            :key="group?.id"
            :group="group"
            :factors="factorsByGroup[group?.id] || []"
            :expanded="expandedFactorGroups[group?.id] || false"
            @toggle-expand="toggleFactorGroupExpand"
            @add-factor="handleAddFactor"
            @edit-group="handleEditFactorGroup"
            @edit-factor="handleEditFactor"
            @connect-factor="handleConnectFactor"
            @factor-ref="registerFactorRef"
            class="earn-studio__card"
          />
          <div v-if="!loadingFactorGroups && !factorGroups?.length" class="earn-studio__empty">
            No earn factor groups yet. Click "Create" to get started.
          </div>
        </div>
      </div>

      <!-- Connection Lines SVG -->
      <div class="earn-studio__connections" ref="connectionsRef">
        <svg class="earn-studio__svg" :width="connectionAreaWidth" :height="connectionAreaHeight">
          <line
            v-for="conn in connectionLines"
            :key="conn.key"
            :x1="conn.x1"
            :y1="conn.y1"
            :x2="conn.x2"
            :y2="conn.y2"
            :stroke="hoveredConnectionKey === conn.key ? (content?.connectionLineActiveColor || '#005BD3') : (content?.connectionLineColor || '#C9CCCF')"
            stroke-width="2"
            :stroke-dasharray="conn.dashed ? '6 4' : 'none'"
            class="earn-studio__connection-line"
            @mouseenter="hoveredConnectionKey = conn.key"
            @mouseleave="hoveredConnectionKey = null"
          />
          <circle
            v-for="conn in connectionLines"
            :key="'dot-l-' + conn.key"
            :cx="conn.x1"
            :cy="conn.y1"
            r="4"
            :fill="content?.connectionLineColor || '#C9CCCF'"
          />
          <circle
            v-for="conn in connectionLines"
            :key="'dot-r-' + conn.key"
            :cx="conn.x2"
            :cy="conn.y2"
            r="4"
            :fill="content?.connectionLineColor || '#C9CCCF'"
          />
        </svg>
      </div>

      <!-- Right Column: Earn Condition Groups -->
      <div class="earn-studio__column earn-studio__column--right" :style="{ width: content?.rightColumnWidth || '420px' }">
        <div class="earn-studio__column-header">
          <h2 class="earn-studio__column-title">Earn Conditions group</h2>
          <button class="earn-studio__create-btn" @click="openCreateConditionGroup">Create</button>
        </div>
        <div class="earn-studio__column-body">
          <div v-if="loadingConditionGroups" class="earn-studio__loading">
            <div class="earn-studio__spinner"></div>
          </div>
          <EarnConditionGroupCard
            v-for="cgEntry in displayedConditionGroups"
            :key="cgEntry.displayKey"
            :group="cgEntry.group"
            :conditions="cgEntry.conditions || []"
            :linked-factor-count="cgEntry.linkedFactorCount"
            :expanded="expandedConditionGroups[cgEntry.displayKey] || false"
            @toggle-expand="toggleConditionGroupExpand(cgEntry.displayKey)"
            @add-condition="handleAddCondition(cgEntry.group)"
            @edit-group="handleEditConditionGroup"
            @dot-ref="registerDotRef"
            class="earn-studio__card"
            :ref="el => registerConditionCardRef(cgEntry.displayKey, el)"
          />
          <div v-if="!loadingConditionGroups && !displayedConditionGroups?.length" class="earn-studio__empty">
            No earn condition groups yet. Click "Create" to get started.
          </div>
        </div>
      </div>
    </div>

    <!-- Sidebar Panels -->
    <transition name="slide-right">
      <EarnFactorConfig
        v-if="activePanel === 'factor-config'"
        :factor="editingFactor"
        :group-id="editingFactorGroupId"
        :condition-groups="allConditionGroupsList"
        :ticket-types="ticketTypes"
        :panel-width="content?.configPanelWidth || '380px'"
        @close="closePanel"
        @save="saveFactorConfig"
      />
    </transition>

    <transition name="slide-right">
      <EarnConditionGroupConfig
        v-if="activePanel === 'condition-config'"
        :group="editingConditionGroup"
        :all-entity-options="allEntityOptions"
        :panel-width="content?.configPanelWidth || '380px'"
        @close="closePanel"
        @save="saveConditionGroupConfig"
      />
    </transition>

    <!-- Modals -->
    <CreateGroupModal
      v-if="showCreateModal"
      :type="createModalType"
      @close="showCreateModal = false"
      @save="handleCreateGroupSave"
    />

    <ConnectPopup
      v-if="connectPopup.open"
      :condition-groups="allConditionGroupsList"
      :position="connectPopup.position"
      @close="connectPopup.open = false"
      @select="handleConnectSelect"
    />
  </div>
</template>

<script>
import { ref, computed, onMounted, watch, nextTick } from 'vue';
import { useApi } from './useApi.js';
import EarnFactorGroupCard from './components/EarnFactorGroupCard.vue';
import EarnConditionGroupCard from './components/EarnConditionGroupCard.vue';
import EarnFactorConfig from './components/EarnFactorConfig.vue';
import EarnConditionGroupConfig from './components/EarnConditionGroupConfig.vue';
import CreateGroupModal from './components/CreateGroupModal.vue';
import ConnectPopup from './components/ConnectPopup.vue';

export default {
  components: {
    EarnFactorGroupCard,
    EarnConditionGroupCard,
    EarnFactorConfig,
    EarnConditionGroupConfig,
    CreateGroupModal,
    ConnectPopup,
  },
  props: {
    uid: { type: String, required: true },
    content: { type: Object, required: true },
    /* wwEditor:start */
    wwEditorState: { type: Object, required: true },
    /* wwEditor:end */
  },
  emits: ['trigger-event'],
  setup(props, { emit }) {
    const api = useApi(props);

    const rootRef = ref(null);
    const columnsRef = ref(null);
    const connectionsRef = ref(null);

    // Data
    const factorGroups = ref([]);
    const factorsByGroup = ref({});
    const allConditionGroupsList = ref([]);
    const conditionGroupDetailsCache = ref({});
    const allEntityOptions = ref([]);
    const ticketTypes = ref([]);
    const loadingFactorGroups = ref(false);
    const loadingConditionGroups = ref(false);

    // UI State
    const expandedFactorGroups = ref({});
    const expandedConditionGroups = ref({});
    const activePanel = ref(null);
    const editingFactor = ref(null);
    const editingFactorGroupId = ref(null);
    const editingConditionGroup = ref(null);
    const showCreateModal = ref(false);
    const createModalType = ref('factor');
    const hoveredConnectionKey = ref(null);

    const connectPopup = ref({ open: false, position: null, factorId: null });

    // Refs for connection lines
    const factorElRefs = ref({});
    const conditionDotRefs = ref({});
    const conditionCardRefs = ref({});

    // Connection line geometry
    const connectionLines = ref([]);
    const connectionAreaWidth = ref(120);
    const connectionAreaHeight = ref(800);

    const rootStyles = computed(() => ({
      '--p-earn-studio-font-family': 'var(--p-font-family-sans)',
    }));

    // Build displayed condition groups with duplication
    const displayedConditionGroups = computed(() => {
      const groups = allConditionGroupsList.value || [];
      const allFactors = Object.values(factorsByGroup.value || {}).flat();
      const result = [];
      const usedGroupIds = new Set();

      for (const factor of allFactors) {
        const cgId = factor?.earn_conditions_group_id;
        if (!cgId) continue;
        const group = groups.find(g => g?.id === cgId);
        if (!group) continue;
        usedGroupIds.add(cgId);

        const linkedCount = allFactors.filter(f => f?.earn_conditions_group_id === cgId).length;
        const details = conditionGroupDetailsCache.value[cgId];
        result.push({
          displayKey: `${cgId}_${factor.id}`,
          group,
          conditions: details?.conditions || group?.conditions || [],
          linkedFactorCount: linkedCount,
          factorId: factor.id,
        });
      }

      for (const group of groups) {
        if (!usedGroupIds.has(group?.id)) {
          result.push({
            displayKey: group.id,
            group,
            conditions: conditionGroupDetailsCache.value[group.id]?.conditions || group?.conditions || [],
            linkedFactorCount: 0,
            factorId: null,
          });
        }
      }

      return result;
    });

    // Data Loading
    async function loadAll() {
      await Promise.all([
        loadFactorGroups(),
        loadConditionGroups(),
        loadEntityOptions(),
      ]);
      emit('trigger-event', {
        name: 'data-loaded',
        event: {
          factorGroupCount: factorGroups.value?.length || 0,
          conditionGroupCount: allConditionGroupsList.value?.length || 0,
        },
      });
    }

    async function loadFactorGroups() {
      loadingFactorGroups.value = true;
      try {
        factorGroups.value = await api.fetchEarnFactorGroups() || [];
        for (const g of factorGroups.value) {
          if (g?.id) {
            const factors = await api.fetchFactorsByGroup(g.id);
            factorsByGroup.value[g.id] = factors || [];
          }
        }
      } catch (e) {
        emitError('Failed to load earn factor groups', e);
      } finally {
        loadingFactorGroups.value = false;
      }
    }

    async function loadConditionGroups() {
      loadingConditionGroups.value = true;
      try {
        const result = await api.fetchAllConditionGroups();
        allConditionGroupsList.value = result || [];
        for (const g of allConditionGroupsList.value) {
          if (g?.id) {
            try {
              const details = await api.fetchConditionGroupDetails(g.id);
              conditionGroupDetailsCache.value[g.id] = details;
            } catch { /* skip */ }
          }
        }
      } catch (e) {
        emitError('Failed to load condition groups', e);
      } finally {
        loadingConditionGroups.value = false;
      }
    }

    async function loadEntityOptions() {
      try {
        allEntityOptions.value = await api.fetchEntityOptions() || [];
      } catch (e) {
        emitError('Failed to load entity options', e);
      }
    }

    function emitError(message, error) {
      console.error(message, error);
      emit('trigger-event', { name: 'error', event: { message, code: 'LOAD_ERROR' } });
    }

    // Expand/Collapse
    function toggleFactorGroupExpand(groupId) {
      expandedFactorGroups.value[groupId] = !expandedFactorGroups.value[groupId];
      nextTick(updateConnectionLines);
    }

    function toggleConditionGroupExpand(displayKey) {
      expandedConditionGroups.value[displayKey] = !expandedConditionGroups.value[displayKey];
      nextTick(updateConnectionLines);
    }

    // Panel Management
    function closePanel() {
      activePanel.value = null;
      editingFactor.value = null;
      editingFactorGroupId.value = null;
      editingConditionGroup.value = null;
    }

    function handleAddFactor(group) {
      editingFactor.value = { earn_factor_type: 'rate', public: true, target_currency: 'points', active_status: true };
      editingFactorGroupId.value = group?.id;
      activePanel.value = 'factor-config';
    }

    function handleEditFactor(factor) {
      editingFactor.value = { ...factor };
      editingFactorGroupId.value = factor?.earn_factor_group_id || null;
      activePanel.value = 'factor-config';
    }

    function handleEditFactorGroup(group) {
      editingFactor.value = null;
      showCreateModal.value = false;
      editingFactorGroupId.value = group?.id;
      activePanel.value = null;
      // For now, reuse the create modal logic - can be enhanced
      createModalType.value = 'factor';
      showCreateModal.value = true;
    }

    function handleAddCondition(group) {
      handleEditConditionGroup(group);
    }

    async function handleEditConditionGroup(group) {
      try {
        if (group?.id) {
          const details = await api.fetchConditionGroupDetails(group.id);
          editingConditionGroup.value = details;
        } else {
          editingConditionGroup.value = { id: null, name: '', conditions: [] };
        }
      } catch {
        editingConditionGroup.value = { ...group, conditions: group?.conditions || [] };
      }
      activePanel.value = 'condition-config';
    }

    // Create Modals
    function openCreateFactorGroup() {
      createModalType.value = 'factor';
      showCreateModal.value = true;
    }

    function openCreateConditionGroup() {
      editingConditionGroup.value = null;
      activePanel.value = 'condition-config';
    }

    async function handleCreateGroupSave(payload) {
      try {
        if (createModalType.value === 'factor') {
          const result = await api.upsertEarnFactorGroup({
            name: payload.name,
            stackable: payload.stackable,
            window_start: payload.window_start,
            window_end: payload.window_end,
            factors: [],
          });
          emit('trigger-event', {
            name: 'earn-factor-group-saved',
            event: { groupId: result?.group_id, groupName: payload.name, action: 'created' },
          });
          await loadFactorGroups();
        }
      } catch (e) {
        emitError('Failed to create group', e);
      }
      showCreateModal.value = false;
    }

    // Factor Save
    async function saveFactorConfig({ groupId, factor }) {
      try {
        const groupDetails = await api.fetchEarnFactorGroupDetails(groupId);
        const existingFactors = groupDetails?.factors || [];
        let updatedFactors;
        if (factor.id) {
          updatedFactors = existingFactors.map(f => f.id === factor.id ? factor : f);
        } else {
          updatedFactors = [...existingFactors, factor];
        }
        await api.upsertEarnFactorGroup({
          id: groupId,
          factors: updatedFactors,
        });
        emit('trigger-event', {
          name: 'earn-factor-saved',
          event: { factorId: factor.id, factorType: factor.earn_factor_type, groupId },
        });
        await loadFactorGroups();
        closePanel();
        nextTick(updateConnectionLines);
      } catch (e) {
        emitError('Failed to save earn factor', e);
      }
    }

    // Condition Group Save
    async function saveConditionGroupConfig(payload) {
      try {
        const result = await api.upsertConditionGroup(payload);
        emit('trigger-event', {
          name: 'earn-condition-group-saved',
          event: {
            groupId: result?.group?.id || payload.id,
            groupName: payload.name,
            action: payload.id ? 'updated' : 'created',
          },
        });
        await loadConditionGroups();
        closePanel();
        nextTick(updateConnectionLines);
      } catch (e) {
        emitError('Failed to save condition group', e);
      }
    }

    // Connection Handling
    function handleConnectFactor(factor) {
      const el = factorElRefs.value[factor?.id];
      if (el) {
        const rect = el.getBoundingClientRect();
        connectPopup.value = {
          open: true,
          position: { x: rect.right + 10, y: rect.top },
          factorId: factor.id,
          factorGroupId: factor.earn_factor_group_id,
        };
      } else {
        connectPopup.value = {
          open: true,
          position: { x: 500, y: 200 },
          factorId: factor.id,
          factorGroupId: factor.earn_factor_group_id,
        };
      }
    }

    async function handleConnectSelect(conditionGroup) {
      const { factorId, factorGroupId } = connectPopup.value;
      connectPopup.value.open = false;

      try {
        const groupDetails = await api.fetchEarnFactorGroupDetails(factorGroupId);
        const factors = (groupDetails?.factors || []).map(f => {
          if (f.id === factorId) {
            return { ...f, earn_conditions_group_id: conditionGroup.id };
          }
          return f;
        });
        await api.upsertEarnFactorGroup({ id: factorGroupId, factors });
        emit('trigger-event', {
          name: 'connection-changed',
          event: { factorId, conditionGroupId: conditionGroup.id, action: 'linked' },
        });
        await loadFactorGroups();
        nextTick(updateConnectionLines);
      } catch (e) {
        emitError('Failed to link factor to condition group', e);
      }
    }

    // Connection Line Geometry
    function registerFactorRef({ factorId, el }) {
      factorElRefs.value[factorId] = el;
      nextTick(updateConnectionLines);
    }

    function registerDotRef({ groupId, el }) {
      if (el) conditionDotRefs.value[groupId] = el;
    }

    function registerConditionCardRef(displayKey, el) {
      if (el?.$el) conditionCardRefs.value[displayKey] = el.$el;
    }

    function updateConnectionLines() {
      const container = columnsRef.value;
      if (!container) return;
      const containerRect = container.getBoundingClientRect();
      const lines = [];

      const allFactors = Object.values(factorsByGroup.value || {}).flat();

      for (const factor of allFactors) {
        const cgId = factor?.earn_conditions_group_id;
        if (!cgId) continue;

        const factorEl = factorElRefs.value[factor.id];
        const displayKey = `${cgId}_${factor.id}`;
        const condCard = conditionCardRefs.value[displayKey];

        if (!factorEl) continue;

        const factorRect = factorEl.getBoundingClientRect();
        const y1 = factorRect.top + factorRect.height / 2 - containerRect.top;
        const x1 = 0;

        let x2 = connectionAreaWidth.value;
        let y2 = y1;

        if (condCard) {
          const condRect = condCard.getBoundingClientRect();
          y2 = condRect.top + 24 - containerRect.top;
          x2 = connectionAreaWidth.value;
        }

        lines.push({
          key: `${factor.id}_${cgId}`,
          x1,
          y1,
          x2,
          y2,
          factorId: factor.id,
          conditionGroupId: cgId,
          dashed: false,
        });
      }

      connectionLines.value = lines;

      const maxY = Math.max(containerRect.height, ...lines.map(l => Math.max(l.y1, l.y2) + 20));
      connectionAreaHeight.value = maxY;
    }

    // Lifecycle
    onMounted(() => {
      if (props.content?.authToken) {
        loadAll();
      }
    });

    watch(() => props.content?.authToken, (newToken) => {
      if (newToken) loadAll();
    });

    // Actions
    const componentActions = {
      refreshData: loadAll,
      closePanel,
    };

    /* wwEditor:start */
    // Editor-only code
    /* wwEditor:end */

    return {
      rootRef,
      columnsRef,
      connectionsRef,
      rootStyles,
      factorGroups,
      factorsByGroup,
      allConditionGroupsList,
      allEntityOptions,
      ticketTypes,
      displayedConditionGroups,
      loadingFactorGroups,
      loadingConditionGroups,
      expandedFactorGroups,
      expandedConditionGroups,
      activePanel,
      editingFactor,
      editingFactorGroupId,
      editingConditionGroup,
      showCreateModal,
      createModalType,
      connectionLines,
      connectionAreaWidth,
      connectionAreaHeight,
      hoveredConnectionKey,
      connectPopup,
      content: computed(() => props.content),
      toggleFactorGroupExpand,
      toggleConditionGroupExpand,
      closePanel,
      handleAddFactor,
      handleEditFactor,
      handleEditFactorGroup,
      handleAddCondition,
      handleEditConditionGroup,
      openCreateFactorGroup,
      openCreateConditionGroup,
      handleCreateGroupSave,
      saveFactorConfig,
      saveConditionGroupConfig,
      handleConnectFactor,
      handleConnectSelect,
      registerFactorRef,
      registerDotRef,
      registerConditionCardRef,
      ...componentActions,
    };
  },
};
</script>

<style scoped lang="scss">
@import 'polaris-weweb-styles';

:root {
  @include polaris-tokens;
}

.earn-studio {
  @include polaris-tokens;
  font-family: var(--p-font-family-sans);
  color: var(--p-color-text);
  background: var(--p-color-bg);
  width: 100%;
  height: 100%;
  overflow: auto;
  padding: var(--p-space-400);

  &__columns {
    display: flex;
    align-items: flex-start;
    gap: 0;
    min-height: 400px;
    position: relative;
  }

  &__column {
    flex-shrink: 0;

    &--left {
      z-index: 2;
    }

    &--right {
      z-index: 2;
    }
  }

  &__column-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: var(--p-space-400);
  }

  &__column-title {
    @include polaris-text-heading-md;
    margin: 0;
  }

  &__create-btn {
    @include polaris-button-primary;
    @include polaris-button-slim;
  }

  &__column-body {
    @include polaris-block-stack(var(--p-space-300));
  }

  &__card {
    /* Spacing handled by block-stack */
  }

  &__connections {
    flex: 1;
    min-width: 80px;
    max-width: 160px;
    position: relative;
    z-index: 1;
  }

  &__svg {
    width: 100%;
    overflow: visible;
  }

  &__connection-line {
    cursor: pointer;
    transition: stroke var(--p-motion-duration-150) var(--p-motion-ease);
  }

  &__loading {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: var(--p-space-800);
  }

  &__spinner {
    @include polaris-spinner;
  }

  &__empty {
    @include polaris-text-body-subdued;
    text-align: center;
    padding: var(--p-space-800);
    background: var(--p-color-bg-surface);
    border-radius: var(--p-border-radius-300);
    border: 2px dashed var(--p-color-border);
  }
}

// Slide transition for panels
.slide-right-enter-active,
.slide-right-leave-active {
  transition: transform var(--p-motion-duration-300) var(--p-motion-ease);
}

.slide-right-enter-from,
.slide-right-leave-to {
  transform: translateX(100%);
}
</style>
