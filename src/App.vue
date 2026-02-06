<script setup lang="ts">
import { useIdle, useMagicKeys } from '@vueuse/core'
import { storeToRefs } from 'pinia'
import { computed, ref, watch, watchEffect, onMounted, onUnmounted } from 'vue'
import NewYearEgg from './components/NewYearEgg.vue'
import SettingsDrawer from './components/SettingsDrawer.vue'
import WeatherEffects from './components/WeatherEffects.vue'
// 引入天气弹窗组件
import WeatherForecastModal from './components/WeatherForecastModal.vue'
import { i18n } from './i18n'
import { useConfigStore } from './stores/config'
import { useWeatherStore } from './stores/weather'
import { isIpadIOS15OrLower } from './utils/device'
import CalendarView from './views/CalendarView.vue'
import ClockWeatherView from './views/ClockWeatherView.vue'
import SmartHomeView from './views/SmartHomeView.vue'
import HomeView2 from './views/HomeView2.vue'
import ThingLikeClock from './views/ThingLikeClock.vue'

const configStore = useConfigStore()
const { showDrawer, layoutConfig } = storeToRefs(configStore)

const currentPage = ref(3)
const calendarRef = ref<any>(null)

const weatherStore = useWeatherStore()
const { 
  weatherData, 
  showRainEffect, 
  showThunderEffect, 
  showSnowEffect, 
  showWeatherDetail 
} = storeToRefs(weatherStore)

const isSwiping = ref(false)

// 强制关闭天气特效
const shouldShowWeatherEffects = computed(() => {
  return false
})

// --- [新增] 睡眠模式检测 (省电逻辑) ---
const isSleepMode = ref(false)
let sleepCheckTimer: number | null = null

function checkSleepMode() {
  const hour = new Date().getHours()
  // 设定：晚上 21 点到次日早上 6 点为睡眠模式
  // 逻辑：大于等于21点 (21, 22, 23) 或者 小于6点 (0, 1, 2, 3, 4, 5)
  isSleepMode.value = hour >= 21 || hour < 6
}

// --- 每日凌晨 3:00 自动刷新逻辑 ---
let refreshTimer: number | null = null

function scheduleDailyRefresh() {
  const now = new Date()
  let targetTime = new Date(
    now.getFullYear(),
    now.getMonth(),
    now.getDate(), 
    3, 0, 0 
  )
  // 如果今天3点已过，设为明天3点
  if (targetTime.getTime() < now.getTime()) {
    targetTime.setDate(targetTime.getDate() + 1)
  }

  const msToTarget = targetTime.getTime() - now.getTime()
  console.log(`[System] 下次自动刷新将在 ${Math.round(msToTarget / 1000 / 60)} 分钟后执行`)

  refreshTimer = window.setTimeout(() => {
    window.location.reload()
  }, msToTarget)
}

onMounted(() => {
  // 1. 启动每日自动刷新
  scheduleDailyRefresh()
  
  // 2. [新增] 启动睡眠模式检测
  checkSleepMode() // 立即检查一次
  sleepCheckTimer = window.setInterval(checkSleepMode, 60 * 1000) // 每分钟检查一次
})

onUnmounted(() => {
  if (refreshTimer) clearTimeout(refreshTimer)
  // [新增] 清除睡眠检测定时器
  if (sleepCheckTimer) clearInterval(sleepCheckTimer)
})
// ----------------------------------------

let startX = 0

function goToPage(page: number) {
  currentPage.value = page
  if (page === 4 && calendarRef.value) {
    calendarRef.value.refreshToday()
  }
}

function handleTouchStart(e: TouchEvent) {
  startX = e.touches[0].clientX
}

function handleTouchEnd(e: TouchEvent) {
  const endX = e.changedTouches[0].clientX
  const diff = startX - endX
  if (Math.abs(diff) > 50) {
    isSwiping.value = true
    setTimeout(() => { isSwiping.value = false }, 50)
    if (diff > 0 && currentPage.value < 4) goToPage(currentPage.value + 1)
    else if (diff < 0 && currentPage.value > 0) goToPage(currentPage.value - 1)
  }
}

function handleMouseDown(e: MouseEvent) {
  startX = e.clientX
}

function handleMouseUp(e: MouseEvent) {
  const diff = startX - e.clientX
  if (Math.abs(diff) > 50) {
    isSwiping.value = true
    setTimeout(() => { isSwiping.value = false }, 50)
    if (diff > 0 && currentPage.value < 4) goToPage(currentPage.value + 1)
    else if (diff < 0 && currentPage.value > 0) goToPage(currentPage.value - 1)
  }
}

function handleGlobalClick(e: MouseEvent) {
  if (isSwiping.value) {
    e.stopImmediatePropagation()
    e.preventDefault()
  }
}

const { left, right } = useMagicKeys()
watchEffect(() => {
  if (showDrawer.value) return
  if (left.value && currentPage.value > 0) goToPage(currentPage.value - 1)
  if (right.value && currentPage.value < 4) goToPage(currentPage.value + 1)
})

const { language } = storeToRefs(configStore)
i18n.global.locale.value = language.value
watch(language, (nextLocale) => {
  i18n.global.locale.value = nextLocale
})
</script>

<template>
  <div
    class="viewport-container overflow-hidden relative w-screen h-screen bg-black"
    @touchstart="handleTouchStart"
    @touchend="handleTouchEnd"
    @mousedown="handleMouseDown"
    @mouseup="handleMouseUp"
    @click.capture="handleGlobalClick"
  >
    <template v-if="!isIpadIOS15OrLower()">
      <div class="absolute top-1/4 left-1/4 w-96 h-96 bg-blue-900/10 rounded-full blur-3xl pointer-events-none" />
      <div class="absolute bottom-1/4 right-1/4 w-96 h-96 bg-purple-900/10 rounded-full blur-3xl pointer-events-none" />
    </template>

    <div
      class="main-slider flex h-full transition-all duration-700 cubic-bezier"
      :style="{ marginLeft: `-${currentPage * 100}vw`, width: '500vw' }"
    >
      <div class="slide-page w-screen h-screen flex items-center justify-center flex-shrink-0">
        <HomeView2 v-if="currentPage === 0" />
      </div>

      <div class="slide-page w-screen h-screen flex items-center justify-center flex-shrink-0">
        <ThingLikeClock v-if="currentPage === 1" />
      </div>

      <div class="slide-page w-screen h-screen flex items-center justify-center flex-shrink-0">
        <SmartHomeView v-if="currentPage === 2" />
      </div>

      <div class="slide-page w-screen h-screen flex items-center justify-center flex-shrink-0">
        <ClockWeatherView />
      </div>

      <div class="slide-page w-screen h-screen flex items-center justify-center flex-shrink-0">
        <CalendarView v-if="currentPage === 4" ref="calendarRef" />
      </div>
    </div>

    <SettingsDrawer />
    
    <WeatherForecastModal 
      :show="showWeatherDetail" 
      @close="showWeatherDetail = false" 
    />

    <NewYearEgg />

    <WeatherEffects v-if="shouldShowWeatherEffects" />

    <div 
      v-if="isSleepMode" 
      class="fixed inset-0 z-[9999] bg-black pointer-events-none transition-opacity duration-1000"
      style="opacity: 0.85;"
    ></div>
  </div>
</template>

<style scoped>
.cubic-bezier {
  transition-timing-function: cubic-bezier(0.23, 1, 0.32, 1);
  transition-property: margin-left;
}
</style>