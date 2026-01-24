<script setup lang="ts">
import { useIdle } from '@vueuse/core'
import { Settings } from 'lucide-vue-next'
import { storeToRefs } from 'pinia'
import { computed, ref, watch } from 'vue'
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

/** 闲置时隐藏设置按钮 */
const showSettingsButton = ref(true)
const { idle } = useIdle(5 * 1000)
watch(idle, (newIdle) => {
  showSettingsButton.value = !newIdle
})

/** * --- 新增逻辑：随机颜色功能 --- 
 * 设定：当 clockConfig.color 为 'random' 时触发
 */
const isRainbowMode = computed(() => clockConfig.value.color === 'random')

// 存储 H1, H2, 冒号, M1, M2 的颜色
const digitColors = ref<string[]>(['', '', '', '', ''])

// 生成鲜艳的随机颜色 (HSL)
function generateBrightColor() {
  const hue = Math.floor(Math.random() * 360)
  // 饱和度 70%-90%，亮度 60%-70%，确保在深色背景下清晰可见
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

// 计算当前的 "10分钟区块" (例如 10:00-10:09 是一个区块)
const tenMinuteBlock = computed(() => Math.floor(now.value.getTime() / (5 * 1000)))

// 当处于随机模式，且进入新的10分钟区块时，更新颜色
watch([tenMinuteBlock, isRainbowMode], ([block, mode]) => {
  if (mode) {
    updateColors()
  }
}, { immediate: true })

/** 获取指定位置的颜色，如果不是随机模式则返回 undefined (使用默认配置颜色) */
function getColor(index: number) {
  return isRainbowMode.value ? digitColors.value[index] : undefined
}

</script>

<template>
  <div
    class="glass-panel relative h-full flex flex-col items-center justify-center text-white w-full overflow-y-auto overflow-x-hidden"
    :class="{ 'clock-only-mode': layoutConfig.clockOnlyMode }"
    @click.stop="showSettingsButton = !showSettingsButton"
  >
    <button
      :class="{ 'opacity-0': !showSettingsButton }"
      class="absolute top-6 right-6 z-10 p-2 rounded-full bg-white/10 hover:bg-white/20 hover:rotate-90" @click="openSettings"
    >
      <Settings class="w-6 h-6 text-white" />
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
      class="clock-display tabular-nums transition-all duration-500"
      :style="{ color: clockConfig.color, fontWeight: clockConfig.fontWeight, opacity: clockConfig.opacity }"
    >
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
/* 完全保留您上一次满意的样式配置 
*/
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
  font-size: 95%;
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