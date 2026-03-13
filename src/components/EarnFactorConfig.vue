<template>
  <div class="config-panel" :style="{ width: panelWidth }">
    <div class="config-panel__header">
      <h3 class="config-panel__title">{{ isNew ? 'Add earn factor' : 'Edit earn factor' }}</h3>
      <button class="config-panel__close" @click="$emit('close')">
        <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
          <path d="M6 6l8 8M14 6l-8 8" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
        </svg>
      </button>
    </div>

    <div class="config-panel__body">
      <div class="config-panel__field">
        <label class="config-panel__label">Earning rule name</label>
        <input class="config-panel__input" v-model="form.name" placeholder="Enter name..." />
      </div>

      <div class="config-panel__field">
        <label class="config-panel__label">Earn factor type</label>
        <div class="config-panel__radio-group">
          <label class="config-panel__radio-item">
            <input type="radio" v-model="form.earn_factor_type" value="rate" />
            <span>Base rate</span>
          </label>
          <label class="config-panel__radio-item">
            <input type="radio" v-model="form.earn_factor_type" value="multiplier" />
            <span>Multiplier</span>
          </label>
        </div>
      </div>

      <div class="config-panel__field">
        <label class="config-panel__label">Amount</label>
        <input class="config-panel__input" type="number" v-model.number="form.earn_factor_amount" placeholder="e.g. 100 or 3" step="any" min="0" />
      </div>

      <div class="config-panel__field-row">
        <div class="config-panel__field config-panel__field--half">
          <label class="config-panel__label">Target currency</label>
          <select class="config-panel__select" v-model="form.target_currency">
            <option value="points">Points</option>
            <option value="ticket">Store credit</option>
          </select>
        </div>
        <div class="config-panel__field config-panel__field--half">
          <label class="config-panel__label">Target currency type</label>
          <select class="config-panel__select" v-model="form.target_entity_id" :disabled="form.target_currency !== 'ticket'">
            <option :value="null">—</option>
            <option v-for="tt in ticketTypes" :key="tt.id" :value="tt.id">{{ tt.name }}</option>
          </select>
        </div>
      </div>

      <div class="config-panel__field-row">
        <div class="config-panel__field config-panel__field--half">
          <label class="config-panel__label">Window start</label>
          <input class="config-panel__date-input" type="date" v-model="form.window_start" />
        </div>
        <div class="config-panel__field-separator">To</div>
        <div class="config-panel__field config-panel__field--half">
          <label class="config-panel__label">Window end</label>
          <input class="config-panel__date-input" type="date" v-model="form.window_end" />
        </div>
      </div>

      <div class="config-panel__field">
        <label class="config-panel__label">Currency expiry (days)</label>
        <div class="config-panel__input-suffix">
          <input class="config-panel__input" type="number" v-model.number="form.window_end_ttl_days" placeholder="e.g. 30" min="0" />
          <span class="config-panel__suffix">Day</span>
        </div>
      </div>

      <div class="config-panel__field">
        <label class="config-panel__label">Is this rule public or private?</label>
        <div class="config-panel__radio-group">
          <label class="config-panel__radio-item">
            <input type="radio" v-model="form.public" :value="true" />
            <span>Public</span>
          </label>
          <label class="config-panel__radio-item">
            <input type="radio" v-model="form.public" :value="false" />
            <span>Private</span>
          </label>
        </div>
      </div>

      <div class="config-panel__field">
        <label class="config-panel__label">Assign earn condition group</label>
        <select class="config-panel__select" v-model="form.earn_conditions_group_id">
          <option :value="null">— None —</option>
          <option v-for="cg in conditionGroups" :key="cg.id" :value="cg.id">
            {{ cg.name || 'Untitled' }}
          </option>
        </select>
      </div>
    </div>

    <div class="config-panel__footer">
      <button class="config-panel__btn-secondary" @click="$emit('close')">Cancel</button>
      <button class="config-panel__btn-primary" @click="handleSave" :disabled="saving">
        {{ saving ? 'Saving...' : 'Save' }}
      </button>
    </div>
  </div>
</template>

<script>
import { ref, watch, computed } from 'vue';

