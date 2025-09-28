<template>
  <Transition name="overlay" appear>
    <div class="number-animation-container" v-show="isVisible">
      <div class="number-overlay">
        <div class="dice-grid">
          <div v-for="(number, index) in displayNumbers" :key="index" class="dice-number-container">
            <div class="dice-number" :class="{ 'final': !isRolling && finalNumbers[index] }">
              {{ number }}
            </div>
          </div>
        </div>
        <div class="animation-text-container">
          <Transition name="text-fade" mode="out-in">
            <div v-if="isRolling" key="rolling" class="animation-text">
              <h3 class="rolling-text">🎲 投掷中...</h3>
              <p class="rolling-subtitle">六个骰子正在滚动</p>
            </div>
            <div v-else-if="allDiceHaveNumbers" key="completed" class="animation-text">
              <h3 class="completed-text">🎉 投掷完成</h3>
              <p class="completed-subtitle">总点数: {{ totalPoints }}</p>
            </div>
          </Transition>
        </div>
      </div>
    </div>
  </Transition>
</template>

<script setup lang="ts">
import { ref, onUnmounted, watch, computed } from 'vue'

interface Props {
  isVisible: boolean
  diceNumbers?: number[]
  minDuration?: number
}

const props = withDefaults(defineProps<Props>(), {
  minDuration: 3000,
  diceNumbers: () => []
})

const emit = defineEmits<{
  animationComplete: []
}>()

const isRolling = ref(false)
const animationStartTime = ref(0)
const displayNumbers = ref<number[]>([1, 2, 3, 4, 5, 6])
const finalNumbers = ref<number[]>([])
let animationInterval: number | null = null

const allDiceHaveNumbers = computed(() => props.diceNumbers.length === 6)
const totalPoints = computed(() => props.diceNumbers.reduce((sum, num) => sum + num, 0))

// 监听可见性变化
watch(() => props.isVisible, (newVisible) => {
  if (newVisible) {
    startAnimation()
  }
})

// 监听骰子点数变化
watch(() => props.diceNumbers, (newNumbers) => {
  console.log('动画组件接收到的骰子数组:', newNumbers);
  if (newNumbers.length === 6 && isRolling.value) {
    const elapsed = Date.now() - animationStartTime.value
    const remaining = Math.max(0, props.minDuration - elapsed)
    
    // 延迟设置最终点数，确保动画至少运行最小持续时间
    setTimeout(() => {
      console.log('动画即将停止，最终骰子数组:', newNumbers);
      stopAnimation(newNumbers)
    }, remaining)
  }
}, { deep: true })

// 清理资源
onUnmounted(() => {
  if (animationInterval) {
    clearInterval(animationInterval)
  }
})

// 开始动画
const startAnimation = () => {
  isRolling.value = true
  animationStartTime.value = Date.now()
  finalNumbers.value = []
  
  // 初始化显示数字
  displayNumbers.value = Array.from({length: 6}, () => Math.floor(Math.random() * 6) + 1)
  
  // 设置动画间隔，随机更新数字
  if (animationInterval) {
    clearInterval(animationInterval)
  }
  
  animationInterval = window.setInterval(() => {
    displayNumbers.value = displayNumbers.value.map(() => Math.floor(Math.random() * 6) + 1)
  }, 100) // 每100毫秒更新一次数字
}

// 停止动画并显示最终结果
const stopAnimation = (finalDiceNumbers: number[]) => {
  console.log('stopAnimation 接收到的最终骰子数组:', finalDiceNumbers);
  
  // 确保 finalDiceNumbers 是有效的数组
  if (!Array.isArray(finalDiceNumbers) || finalDiceNumbers.length !== 6) {
    console.error('无效的骰子数组:', finalDiceNumbers);
    return;
  }
  
  // 保存最终数字数组的副本
  finalNumbers.value = [...finalDiceNumbers];
  
  // 逐个停止骰子
  const stopInterval = 200; // 每个骰子停止的间隔时间
  
  // 停止随机更新
  if (animationInterval) {
    clearInterval(animationInterval);
    animationInterval = null;
  }
  
  // 逐个设置最终数字
  finalDiceNumbers.forEach((num, index) => {
    setTimeout(() => {
      console.log(`设置第 ${index} 个骰子为:`, num);
      
      // 更新显示的数字为最终结果
      const newDisplayNumbers = [...displayNumbers.value];
      newDisplayNumbers[index] = num;
      displayNumbers.value = newDisplayNumbers;
      
      // 当最后一个骰子停止时，完成动画
      if (index === finalDiceNumbers.length - 1) {
        setTimeout(() => {
          console.log('所有骰子已停止，最终显示:', displayNumbers.value);
          isRolling.value = false;
          
          // 延迟发出动画完成事件，让用户有时间查看结果
          setTimeout(() => {
            emit('animationComplete');
          }, 4000); // 保持结果显示4秒
        }, 500); // 短暂延迟确保最终状态渲染完成
      }
    }, index * stopInterval); // 依次停止每个骰子
  });
}
</script>

<style scoped>
.number-animation-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 1000;
}

.number-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(12px);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.dice-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: repeat(2, 1fr);
  gap: 20px;
  margin-bottom: 60px;
}

.dice-number-container {
  width: 80px;
  height: 80px;
  display: flex;
  justify-content: center;
  align-items: center;
  perspective: 600px;
}

.dice-number {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: white;
  border-radius: 12px;
  font-size: 40px;
  font-weight: bold;
  color: #dc3545;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  transform-style: preserve-3d;
  animation: pulse 0.5s infinite alternate;
}

.dice-number.final {
  animation: none;
  transform: scale(1.1);
  box-shadow: 0 12px 24px rgba(220, 53, 69, 0.4);
  border: 2px solid #dc3545;
}

@keyframes pulse {
  from {
    transform: scale(1) rotate(0deg);
  }
  to {
    transform: scale(1.05) rotate(2deg);
  }
}

.animation-text-container {
  position: absolute;
  bottom: 10%;
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
  color: white;
}

.rolling-text, .completed-text {
  font-size: 3rem;
  margin-bottom: 0.8rem;
  color: #4ade80;
  font-weight: bold;
  background: linear-gradient(45deg, #4ade80, #22c55e);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.rolling-subtitle, .completed-subtitle {
  font-size: 1.4rem;
  opacity: 0.9;
  color: #e5e7eb;
  font-weight: 500;
}

.text-fade-enter-active,
.text-fade-leave-active {
  transition: all 0.4s ease-in-out;
}

.text-fade-enter-from {
  opacity: 0;
  transform: translateY(30px);
}

.text-fade-leave-to {
  opacity: 0;
  transform: translateY(-30px);
}

.overlay-enter-active,
.overlay-leave-active {
  transition: opacity 0.5s ease;
}

.overlay-enter-from,
.overlay-leave-to {
  opacity: 0;
}
</style>