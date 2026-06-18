<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useGameStore } from '../stores/gameStore'
import gameConfig from '../config/gameConfig'
import { isGiftLiked, isGiftDisliked, calculateGiftAffinity } from '../utils/gameUtils'

const emit = defineEmits<{
  (e: 'close'): void
}>()

const gameStore = useGameStore()
const selectedGiftId = ref<string | null>(null)
const showResult = ref(false)
const giftResult = ref<{ liked: boolean; disliked: boolean; affinityChange: number; giftName: string } | null>(null)
const isVisible = ref(false)

onMounted(() => {
  requestAnimationFrame(() => {
    isVisible.value = true
  })
})

const currentCharacterConfig = computed(() => gameStore.currentCharacterConfig)

const canAfford = computed(() => {
  if (!selectedGiftId.value) return false
  const gift = gameConfig.gifts.find(g => g.id === selectedGiftId.value)
  return gift ? gameStore.resources >= gift.price : false
})

function getGiftStatus(giftId: string): string {
  if (!currentCharacterConfig.value) return ''
  if (isGiftLiked(giftId, currentCharacterConfig.value)) return '喜欢'
  if (isGiftDisliked(giftId, currentCharacterConfig.value)) return '讨厌'
  return '普通'
}

function getGiftStatusClass(giftId: string): string {
  if (!currentCharacterConfig.value) return ''
  if (isGiftLiked(giftId, currentCharacterConfig.value)) return 'status-liked'
  if (isGiftDisliked(giftId, currentCharacterConfig.value)) return 'status-disliked'
  return 'status-normal'
}

function sendGift() {
  if (!selectedGiftId.value || !gameStore.selectedCharacterId) return
  
  const gift = gameConfig.gifts.find(g => g.id === selectedGiftId.value)
  if (!gift || !currentCharacterConfig.value) return
  
  const charState = gameStore.characters.find(c => c.id === gameStore.selectedCharacterId)
  if (!charState) return
  
  const affinityChange = calculateGiftAffinity(
    selectedGiftId.value,
    currentCharacterConfig.value,
    gift.price,
    charState.mood
  )
  
  const liked = isGiftLiked(selectedGiftId.value, currentCharacterConfig.value)
  const disliked = isGiftDisliked(selectedGiftId.value, currentCharacterConfig.value)
  
  giftResult.value = {
    liked,
    disliked,
    affinityChange,
    giftName: gift.name
  }
  
  showResult.value = true
  
  gameStore.performAction('gift', gameStore.selectedCharacterId, selectedGiftId.value)
  
  setTimeout(() => {
    closeModal()
  }, 1500)
}

function closeModal() {
  isVisible.value = false
  setTimeout(() => {
    emit('close')
  }, 300)
}

function getReactionEmoji(): string {
  if (!giftResult.value) return '😊'
  if (giftResult.value.liked) return '🥰'
  if (giftResult.value.disliked) return '😕'
  return '😊'
}

function getReactionText(): string {
  if (!giftResult.value) return ''
  if (giftResult.value.liked) return 'ta非常喜欢！'
  if (giftResult.value.disliked) return 'ta好像不太喜欢...'
  return 'ta收下了礼物'
}
</script>

<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isVisible" class="modal-overlay" @click.self="showResult || closeModal()">
        <div class="modal-content gift-modal" :class="{ 'result-mode': showResult }">
          <Transition name="fade" mode="out-in">
            <div v-if="!showResult" key="select" class="gift-select">
              <div class="modal-header">
                <h2>🎁 选择礼物</h2>
                <button class="close-btn" @click="closeModal()">✕</button>
              </div>

              <div v-if="!currentCharacterConfig" class="no-character">
                请先选择一个角色
              </div>

              <div v-else class="gift-target">
                <span class="target-avatar">{{ currentCharacterConfig.avatar }}</span>
                <span class="target-name">送给 {{ currentCharacterConfig.name }}</span>
              </div>

              <div class="gift-grid">
                <div
                  v-for="gift in gameConfig.gifts"
                  :key="gift.id"
                  class="gift-item"
                  :class="{ selected: selectedGiftId === gift.id, disabled: gameStore.resources < gift.price }"
                  @click="gameStore.resources >= gift.price && (selectedGiftId = gift.id)"
                >
                  <div class="gift-icon">{{ gift.icon }}</div>
                  <div class="gift-info">
                    <span class="gift-name">{{ gift.name }}</span>
                    <span class="gift-desc">{{ gift.description }}</span>
                  </div>
                  <div class="gift-meta">
                    <span class="gift-price">💰 {{ gift.price }}</span>
                    <span class="gift-status" :class="getGiftStatusClass(gift.id)">
                      {{ getGiftStatus(gift.id) }}
                    </span>
                  </div>
                </div>
              </div>

              <div class="modal-footer">
                <span class="current-resources">
                  当前代币：{{ gameStore.resources }}
                </span>
                <div class="footer-buttons">
                  <button class="btn btn-secondary" @click="closeModal()">取消</button>
                  <button 
                    class="btn btn-primary" 
                    :disabled="!selectedGiftId || !canAfford || gameStore.actionsRemaining <= 0"
                    @click="sendGift"
                  >
                    送出
                  </button>
                </div>
              </div>
            </div>

            <div v-else key="result" class="gift-result">
              <div class="result-avatar" :class="{ liked: giftResult?.liked, disliked: giftResult?.disliked }">
                <span class="avatar-emoji">{{ currentCharacterConfig?.avatar }}</span>
                <span class="reaction-emoji">{{ getReactionEmoji() }}</span>
              </div>
              <h3 class="result-title">{{ getReactionText() }}</h3>
              <p class="result-gift-name">「{{ giftResult?.giftName }}」</p>
              <div class="result-affinity" :class="{ positive: (giftResult?.affinityChange || 0) > 0, negative: (giftResult?.affinityChange || 0) < 0 }">
                好感 {{ (giftResult?.affinityChange || 0) > 0 ? '+' : '' }}{{ giftResult?.affinityChange }}
              </div>
              <div class="result-hint">即将返回...</div>
            </div>
          </Transition>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style scoped>
