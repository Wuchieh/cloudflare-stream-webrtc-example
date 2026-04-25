<template>
  <section class="view-card">
    <div class="topbar" style="margin-bottom:6px; display:flex; gap:8px; align-items:center;">
      <button class="button" @click="goBack">← 返回</button>
    </div>
    <h2>拉流 (Pull) - WHEP</h2>
    <div class="field-group" style="margin-bottom:8px;">
      <label class="label">WHEP URL</label>
      <div class="row" style="gap:8px; align-items: center; flex-wrap:wrap;">
        <input class="field" style="flex:1;min-width:0;" v-model="webrtcUrl" placeholder="輸入拉流網址 (WHEP URL)" />
        <button class="button" @click="startPlay" :disabled="!webrtcUrl">播放</button>
        <button class="button" @click="stopPlay" v-if="isPlaying">停止</button>
      </div>
    </div>

    <div class="view-card" style="margin-top:8px;">
      <video
        ref="videoEl"
        class="video"
        playsinline
        autoplay
        muted
        width="100%"
        @playing="onVideoPlaying"
        @waiting="onVideoWaiting"
        @error="onVideoError"
      ></video>
      <div class="status-row" style="margin-top:6px; display:flex; gap:8px; align-items:center; flex-wrap:wrap;">
        <span class="status" :class="statusClass">{{ statusText }}</span>
        <span v-if="error" style="color:#f87171; font-size:12px;">{{ error }}</span>
      </div>
    </div>
  </section>
</template>

<script setup lang="ts">
import { ref, onUnmounted, computed } from 'vue'
import { WebRTCPlayer } from '@eyevinn/webrtc-player'

const emit = defineEmits<{
  (e: 'goBack'): void
}>()
function goBack() { emit('goBack') }

const videoEl = ref<HTMLVideoElement | null>(null)
const webrtcUrl = ref<string>('')
const player = ref<WebRTCPlayer | null>(null)
const status = ref<'idle' | 'connecting' | 'playing' | 'stopped' | 'failed'>('idle')
const error = ref<string>('')

const isPlaying = computed(() => status.value === 'playing' || status.value === 'connecting')
const statusText = computed(() => {
  switch (status.value) {
    case 'connecting': return '連線中...'
    case 'playing': return '播放中'
    case 'stopped': return '已停止'
    case 'failed': return '連線失敗'
    default: return '等待播放'
  }
})
const statusClass = computed(() => {
  if (status.value === 'playing') return 'green'
  if (status.value === 'connecting') return 'yellow'
  if (status.value === 'failed') return 'red'
  return ''
})

function onVideoPlaying() {
  status.value = 'playing'
  error.value = ''
}

function onVideoWaiting() {
  if (status.value === 'playing') {
    status.value = 'connecting'
  }
}

function onVideoError() {
  if (status.value === 'connecting') {
    status.value = 'failed'
    error.value = '影片載入失敗'
  }
}

async function startPlay() {
  error.value = ''
  if (!videoEl.value || !webrtcUrl.value) return
  if (player.value) {
    await stopPlay()
  }
  try {
    status.value = 'connecting'
    player.value = new WebRTCPlayer({
      video: videoEl.value,
      type: 'whep',
      iceServers: [{ urls: 'stun:stun.l.google.com:19302' }],
      debug: true,
    })
    await player.value.load(new URL(webrtcUrl.value))
    player.value.unmute()
    status.value = 'playing'
  } catch (e: unknown) {
    const msg = e instanceof Error ? e.message : String(e)
    console.error('startPlay failed:', msg)
    status.value = 'failed'
    error.value = '播放失敗，請檢查 URL 是否正確'
  }
}

async function stopPlay() {
  if (player.value) {
    try {
      await player.value.unload()
    } catch (e: unknown) {
      console.warn('unload error:', e instanceof Error ? e.message : e)
    }
    try {
      player.value.destroy()
    } catch (e: unknown) {
      console.warn('destroy error:', e instanceof Error ? e.message : e)
    }
    player.value = null
  }
  if (videoEl.value) {
    videoEl.value.srcObject = null
  }
  status.value = 'stopped'
  error.value = ''
}

onUnmounted(() => {
  if (player.value) {
    try { player.value.unload() } catch (e: unknown) { console.warn('cleanup unload:', e) }
    try { player.value.destroy() } catch (e: unknown) { console.warn('cleanup destroy:', e) }
    player.value = null
  }
})
</script>

<style scoped>
.video { width: 100%; max-height: 60vh; background: #000; display: block; }
</style>
