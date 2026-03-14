<template>
  <div class="earn-studio" ref="rootRef">
    <div class="earn-studio__layout" ref="layoutRef">
      <!-- LEFT COLUMN -->
      <div class="earn-studio__col earn-studio__col--left">
        <div class="earn-studio__col-head">
          <h2 class="earn-studio__title">Earn factor group</h2>
          <button class="earn-studio__primary-btn" @click="openCreateFactorGroup">Create</button>
        </div>
        <div class="earn-studio__col-body">
          <div v-if="loadingFactorGroups" class="earn-studio__loading"><div class="earn-studio__spinner"></div></div>
          <template v-else>
            <div v-for="g in factorGroups" :key="g?.id" class="earn-studio__factor-section">
              <!-- Group label -->
              <div class="earn-studio__group-label">
                <div class="earn-studio__group-icon">
                  <svg width="13" height="13" viewBox="0 0 20 20" fill="none"><rect x="2" y="4" width="7" height="5" rx="1" fill="currentColor" opacity=".8"/><rect x="2" y="11" width="7" height="5" rx="1" fill="currentColor" opacity=".6"/><rect x="11" y="4" width="7" height="5" rx="1" fill="currentColor" opacity=".4"/><rect x="11" y="11" width="7" height="5" rx="1" fill="currentColor" opacity=".3"/></svg>
                </div>
                <span class="earn-studio__group-name">{{ g?.name || 'Untitled Group' }}</span>
                <button class="earn-studio__link-btn" @click="handleAddFactor(g)">+ Add earn factor</button>
                <button class="earn-studio__icon-btn" @click="handleEditFactorGroup(g)">
                  <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
                </button>
              </div>
              <!-- Factor rows — each has data-factor-id for DOM lookup -->
              <div
                v-for="f in (factorsByGroup[g?.id] || [])"
                :key="f?.id"
                :data-factor-id="f?.id"
                class="earn-studio__factor-row"
              >
                <div class="earn-studio__factor-bar" :class="f?.target_currency === 'ticket' ? 'earn-studio__factor-bar--credit' : 'earn-studio__factor-bar--points'"></div>
                <div class="earn-studio__factor-body">
                  <div class="earn-studio__factor-icon" :class="f?.target_currency === 'ticket' ? 'earn-studio__factor-icon--credit' : 'earn-studio__factor-icon--points'">
                    <svg v-if="f?.earn_factor_type === 'rate'" width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M10 2l2.39 4.84L17.27 7.6l-3.64 3.54.86 5.01L10 13.77l-4.49 2.36.86-5L2.73 7.6l4.88-.76L10 2z" fill="currentColor"/></svg>
                    <svg v-else width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M10 3L12.5 8H17L13.5 11.5L15 17L10 14L5 17L6.5 11.5L3 8H7.5L10 3Z" fill="currentColor"/></svg>
                  </div>
                  <div class="earn-studio__factor-info">
                    <div class="earn-studio__factor-title">{{ getFactorTitle(f) }}</div>
                    <div class="earn-studio__factor-meta">{{ getFactorMeta(f) }}</div>
                  </div>
                  <div v-if="f?.earn_factor_type === 'multiplier'" class="earn-studio__factor-mult">
                    Total Multiplier : <strong>{{ f?.earn_factor_amount || 0 }}x</strong>
                  </div>
                  <button class="earn-studio__icon-btn earn-studio__icon-btn--hover" @click="handleEditFactor(f)">
                    <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
                  </button>
                </div>
                <button class="earn-studio__connect-btn" @click="handleConnectFactor(f, $event)">
                  <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/></svg>
                </button>
              </div>
              <div v-if="!(factorsByGroup[g?.id] || []).length" class="earn-studio__no-factors">No factors</div>
            </div>
            <div v-if="!factorGroups?.length" class="earn-studio__empty">No earn factor groups yet.</div>
          </template>
        </div>
      </div>

      <!-- SVG CONNECTIONS -->
      <svg class="earn-studio__svg" ref="svgRef">
        <path
          v-for="ln in lines"
          :key="ln.key"
          :d="ln.d"
          fill="none"
          :stroke="hoveredLine === ln.key ? '#0262E0' : '#CCCCCC'"
          :stroke-width="hoveredLine === ln.key ? 2.5 : 1.5"
          :stroke-dasharray="ln.dashed ? '4 4' : ''"
          @mouseenter="hoveredLine = ln.key"
          @mouseleave="hoveredLine = null"
          style="cursor:pointer"
        />
      </svg>

      <!-- RIGHT COLUMN -->
      <div class="earn-studio__col earn-studio__col--right">
        <div class="earn-studio__col-head">
          <h2 class="earn-studio__title">Earn Conditions group</h2>
          <button class="earn-studio__primary-btn" @click="openCreateConditionGroup">Create</button>
        </div>
        <div class="earn-studio__col-body">
          <div v-if="loadingConditionGroups" class="earn-studio__loading"><div class="earn-studio__spinner"></div></div>
          <template v-else>
            <!-- Each condition group pill has data-cg-key for DOM lookup -->
            <EarnConditionGroupCard
              v-for="entry in rightEntries"
              :key="entry.dk"
              :data-cg-key="entry.dk"
              :group="entry.group"
              :conditions="entry.conditions"
              :linked-factor-count="entry.linkedCount"
              :is-active="!!expandedRight[entry.dk]"
              :display-key="entry.dk"
              @select-group="(g, dk) => { expandedRight[dk] = !expandedRight[dk]; scheduleLineUpdate(); }"
              @add-condition="g => handleEditConditionGroup(g)"
              @edit-group="handleEditConditionGroup"
            />
            <div v-if="!rightEntries?.length" class="earn-studio__empty">No condition groups yet.</div>
          </template>
        </div>
      </div>
    </div>

    <!-- Panels -->
    <transition name="slide">
      <EarnFactorConfig v-if="panel === 'factor'" :factor="editingFactor" :group-id="editingGroupId"
        :condition-groups="allCondGroups" :ticket-types="ticketTypes" :panel-width="content?.configPanelWidth || '380px'"
        @close="panel = null" @save="saveFactorConfig" />
    </transition>
    <transition name="slide">
      <EarnConditionGroupConfig v-if="panel === 'condition'" :group="editingCondGroup"
        :all-entity-options="entityOptions" :panel-width="content?.configPanelWidth || '380px'"
        @close="panel = null" @save="saveCondGroupConfig" />
    </transition>

    <CreateGroupModal v-if="showModal" :type="modalType" @close="showModal = false" @save="handleModalSave" />
    <ConnectPopup v-if="connectPopup.open" :condition-groups="allCondGroups" :position="connectPopup.pos"
      @close="connectPopup.open = false" @select="handleConnectSelect" />
  </div>
