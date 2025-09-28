<template>
  <Transition name="overlay" appear>
    <div class="dice-animation-container" v-show="isVisible">
      <div class="dice-overlay">
        <canvas ref="diceCanvas" class="dice-canvas"></canvas>
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
import { ref, onMounted, onUnmounted, watch, computed } from 'vue'
import * as THREE from 'three'

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

const diceCanvas = ref<HTMLCanvasElement | null>(null)
const isRolling = ref(false)
const animationStartTime = ref(0)
let animationFrameId: number;

let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
const diceMeshes: THREE.Mesh[] = []

const allDiceHaveNumbers = computed(() => props.diceNumbers.length === 6)
const totalPoints = computed(() => props.diceNumbers.reduce((sum, num) => sum + num, 0))

onMounted(() => {
  if (diceCanvas.value) {
    initThree()
    createDice()
    animate()
  }
  window.addEventListener('resize', onWindowResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', onWindowResize)
  cancelAnimationFrame(animationFrameId)
  // Clean up Three.js resources
  renderer.dispose()
  scene.traverse(object => {
    if (object instanceof THREE.Mesh) {
      object.geometry.dispose();
      if (Array.isArray(object.material)) {
        object.material.forEach(material => material.dispose());
      } else {
        object.material.dispose();
      }
    }
  });
})

watch(() => props.isVisible, (newVisible) => {
  if (newVisible) {
    startAnimation()
  }
})

watch(() => props.diceNumbers, (newNumbers) => {
  if (newNumbers.length === 6 && isRolling.value) {
    const elapsed = Date.now() - animationStartTime.value
    const remaining = Math.max(0, props.minDuration - elapsed)
    setTimeout(() => {
      setDiceFinalState(newNumbers)
    }, remaining)
  }
}, { deep: true })

const onWindowResize = () => {
    if (camera && renderer) {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    }
}

const initThree = () => {
  scene = new THREE.Scene()
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(0, 8, 12)
  camera.lookAt(0, 0, 0)

  renderer = new THREE.WebGLRenderer({ canvas: diceCanvas.value!, alpha: true, antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.shadowMap.enabled = true

  const ambientLight = new THREE.AmbientLight(0xffffff, 0.8)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(10, 20, 5)
  directionalLight.castShadow = true
  scene.add(directionalLight)
}

const createDiceMaterials = (): THREE.MeshStandardMaterial[] => {
    const materials: THREE.MeshStandardMaterial[] = [];
    const faceNumbers = [3, 4, 2, 5, 1, 6]; // right, left, top, bottom, front, back

    for (const number of faceNumbers) {
        // 为每个面创建新的canvas，避免共享同一个canvas导致的纹理问题
        const canvas = document.createElement('canvas');
        canvas.width = 256;
        canvas.height = 256;
        const context = canvas.getContext('2d')!;
        
        // 绘制骰子面
        context.fillStyle = '#f8f9fa'; // 浅灰色/白色背景
        context.fillRect(0, 0, 256, 256);
        context.fillStyle = '#dc3545'; // 红色点数
        context.font = 'bold 160px Arial';
        context.textAlign = 'center';
        context.textBaseline = 'middle';
        context.fillText(number.toString(), 128, 128);
        
        // 创建纹理并应用到材质
        const texture = new THREE.CanvasTexture(canvas);
        materials.push(new THREE.MeshStandardMaterial({ 
            map: texture, 
            roughness: 0.2, 
            metalness: 0.1 
        }));
    }
    return materials;
};

const createDice = () => {
  const diceGeometry = new THREE.BoxGeometry(2, 2, 2)
  
  for (let i = 0; i < 6; i++) {
    const materials = createDiceMaterials(); // Create unique materials for each die
    const mesh = new THREE.Mesh(diceGeometry, materials)
    mesh.castShadow = true
    
    // Position dice in a 2x3 grid
    mesh.position.set(
        (i % 3 - 1) * 3.5,
        Math.floor(i / 3) * -3.5 + 2,
        0
    );

    scene.add(mesh)
    diceMeshes.push(mesh)
  }
}

const getQuaternionForNumber = (number: number): THREE.Quaternion => {
    const q = new THREE.Quaternion();
    // 修正旋转角度，确保朝向用户的面是结果点数
    switch (number) {
        case 1: q.setFromEuler(new THREE.Euler(0, 0, 0)); break;        // 1朝向用户
        case 2: q.setFromEuler(new THREE.Euler(-Math.PI / 2, 0, 0)); break; // 2朝向用户
        case 3: q.setFromEuler(new THREE.Euler(0, Math.PI / 2, 0)); break;  // 3朝向用户
        case 4: q.setFromEuler(new THREE.Euler(0, -Math.PI / 2, 0)); break; // 4朝向用户
        case 5: q.setFromEuler(new THREE.Euler(Math.PI / 2, 0, 0)); break;  // 5朝向用户
        case 6: q.setFromEuler(new THREE.Euler(Math.PI, 0, 0)); break;      // 6朝向用户
    }
    return q;
};

const startAnimation = () => {
  isRolling.value = true
  animationStartTime.value = Date.now()
  diceMeshes.forEach(mesh => {
      mesh.rotation.set(
          Math.random() * Math.PI * 2,
          Math.random() * Math.PI * 2,
          Math.random() * Math.PI * 2
      )
  })
}

const setDiceFinalState = (numbers: number[]) => {
  numbers.forEach((num, i) => {
    const mesh = diceMeshes[i]
    const finalQuaternion = getQuaternionForNumber(num);
    
    // Simple and instant rotation
    mesh.quaternion.copy(finalQuaternion);
  })
  
  setTimeout(() => {
      isRolling.value = false
      setTimeout(() => {
        emit('animationComplete')
      }, 4000) // Keep result visible for 4 seconds
  }, 500) // Short delay to ensure final state is rendered
}

const animate = () => {
  animationFrameId = requestAnimationFrame(animate)
  
  if (isRolling.value) {
      const delta = 0.1; // 增加旋转速度，从0.02提高到0.1
      diceMeshes.forEach(mesh => {
          mesh.rotation.x += delta;
          mesh.rotation.y += delta;
      });
  }

  renderer.render(scene, camera)
}

</script>

<style scoped>
.dice-animation-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 1000;
}

.dice-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(12px);
}

.dice-canvas {
  width: 100%;
  height: 100%;
  display: block;
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
</style>