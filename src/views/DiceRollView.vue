<template>
  <div class="px-2 md:px-5 surface-ground min-h-screen">
    <div class="grid px-2 md:px-5 py-3 md:py-5 justify-content-center">
      <div class="col-12 sm:col-10 md:col-8 lg:col-6 xl:col-5">
        <div class="flex align-items-center mb-3">
          <Button label="返回" icon="pi pi-arrow-left" text severity="secondary" @click="goBack" />
        </div>

        <Card class="w-full">
          <template #title>
            <span class="text-3xl font-bold pi pi-box"> 投掷骰子</span>
          </template>
        <template #subtitle>
          <div class="mb-4">房间号: {{ roomId }}</div>
        </template>
          <template #content>
            <form @submit.prevent="rollDice" class="grid gap-3">
              <div class="col-12">
                <FloatLabel variant="on" class="w-full">
                  <InputText
                    id="studentId"
                    v-model="studentId"
                    class="w-full"
                    autocomplete="off"
                    required
                  />
                  <label for="studentId">学号</label>
                </FloatLabel>
              </div>
              <div class="col-12">
                <FloatLabel variant="on" class="w-full">
                  <InputText
                    id="studentName"
                    v-model="studentName"
                    class="w-full"
                    autocomplete="off"
                    required
                  />
                  <label for="studentName">姓名</label>
                </FloatLabel>
              </div>
              <div class="col-12">
                <Button type="submit" :label="isRolling ? '投掷中...' : '投掷骰子'" :icon="isRolling ? 'pi pi-spinner pi-spin' : 'pi pi-play'" class="w-full" :disabled="!studentId || !studentName || isRolling" />
              </div>

              <Message v-if="errorMessage" severity="error" class="col-12">{{ errorMessage }}</Message>
              <Message v-if="successMessage" severity="success" class="col-12">{{ successMessage }}</Message>
            </form>

            <Divider />

            <div v-if="rollResult" class="grid">
              <div class="col-12">
                <Fieldset class="w-full">
                  <template #legend>
                    <div class="flex align-items-center pl-2">
                      <i class="pi pi-check-circle mr-2 text-primary"></i>
                      <span class="font-bold p-2 text-xl">投掷结果</span>
                    </div>
                  </template>
                  <div class="grid">
                    <div class="col-12 md:col-6">
                      <div class="flex justify-content-between"><span class="text-600">学号</span><span class="font-medium">{{ rollResult.cardnum }}</span></div>
                      <div class="flex justify-content-between"><span class="text-600">姓名</span><span class="font-medium">{{ rollResult.name }}</span></div>
                      <div class="flex justify-content-between"><span class="text-600">房间号</span><span class="font-medium">{{ rollResult.roomId }}</span></div>
                      <div class="flex justify-content-between"><span class="text-600">玩家ID</span><span class="font-medium">{{ rollResult.playerId }}</span></div>
                    </div>
                    <div class="col-12 md:col-6">
                      <div class="flex justify-content-between"><span class="text-600">轮次</span><span class="font-medium">第 {{ rollResult.round }} 轮</span></div>
                      <div class="flex justify-content-between"><span class="text-600">骰子点数</span><span class="font-medium">{{ rollResult.dice }}</span></div>
                      <div class="flex justify-content-between"><span class="text-600">结果</span><span class="font-medium">{{ rollResult.outcome }}</span></div>
                      <div class="flex justify-content-between"><span class="text-600">得分</span><span class="font-bold text-primary">{{ rollResult.score }} 分</span></div>
                    </div>
                  </div>
                </Fieldset>
              </div>
            </div>

            <div v-if="rollHistory.length > 0" class="grid mt-3">
              <div class="col-12">
                <Fieldset class="w-full">
                  <template #legend>
                    <div class="flex align-items-center pl-2">
                      <i class="pi pi-history mr-2 text-primary"></i>
                      <span class="font-bold p-2 text-xl">投掷历史</span>
                    </div>
                  </template>
                  <div class="flex flex-column gap-2">
                    <Panel v-for="(record, index) in rollHistory.slice()" :key="index" class="mb-2" :toggleable="false">
                      <template #header>
                        <div class="flex justify-content-between align-items-center w-full">
                          <span class="font-bold">第 {{ record.round }} 轮</span>
                          <Tag :value="formatOutcome(record.diceOutcome)" severity="success" class="font-bold" />
                        </div>
                      </template>
                      <div class="flex gap-3 text-600">
                        <span>骰子: {{ record.dice }}</span>
                        <span>得分: {{ record.score }} 分</span>
                      </div>
                    </Panel>
                  </div>
                </Fieldset>
              </div>
            </div>
          </template>
        </Card>
      </div>
    </div>

    <!-- 骰子动画覆盖层 -->
    <NumberRollAnimation
      :isVisible="showDiceAnimation"
      :diceNumbers="animationDiceNumbers"
      :minDuration="animationMinDuration"
      :resultHoldDuration="animationResultHoldDuration"
      @animationComplete="onAnimationComplete"
    />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import Card from 'primevue/card'
import InputText from 'primevue/inputtext'
import Button from 'primevue/button'
import Message from 'primevue/message'
import Divider from 'primevue/divider'
import Tag from 'primevue/tag'
import FloatLabel from 'primevue/floatlabel'
import Fieldset from 'primevue/fieldset'
import Panel from 'primevue/panel'
import NumberRollAnimation from '../components/NumberRollAnimation.vue'

