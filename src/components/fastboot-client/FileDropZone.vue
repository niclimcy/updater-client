<template>
  <div
    class="flex cursor-pointer flex-col items-center justify-center gap-3 rounded-2xl border-2 border-dashed p-8 transition duration-150"
    :class="
      isDragging
        ? 'border-brand-primary bg-brand-primary/5'
        : 'hover:border-brand-primary/50 dark:hover:border-brand-primary/50 border-black/20 dark:border-white/20'
    "
    @dragover.prevent="isDragging = true"
    @dragleave.prevent="isDragging = false"
    @drop.prevent="onDrop"
    @click="fileInput?.click()"
  >
    <input ref="fileInput" class="hidden" type="file" :accept="accept" @change="onInputChange" />

    <template v-if="!selectedFile">
      <MdiIcon class="h-10 w-10 text-black/40 dark:text-white/40" :path="mdiUpload" />
      <div class="text-center">
        <p class="m-0 text-sm font-medium">{{ label }}</p>
        <p v-if="hint" class="m-0 mt-1 text-xs text-black/50 dark:text-white/50">{{ hint }}</p>
        <p class="m-0 mt-1 text-xs text-black/40 dark:text-white/40">
          Drag and drop or click to select
        </p>
      </div>
    </template>

    <template v-else>
      <MdiIcon class="text-brand-primary h-8 w-8" :path="mdiFileOutline" />
      <div class="text-center">
        <p class="m-0 text-sm font-medium">{{ selectedFile.name }}</p>
        <p class="m-0 mt-1 text-xs text-black/50 dark:text-white/50">
          {{ formatSize(selectedFile.size) }}
        </p>
        <button class="text-brand-primary mt-2 text-xs underline" @click.stop="fileInput?.click()">
          Change
        </button>
      </div>
    </template>
  </div>
</template>

<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'
import { mdiUpload, mdiFileOutline } from '@mdi/js'
import MdiIcon from '@/components/mdi-icon/MdiIcon.vue'

defineProps<{
  label: string
  hint?: string
  accept?: string
}>()

const emit = defineEmits<{
  'file-selected': [file: File]
}>()

const fileInput = useTemplateRef('fileInput')
const isDragging = ref(false)
const selectedFile = ref<File | null>(null)

function onDrop(e: DragEvent) {
  isDragging.value = false
  const file = e.dataTransfer?.files?.[0]
  if (file) selectFile(file)
}

function onInputChange(e: Event) {
  const file = (e.currentTarget as HTMLInputElement).files?.[0]
  if (file) selectFile(file)
}

function selectFile(file: File) {
  selectedFile.value = file
  emit('file-selected', file)
}

function formatSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
}
</script>
