<script setup lang="ts">
import { useIdle } from '@vueuse/core'
import { Settings, RefreshCw } from 'lucide-vue-next' // [修改] 引入 RefreshCw 图标
import { storeToRefs } from 'pinia'
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import Digit from '../components/Digit.vue'
import Weather from '../components/Weather.vue'
import { useTime } from '../hooks/useTime'
import { useConfigStore } from '../stores/config'

const configStore = useConfigStore()
const { clockConfig, layoutConfig, showDrawer, activeTab } = storeToRefs(configStore)
const { locale } = useI18n()

const { h1, h2, m1, m2, s1, s2, lunar, now } = useTime({
  is24Hour: computed(() => clockConfig.value.is24Hour),
})

function openSettings() {
  activeTab.value = 'general'
  showDrawer.value = true
}

// [新增] 刷新页面函数
function reloadPage() {
  window.location.reload()
}

const weekdayLabel = computed(() => {
  const formatter = new Intl.DateTimeFormat(locale.value, { weekday: 'long' })
  return formatter.format(now.value)
})

const yearMonthLabel = computed(() => {
  const date = now.value
  if (locale.value === 'en-US') {
    const month = String(date.getMonth() + 1).padStart(2, '0')
    return `${month}-${date.getFullYear()}`
  }
  const formatter = new Intl.DateTimeFormat(locale.value, { year: 'numeric', month: 'long' })
  return formatter.format(date)
})

const showLunar = computed(() => locale.value !== 'en-US')

const baseDelay = computed(() => {
  return 0
})

const showSettingsButton = ref(true)
const { idle } = useIdle(5 * 1000)
watch(idle, (newIdle) => {
  showSettingsButton.value = !newIdle
})

/** 随机颜色功能 */
const isRainbowMode = computed(() => clockConfig.value.color === 'random')
const digitColors = ref<string[]>(['', '', '', '', ''])

function generateBrightColor() {
  const hue = Math.floor(Math.random() * 360)
  return `hsl(${hue}, 80%, 65%)`
}

function updateColors() {
  digitColors.value = [
    generateBrightColor(),
    generateBrightColor(),
    generateBrightColor(),
    generateBrightColor(),
    generateBrightColor()
  ]
}

const tenMinuteBlock = computed(() => Math.floor(now.value.getTime() / (60 * 1000)))

watch([tenMinuteBlock, isRainbowMode], ([block, mode]) => {
  if (mode) {
    updateColors()
  }
}, { immediate: true })

function getColor(index: number) {
  return isRainbowMode.value ? digitColors.value[index] : undefined
}

// --- 红点环绕核心逻辑 (12点方向修正版) ---
const dotStyle = ref({ top: '0px', left: '50%' })
let animationFrameId: number

// [省电模式] 判断是否为睡眠时间 (晚上21点 - 次日6点)
function isSleepTime() {
  const h = new Date().getHours()
  // 大于等于21点 或 小于6点
  return h >= 21 || h < 6
}

const updateDotPosition = () => {
  // [省电模式逻辑]
  if (isSleepTime()) {
    // 睡眠时间：隐藏红点 (设置 opacity 为 0)
    dotStyle.value = { ...dotStyle.value, opacity: 0 } as any
    
    // 降低检查频率：每秒检查一次时间
    setTimeout(updateDotPosition, 1000)
    return
  }

  // 正常模式
  const date = new Date()
  const s = date.getSeconds()
  const ms = date.getMilliseconds()
  // 当前秒数
  const currentSec = s + ms / 1000
  
  // 偏移量：直径14px，半径7px -> offset -7px
  const offset = '-7px' 

  let style = {}
  
  if (currentSec < 7.5) {
    const progress = (currentSec / 7.5) * 50 + 50
    style = { top: offset, left: `${progress}%` }
  } 
  else if (currentSec < 22.5) {
    const progress = ((currentSec - 7.5) / 15) * 100
    style = { top: `${progress}%`, left: `calc(100% + ${offset})` }
  } 
  else if (currentSec < 37.5) {
    const progress = 100 - ((currentSec - 22.5) / 15) * 100
    style = { top: `calc(100% + ${offset})`, left: `${progress}%` }
  } 
  else if (currentSec < 52.5) {
    const progress = 100 - ((currentSec - 37.5) / 15) * 100
    style = { top: `${progress}%`, left: offset }
  } 
  else {
    const progress = ((currentSec - 52.5) / 7.5) * 50
    style = { top: offset, left: `${progress}%` }
  }

  dotStyle.value = style as any
  animationFrameId = requestAnimationFrame(updateDotPosition)
}