</template>

<script>
import { ref, computed, onMounted, watch, nextTick, onBeforeUnmount } from 'vue';
import { useApi } from './useApi.js';
import EarnConditionGroupCard from './components/EarnConditionGroupCard.vue';
import EarnFactorConfig from './components/EarnFactorConfig.vue';
import EarnConditionGroupConfig from './components/EarnConditionGroupConfig.vue';
import CreateGroupModal from './components/CreateGroupModal.vue';
import ConnectPopup from './components/ConnectPopup.vue';

export default {
  components: { EarnConditionGroupCard, EarnFactorConfig, EarnConditionGroupConfig, CreateGroupModal, ConnectPopup },
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
    const layoutRef = ref(null);
    const svgRef = ref(null);

    const factorGroups = ref([]);
    const factorsByGroup = ref({});
    const allCondGroups = ref([]);
    const condCache = ref({});
    const entityOptions = ref([]);
    const ticketTypes = ref([]);
    const loadingFactorGroups = ref(false);
    const loadingConditionGroups = ref(false);

    const expandedRight = ref({});
    const panel = ref(null);
    const editingFactor = ref(null);
    const editingGroupId = ref(null);
    const editingCondGroup = ref(null);
    const showModal = ref(false);
    const modalType = ref('factor');
    const hoveredLine = ref(null);
    const connectPopup = ref({ open: false, pos: null, factorId: null, groupId: null });
    const lines = ref([]);
    let lineTimer = null;
    let ro = null;

    // ─── Right column entries with duplication rule ───
    const rightEntries = computed(() => {
      const groups = allCondGroups.value || [];
      const allF = Object.values(factorsByGroup.value || {}).flat();
      const out = [];
      const used = new Set();

      for (const f of allF) {
        const cid = f?.earn_conditions_group_id;
        if (!cid) continue;
        const g = groups.find(x => x?.id === cid);
        if (!g) continue;
        used.add(cid);
        out.push({
          dk: `${cid}__${f.id}`,
          group: g,
          conditions: condCache.value[cid]?.conditions || g?.conditions || [],
          linkedCount: allF.filter(x => x?.earn_conditions_group_id === cid).length,
          factorId: f.id,
        });
      }
      for (const g of groups) {
        if (!used.has(g?.id)) {
          out.push({ dk: g.id, group: g, conditions: condCache.value[g.id]?.conditions || g?.conditions || [], linkedCount: 0, factorId: null });
        }
      }
      return out;
    });

    // ─── Data loading ───
    async function loadAll() {
      await Promise.all([loadFactorGroups(), loadCondGroups(), loadEntities()]);
      emit('trigger-event', { name: 'data-loaded', event: { factorGroupCount: factorGroups.value?.length, conditionGroupCount: allCondGroups.value?.length } });
      scheduleLineUpdate();
    }

    async function loadFactorGroups() {
      loadingFactorGroups.value = true;
      try {
        factorGroups.value = await api.fetchEarnFactorGroups() || [];
        const m = {};
        for (const g of factorGroups.value) { if (g?.id) m[g.id] = await api.fetchFactorsByGroup(g.id) || []; }
        factorsByGroup.value = m;
      } catch (e) { err('Load factor groups failed', e); }
      finally { loadingFactorGroups.value = false; }
    }

    async function loadCondGroups() {
      loadingConditionGroups.value = true;
      try {
        allCondGroups.value = await api.fetchAllConditionGroups() || [];
        const c = {};
        for (const g of allCondGroups.value) { if (g?.id) try { c[g.id] = await api.fetchConditionGroupDetails(g.id); } catch {} }
        condCache.value = c;
      } catch (e) { err('Load condition groups failed', e); }
      finally { loadingConditionGroups.value = false; }
    }

    async function loadEntities() {
      try { entityOptions.value = await api.fetchEntityOptions() || []; }
      catch (e) { err('Load entities failed', e); }
    }

    function err(m, e) {
      console.error(m, e);
      emit('trigger-event', { name: 'error', event: { message: m, code: 'ERR' } });
    }

    // ─── Connection lines via DOM query (no ref emit chains) ───
    function scheduleLineUpdate() {
      clearTimeout(lineTimer);
      lineTimer = setTimeout(() => nextTick(rebuildLines), 80);
    }

    function rebuildLines() {
      const layout = layoutRef.value;
      if (!layout) return;

      const layoutRect = layout.getBoundingClientRect();
      const newLines = [];
      const allF = Object.values(factorsByGroup.value || {}).flat();

      for (const f of allF) {
        const cid = f?.earn_conditions_group_id;
        if (!cid) continue;

        const fEl = layout.querySelector(`[data-factor-id="${f.id}"]`);
        const dk = `${cid}__${f.id}`;
        const cEl = layout.querySelector(`[data-cg-key="${dk}"]`);

        if (!fEl || !cEl) continue;

        const fRect = fEl.getBoundingClientRect();
        const cRect = cEl.getBoundingClientRect();

        const x1 = fRect.right - layoutRect.left;
        const y1 = fRect.top + fRect.height / 2 - layoutRect.top;
        const x2 = cRect.left - layoutRect.left;
        const y2 = cRect.top + cRect.height / 2 - layoutRect.top;

        const dx = x2 - x1;
        const cp = Math.max(dx * 0.4, 30);
        const d = `M ${x1} ${y1} C ${x1 + cp} ${y1}, ${x2 - cp} ${y2}, ${x2} ${y2}`;
        newLines.push({ key: `${f.id}__${cid}`, d, dashed: false });
      }

      lines.value = newLines;
    }

    // ─── Factor helpers ───
    function getFactorTitle(f) {
      if (!f) return 'Unnamed';
      if (f.name || f.target_entity_name) return f.name || f.target_entity_name;
      const c = f.target_currency === 'ticket' ? 'Credit' : 'Points';
      return f.earn_factor_type === 'rate' ? `${c} Starter Rules` : `${c} Power Boost`;
    }

    function getFactorMeta(f) {
      if (!f) return '';
      const c = f.target_currency === 'ticket' ? 'Store credit' : 'Points';
      const t = f.earn_factor_type === 'rate' ? 'Base rate' : 'Multiplier';
      const parts = [`${c} (${t})`];
      if (f.earn_factor_type === 'rate' && f.earn_factor_amount) parts.push(`Rate : ฿${f.earn_factor_amount} = 1 point`);
      if (f.window_start || f.window_end) {
        const fmt = d => { if (!d) return '—'; try { const dt = new Date(d); return `${String(dt.getDate()).padStart(2,'0')}.${String(dt.getMonth()+1).padStart(2,'0')}.${String(dt.getFullYear()).slice(2)}`; } catch { return '—'; } };
        parts.push(`${fmt(f.window_start)} - ${fmt(f.window_end)}`);
      }
      return parts.join('  |  ');
    }

    // ─── UI actions ───
    function handleAddFactor(g) {
      editingFactor.value = { earn_factor_type: 'rate', public: true, target_currency: 'points', active_status: true };
      editingGroupId.value = g?.id;
      panel.value = 'factor';
    }

    function handleEditFactor(f) {
      editingFactor.value = { ...f };
      editingGroupId.value = f?.earn_factor_group_id;
      panel.value = 'factor';
    }

    function handleEditFactorGroup() {
      modalType.value = 'factor';
      showModal.value = true;
    }

    async function handleEditConditionGroup(g) {
      try {
        editingCondGroup.value = g?.id
          ? await api.fetchConditionGroupDetails(g.id)
          : { id: null, name: '', conditions: [] };
      } catch {
        editingCondGroup.value = { ...g, conditions: g?.conditions || [] };
      }
      panel.value = 'condition';
    }

    function openCreateFactorGroup() { modalType.value = 'factor'; showModal.value = true; }
    function openCreateConditionGroup() { editingCondGroup.value = null; panel.value = 'condition'; }

    async function handleModalSave(p) {
      try {
        if (modalType.value === 'factor') {
          await api.upsertEarnFactorGroup({ name: p.name, stackable: p.stackable, window_start: p.window_start, window_end: p.window_end, factors: [] });
          await loadFactorGroups();
        }
      } catch (e) { err('Create failed', e); }
      showModal.value = false;
      scheduleLineUpdate();
    }

    async function saveFactorConfig({ groupId, factor }) {
      try {
        const det = await api.fetchEarnFactorGroupDetails(groupId);
        const ex = det?.factors || [];
        const up = factor.id ? ex.map(f => f.id === factor.id ? factor : f) : [...ex, factor];
        await api.upsertEarnFactorGroup({ id: groupId, factors: up });
        await loadFactorGroups();
        panel.value = null;
      } catch (e) { err('Save factor failed', e); }
      scheduleLineUpdate();
    }

    async function saveCondGroupConfig(payload) {
      try {
        await api.upsertConditionGroup(payload);
        await loadCondGroups();
        panel.value = null;
      } catch (e) { err('Save condition group failed', e); }
      scheduleLineUpdate();
    }

    function handleConnectFactor(f, ev) {
      const r = ev?.target?.closest?.('.earn-studio__connect-btn')?.getBoundingClientRect?.()
                || ev?.target?.getBoundingClientRect?.();
      connectPopup.value = {
        open: true,
        pos: r ? { x: r.right + 8, y: r.top } : { x: 400, y: 200 },
        factorId: f.id,
        groupId: f.earn_factor_group_id,
      };
    }

    async function handleConnectSelect(cg) {
      const { factorId, groupId } = connectPopup.value;
      connectPopup.value.open = false;
      try {
        const det = await api.fetchEarnFactorGroupDetails(groupId);
        const fs = (det?.factors || []).map(f => f.id === factorId ? { ...f, earn_conditions_group_id: cg.id } : f);
        await api.upsertEarnFactorGroup({ id: groupId, factors: fs });
        await loadFactorGroups();
        await loadCondGroups();
      } catch (e) { err('Link failed', e); }
      scheduleLineUpdate();
    }

    // ─── Lifecycle ───
    onMounted(() => {
      if (props.content?.authToken) loadAll();
      ro = new ResizeObserver(scheduleLineUpdate);
      nextTick(() => { if (layoutRef.value) ro.observe(layoutRef.value); });
    });

    onBeforeUnmount(() => { ro?.disconnect(); clearTimeout(lineTimer); });

    watch(() => props.content?.authToken, t => { if (t) loadAll(); });

    return {
      rootRef, layoutRef, svgRef,
      content: computed(() => props.content),
      factorGroups, factorsByGroup, allCondGroups, entityOptions, ticketTypes,
      loadingFactorGroups, loadingConditionGroups, expandedRight,
      panel, editingFactor, editingGroupId, editingCondGroup,
      showModal, modalType, hoveredLine, connectPopup,
      rightEntries, lines,
      getFactorTitle, getFactorMeta,
      handleAddFactor, handleEditFactor, handleEditFactorGroup,
      handleEditConditionGroup, handleConnectFactor,
      openCreateFactorGroup, openCreateConditionGroup, handleModalSave,
      saveFactorConfig, saveCondGroupConfig, handleConnectSelect,
      scheduleLineUpdate,
      refreshData: loadAll,
      closePanel: () => { panel.value = null; },
    };
  },
};
</script>

