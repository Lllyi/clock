<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'

const props = defineProps<{
  value: number
  trigger?: any
  showSeconds?: boolean
  enableTilt?: boolean
  delay?: number
  // 新增：是否使用窄间距
  narrowGap?: boolean 
}>()

const displayValue = ref(props.value)
const nextValue = ref(props.value)
const isAnimating = ref(false)
const containerRotate = ref(`rotate(${(Math.random() * 12 - 6).toFixed(1)}deg)`)

watch(() => props.trigger, () => {
  containerRotate.value = `rotate(${(Math.random() * 12 - 6).toFixed(1)}deg)`
})

watch(() => props.value, (newVal, oldVal) => {
  if (newVal === oldVal) return
  containerRotate.value = `rotate(${(Math.random() * 12 - 6).toFixed(1)}deg)`
  nextValue.value = newVal
  isAnimating.value = true
  setTimeout(() => {
    isAnimating.value = false
    displayValue.value = newVal
  }, 800 + (props.delay || 0))
})

onMounted(() => {
  displayValue.value = props.value
  nextValue.value = props.value
})
</script>

<template>
  <div
    class="digit-container" 
    :class="{ 
      'show-seconds': showSeconds,
      'narrow-gap': narrowGap  /* 绑定 class */
    }"
    :style="{ 'transform': enableTilt ? containerRotate : 'none', '--delay': `${delay || 0}ms` }"
  >
    <div class="digit-window" :class="{ animating: isAnimating }">
      <div class="digit-item">
        {{ displayValue }}
      </div>
      <div class="digit-item">
        {{ nextValue }}
      </div>
    </div>
  </div>
</template>

<style scoped>
.digit-container {
  --digit-item-height: 1.0em;
  position: relative;
  display: inline-block;
  
  /* --- 关键修改开始 --- */
  height: var(--digit-item-height);
  /* 同时设置 width 和 min-width，强制宽度不被压缩 */
  width: 1.05em;
  min-width: 1.05em; 
  /* 确保不被 flex 压缩 */
  flex-shrink: 0;
  /* --- 关键修改结束 --- */

  overflow: hidden;
  margin: 0 -0.18em;
  vertical-align: middle;
  transition: transform 0.6s linear;
  mix-blend-mode: screen;
  z-index: 1;
}

/* 新增：窄间距样式，增加负 margin 让数字更紧凑 */
.digit-container.narrow-gap {
  margin: 0 -0.30em; 
}

.digit-container.show-seconds {
  margin: 0 -0.18em;
}

.digit-window {
  display: flex;
  flex-direction: column;
  width: 100%;
  transform: translateY(0);
}

.digit-window.animating {
  transform: translateY(calc(var(--digit-item-height) * -1));
  transition-property: transform;
  transition-timing-function: cubic-bezier(0.18, 0.18, 0.43, 1.34);
  transition-duration: 0.8s;
  transition-delay: var(--delay);
}

.digit-item {
  display: block;
  height: var(--digit-item-height);
  line-height: var(--digit-item-height);
  text-align: center;
  flex-shrink: 0;
}
</style>