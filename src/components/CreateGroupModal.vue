<template>
  <div class="modal-backdrop" @click.self="$emit('close')">
    <div class="modal">
      <div class="modal__header">
        <h3 class="modal__title">{{ title }}</h3>
        <button class="modal__close" @click="$emit('close')">
          <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
            <path d="M6 6l8 8M14 6l-8 8" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
          </svg>
        </button>
      </div>

      <div class="modal__body">
        <div class="modal__field">
          <label class="modal__label">Group name</label>
          <input class="modal__input" v-model="form.name" :placeholder="namePlaceholder" />
        </div>

        <template v-if="type === 'factor'">
          <div class="modal__field">
            <label class="modal__label">Stackable</label>
            <label class="modal__switch-wrapper">
              <input type="checkbox" class="modal__switch" v-model="form.stackable" />
              <span class="modal__switch-label">{{ form.stackable ? 'Yes — multipliers stack' : 'No — best multiplier wins' }}</span>
            </label>
          </div>

          <div class="modal__field-row">
            <div class="modal__field modal__field--half">
              <label class="modal__label">Window start</label>
              <input class="modal__date-input" type="date" v-model="form.window_start" />
            </div>
            <div class="modal__field modal__field--half">
              <label class="modal__label">Window end</label>
              <input class="modal__date-input" type="date" v-model="form.window_end" />
            </div>
          </div>
        </template>
      </div>

      <div class="modal__footer">
        <button class="modal__btn-secondary" @click="$emit('close')">Cancel</button>
        <button class="modal__btn-primary" @click="handleSave" :disabled="saving || !form.name?.trim()">
          {{ saving ? 'Creating...' : 'Create' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue';

export default {
  props: {
    type: { type: String, default: 'factor' },
  },
  emits: ['close', 'save'],
  setup(props, { emit }) {
    const saving = ref(false);

    const title = computed(() =>
      props.type === 'factor' ? 'Create Earn Factor Group' : 'Create Earn Condition Group'
    );

    const namePlaceholder = computed(() =>
      props.type === 'factor' ? 'e.g. Standard Earning Rule' : 'e.g. Tier Perks'
    );

    const form = ref({
      name: '',
      stackable: true,
      window_start: '',
      window_end: '',
    });

    async function handleSave() {
      if (!form.value.name?.trim()) return;
      saving.value = true;
      try {
        const payload = { name: form.value.name.trim() };
        if (props.type === 'factor') {
          payload.stackable = form.value.stackable;
          payload.window_start = form.value.window_start || null;
          payload.window_end = form.value.window_end || null;
          if (payload.window_start) payload.window_start = new Date(payload.window_start).toISOString();
          if (payload.window_end) payload.window_end = new Date(payload.window_end).toISOString();
        }
        emit('save', payload);
      } finally {
        saving.value = false;
      }
    }

    return { form, title, namePlaceholder, saving, handleSave };
  },
};
</script>

<style scoped lang="scss">
@import 'polaris-weweb-styles';

.modal-backdrop {
  @include polaris-modal-backdrop;
}

.modal {
  @include polaris-modal;
  max-width: 480px;

  &__header { @include polaris-modal-header; }
  &__title { @include polaris-text-heading-sm; margin: 0; }
  &__close {
    width: 32px; height: 32px;
    display: flex; align-items: center; justify-content: center;
    background: transparent; border: none; border-radius: var(--p-border-radius-100);
    color: var(--p-color-icon); cursor: pointer;
    &:hover { background: var(--p-color-bg-surface-hover); }
  }
  &__body { @include polaris-modal-content; }
  &__field {
    margin-bottom: var(--p-space-400);
    &--half { flex: 1; min-width: 0; }
  }
  &__field-row { display: flex; gap: var(--p-space-300); margin-bottom: var(--p-space-400); }
  &__label { @include polaris-label; display: block; margin-bottom: var(--p-space-100); }
  &__input { @include polaris-input; }
  &__date-input { @include polaris-date-field; }
  &__switch-wrapper { @include polaris-switch-wrapper; }
  &__switch { @include polaris-switch; }
  &__switch-label { @include polaris-text-body; }
  &__footer { @include polaris-modal-footer; }
  &__btn-primary { @include polaris-button-primary; }
  &__btn-secondary { @include polaris-button-default; }
}
</style>