<style scoped lang="scss">
@import 'polaris-weweb-styles';

.earn-studio {
  @include polaris-tokens;
  font-family: var(--p-font-family-sans);
  color: var(--p-color-text);
  background: #F8FAFC;
  width: 100%;
  height: 100%;
  overflow: auto;
  padding: 0 var(--p-space-600) 64px;

  &__layout {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    position: relative;
  }

  &__col {
    flex-shrink: 0;
    z-index: 2;
    position: relative;
    &--left { width: 580px; }
    &--right { width: 500px; }
  }

  &__col-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: var(--p-space-600) 0 var(--p-space-500);
  }

  &__title { @include polaris-text-title; margin: 0; }

  &__primary-btn {
    @include polaris-button-primary;
    @include polaris-button-slim;
    font-size: var(--p-font-size-300);
  }

  &__col-body {
    display: flex;
    flex-direction: column;
    gap: var(--p-space-400);
  }

  // ─── SVG overlay ───
  &__svg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;
    pointer-events: none;
    overflow: visible;
    path { pointer-events: stroke; }
  }

  // ─── Left column: group labels + factor rows ───
  &__factor-section {
    margin-bottom: var(--p-space-100);
  }

  &__group-label {
    display: flex;
    align-items: center;
    gap: var(--p-space-200);
    padding: var(--p-space-050) 0;
    margin-bottom: var(--p-space-200);
  }

  &__group-icon {
    width: 22px; height: 22px; min-width: 22px;
    border-radius: var(--p-border-radius-100);
    background: #F54239; color: #fff;
    display: flex; align-items: center; justify-content: center;
  }

  &__group-name {
    @include polaris-text-subtitle-sm;
    flex: 1; min-width: 0;
    white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  }

  &__link-btn {
    @include polaris-button-plain;
    font-size: var(--p-font-size-300);
    white-space: nowrap; padding: 2px 6px; min-height: auto;
  }

  &__icon-btn {
    width: 22px; height: 22px;
    display: flex; align-items: center; justify-content: center;
    background: none; border: none; border-radius: var(--p-border-radius-100);
    color: var(--p-color-icon); cursor: pointer;
    &:hover { background: var(--p-color-bg-surface-hover); }
    &--hover { opacity: 0; transition: opacity 0.1s; }
  }

  &__factor-row {
    position: relative;
    display: flex; align-items: stretch;
    background: var(--p-color-bg-surface);
    border: 1px solid var(--p-color-border-info);
    border-radius: var(--p-border-radius-200);
    box-shadow: var(--p-shadow-card-sm);
    margin-bottom: var(--p-space-200);
    overflow: visible;
    &:last-child { margin-bottom: 0; }
    &:hover {
      box-shadow: var(--p-shadow-card-hover);
      .earn-studio__icon-btn--hover { opacity: 1; }
      .earn-studio__connect-btn { opacity: 1; }
    }
  }

  &__factor-bar {
    width: 3px; min-width: 3px;
    border-radius: var(--p-border-radius-200) 0 0 var(--p-border-radius-200);
    &--points { background: var(--p-color-text-info); }
    &--credit { background: var(--p-color-text-success); }
  }

  &__factor-body {
    flex: 1; display: flex; align-items: center; gap: var(--p-space-200);
    padding: var(--p-space-300); min-width: 0;
  }

  &__factor-icon {
    width: 30px; height: 30px; min-width: 30px;
    border-radius: 5px; border: 0.7px solid var(--p-color-border);
    display: flex; align-items: center; justify-content: center;
    &--points { color: var(--p-color-text-info); }
    &--credit { color: var(--p-color-text); }
  }

  &__factor-info { flex: 1; min-width: 0; }

  &__factor-title {
    @include polaris-text-subtitle;
    white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  }

  &__factor-meta {
    @include polaris-text-description;
    white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  }

  &__factor-mult {
    font-size: var(--p-font-size-300); color: var(--p-color-text-success);
    white-space: nowrap; flex-shrink: 0;
    strong { font-weight: var(--p-font-weight-bold); }
  }

  &__connect-btn {
    position: absolute; right: -15px; top: 50%; transform: translateY(-50%);
    width: 28px; height: 28px; border-radius: var(--p-border-radius-full);
    background: var(--p-color-bg-fill-brand); color: var(--p-color-text-on-color);
    border: 3px solid var(--p-color-bg-surface);
    display: flex; align-items: center; justify-content: center;
    cursor: pointer; opacity: 0; z-index: 5;
    box-shadow: var(--p-shadow-button);
    transition: opacity 0.1s, background 0.1s, transform 0.1s;
    &:hover { background: var(--p-color-bg-fill-brand-hover); transform: translateY(-50%) scale(1.1); }
  }

  &__no-factors {
    @include polaris-text-description;
    color: var(--p-color-text-disabled);
    text-align: center; padding: var(--p-space-200) 0;
  }

  // ─── Common ───
  &__loading { display: flex; justify-content: center; padding: var(--p-space-1200) 0; }
  &__spinner { @include polaris-spinner; }
  &__empty {
    @include polaris-text-description;
    text-align: center; padding: var(--p-space-1000) var(--p-space-400);
    border: 2px dashed var(--p-color-border);
    border-radius: var(--p-border-radius-200);
    color: var(--p-color-text-disabled);
  }
}

.slide-enter-active, .slide-leave-active { transition: transform var(--p-motion-duration-300) var(--p-motion-ease); }
.slide-enter-from, .slide-leave-to { transform: translateX(100%); }
</style>
