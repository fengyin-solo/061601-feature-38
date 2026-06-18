<script setup lang="ts">
import { ref, onMounted, watch, computed } from 'vue'
import { useGameStore } from './stores/gameStore'
import { useSaveStore } from './stores/saveStore'
import TopBar from './components/TopBar.vue'
import CharacterPanel from './components/CharacterPanel.vue'
import ActionPanel from './components/ActionPanel.vue'
import LogPanel from './components/LogPanel.vue'
import EventModal from './components/EventModal.vue'
import SaveModal from './components/SaveModal.vue'
import CardCollection from './components/CardCollection.vue'
import HistoryPanel from './components/HistoryPanel.vue'
import GiftModal from './components/GiftModal.vue'

const gameStore = useGameStore()
const saveStore = useSaveStore()

const showSaveModal = ref(false)
const showCards = ref(false)
const showHistory = ref(false)
const showGiftModal = ref(false)
const showContextBanner = ref(false)

const theme = computed(() => gameStore.darkMode ? 'dark' : 'light')

watch(() => gameStore.day, () => {
  saveStore.autoSave()
})

watch(() => gameStore.lastActionContext, () => {
  saveStore.autoSave()
})

watch(theme, (newTheme) => {
  document.documentElement.setAttribute('data-theme', newTheme)
})

function dismissContextBanner() {
  showContextBanner.value = false
}

onMounted(() => {
  document.documentElement.setAttribute('data-theme', theme.value)
  
  const hasSave = saveStore.hasAutoSave()
  if (hasSave) {
    if (confirm('检测到自动存档，是否继续游戏？')) {
      const success = saveStore.loadAutoSave()
      if (success && gameStore.lastActionContext) {
        showContextBanner.value = true
        setTimeout(() => {
          showContextBanner.value = false
        }, 5000)
      }
      if (!gameStore.currentEvent) {
        gameStore.checkAndTriggerEvent()
      }
    } else {
      gameStore.resetGame()
    }
  } else {
    gameStore.resetGame()
  }
})
</script>

<template>
  <div class="game-container">
    <TopBar 
      @toggle-save="showSaveModal = true"
      @toggle-cards="showCards = true"
      @toggle-history="showHistory = true"
      @toggle-theme="gameStore.toggleDarkMode()"
      @reset="gameStore.resetGame()"
    />

    <Transition name="slide-down">
      <div v-if="showContextBanner && gameStore.lastActionContext" class="context-banner" @click="dismissContextBanner">
        <span class="context-icon">
          {{ gameStore.lastActionContext.type === 'chat' ? '💬' : gameStore.lastActionContext.type === 'gift' ? '🎁' : gameStore.lastActionContext.type === 'work' ? '💼' : gameStore.lastActionContext.type === 'event' ? '📖' : '📌' }}
        </span>
        <span class="context-text">
          上次操作：{{ gameStore.lastActionContext.summary }}
        </span>
        <span class="context-hint">点击关闭 · 继续你的旅程 →</span>
      </div>
    </Transition>
    
    <div class="main-content">
      <div class="left-column">
        <CharacterPanel />
        <ActionPanel @open-gift="showGiftModal = true" />
      </div>
      
      <div class="right-column">
        <LogPanel />
      </div>
    </div>

    <EventModal />
    <SaveModal v-if="showSaveModal" @close="showSaveModal = false" />
    <CardCollection v-if="showCards" @close="showCards = false" />
    <HistoryPanel v-if="showHistory" @close="showHistory = false" />
    <GiftModal v-if="showGiftModal" @close="showGiftModal = false" />
  </div>
</template>

<style scoped>
.game-container {
  min-height: 100vh;
  padding: 16px;
  max-width: 1400px;
  margin: 0 auto;
}

.context-banner {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 20px;
  background: linear-gradient(135deg, var(--accent-light), var(--bg-tertiary));
  border: 1px solid var(--border-color);
  border-radius: var(--radius-lg);
  margin-bottom: 12px;
  cursor: pointer;
  transition: all 0.2s;
}

.context-banner:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px var(--shadow-color);
}

.context-icon {
  font-size: 24px;
  flex-shrink: 0;
}

.context-text {
  flex: 1;
  font-size: 14px;
  font-weight: 500;
  color: var(--text-primary);
}

.context-hint {
  font-size: 12px;
  color: var(--text-muted);
  flex-shrink: 0;
}

.slide-down-enter-active {
  transition: all 0.4s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.slide-down-leave-active {
  transition: all 0.3s ease;
}

.slide-down-enter-from {
  opacity: 0;
  transform: translateY(-20px);
}

.slide-down-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}

.main-content {
  display: grid;
  grid-template-columns: 1fr 400px;
  gap: 16px;
  margin-top: 16px;
}

.left-column {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.right-column {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

@media (max-width: 900px) {
  .main-content {
    grid-template-columns: 1fr;
  }
}
</style>