// 接口返回数据类型
interface RollData {
  playerId: number
  round: number
  dice: string
  outcome: string
  score: number
  cardnum: string
  name: string
  roomId: string
}

// 历史记录数据类型
interface PlayerRecord {
  playerId: number
  cardnum: string
  name: string
  round: number
  dice: string
  diceOutcome: string
  score: number
  rollTime: string
}

const router = useRouter()
const route = useRoute()

const roomId = ref('')
const studentId = ref('')
const studentName = ref('')
const isRolling = ref(false)
const errorMessage = ref('')
const successMessage = ref('')
const rollResult = ref<RollData | null>(null)

// 历史记录数组
const rollHistory = ref<PlayerRecord[]>([])

// 动画相关状态
const showDiceAnimation = ref(false)
const animationDiceNumbers = ref<number[]>([]) // 六个骰子的点数数组
const animationMinDuration = ref(4000) // 最小动画时长4秒，可配置
const animationResultHoldDuration = ref(3000) // 结果停留时长3秒，可配置

// 获取房间ID
onMounted(async () => {
  // 同时检查路由参数和查询参数
  const urlRoomId = route.params.roomId || route.query.roomId
  if (urlRoomId) {
    roomId.value = String(urlRoomId)
    // 页面加载时尝试获取历史记录
    // 注意：由于此时还没有playerId，所以暂时无法获取历史记录
    // 可以考虑在用户第一次投掷后保存playerId到localStorage，下次访问时使用
  } else {
    // 如果没有房间ID，返回加入房间页面
    router.push({ name: 'join-room' })
  }
})

const goBack = () => {
  router.push({ name: 'join-room' })
}

// 动画完成处理
const onAnimationComplete = () => {
  // 动画完成后，组件内部已延迟触发事件，这里只负责状态复位
  showDiceAnimation.value = false
  isRolling.value = false
}

const rollDice = async () => {
  if (!studentId.value || !studentName.value) {
    errorMessage.value = '请填写完整信息'
    return
  }

  isRolling.value = true
  errorMessage.value = ''
  successMessage.value = ''

  // 开始动画
  showDiceAnimation.value = true
  animationDiceNumbers.value = []

  try {
    const requestData = {
      roomId: roomId.value, // 直接使用字符串类型的 roomId
      cardnum: studentId.value,
      name: studentName.value
    }

    const response = await fetch('/api/player/roll', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(requestData)
    })

    const result = await response.json()

    if (result.code === 200) {
      // 解析后端返回的 dice 字符串
      try {
        const diceArray = JSON.parse(result.data.dice);
        if (Array.isArray(diceArray) && diceArray.length === 6) {
          // 确保使用后端返回的6个骰子数组，并转换为数字类型
          const parsedDiceNumbers = diceArray.map((n: unknown) => Number(n));
          console.log('后端返回的骰子数组:', parsedDiceNumbers);
          animationDiceNumbers.value = parsedDiceNumbers;
        } else {
          // 如果解析失败或不是数组，则使用备用方案
          throw new Error('Dice data is not a valid array.');
        }
      } catch (e) {
        console.error('Failed to parse dice data, using fallback:', e);
        // 临时方案：生成六个随机点数
        animationDiceNumbers.value = Array.from({length: 6}, () => Math.floor(Math.random() * 6) + 1);
      }

      // 创建完整的结果对象
      const completeResult = {
        roomId: roomId.value,
        cardnum: studentId.value,
        name: studentName.value,
        ...result.data
      }

      // 当前结果展示
      rollResult.value = completeResult

      successMessage.value = '投掷成功！'

      // 使用真实的playerId获取最新的历史记录
      await fetchHistoryRecords(result.data.playerId)
    } else {
      errorMessage.value = result.message || '投掷失败，请重试'
      // 如果失败，关闭动画
      showDiceAnimation.value = false
      isRolling.value = false
    }
  } catch (error) {
    errorMessage.value = '网络错误，请检查连接后重试'
    console.error('Roll dice error:', error)
    // 如果出错，关闭动画
    showDiceAnimation.value = false
    isRolling.value = false
  }
  // 注意：不在这里设置 isRolling.value = false，就是在动画完成后设置
}

// 获取历史记录
const fetchHistoryRecords = async (playerId: number) => {
  try {
    const response = await fetch(`/api/player/history-records?roomId=${roomId.value}&playerId=${playerId}`, {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
      }
    });

    if (!response.ok) {
      console.error('获取历史记录失败:', response.status);
      return;
    }

    const result = await response.json();

    if (result.code === 200 && result.data && result.data.historyRecords) {
      // 更新历史记录
      rollHistory.value = result.data.historyRecords;
    }
  } catch (error) {
    console.error('获取历史记录错误:', error);
  }
}

// 格式化骰子结果
const formatOutcome = (outcome: string) => {
  const outcomeMap: Record<string, string> = {
    'LIU_PAO_HONG': '六豹红',
    'ZHUANG_YUAN_CJH': '状元插金花',
    'WU_HUANG': '五王',
    'WU_ZI': '五子',
    'ZHUANG_YUAN': '状元',
    'DUI_TANG': '对堂',
    'SAN_HONG': '三红',
    'SI_JIN': '四进',
    'ER_JU': '二举',
    'YI_XIU': '一秀',
    'PU_TONG': '普通'
  };

  return outcomeMap[outcome] || outcome;
}

</script>

<style scoped>
</style>
