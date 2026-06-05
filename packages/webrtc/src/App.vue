<script setup lang="ts">
import { ref, provide, onMounted, onUnmounted, shallowRef } from 'vue'
import { HIDTransportKey, loadWasm } from '@openterface/core'
import { useWebRtcTransport } from './composables/useWebRtcTransport'
import { useWebRtcJsonCommands } from './composables/useWebRtcJsonCommands'

const transport = useWebRtcTransport()
provide(HIDTransportKey, transport)

const jsonCommands = useWebRtcJsonCommands(transport)

const videoRef = shallowRef<HTMLVideoElement>()
const inputRef = shallowRef<HTMLInputElement>()

function focusInput() {
  inputRef.value?.focus()
}

function onInputBlur() {
  // Re-focus after a short delay to prevent immediate re-blur
  setTimeout(() => inputRef.value?.focus(), 100)
}

const isConnecting = ref(false)
const error = ref<string | null>(null)

// Track mouse state for click detection
let lastButtonState = 0

async function connect() {
  isConnecting.value = true
  error.value = null
  try {
    const success = await transport.connect()
    if (!success) {
      error.value = 'Failed to connect'
    }
  } catch (err) {
    error.value = String(err)
  } finally {
    isConnecting.value = false
  }
}

function disconnect() {
  transport.disconnect()
}

// Mouse event handlers
function onMouseMove(e: MouseEvent) {
  if (!transport.isConnected.value) return
  const video = e.target as HTMLVideoElement
  const rect = video.getBoundingClientRect()
  const x = Math.round((e.clientX - rect.left) * 1920 / rect.width)
  const y = Math.round((e.clientY - rect.top) * 1080 / rect.height)
  jsonCommands.sendMouseAbsolute(x, y, lastButtonState)
}

function onMouseDown(e: MouseEvent) {
  if (!transport.isConnected.value) return
  const video = e.target as HTMLVideoElement
  const rect = video.getBoundingClientRect()
  const x = Math.round((e.clientX - rect.left) * 1920 / rect.width)
  const y = Math.round((e.clientY - rect.top) * 1080 / rect.height)
  lastButtonState = e.button === 0 ? 1 : e.button === 1 ? 2 : e.button === 2 ? 4 : 0
  jsonCommands.sendMouseButton(x, y, e.button === 0 ? 'left' : e.button === 1 ? 'middle' : 'right', true)
}

function onMouseUp(e: MouseEvent) {
  if (!transport.isConnected.value) return
  const video = e.target as HTMLVideoElement
  const rect = video.getBoundingClientRect()
  const x = Math.round((e.clientX - rect.left) * 1920 / rect.width)
  const y = Math.round((e.clientY - rect.top) * 1080 / rect.height)
  jsonCommands.sendMouseButton(x, y, e.button === 0 ? 'left' : e.button === 1 ? 'middle' : 'right', false)
  lastButtonState = 0
}

// Keyboard event handlers
function onKeyDown(e: KeyboardEvent) {
  console.log('[App] Key down:', e.code, 'connected:', transport.isConnected.value)
  if (!transport.isConnected.value) return
  e.preventDefault()
  const keysym = keyCodeToKeysym(e.code, e.key)
  console.log('[App] Keysym:', keysym)
  if (keysym) {
    jsonCommands.sendKeyDown(keysym)
  }
}

function onKeyUp(e: KeyboardEvent) {
  if (!transport.isConnected.value) return
  e.preventDefault()
  const keysym = keyCodeToKeysym(e.code, e.key)
  if (keysym) {
    jsonCommands.sendKeyUp(keysym)
  }
}

// Simple keyCode to RFB keysym mapping
function keyCodeToKeysym(code: string, key: string): number | null {
  // Letter keys
  if (code.startsWith('Key')) {
    return code.charCodeAt(3) | 0x20 // Convert to lowercase ASCII
  }
  // Number keys
  if (code.startsWith('Digit')) {
    return code.charCodeAt(5)
  }
  // Special keys
  const specialKeys: Record<string, number> = {
    'Enter': 0xff0d,
    'Escape': 0xff1b,
    'Tab': 0xff09,
    'Backspace': 0xff08,
    'Space': 0x20,
    'Delete': 0xffff,
    'ArrowUp': 0xff52,
    'ArrowDown': 0xff54,
    'ArrowLeft': 0xff51,
    'ArrowRight': 0xff53,
    'ShiftLeft': 0xffe1,
    'ShiftRight': 0xffe2,
    'ControlLeft': 0xffe3,
    'ControlRight': 0xffe4,
    'AltLeft': 0xffe9,
    'AltRight': 0xffea,
  }
  return specialKeys[code] || null
}

onMounted(async () => {
  try {
    await loadWasm()
    console.log('[App] WASM loaded')
  } catch (err) {
    console.error('[App] WASM load failed:', err)
  }
  // Add keyboard listeners to document (higher priority)
  document.addEventListener('keydown', onKeyDown)
  document.addEventListener('keyup', onKeyUp)

  // Also add to the root element with tabindex for focus
  const rootEl = document.querySelector('#app')
  if (rootEl) {
    rootEl.setAttribute('tabindex', '0')
    rootEl.focus()
    console.log('[App] Root element focused')
  }
  // Focus the window
  window.focus()
  console.log('[App] Keyboard listeners registered')
})

onUnmounted(() => {
  disconnect()
  document.removeEventListener('keydown', onKeyDown)
  document.removeEventListener('keyup', onKeyUp)
})
</script>

<template>
  <div class="flex flex-col h-screen w-screen overflow-hidden bg-slate-950 text-white">
    <!-- Header -->
    <div class="flex items-center justify-between px-4 py-2 bg-slate-900 border-b border-slate-800">
      <h1 class="text-lg font-bold">Openterface WebRTC Client</h1>
      <div class="flex items-center gap-3">
        <span class="text-sm" :class="transport.isConnected.value ? 'text-green-400' : 'text-slate-400'">
          {{ transport.state.value }}
        </span>
        <button
          v-if="!transport.isConnected.value"
          @click="connect"
          :disabled="isConnecting"
          class="px-4 py-1.5 bg-blue-600 hover:bg-blue-500 disabled:bg-slate-700 rounded text-sm font-medium transition-colors"
        >
          {{ isConnecting ? 'Connecting...' : 'Connect' }}
        </button>
        <button
          v-else
          @click="disconnect"
          class="px-4 py-1.5 bg-red-600 hover:bg-red-500 rounded text-sm font-medium transition-colors"
        >
          Disconnect
        </button>
      </div>
    </div>

    <!-- Error -->
    <div v-if="error" class="px-4 py-2 bg-red-900/50 text-red-200 text-sm">
      {{ error }}
    </div>

    <!-- Hidden input for keyboard capture -->
    <input
      ref="inputRef"
      type="text"
      class="absolute top-0 left-0 w-0 h-0 opacity-0"
      autocomplete="off"
      @blur="onInputBlur"
    />

    <!-- Video -->
    <div class="flex-1 flex items-center justify-center bg-black overflow-hidden" @click="focusInput">
      <video
        ref="videoRef"
        class="w-full h-full object-contain"
        autoplay
        playsinline
        muted
        @mousemove="onMouseMove"
        @mousedown="onMouseDown"
        @mouseup="onMouseUp"
        @contextmenu.prevent
      />
    </div>

    <!-- Instructions -->
    <div class="px-4 py-3 bg-slate-900 border-t border-slate-800 text-sm text-slate-400">
      <p>Connect to start viewing. Keyboard and mouse input will be sent via the WebRTC data channel.</p>
    </div>
  </div>
</template>