export default {
  props: {
    factor: { type: Object, default: null },
    groupId: { type: String, default: null },
    conditionGroups: { type: Array, default: () => [] },
    ticketTypes: { type: Array, default: () => [] },
    panelWidth: { type: String, default: '380px' },
  },
  emits: ['close', 'save'],
  setup(props, { emit }) {
    const saving = ref(false);
    const isNew = computed(() => !props.factor?.id);

    const defaultForm = () => ({
      id: null,
      name: '',
      earn_factor_type: 'rate',
      earn_factor_amount: null,
      target_currency: 'points',
      target_entity_id: null,
      window_start: '',
      window_end: '',
      window_end_ttl_days: null,
      public: true,
      earn_conditions_group_id: null,
      active_status: true,
    });

    const form = ref(defaultForm());

    function toDateStr(val) {
      if (!val) return '';
      try {
        return new Date(val).toISOString().split('T')[0];
      } catch { return ''; }
    }

    watch(() => props.factor, (f) => {
      if (f) {
        form.value = {
          id: f.id || null,
          name: f.name || '',
          earn_factor_type: f.earn_factor_type || 'rate',
          earn_factor_amount: f.earn_factor_amount ?? null,
          target_currency: f.target_currency || 'points',
          target_entity_id: f.target_entity_id || null,
          window_start: toDateStr(f.window_start),
          window_end: toDateStr(f.window_end),
          window_end_ttl_days: f.window_end_ttl_days ?? null,
          public: f.public !== false,
          earn_conditions_group_id: f.earn_conditions_group_id || null,
          active_status: f.active_status !== false,
        };
      } else {
        form.value = defaultForm();
      }
    }, { immediate: true });

    async function handleSave() {
      saving.value = true;
      try {
        const payload = { ...form.value };
        if (!payload.id) delete payload.id;
        if (payload.window_start) payload.window_start = new Date(payload.window_start).toISOString();
        else payload.window_start = null;
        if (payload.window_end) payload.window_end = new Date(payload.window_end).toISOString();
        else payload.window_end = null;
        if (payload.target_currency !== 'ticket') payload.target_entity_id = null;

        emit('save', {
          groupId: props.groupId,
          factor: payload,
        });
      } finally {
        saving.value = false;
      }
    }

    return { form, isNew, saving, handleSave };
  },
};
</script>

<style scoped lang="scss">
@import 'polaris-weweb-styles';

.config-panel {
  position: fixed;
  top: 0;
  right: 0;
  height: 100vh;
  background: var(--p-color-bg-surface);
  box-shadow: -4px 0 12px rgba(0, 0, 0, 0.1);
  z-index: 300;
  display: flex;
  flex-direction: column;
  overflow: hidden;

  &__header {
    @include polaris-modal-header;
  }

  &__title {
    @include polaris-text-heading-sm;
    margin: 0;
  }

  &__close {
    width: 32px;
    height: 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: transparent;
    border: none;
    border-radius: var(--p-border-radius-100);
    color: var(--p-color-icon);
    cursor: pointer;

    &:hover {
      background: var(--p-color-bg-surface-hover);
    }
  }

  &__body {
    flex: 1;
    overflow-y: auto;
    padding: var(--p-space-400);
  }

  &__field {
    margin-bottom: var(--p-space-400);

    &--half {
      flex: 1;
      min-width: 0;
    }
  }

  &__field-row {
    display: flex;
    gap: var(--p-space-200);
    align-items: flex-end;
    margin-bottom: var(--p-space-400);
  }

  &__field-separator {
    @include polaris-text-body-subdued;
    padding-bottom: var(--p-space-200);
    flex-shrink: 0;
  }

  &__label {
    @include polaris-label;
    display: block;
    margin-bottom: var(--p-space-100);
  }

  &__input {
    @include polaris-input;
  }

  &__select {
    @include polaris-select;
  }

  &__date-input {
    @include polaris-date-field;
  }

  &__input-suffix {
    position: relative;
  }

  &__suffix {
    position: absolute;
    right: var(--p-space-300);
    top: 50%;
    transform: translateY(-50%);
    color: var(--p-color-text-secondary);
    font-size: var(--p-font-size-300);
    pointer-events: none;
  }

  &__radio-group {
    display: flex;
    gap: var(--p-space-400);
  }

  &__radio-item {
    @include polaris-radio-wrapper;
    font-size: var(--p-font-size-350);

    input[type="radio"] {
      @include polaris-radio;
    }
  }

  &__footer {
    @include polaris-modal-footer;
  }

  &__btn-primary {
    @include polaris-button-primary;
  }

  &__btn-secondary {
    @include polaris-button-default;
  }
}
</style>