.gift-modal {
  padding: 24px;
  max-width: 600px;
}

.gift-modal.result-mode {
  max-width: 400px;
  text-align: center;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.25s ease, transform 0.25s ease;
}

.fade-enter-from {
  opacity: 0;
  transform: translateY(10px);
}

.fade-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}

.gift-result {
  padding: 20px 0;
  animation: slideUp 0.4s ease-out;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.result-avatar {
  position: relative;
  width: 100px;
  height: 100px;
  margin: 0 auto 20px;
  background: var(--accent-light);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  animation: bounce 0.6s ease;
}

.result-avatar.liked {
  background: #dcfce7;
}

.result-avatar.disliked {
  background: #fee2e2;
}

[data-theme='dark'] .result-avatar.liked {
  background: #14532d;
}

[data-theme='dark'] .result-avatar.disliked {
  background: #7f1d1d;
}

@keyframes bounce {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

.avatar-emoji {
  font-size: 48px;
}

.reaction-emoji {
  position: absolute;
  bottom: -5px;
  right: -5px;
  font-size: 32px;
  animation: popIn 0.4s ease 0.3s both;
}

@keyframes popIn {
  0% {
    opacity: 0;
    transform: scale(0);
  }
  70% {
    transform: scale(1.2);
  }
  100% {
    opacity: 1;
    transform: scale(1);
  }
}

.result-title {
  font-size: 20px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 8px;
}

.result-gift-name {
  font-size: 16px;
  color: var(--text-secondary);
  margin-bottom: 20px;
}

.result-affinity {
  display: inline-block;
  padding: 8px 20px;
  border-radius: 9999px;
  font-size: 18px;
  font-weight: 600;
  margin-bottom: 20px;
  background: var(--bg-tertiary);
}

.result-affinity.positive {
  background: #dcfce7;
  color: #166534;
}

.result-affinity.negative {
  background: #fee2e2;
  color: #991b1b;
}

[data-theme='dark'] .result-affinity.positive {
  background: #14532d;
  color: #86efac;
}

[data-theme='dark'] .result-affinity.negative {
  background: #7f1d1d;
  color: #fca5a5;
}

.result-hint {
  font-size: 13px;
  color: var(--text-muted);
  animation: pulse 1.5s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 0.5; }
  50% { opacity: 1; }
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.modal-header h2 {
  font-size: 20px;
  font-weight: 600;
}

.close-btn {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: var(--bg-tertiary);
  font-size: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.close-btn:hover {
  background: var(--accent-light);
}

.no-character {
  text-align: center;
  padding: 40px;
  color: var(--text-muted);
}

.gift-target {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 12px 16px;
  background: var(--accent-light);
  border-radius: var(--radius-md);
  margin-bottom: 16px;
}

.target-avatar {
  font-size: 24px;
}

.target-name {
  font-weight: 500;
}

.gift-grid {
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-height: 400px;
  overflow-y: auto;
  padding-right: 8px;
}

.gift-item {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 14px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-md);
  cursor: pointer;
  border: 2px solid transparent;
  transition: all 0.2s;
}

.gift-item:hover:not(.disabled) {
  background: var(--accent-light);
}

.gift-item.selected {
  border-color: var(--accent-primary);
  background: var(--accent-light);
}

.gift-item.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.gift-icon {
  font-size: 36px;
  width: 50px;
  text-align: center;
  flex-shrink: 0;
}

.gift-info {
  flex: 1;
  min-width: 0;
}

.gift-name {
  display: block;
  font-weight: 600;
  font-size: 15px;
  margin-bottom: 4px;
}

.gift-desc {
  font-size: 12px;
  color: var(--text-secondary);
}

.gift-meta {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 4px;
  flex-shrink: 0;
}

.gift-price {
  font-weight: 600;
  color: var(--accent-primary);
  font-size: 14px;
}

.gift-status {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 4px;
  background: var(--bg-secondary);
  color: var(--text-muted);
}

.gift-status.status-liked {
  background: #dcfce7;
  color: #166534;
}

[data-theme='dark'] .gift-status.status-liked {
  background: #14532d;
  color: #86efac;
}

.gift-status.status-disliked {
  background: #fee2e2;
  color: #991b1b;
}

[data-theme='dark'] .gift-status.status-disliked {
  background: #7f1d1d;
  color: #fca5a5;
}

.modal-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 20px;
  padding-top: 16px;
  border-top: 1px solid var(--border-light);
}

.current-resources {
  font-size: 14px;
  color: var(--text-secondary);
}

.footer-buttons {
  display: flex;
  gap: 10px;
}
</style>
