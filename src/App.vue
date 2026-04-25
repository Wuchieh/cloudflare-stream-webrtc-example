<template>
  <div class="app-container">
    <header class="app-header">
      <h1>Cloudflare Stream WebRTC</h1>
    </header>

    <main class="view-area">
      <component :is="currentView" @navigate="navigate" @goBack="goBack" />
    </main>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import PushStream from './views/PushStream.vue'
import PullStream from './views/PullStream.vue'
import Home from './views/Home.vue'

type View = 'home'|'push'|'pull'
const currentViewName = ref<View>('home')

const currentView = computed(() => {
  switch (currentViewName.value) {
    case 'home': return Home
    case 'push': return PushStream
    case 'pull': return PullStream
  }
})

function navigate(target: 'push'|'pull') {
  currentViewName.value = target
}

function goBack() {
  currentViewName.value = 'home'
}
</script>

<style scoped>
/* Basic page resets for a clean, minimal UI */
.app-container {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  background: var(--bg, #0b0f17);
  color: var(--text, #eaeaff);
}

.app-header {
  padding: 16px;
  text-align: center;
  border-bottom: 1px solid rgba(255,255,255,0.08);
}
.app-header h1 { font-family: Georgia, 'Times New Roman', serif; margin: 0; font-size: 20px; }

.view-area {
  flex: 1;
  padding: 12px;
}
</style>