onMounted(() => {
  updateDotPosition()
})

onUnmounted(() => {
  cancelAnimationFrame(animationFrameId)
})

</script>

<template>
  <div
    class="glass-panel relative h-full flex flex-col items-center justify-center text-white w-full overflow-y-auto overflow-x-hidden"
    :class="{ 'clock-only-mode': layoutConfig.clockOnlyMode }"
    @click.stop="showSettingsButton = !showSettingsButton"
  >
    <button
      :class="{ 'opacity-0': !showSettingsButton }"
      class="absolute top-6 right-6 z-10 p-2 rounded-full bg-white/10 hover:bg-white/20 hover:rotate-90 transition-opacity" @click="openSettings"
    >
      <Settings class="w-6 h-6 text-white" />
    </button>

    <button
      :class="{ 'opacity-0': !showSettingsButton }"
      class="absolute top-6 left-6 z-10 p-2 rounded-full bg-white/10 hover:bg-white/20 transition-opacity"
      @click="reloadPage"
    >
      <RefreshCw class="w-6 h-6 text-white" />
    </button>

    <div v-if="!layoutConfig.clockOnlyMode" class="flex flex-col sm:flex-row items-center md:items-start w-full justify-center">
      <div class="flex items-center">
        <div class="date-day-big">
          {{ now.getDate() }}
        </div>
        <div class="flex flex-col mr-[5.6vh]">
          <span class="weekday-label">
            {{ weekdayLabel }}
          </span>
          <span class="year-label">
            {{ yearMonthLabel }}
          </span>
        </div>
        <div v-if="showLunar" class="flex flex-col">
          <div class="lunar-date-label">
            {{ lunar.date }}<span v-if="lunar.festival">·{{ lunar.festival }}</span>
          </div>
          <span class="lunar-year-label">{{ lunar.year }}({{ lunar.yearShengxiao }})年{{ lunar.month }}月</span>
        </div>
      </div>
    </div>

    <div
      class="clock-display tabular-nums transition-all duration-500 relative"
      :style="{ color: clockConfig.color, fontWeight: clockConfig.fontWeight, opacity: clockConfig.opacity }"
    >
      
      <div class="seconds-dot-wrapper">
        <div class="seconds-dot animate-pulse-dot" :style="dotStyle"></div>
      </div>

      <Digit
        v-if="clockConfig.is24Hour || h1 !== 0"
        :value="h1" :enable-tilt="clockConfig.enableTilt"
        :trigger="Math.floor(now.getTime() / 5000)"
        :delay="(5 - baseDelay) * 100"
        :narrow-gap="true" 
        class="opacity-95"
        :style="{ color: getColor(0) }" 
      />
      
      <Digit
        :value="h2" :enable-tilt="clockConfig.enableTilt"
        :trigger="Math.floor(now.getTime() / 5000)"
        :delay="(4 - baseDelay) * 100"
        :narrow-gap="true"
        class="opacity-95"
        :class="[{
          brightness: clockConfig.is24Hour || (!clockConfig.is24Hour && h1 !== 0),
        }]"
        :style="{ color: getColor(1) }"
      />

      <div 
        class="clock-separator animate-blink"
        :style="{ color: getColor(2) }"
      >
        :
      </div>

      <Digit
        :value="m1" :enable-tilt="clockConfig.enableTilt"
        :trigger="Math.floor(now.getTime() / 5000)"
        :delay="(3 - baseDelay) * 100"
        :narrow-gap="true"
        class="opacity-95"
        :style="{ color: getColor(3) }"
      />
      
      <Digit
        :value="m2" :enable-tilt="clockConfig.enableTilt"
        :trigger="Math.floor(now.getTime() / 5000)"
        :delay="(2 - baseDelay) * 100"
        :narrow-gap="true"
        class="opacity-95 brightness"
        :style="{ color: getColor(4) }"
      />
    </div>

    <Weather v-if="!layoutConfig.clockOnlyMode" />
  </div>
