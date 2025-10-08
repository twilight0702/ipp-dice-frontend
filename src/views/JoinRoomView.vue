<template>
  <div class="grid justify-content-center p-3 surface-ground min-h-screen">
    <div class="col-12 md:col-6">
      <Card>
        <template #title>
          <span class="text-3xl font-bold pi pi-box"> 骰子游戏</span>
        </template>
        <template #subtitle>
          <div class="mb-4">输入房间号加入游戏</div>
        </template>
        <template #content>
          <form @submit.prevent="joinRoom" class="flex flex-column gap-5">
            <div class="flex flex-column gap-1">
              <FloatLabel variant="on">
                <InputText
                  id="roomId"
                  v-model="roomId"
                  class="w-full"
                  autocomplete="off"
                  required
                />
                <label for="roomId">房间号</label>
              </FloatLabel>
              <small class="text-600">房间号由数字组成</small>
            </div>

            <Button
              type="submit"
              :label="isLoading ? '正在进入...' : '进入房间'"
              :icon="isLoading ? 'pi pi-spinner pi-spin' : 'pi pi-sign-in'"
              class="w-full"
              :disabled="!roomId || isLoading"
            />

            <Message v-if="errorMessage" severity="error" class="w-full">
              {{ errorMessage }}
            </Message>
          </form>
        </template>
      </Card>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import Card from 'primevue/card'
import InputText from 'primevue/inputtext'
import FloatLabel from 'primevue/floatlabel';
import Button from 'primevue/button'
import Message from 'primevue/message'
import 'primeicons/primeicons.css'

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
    // 调用后端接口检查房间是否存在
    const response = await fetch(`/api/room/info/${roomId.value}`, {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
      }
    });

    if (response.status === 404) {
      // 房间不存在
      errorMessage.value = '房间不存在，请检查房间号是否正确'
      isLoading.value = false
      return
    }

    if (!response.ok) {
      // 其他错误
      errorMessage.value = '获取房间信息失败，请稍后重试'
      isLoading.value = false
      return
    }

    // 解析响应数据
    const result = await response.json();
    
    // 检查房间是否开放
    if (result.data && result.data.isOpen === 0) {
      // 房间已关闭
      errorMessage.value = '房间已关闭，无法进入'
      isLoading.value = false
      return
    }

    // 房间开放，跳转到投掷页面
    await router.push({
      name: 'dice-roll',
      params: { roomId: roomId.value }
    })
  } catch (error) {
    errorMessage.value = '网络错误，请检查连接后重试'
    console.error('Join room error:', error)
  } finally {
    isLoading.value = false
  }
}
</script>

<style scoped>
</style>