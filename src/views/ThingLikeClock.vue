<script setup lang="ts">
import { useIdle } from '@vueuse/core'
import { Settings } from 'lucide-vue-next'
import { storeToRefs } from 'pinia'
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import { useTime } from '../hooks/useTime'
import { useConfigStore } from '../stores/config'

const configStore = useConfigStore()
const { showDrawer, activeTab } = storeToRefs(configStore)

// 使用 useTime 获取时间
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

// --- 模拟时钟核心逻辑 (60FPS) ---
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

  // 秒针：平滑扫秒
  secondDeg.value = (s + ms / 1000) * 6
  // 分针：随秒微动
  minuteDeg.value = m * 6 + s * 0.1
  // 时针：随分微动
  hourDeg.value = h * 30 + m * 0.5 + (s / 120)

  animationFrameId = requestAnimationFrame(updateClockLoop)
}

onMounted(() => {
  updateClockLoop()
})

onUnmounted(() => {
  cancelAnimationFrame(animationFrameId)
})

// --- 日期和星期格式化 ---
const dayLabel = computed(() => {
  return new Intl.DateTimeFormat('zh-CN', { weekday: 'short' }).format(now.value)
})

const dateLabel = computed(() => {
  return now.value.getDate()
})

// --- SVG 矩形刻度计算逻辑 ---
const VIEW_W = 400
const VIEW_H = 300
const CENTER_X = VIEW_W / 2
const CENTER_Y = VIEW_H / 2

// 核心几何函数：计算射线与矩形的交点
function getRectPoint(angleDeg: number, w: number, h: number) {
  const rad = (angleDeg - 90) * (Math.PI / 180)
  const dx = Math.cos(rad)
  const dy = Math.sin(rad)
  const xLimit = w / 2
  const yLimit = h / 2
  
  const absDx = Math.abs(dx) < 1e-6 ? 1e-6 : Math.abs(dx)
  const absDy = Math.abs(dy) < 1e-6 ? 1e-6 : Math.abs(dy)

  const scale = Math.min(xLimit / absDx, yLimit / absDy)
  return { x: CENTER_X + dx * scale, y: CENTER_Y + dy * scale }
}

// 生成60个刻度的坐标
const ticks = computed(() => {
  const items = []
  const outerW = VIEW_W - 20 
  const outerH = VIEW_H - 20
  
  for (let i = 0; i < 60; i++) {
    const angle = i * 6
    const isHourMark = i % 5 === 0
    let inset = 0
    
    // --- 刻度长度核心逻辑 ---
    if (isHourMark) {
      const hourIndex = i / 5 // 0-11
      
      // 1. 最长刻度: 2, 4, 8, 10
      if ([2, 4, 8, 10].includes(hourIndex)) {
        inset = 85 
      } 
      // 2. 中等刻度: 1, 5, 7, 11
      else if ([1, 5, 7, 11].includes(hourIndex)) {
        inset = 50 
      } 
      // 3. 最短刻度: 0(12), 3, 6, 9
      else {
        inset = 25 
      }
    } else {
      // 4. 分钟刻度
      inset = 10 
    }
    
    const start = getRectPoint(angle, outerW, outerH)
    const end = getRectPoint(angle, outerW - inset, outerH - inset)

    items.push({
      x1: start.x, y1: start.y,
      x2: end.x, y2: end.y,
      isHour: isHourMark
    })
  }
  return items
})
</script>

<template>
  <div
    class="relative w-full h-full overflow-hidden bg-black select-none flex items-center justify-center font-sans"
    @click.stop="showSettingsButton = !showSettingsButton"
  >
    <button
      :class="{ 'opacity-0': !showSettingsButton }"
      class="absolute top-8 right-8 z-50 p-3 rounded-full bg-gray-800/50 backdrop-blur-md hover:bg-gray-700 transition-all duration-300"
      @click="openSettings"
    >
      <Settings class="w-6 h-6 text-white/80" />
    </button>

    <div class="relative w-full h-full max-w-[133.33vh] max-h-[75vw]" style="aspect-ratio: 4/3;">
      
      <svg class="absolute inset-0 w-full h-full pointer-events-none" :viewBox="`0 0 ${VIEW_W} ${VIEW_H}`">
        <line
          v-for="(tick, index) in ticks"
          :key="index"
          :x1="tick.x1" :y1="tick.y1"
          :x2="tick.x2" :y2="tick.y2"
          :stroke="tick.isHour ? 'rgba(255,255,255,0.7)' : 'rgba(255,255,255,0.25)'"
          :stroke-width="tick.isHour ? 3 : 1.5"
          stroke-linecap="round"
        />
      </svg>

      <div class="absolute inset-0 pointer-events-none">
        <div class="absolute top-[10%] left-1/2 -translate-x-1/2 text-white font-bold text-[13vmin] leading-none">12</div>
        <div class="absolute bottom-[10%] left-1/2 -translate-x-1/2 text-white font-bold text-[13vmin] leading-none">6</div>
        <div class="absolute left-[10%] top-1/2 -translate-y-1/2 text-white font-bold text-[13vmin] leading-none">9</div>
        <div class="absolute right-[10%] top-1/2 -translate-y-1/2 text-white font-bold text-[13vmin] leading-none">3</div>
      </div>

      <div class="absolute top-1/2 right-[22%] -translate-y-1/2 flex items-center gap-3 pointer-events-none">
        <span class="text-white font-bold text-[5vmin]">{{ dateLabel }}</span>
        <span class="text-[#ff453a] font-bold text-[5vmin]">{{ dayLabel }}</span>
      </div>

      <div class="absolute inset-0 pointer-events-none">
        
        <div 
          class="absolute left-1/2 bg-white rounded-full shadow-lg z-10"
          :style="{ 
            width: '2.5vmin', 
            height: '16vmin', 
            transformOrigin: 'bottom center',  
            transform: `translateX(-50%) rotate(${hourDeg}deg)`,
            bottom: '50%' 
          }"
        ></div>

        <div 
          class="absolute left-1/2 bg-white rounded-full shadow-lg z-20"
          :style="{ 
            width: '2vmin', 
            height: '24vmin', 
            transformOrigin: 'bottom center', 
            transform: `translateX(-50%) rotate(${minuteDeg}deg)`,
            bottom: '50%' 
          }"
        ></div>

        <div 
          class="absolute top-1/2 left-1/2 w-0 h-0 z-30"
          :style="{ transform: `rotate(${secondDeg}deg)` }"
        >
          <div 
            class="absolute border-[0.5vmin] border-[#ff9f0a] rounded-full box-border bg-black"
            style="
              width: 3vmin; 
              height: 3vmin; 
              transform: translate(-50%, -50%);
            "
          ></div>

          <div 
            class="absolute bg-[#ff9f0a] rounded-full"
            style="
              width: 0.6vmin; 
              height: 30vmin; 
              left: -0.3vmin;
              bottom: 1.5vmin; 
            "
          ></div>

          <div 
            class="absolute bg-[#ff9f0a] rounded-full"
            style="
              width: 0.6vmin; 
              height: 5vmin; 
              left: -0.3vmin;
              top: 1.5vmin; 
            "
          ></div>
        </div>

      </div>

    </div>
  </div>
</template>

<style scoped>
div {
  font-variant-numeric: tabular-nums;
}
</style>