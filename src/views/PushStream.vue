<template>
  <section class="view-card">
    <div class="topbar" style="margin-bottom:6px; display:flex; gap:8px; align-items:center;">
      <button class="button" @click="goBack">← 返回</button>
    </div>
    <h2>推流 (Push) - WHIP</h2>

    <div class="field-group">
      <label class="label">來源</label>
      <div class="row" role="group" aria-label="source selection">
        <button class="button" :class="{ 'active': mode==='camera' }" @click="mode='camera'">攝像頭</button>
        <button class="button" :class="{ 'active': mode==='screen' }" @click="mode='screen'" :disabled="!showScreenOption">螢幕分享</button>
      </div>
      <div v-if="mode==='camera'" style="margin-top:8px;">
        <label class="label">攝像頭清單</label>
        <select class="field" v-model="selectedDeviceId" @change="startPreview">
          <option value="">選擇攝像頭裝置</option>
          <option v-for="d in devices" :key="d.deviceId" :value="d.deviceId">{{ d.label || ('Device ' + d.index) }}</option>
        </select>
      </div>
      <div v-else-if="mode==='screen'" style="margin-top:8px;">
        <div class="muted">注意：螢幕分享在行動裝置通常不可用，僅在桌面瀏覽器可用。</div>
      </div>
    </div>

    <div class="view-card" style="margin-top:12px;">
      <label class="label">WHIP 伺服端 URL</label>
      <input class="field" v-model="whipUrl" placeholder="輸入推流網址 (WHIP URL)" />
      <div class="status-row" style="margin-top:6px; display:flex; align-items:center; gap:8px; flex-wrap:wrap;">
        <span class="status" :class="statusClass">{{ statusText }}</span>
        <span v-if="error" style="color: #f87171; font-size:12px;">{{ error }}</span>
      </div>
    </div>

    <div class="row" style="margin-top:12px; flex-wrap: wrap; gap: 8px;">
      <button class="button" :disabled="!canStart" @click="startPush">開始推流</button>
      <button class="button" v-if="isStreaming" @click="stopPush">停止推流</button>
    </div>

    <div class="view-card" style="margin-top:12px;">
      <label class="label">本地預覽</label>
      <video ref="previewVideo" class="video" playsinline autoplay muted style="background:#000; max-width:100%;" @error="onVideoError"></video>
    </div>
  </section>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, computed, watch } from 'vue'
import { WHIPClient } from '@eyevinn/whip-web-client'

const emit = defineEmits<{
  (e: 'goBack'): void
}>()
function goBack() { emit('goBack') }

const whipUrl = ref<string>('')
const mode = ref<'camera'|'screen'>('camera')
const devices = ref<Array<{ deviceId: string; label?: string; index?: number }>>([])
const selectedDeviceId = ref<string>('')
const previewVideo = ref<HTMLVideoElement | null>(null)
const client = ref<WHIPClient | null>(null)
const stream = ref<MediaStream | null>(null)
const status = ref<'idle'|'connecting'|'streaming'|'stopped'|'failed'>('idle')
const error = ref<string>('')
const showScreenOption = ref<boolean>(false)

const isStreaming = computed(() => status.value === 'streaming')
const canStart = computed(() => !!whipUrl.value && !!stream.value && mode.value)
const statusText = computed(() => {
  switch (status.value) {
    case 'connecting': return '連線中...'
    case 'streaming': return '推流中'
    case 'stopped': return '已停止'
    case 'failed': return '連線失敗'
    default: return '未開始'
  }
})
const statusClass = computed(() => {
  if (status.value === 'streaming') return 'green'
  if (status.value === 'connecting') return 'yellow'
  if (status.value === 'failed') return 'red'
  return ''
})

async function ensureDisplayOption() {
  showScreenOption.value = typeof navigator.mediaDevices?.getDisplayMedia === 'function'
}

async function enumerateCameraDevices() {
  devices.value = []
  try {
    const infos = await navigator.mediaDevices.enumerateDevices()
    let idx = 0
    for (const d of infos) {
      if (d.kind === 'videoinput') {
        devices.value.push({ deviceId: d.deviceId, label: d.label, index: idx++ })
      }
    }
  } catch (e: unknown) {
    console.warn('enumerateDevices failed:', e instanceof Error ? e.message : e)
  }
}

async function startPreview() {
  if (stream.value) {
    stream.value.getTracks().forEach(t => t.stop())
    stream.value = null
  }
  try {
    if (mode.value === 'camera') {
      const constraints: MediaStreamConstraints = {
        video: selectedDeviceId.value ? { deviceId: { exact: selectedDeviceId.value } } : true,
        audio: true
      }
      stream.value = await navigator.mediaDevices.getUserMedia(constraints)
    } else {
      stream.value = await navigator.mediaDevices.getDisplayMedia({ video: true, audio: true })
    }
    if (previewVideo.value && stream.value) {
      previewVideo.value.srcObject = stream.value
      await previewVideo.value.play()
    }
    status.value = 'idle'
  } catch (e: unknown) {
    const msg = e instanceof Error ? e.message : String(e)
    console.error('startPreview failed:', msg)
    error.value = '無法取得媒體來源'
    status.value = 'failed'
  }
}

async function startPush() {
  if (!stream.value) {
    error.value = '請先選擇媒體來源並啟動預覽'
    status.value = 'failed'
    return
  }
  if (!whipUrl.value) {
    error.value = '請輸入 WHIP URL'
    status.value = 'failed'
    return
  }
  error.value = ''
  status.value = 'connecting'
  try {
    client.value = new WHIPClient({
      endpoint: whipUrl.value,
      opts: {
        debug: true,
        iceServers: [{ urls: 'stun:stun.l.google.com:19302' }],
      }
    })
    await client.value.ingest(stream.value!)
    status.value = 'streaming'
  } catch (e: unknown) {
    console.error('WHIP ingest failed:', e instanceof Error ? e.message : e)
    status.value = 'failed'
    error.value = '推流失敗，請檢查 URL 與來源'
  }
}

async function stopPush() {
  if (client.value) {
    try { await client.value.destroy() } catch (e: unknown) { console.warn('WHIP destroy:', e instanceof Error ? e.message : e) }
    client.value = null
  }
  if (stream.value) {
    stream.value.getTracks().forEach(t => t.stop())
    stream.value = null
  }
  if (previewVideo.value) {
    previewVideo.value.srcObject = null
  }
  status.value = 'stopped'
}

function onVideoError() { if (status.value === 'connecting') error.value = '本地預覽影片錯誤' }

onMounted(async () => {
  await ensureDisplayOption()
  await enumerateCameraDevices()
  if (mode.value === 'camera' && devices.value.length > 0) {
    selectedDeviceId.value = devices.value[0].deviceId
    await startPreview()
  }
})

watch(mode, async (newMode) => {
  if (newMode === 'camera') {
    await enumerateCameraDevices()
    if (devices.value.length > 0) {
      selectedDeviceId.value = devices.value[0].deviceId
    }
  }
  await startPreview()
})

onBeforeUnmount(() => {
  if (stream.value) {
    stream.value.getTracks().forEach(t => t.stop())
  }
  if (client.value) {
    client.value.destroy()
  }
})

</script>

<style scoped>
.view-card { padding: 12px; border-radius: 8px; background: rgba(20,24,40,0.5); border: 1px solid rgba(255,255,255,0.12); }
.muted { color: var(--muted); font-size: 12px; }
.active { outline: 2px solid rgba(108,92,231,0.5); }
</style>