</template>

<style scoped>
.glass-panel {
  max-width: 180vh;
  margin: 0 auto;
}

.glass-panel.clock-only-mode {
  max-width: 100vw;
}

.date-day-big {
  font-size: 11vh;
  line-height: 1.1;
  font-weight: 800;
  background: linear-gradient(to bottom, #ffffff, rgba(255, 255, 255, 0.7));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  margin-right: 1vh;
}

.weekday-label {
  font-size: 4vh;
  letter-spacing: 0.2em;
  line-height: 1.1;
  opacity: 0.9;
}

.year-label {
  font-size: 3vh;
  letter-spacing: 0.2em;
  line-height: 1.1;
  opacity: 0.8;
  margin-top: 0.5vh;
}

.lunar-date-label {
  font-size: 4vh;
  letter-spacing: 0.2em;
  line-height: 1.1;
  opacity: 0.9;
}

.lunar-year-label {
  font-size: 3vh;
  letter-spacing: 0.2em;
  line-height: 1.1;
  opacity: 0.8;
  margin-top: 0.3vh;
}

/* 时钟容器配置 */
.clock-display {
  display: flex;
  flex-direction: row !important;
  flex-wrap: nowrap !important;
  align-items: center;
  justify-content: center;
  font-family: 'SFCompactRounded', 'Huninn', sans-serif;
  font-size: 68vh;
  margin-top: 6vh;
  margin-bottom: 6vh;
  position: relative; 
  padding: 2vh; 
}

.clock-display.with-seconds {
  font-size: 30vh;
}

.clock-only-mode .clock-display {
  font-size: 50vw;
  margin-top: 0;
  margin-bottom: 0;
}

.clock-only-mode .clock-display.with-seconds {
  font-size: 25vw;
  margin-top: 0;
  margin-bottom: 0;
}

.clock-separator {
  font-size: 60%;
  opacity: 0.98;
  text-align: center;
  margin: 0 -0.08em;
  display: flex;
  justify-content: center;
  line-height: 0.8em;
  position: relative;
  top: -0.05em;
  z-index: 10;
  filter: brightness(1.8);
  flex-shrink: 0;
  min-width: 0.25em;
}

.brightness {
  filter: brightness(1.25);
}

/* --- 环绕红点样式 --- */

.seconds-dot-wrapper {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 50; 
}

.seconds-dot {
  position: absolute;
  /* 直径 14px */
  width: 14px;
  height: 14px;
  background-color: #FF1111; 
  border-radius: 50%;
  box-shadow: 0 0 12px #FF1111, 0 0 6px rgba(255, 255, 255, 0.8);
  will-change: transform, top, left; 
}

/* 呼吸闪烁：scale 1.0 <-> 0.55 */
@keyframes pulse-dot {
  0%, 100% {
    transform: scale(1);
    opacity: 1; 
  }
  50% {
    transform: scale(0.55); 
    opacity: 1; 
  }
}

.animate-pulse-dot {
  animation: pulse-dot 1s ease-in-out infinite;
}
</style>

<style>
@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0; }
}

.animate-blink {
  animation: blink 2s step-end infinite;
}
</style>