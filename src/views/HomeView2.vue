<script setup lang="ts">
import { useIdle } from '@vueuse/core'
import { Settings } from 'lucide-vue-next'
import { storeToRefs } from 'pinia'
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import { useTime } from '../hooks/useTime'
import { useConfigStore } from '../stores/config'

const configStore = useConfigStore()
const { showDrawer, activeTab } = storeToRefs(configStore)

// useTime 仅用于日期和星期的低频更新
const { now } = useTime()

function openSettings() {
  activeTab.value = 'smart'
  showDrawer.value = true
}

/** 闲置时隐藏设置按钮 */
const showSettingsButton = ref(true)
const { idle } = useIdle(5 * 1000)
watch(idle, (newIdle) => {
  showSettingsButton.value = !newIdle
})

// --- 模拟时钟核心逻辑 (高帧率模式) ---
const secondDeg = ref(0)
const minuteDeg = ref(0)
const hourDeg = ref(0)
let animationFrameId: number

const updateClockLoop = () => {
  const date = new Date()
  const s = date.getSeconds()
  const ms = date.getMilliseconds()
  const m = date.getMinutes()
  const h = date.getHours() % 12

  // 1. 秒针：利用毫秒实现极致丝滑的“扫秒” (60FPS)
  secondDeg.value = (s + ms / 1000) * 6

  // 2. 分针：随秒针微动 (让分针也平滑移动，而不是一分钟跳一次)
  minuteDeg.value = m * 6 + s * 0.1 + (ms / 1000) * 0.1

  // 3. 时针：随分针微动
  hourDeg.value = h * 30 + m * 0.5 + (s / 120)

  // 循环调用
  animationFrameId = requestAnimationFrame(updateClockLoop)
}

onMounted(() => {
  updateClockLoop()
})

onUnmounted(() => {
  cancelAnimationFrame(animationFrameId)
})

// --- 星期显示 (中文) ---
const dayLabel = computed(() => {
  return new Intl.DateTimeFormat('zh-CN', { weekday: 'short' }).format(now.value)
})
</script>

<template>
  <div
    class="glass-panel relative h-full flex flex-col items-center justify-center w-full overflow-hidden"
    @click.stop="showSettingsButton = !showSettingsButton"
  >
    <button
      :class="{ 'opacity-0': !showSettingsButton }"
      class="absolute top-6 right-6 z-10 p-2 rounded-full bg-white/10 hover:bg-white/20 hover:rotate-90 transition-all duration-300"
      @click="openSettings"
    >
      <Settings class="w-6 h-6 text-white" />
    </button>

    <div class="analog-clock">
      <div class="clock-face">
        
        <div 
          v-for="i in 12" 
          :key="`h-${i}`" 
          class="mark hour-mark"
          :style="{ transform: `rotate(${i * 30}deg) translateY(calc(var(--clock-size) / -2 + 15px))` }"
        ></div>
        
        <div 
          v-for="i in 60" 
          :key="`m-${i}`" 
          class="mark minute-mark"
          :class="{ 'hidden': i % 5 === 0 }"
          :style="{ transform: `rotate(${i * 6}deg) translateY(calc(var(--clock-size) / -2 + 15px))` }"
        ></div>

        <div class="clock-number num-12">12</div>
        <div class="clock-number num-3">3</div>
        <div class="clock-number num-6">6</div>
        <div class="clock-number num-9">9</div>

        <div class="day-display">
          {{ dayLabel }}
        </div>

        <div class="date-window">
          {{ now.getDate() }}
        </div>

      </div>

      <div 
        class="hand hour-hand shadow-lg"
        :style="{ transform: `translate(-50%, -100%) rotate(${hourDeg}deg)` }"
      ></div>

      <div class="hand minute-hand shadow-lg"
        :style="{ transform: `translate(-50%, -100%) rotate(${minuteDeg}deg)` }"
      ></div>

      <div class="hand second-hand"
        :style="{ transform: `translate(-50%, -100%) rotate(${secondDeg}deg)` }"
      >
        <div class="second-counterweight"></div>
      </div>

      <div class="center-cap shadow-md"></div>
    </div>
  </div>
</template>

<style scoped>
.glass-panel {
  max-width: 100%;
  margin: 0;
  background-color: black; /* 设置为纯黑色背景 */
}

/* --- 核心变量 --- */
.analog-clock {
  /* 调整时钟大小以适应 iPad Air 屏幕，并铺满 */
  --clock-size: min(90vh, 90vw); /* 使用vh和vw的最小值，确保在不同方向上都能铺满 */
  
  width: var(--clock-size);
  height: var(--clock-size);
  min-width: 300px;
  min-height: 300px;
  border-radius: 50%;
  position: relative;
  
  /* 移除玻璃面板效果，改为纯黑色背景 */
  background: none;
  backdrop-filter: none;
  border: none;
  box-shadow: none;
}

