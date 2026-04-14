<template>
  <div class="order-1 flex-none grow-0 self-stretch">
    <div v-show="connected" class="mb-4">
      <!-- Active mode: cancel + input/dropzone -->
      <div v-if="inputMode !== 'none'">
        <button
          class="mb-4 inline-flex items-center justify-center gap-1.5 rounded-full pl-2 pr-4 py-1.5 text-sm font-medium text-black/60 transition duration-150 hover:bg-black/5 hover:text-black/85 dark:text-white/60 dark:hover:bg-white/10 dark:hover:text-white/85"
          @click="cancelAction"
        >
          <MdiIcon class="size-4" :path="mdiArrowLeft" />
          Cancel
        </button>

        <div
          v-show="isTextInputMode"
          class="mb-4 flex items-center rounded-2xl border border-solid border-black/25 bg-black px-6 md:px-4 dark:border-white/25"
        >
          <span class="font-mono select-none">$</span>
          <input
            ref="inputRef"
            v-model="inputValue"
            class="w-full bg-transparent p-2 font-mono outline-none"
            type="text"
            :placeholder="inputPlaceholder"
            @keyup.enter="submitInput"
          />
        </div>

        <FileDropZone
          v-if="inputMode === 'boot'"
          class="mb-4"
          label="Select boot image"
          hint="Temporarily boots the device into the provided image"
          @file-selected="bootImageExec"
        />
        <FileDropZone
          v-if="inputMode === 'flash-file'"
          class="mb-4"
          :label="`Flash to '${partition}' partition`"
          @file-selected="flashImageExec"
        />
        <FileDropZone
          v-if="inputMode === 'wipe-super-file'"
          class="mb-4"
          label="Select super_empty.img"
          hint="This will overwrite the super partition"
          @file-selected="onWipeSuperFileSelected"
        />
      </div>

      <!-- Button grid (hidden while an action is active) -->
      <div v-else class="grid grid-cols-3 gap-3">
        <button
          class="bg-brand-light/60 hover:bg-brand-light dark:bg-brand-dark/60 dark:text-brand-light dark:hover:bg-brand-dark flex flex-col items-center justify-center gap-2 rounded-2xl p-4 text-center text-sm font-medium transition duration-150"
          @click="bootImage"
        >
          <MdiIcon class="h-6 w-6" :path="mdiAndroid" />
          Boot image
        </button>

        <button
          class="bg-brand-light/60 hover:bg-brand-light dark:bg-brand-dark/60 dark:text-brand-light dark:hover:bg-brand-dark flex flex-col items-center justify-center gap-2 rounded-2xl p-4 text-center text-sm font-medium transition duration-150"
          @click="promptFlashImage"
        >
          <MdiIcon class="h-6 w-6" :path="mdiFlash" />
          Flash image
        </button>

        <button
          class="flex flex-col items-center justify-center gap-2 rounded-2xl bg-red-100/60 p-4 text-center text-sm font-medium text-red-700 transition duration-150 hover:bg-red-100 dark:bg-red-950/30 dark:text-red-400 dark:hover:bg-red-950/50"
          @click="wipeSuper"
        >
          <MdiIcon class="h-6 w-6" :path="mdiDeleteForever" />
          Wipe super
        </button>

        <button
          class="bg-brand-light/60 hover:bg-brand-light dark:bg-brand-dark/60 dark:text-brand-light dark:hover:bg-brand-dark flex flex-col items-center justify-center gap-2 rounded-2xl p-4 text-center text-sm font-medium transition duration-150"
          @click="promptGetVariable"
        >
          <MdiIcon class="h-6 w-6" :path="mdiInformation" />
          Get variable
        </button>

        <button
          class="bg-brand-light/60 hover:bg-brand-light dark:bg-brand-dark/60 dark:text-brand-light dark:hover:bg-brand-dark flex flex-col items-center justify-center gap-2 rounded-2xl p-4 text-center text-sm font-medium transition duration-150"
          @click="promptReboot"
        >
          <MdiIcon class="h-6 w-6" :path="mdiRestart" />
          Reboot
        </button>

        <button
          class="bg-brand-light/60 hover:bg-brand-light dark:bg-brand-dark/60 dark:text-brand-light dark:hover:bg-brand-dark flex flex-col items-center justify-center gap-2 rounded-2xl p-4 text-center text-sm font-medium transition duration-150"
          @click="promptRunCommand"
        >
          <MdiIcon class="h-6 w-6" :path="mdiConsole" />
          Run command
        </button>
      </div>
    </div>

    <div v-show="!connected" class="mb-4 w-full text-center">
      <button class="btn btn-primary mx-auto px-4 py-1" @click="connect">Connect</button>
      <div class="mt-3 flex justify-center text-sm">Reboot to fastboot before connecting.</div>
    </div>

    <ConfirmDialog
      :open="confirmOpen"
      title="Wipe super partition?"
      :body="wipeSuperConfirmBody"
      confirm-label="Wipe"
      :destructive="true"
      @confirm="onWipeSuperConfirm"
      @cancel="onWipeSuperCancel"
    />
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onMounted, onUnmounted, ref, useTemplateRef, watch } from 'vue'
import * as fastboot from 'android-fastboot'
import {
  mdiAndroid,
  mdiArrowLeft,
  mdiConsole,
  mdiDeleteForever,
  mdiFlash,
  mdiInformation,
  mdiRestart
} from '@mdi/js'
import MdiIcon from '@/components/mdi-icon/MdiIcon.vue'
import ConfirmDialog from './ConfirmDialog.vue'
import FileDropZone from './FileDropZone.vue'

