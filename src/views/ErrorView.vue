<template>
  <main class="flex flex-col items-center justify-center text-center">
    <span class="text-2xl">{{ store.error || message }}</span>
    <button class="btn mt-2 px-4 py-1" @click="router.push('/')">Refresh</button>
  </main>
</template>

<script setup lang="ts">
import { useUiStore } from '@/stores/ui'
import { useRouter } from 'vue-router'
const router = useRouter()
const store = useUiStore()
defineProps({
  message: {
    type: String,
    default: 'An unknown error occurred'
  }
})
</script>
