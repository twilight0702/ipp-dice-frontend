<template>
  <div class="page-container">
    <div class="content">
      <div class="page-header">
        <h1 class="page-title">🎲 骰子游戏</h1>
        <p class="page-description">输入房间号加入游戏</p>
      </div>

      <div class="form-container">
        <div class="form-card">
          <form @submit.prevent="joinRoom">
            <div class="form-group">
              <label class="form-label">
                <span class="label-icon">🏠</span>
                房间号
              </label>
              <input
                v-model="roomId"
                type="text"
                class="form-input"
                placeholder="请输入房间号"
                required
              />
              <div class="form-hint">
                房间号由数字组成
              </div>
            </div>

            <button
              type="submit"
              class="btn btn-primary btn-full-width"
              :disabled="!roomId || isLoading"
            >
              <span v-if="isLoading" class="loading-spinner"></span>
              <span class="btn-icon" v-else>🚪</span>
              {{ isLoading ? '正在进入...' : '进入房间' }}
            </button>
          </form>

          <div v-if="errorMessage" class="message error-message">
            {{ errorMessage }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route = useRoute()

const roomId = ref('')
const isLoading = ref(false)
const errorMessage = ref('')

// 检查URL参数中是否有roomId
onMounted(() => {
  const urlRoomId = route.params.roomId || route.query.roomId
  if (urlRoomId) {
    roomId.value = String(urlRoomId)
  }
})

const joinRoom = async () => {
  if (!roomId.value) {
    errorMessage.value = '请输入房间号'
    return
  }

  // 验证房间号是否为数字
  if (!/^\d+$/.test(roomId.value)) {
    errorMessage.value = '房间号必须是数字'
    return
  }

  isLoading.value = true
  errorMessage.value = ''

  try {
    // 这里可以添加验证房间是否存在的API调用
    // 暂时直接跳转到投掷页面
    await router.push({
      name: 'dice-roll',
      params: { roomId: roomId.value }
    })
  } catch (error) {
    errorMessage.value = '进入房间失败，请重试'
    console.error('Join room error:', error)
  } finally {
    isLoading.value = false
  }
}
</script>

<style scoped>
/* 使用公共样式，无需额外样式 */
</style>