defineOptions({ name: 'FastbootView' })

const connected = ref(false)
const device = ref<fastboot.FastbootDevice | null>(null)
const partition = ref('')
const inputMode = ref<
  | 'none'
  | 'flash'
  | 'variable'
  | 'reboot'
  | 'run-command'
  | 'boot'
  | 'flash-file'
  | 'wipe-super-file'
>('none')
const inputValue = ref('')
const confirmOpen = ref(false)
const pendingWipeSuperFile = ref<File | null>(null)

const isTextInputMode = computed(() =>
  ['flash', 'variable', 'reboot', 'run-command'].includes(inputMode.value)
)

const inputPlaceholder = computed(() => {
  if (inputMode.value === 'flash') return 'Partition name (e.g. boot)'
  if (inputMode.value === 'variable') return 'Variable name (e.g. version-bootloader)'
  if (inputMode.value === 'reboot') return 'Reboot target (e.g. recovery)'
  if (inputMode.value === 'run-command') return 'Command (e.g. flashing unlock, oem unlock)'
  return ''
})

const wipeSuperConfirmBody = computed(() => {
  const name = pendingWipeSuperFile.value?.name
  return `You are about to wipe the super partition${name ? ` using "${name}"` : ''}. This will erase all user data on devices using dynamic partitions (shipping with Android 10+). Make sure you have a backup before continuing.`
})

const inputRef = useTemplateRef('inputRef')
const props = defineProps<{
  appendLog: (message: string) => void
  clearLog: () => void
}>()

onMounted(() => {
  device.value = new fastboot.FastbootDevice()

  fastboot.setDebugLevel(2 /* verbose */)
  fastboot.setDebugLogger((...data) => {
    console.log(...data)
    props.appendLog([...data].join(' '))
  })
})

onUnmounted(() => {
  connected.value = false
  device.value = null
  partition.value = ''
})

async function connect() {
  try {
    await device.value?.connect()

    props.clearLog()
    connected.value = true

    await device.value?.getVariable('product')
    await device.value?.getVariable('serialno')
  } catch (err) {
    console.log(err)
  }
}

function cancelAction() {
  inputMode.value = 'none'
  inputValue.value = ''
  pendingWipeSuperFile.value = null
}

function bootImage() {
  inputMode.value = 'boot'
}

async function bootImageExec(file: File) {
  inputMode.value = 'none'
  await device.value?.bootBlob(file)
}

function promptFlashImage() {
  inputMode.value = 'flash'
  inputValue.value = ''
}

function promptGetVariable() {
  inputMode.value = 'variable'
  inputValue.value = ''
}

function promptReboot() {
  inputMode.value = 'reboot'
  inputValue.value = ''
}

function promptRunCommand() {
  inputMode.value = 'run-command'
  inputValue.value = ''
}

async function flashImageExec(file: File) {
  inputMode.value = 'none'
  await device.value?.flashBlob(partition.value, 'current', file)
}

function wipeSuper() {
  inputMode.value = 'wipe-super-file'
}

function onWipeSuperFileSelected(file: File) {
  pendingWipeSuperFile.value = file
  confirmOpen.value = true
}

async function onWipeSuperConfirm() {
  confirmOpen.value = false
  inputMode.value = 'none'
  if (pendingWipeSuperFile.value) {
    await device.value?.wipeSuper(pendingWipeSuperFile.value)
    pendingWipeSuperFile.value = null
  }
}

function onWipeSuperCancel() {
  confirmOpen.value = false
  pendingWipeSuperFile.value = null
  inputMode.value = 'none'
}

function flashImage() {
  const value = inputValue.value.trim()
  if (!value) return
  partition.value = value
  inputMode.value = 'flash-file'
  inputValue.value = ''
}

async function getVariable() {
  const value = inputValue.value.trim()
  if (!value) return
  inputMode.value = 'none'
  inputValue.value = ''
  await device.value?.getVariable(value)
}

async function reboot() {
  const value = inputValue.value.trim()
  inputMode.value = 'none'
  inputValue.value = ''
  await device.value?.reboot(value)
}

async function runCommand() {
  const raw = inputValue.value.trim()
  if (!raw) return
  inputMode.value = 'none'
  inputValue.value = ''
  const value = raw.replace(/^fastboot\s+/i, '')
  await device.value?.runCommand(value)
}

async function submitInput() {
  if (inputMode.value === 'flash') {
    flashImage()
  } else if (inputMode.value === 'variable') {
    await getVariable()
  } else if (inputMode.value === 'reboot') {
    await reboot()
  } else if (inputMode.value === 'run-command') {
    await runCommand()
  }
}

watch(inputMode, async () => {
  if (isTextInputMode.value) {
    await nextTick(() => inputRef.value?.focus())
  }
})
</script>