.clock-face {
  width: 100%;
  height: 100%;
  position: relative;
}

/* --- 刻度 --- */
.mark {
  position: absolute;
  left: 50%;
  top: 50%;
  background-color: rgba(255, 255, 255, 0.8);
  transform-origin: center center;
}

.hour-mark {
  width: 8px;
  height: 35px;
  margin-left: -4px; 
  margin-top: -17.5px;
  border-radius: 4px;
}

.minute-mark {
  width: 3px;
  height: 15px;
  margin-left: -1.5px;
  margin-top: -7.5px;
  opacity: 0.4;
}

/* --- 数字 --- */
.clock-number {
  position: absolute;
  top: 50%;
  left: 50%;
  font-family: 'SFCompactRounded', 'Huninn', sans-serif;
  font-size: 11vh;
  font-weight: 700;
  color: rgba(255, 255, 255, 0.9);
  display: flex;
  align-items: center;
  justify-content: center;
  width: 12vh;
  height: 12vh;
  z-index: 1;
}

.num-12 { transform: translate(-50%, -50%) translateY(calc(var(--clock-size) / -2 + 80px)); }
.num-6  { transform: translate(-50%, -50%) translateY(calc(var(--clock-size) / 2 - 80px)); }
.num-3  { transform: translate(-50%, -50%) translateX(calc(var(--clock-size) / 2 - 80px)); }
.num-9  { transform: translate(-50%, -50%) translateX(calc(var(--clock-size) / -2 + 80px)); }

/* --- 星期显示 --- */
.day-display {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%) translateY(-15vh);
  
  font-family: 'SFCompactRounded', 'Huninn', sans-serif;
  font-size: 3vh;
  letter-spacing: 0.2em;
  color: rgba(255, 255, 255, 0.8);
  font-weight: 600;
  z-index: 1;
  text-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

/* --- 日期窗口 --- */
.date-window {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%) translateY(15vh);
  
  width: 6vh;
  height: 6vh;
  border-radius: 50%;
  border: 1.5px solid rgba(255, 255, 255, 0.3);
  background-color: rgba(255, 255, 255, 0.05);
  
  display: flex;
  align-items: center;
  justify-content: center;
  
  font-family: 'SFCompactRounded', 'Huninn', sans-serif;
  font-size: 2.8vh;
  font-weight: 700;
  color: #ff6b6b; 
  z-index: 1;
}

/* --- 指针 --- */
.hand {
  position: absolute;
  top: 50%;
  left: 50%;
  transform-origin: bottom center; 
  border-radius: 999px 999px 0 0;
  z-index: 10;
  /* 关键：使用 transform 修改时，不加 transition，否则会跟不上 RAF 的高频更新，导致抖动 */
  will-change: transform; 
}

.hour-hand {
  width: 16px;
  height: 28%;
  margin-left: -8px;
  background-color: rgba(255, 255, 255, 0.9);
  z-index: 2;
}

.minute-hand {
  width: 8px;
  height: 40%;
  margin-left: -4px;
  background-color: rgba(255, 255, 255, 0.8);
  z-index: 3;
}

.second-hand {
  width: 3px;
  height: 44%;
  margin-left: -1.5px;
  background-color: #ff6b6b;
  z-index: 4;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
}

.second-counterweight {
  width: 8px;
  height: 30px;
  background-color: #ff6b6b;
  border-radius: 999px;
  position: absolute;
  bottom: -25px; 
  left: -2.5px;
}

/* 中心帽 */
.center-cap {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 22px;
  height: 22px;
  margin-left: -11px;
  margin-top: -11px;
  background: linear-gradient(135deg, #ffffff, #d4d4d4);
  border: 3px solid #ff6b6b;
  border-radius: 50%;
  box-shadow: 0 2px 5px rgba(0,0,0,0.3), inset 0 1px 2px rgba(255,255,255,0.8);
  z-index: 20;
}

/* --- 移动端适配 --- */
@media (max-width: 768px) {
  .analog-clock {
    --clock-size: 90vw;
  }
  
  .clock-number {
    font-size: 11vw;
    width: 12vw;
    height: 12vw;
  }

  .num-12 { transform: translate(-50%, -50%) translateY(calc(var(--clock-size) / -2 + 16vw)); }
  .num-6  { transform: translate(-50%, -50%) translateY(calc(var(--clock-size) / 2 - 16vw)); }
  .num-3  { transform: translate(-50%, -50%) translateX(calc(var(--clock-size) / 2 - 16vw)); }
  .num-9  { transform: translate(-50%, -50%) translateX(calc(var(--clock-size) / -2 + 16vw)); }

  /* 移动端调整 */
  .day-display { 
    transform: translate(-50%, -50%) translateY(-15vw); 
    font-size: 3.5vw;
  }
  .date-window { 
    transform: translate(-50%, -50%) translateY(15vw);
    width: 7vw; height: 7vw; font-size: 3.2vw;
  }
}
</style>