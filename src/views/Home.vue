<template>
  <section class="home">
    <div class="home-grid">
      <button class="home-card" @click="goPush" aria-label="推流">
        <div class="icon">📤</div>
        <div class="title">推流</div>
        <div class="desc">將攝像頭或螢幕畫面推送到 Cloudflare Stream</div>
      </button>

      <button class="home-card" @click="goPull" aria-label="拉流">
        <div class="icon">📥</div>
        <div class="title">拉流</div>
        <div class="desc">播放 Cloudflare Stream 的即時影像</div>
      </button>
    </div>
  </section>
  </template>

<script setup lang="ts">
const emit = defineEmits<{
  (e: 'navigate', payload: 'push'|'pull'): void
}>()

function goPush() { emit('navigate', 'push') }
function goPull() { emit('navigate', 'pull') }
</script>

<style scoped>
.home { padding: 24px; }
.home-grid { display: grid; grid-template-columns: 1fr; gap: 16px; justify-items: center; }
.home-card {
  width: min(520px, 100%);
  border-radius: 16px;
  padding: 28px;
  text-align: left;
  background: rgba(20,24,40,0.6);
  border: 1px solid rgba(255,255,255,0.14);
  color: var(--text);
  cursor: pointer;
  transition: transform 0.2s ease, background 0.2s ease;
}
.home-card:hover { transform: translateY(-2px); background: rgba(108,92,231,0.25); }
.home-card .icon { font-size: 28px; }
.home-card .title { font-weight: 700; font-size: 20px; margin-top: 6px; }
.home-card .desc { font-size: 13px; color: var(--muted); margin-top: 6px; }

@media (min-width: 600px) {
  .home-grid { grid-template-columns: 1fr 1fr; }
}
</style>
