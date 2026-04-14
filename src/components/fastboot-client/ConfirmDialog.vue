<template>
  <Teleport to="body">
    <!-- Portal container: always mounted so child leave transitions can complete -->
    <div
      class="fixed inset-0 z-50 flex items-center justify-center p-6"
      :class="open ? 'pointer-events-auto' : 'pointer-events-none'"
    >
      <!-- Scrim -->
      <Transition
        enter-from-class="opacity-0"
        enter-to-class="opacity-100"
        enter-active-class="transition-opacity duration-200"
        leave-from-class="opacity-100"
        leave-to-class="opacity-0"
        leave-active-class="transition-opacity duration-150"
      >
        <div v-if="open" class="absolute inset-0 bg-black/32" @click="handleCancel" />
      </Transition>

      <!-- Dialog panel -->
      <Transition
        enter-from-class="opacity-0 scale-[0.85]"
        enter-to-class="opacity-100 scale-100"
        enter-active-class="transition-[opacity,scale] duration-200 ease-out"
        leave-from-class="opacity-100 scale-100"
        leave-to-class="opacity-0 scale-[0.85]"
        leave-active-class="transition-[opacity,scale] duration-150 ease-in"
      >
        <div
          v-if="open"
          ref="panelRef"
          role="dialog"
          aria-modal="true"
          tabindex="-1"
          class="relative w-[min(560px,calc(100vw-48px))] min-w-70 rounded-[28px] bg-white p-6 shadow-xl outline-none dark:bg-[#2c2c2c] dark:text-white/85"
          @keydown.esc.prevent="handleCancel"
        >
          <slot name="icon" />
          <h2 class="mt-0 mb-4 text-xl font-medium">{{ title }}</h2>
          <p class="mt-0 mb-6 text-sm text-black/60 dark:text-white/60">{{ body }}</p>
          <div class="flex justify-end gap-2">
            <button
              class="text-brand-primary hover:bg-brand-primary/10 rounded-full px-4 py-2 text-sm font-medium transition duration-150"
              @click="handleCancel"
            >
              {{ cancelLabel ?? 'Cancel' }}
            </button>
            <button
              class="rounded-full px-4 py-2 text-sm font-medium transition duration-150"
              :class="
                destructive
                  ? 'text-red-600 hover:bg-red-600/10 dark:text-red-400 dark:hover:bg-red-400/10'
                  : 'text-brand-primary hover:bg-brand-primary/10'
              "
              @click="handleConfirm"
            >
              {{ confirmLabel ?? 'Confirm' }}
            </button>
          </div>
        </div>
      </Transition>
    </div>
  </Teleport>
</template>

<script setup lang="ts">
import { nextTick, useTemplateRef, watch } from 'vue'

const props = defineProps<{
  open: boolean
  title: string
  body: string
  confirmLabel?: string
  cancelLabel?: string
  destructive?: boolean
}>()

const emit = defineEmits<{
  confirm: []
  cancel: []
}>()

const panelRef = useTemplateRef('panelRef')

watch(
  () => props.open,
  async (open) => {
    if (open) {
      await nextTick()
      panelRef.value?.focus()
    }
  }
)

function handleConfirm() {
  emit('confirm')
}

function handleCancel() {
  emit('cancel')
}
</script>
