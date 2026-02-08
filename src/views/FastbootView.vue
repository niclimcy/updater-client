<template>
  <div class="flex h-full w-full flex-col">
    <div class="h-full w-full grow overflow-auto">
      <div class="mx-auto max-w-189 min-w-0 px-8">
        <div class="flex flex-col items-start gap-4 px-6 py-10 sm:px-4">
          <h1 class="m-0 flex-none self-stretch text-3xl font-medium">Fastboot client</h1>
          <div v-show="webUsbSupported" class="order-1 flex-none grow-0 self-stretch">
            <div v-show="connected" class="mb-4 justify-center">
              <textarea ref="log" class="resize-none font-mono" cols="80" rows="20"></textarea>

              <input ref="bootImage" class="hidden" type="file" @change="bootImageExec" />
              <input ref="flashImage" class="hidden" type="file" @change="flashImageExec" />

              <button class="btn mr-3 mb-3 px-4 py-1" @click="bootImage">Boot image</button>
              <button class="btn mr-3 mb-3 px-4 py-1" @click="flashImage">Flash image</button>
              <button class="btn mr-3 mb-3 px-4 py-1" @click="getVariable">Get variable</button>
              <button class="btn mr-3 mb-3 px-4 py-1" @click="rebootToRecovery">
                Reboot to recovery
              </button>
            </div>
            <div v-show="!connected" class="mb-4 flex justify-center">
              <button class="btn px-4 py-1" @click="connect">Connect</button>
            </div>
          </div>
          <p v-show="!webUsbSupported">Your browser does not support WebUSB!</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onUnmounted, ref, useTemplateRef } from 'vue'
import { FastbootClient } from '@aepyornis/fastboot.ts'

const connected = ref(false)
const client = ref<FastbootClient | null>(null)
const partition = ref('')
// @ts-expect-error: Some browsers have WebUSB, do not enforce strict type check here
const webUsbSupported = typeof navigator !== 'undefined' && navigator.usb !== undefined

const log = useTemplateRef('log')
const bootImageInput = useTemplateRef('bootImage')
const flashImageInput = useTemplateRef('flashImage')

function appendLog(message: string) {
  if (log.value) {
    log.value.value += message + '\n'
    log.value.scrollTop = log.value.scrollHeight
  }
}

const logger = { log: appendLog }

onUnmounted(() => {
  connected.value = false
  client.value = null
  partition.value = ''
})

async function connect() {
  try {
    const usbDevice = await FastbootClient.requestUsbDevice()
    client.value = new FastbootClient(usbDevice, logger)

    await client.value?.getVar('product')
    await client.value?.getVar('serialno')
    connected.value = true
  } catch (err) {
    console.log(err)
  }
}

function bootImage() {
  bootImageInput.value?.click()
}
async function bootImageExec(event: Event) {
  const file = (event?.currentTarget as HTMLInputElement)?.files?.[0]
  if (!file || !client.value) return
  const buffer = await file.arrayBuffer()
  await client.value.fd.transferData(buffer)
  const res = await client.value.fd.sendCommand('boot')

  if (res && res.status === 'OKAY') {
    client.value = null
    connected.value = false
  }
}

function flashImage() {
  const promptValue = window.prompt('Partition name (e.g. boot)', '')
  partition.value = promptValue ?? ''

  if (partition.value.length > 0) {
    flashImageInput.value?.click()
  }
}

async function flashImageExec(event: Event) {
  const file = (event?.currentTarget as HTMLInputElement)?.files?.[0]
  if (!file || !partition.value) return
  await client.value?.doFlash(partition.value, file, 'current')
}

async function getVariable() {
  const variable = window.prompt('Variable name (e.g. version-bootloader)', '')
  if (!variable) return
  await client.value?.getVar(variable)
}

async function rebootToRecovery() {
  const res = await client.value?.fd.exec('reboot-recovery')
  if (res && res.status === 'OKAY') {
    client.value = null
    connected.value = false
  }
}
</script>
