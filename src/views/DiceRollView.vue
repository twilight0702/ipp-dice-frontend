<template>
  <div class="page-container">
    <div class="content">
      <!-- 返回按钮 -->
      <button @click="goBack" class="back-button">
        <span class="back-icon">←</span>
        返回
      </button>

      <div class="page-header">
        <h1 class="page-title">🎲 投掷骰子</h1>
        <p class="page-description">房间号: {{ roomId }}</p>
      </div>

      <div class="form-container">
        <div class="form-card">
          <form @submit.prevent="rollDice">
            <div class="form-group">
              <label class="form-label">
                <span class="label-icon">🎓</span>
                学号
              </label>
              <input
                v-model="studentId"
                type="text"
                class="form-input"
                placeholder="请输入学号"
                required
              />
            </div>

            <div class="form-group">
              <label class="form-label">
                <span class="label-icon">👤</span>
                姓名
              </label>
              <input
                v-model="studentName"
                type="text"
                class="form-input"
                placeholder="请输入姓名"
                required
              />
            </div>

            <button
              type="submit"
              class="btn btn-primary btn-full-width"
              :disabled="!studentId || !studentName || isRolling"
            >
              <span v-if="isRolling" class="loading-spinner"></span>
              <span class="btn-icon" v-else>🎲</span>
              {{ isRolling ? '投掷中...' : '投掷骰子' }}
            </button>
          </form>

          <!-- 错误信息显示 -->
          <div v-if="errorMessage" class="message error-message">
            {{ errorMessage }}
          </div>

          <!-- 成功信息显示 -->
          <div v-if="successMessage" class="message success-message">
            {{ successMessage }}
          </div>

          <!-- 投掷结果显示 -->
          <div v-if="rollResult" class="info-card">
            <h3 class="card-title">🎉 投掷结果</h3>
            <div class="info-item">
              <span class="info-label">学号</span>
              <span class="info-value">{{ rollResult.cardnum }}</span>
            </div>
            <div class="info-item">
              <span class="info-label">姓名</span>
              <span class="info-value">{{ rollResult.name }}</span>
            </div>
            <div class="info-item">
              <span class="info-label">房间号</span>
              <span class="info-value">{{ rollResult.roomId }}</span>
            </div>
            <div class="info-item">
              <span class="info-label">玩家ID</span>
              <span class="info-value">{{ rollResult.playerId }}</span>
            </div>
            <div class="info-item">
              <span class="info-label">轮次</span>
              <span class="info-value">第 {{ rollResult.round }} 轮</span>
            </div>
            <div class="info-item">
              <span class="info-label">骰子点数</span>
              <span class="info-value">{{ rollResult.dice }}</span>
            </div>
            <div class="info-item">
              <span class="info-label">结果</span>
              <span class="info-value">{{ rollResult.outcome }}</span>
            </div>
            <div class="info-item">
              <span class="info-label">得分</span>
              <span class="info-value">{{ rollResult.score }} 分</span>
            </div>
          </div>

          <!-- 历史记录显示 -->
          <div v-if="rollHistory.length > 0" class="info-card">
            <h3 class="card-title">📊 投掷历史</h3>
            <div v-for="(record, index) in rollHistory.slice()" :key="index" class="history-item">
              <div class="history-header">
                <span class="history-round">第 {{ record.round }} 轮</span>
                <span class="history-score">{{ record.score }} 分</span>
              </div>
              <div class="history-details">
                <span>骰子: {{ record.dice }}</span>
                <span>结果: {{ formatOutcome(record.diceOutcome) }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 骰子动画组件 -->
    <NumberRollAnimation
      :is-visible="showDiceAnimation"
      :dice-numbers="animationDiceNumbers"
      :min-duration="animationMinDuration"
      @animation-complete="onAnimationComplete"
    />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import NumberRollAnimation from '@/components/NumberRollAnimation.vue'

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
const animationMinDuration = ref(5000) // 最小动画时长5秒，可配置

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
        if (Array.isArray(diceArray) && diceArray.length > 0) {
          // 确保使用后端返回的骰子数组，并转换为数字类型
          const parsedDiceNumbers = diceArray.map(n => Number(n));
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
/* 投掷结果样式 */
.info-card {
  margin-top: 24px;
}

/* 历史记录样式 */
.history-item {
  padding: var(--spacing-md);
  margin-bottom: var(--spacing-sm);
  background: var(--bg-light);
  border-radius: var(--radius-medium);
  border-left: 4px solid var(--primary-color);
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--spacing-xs);
}

.history-round {
  font-weight: 600;
  color: var(--text-primary);
}

.history-score {
  font-weight: 600;
  color: var(--primary-color);
  font-size: var(--font-lg);
}

.history-details {
  display: flex;
  gap: var(--spacing-lg);
  font-size: var(--font-xs);
  color: var(--text-secondary);
}

.history-details span {
  padding: var(--spacing-xs) var(--spacing-sm);
  background: var(--bg-white);
  border-radius: var(--radius-small);
}
</style>
