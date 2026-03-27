<script setup>
import { ref, computed, onMounted, watch, onUnmounted } from 'vue'
import { WebviewWindow } from '@tauri-apps/api/webviewWindow'
import { emit } from '@tauri-apps/api/event'

// 状态
const appState = ref('idle') // idle, running, paused
const hours = ref(0)
const minutes = ref(5)
const seconds = ref(0)
const reminderText = ref('')
const isLoop = ref(false)
const showOverlay = ref(false)
const showSettings = ref(false)

// 设置
const settings = ref({
  autoCloseTime: 10,
  defaultReminder: '时间到了！'
})

// 计时器
let timerInterval = null
const remainingSeconds = ref(0)
let isPaused = false
let initialTotalSeconds = 0  // 保存初始设定的总秒数

// 生成数字数组
const generateNumbers = (start, end, step = 1) => {
  const nums = []
  for (let i = start; i <= end; i += step) {
    nums.push(i.toString().padStart(2, '0'))
  }
  return nums
}

const hourOptions = generateNumbers(0, 23)
const minuteOptions = generateNumbers(0, 59)
const secondOptions = generateNumbers(0, 59)

// 转盘偏移量（像素）
const hourOffset = ref(0)
const minuteOffset = ref(-60)  // 默认选中第2个（05分）
const secondOffset = ref(0)

// 拖拽状态
let isDragging = false
let startY = 0
let startOffset = 0
let dragType = null  // 当前拖拽的列

const itemHeight = 60

// 计算当前选中的索引
const hourIndex = computed(() => Math.round(-hourOffset.value / itemHeight))
const minuteIndex = computed(() => Math.round(-minuteOffset.value / itemHeight))
const secondIndex = computed(() => Math.round(-secondOffset.value / itemHeight))

// 同步到时间值
watch(hourIndex, (idx) => {
  if (appState.value !== 'idle') return
  const normalized = ((idx % 24) + 24) % 24
  hours.value = normalized
})

watch(minuteIndex, (idx) => {
  if (appState.value !== 'idle') return
  const normalized = ((idx % 60) + 60) % 60
  minutes.value = normalized
})

watch(secondIndex, (idx) => {
  if (appState.value !== 'idle') return
  const normalized = ((idx % 60) + 60) % 60
  seconds.value = normalized
})

// 初始化偏移量
onMounted(() => {
  minuteOffset.value = -60  // 默认05:00
})

// 鼠标/触摸处理
function handleWheel(e, type) {
  e.preventDefault()
  const delta = e.deltaY > 0 ? itemHeight : -itemHeight

  if (type === 'hour') {
    hourOffset.value += delta
    // 循环
    if (hourOffset.value > 0) hourOffset.value = -23 * itemHeight
    if (hourOffset.value < -23 * itemHeight) hourOffset.value = 0
  } else if (type === 'minute') {
    minuteOffset.value += delta
    if (minuteOffset.value > 0) minuteOffset.value = -59 * itemHeight
    if (minuteOffset.value < -59 * itemHeight) minuteOffset.value = 0
  } else {
    secondOffset.value += delta
    if (secondOffset.value > 0) secondOffset.value = -59 * itemHeight
    if (secondOffset.value < -59 * itemHeight) secondOffset.value = 0
  }
}

function handleMouseDown(e, type) {
  isDragging = true
  dragType = type
  startY = e.clientY

  if (type === 'hour') startOffset = hourOffset.value
  else if (type === 'minute') startOffset = minuteOffset.value
  else startOffset = secondOffset.value
}

function handleMouseMove(e) {
  if (!isDragging || !dragType) return

  const delta = e.clientY - startY

  if (dragType === 'hour') {
    hourOffset.value = startOffset + delta
  } else if (dragType === 'minute') {
    minuteOffset.value = startOffset + delta
  } else {
    secondOffset.value = startOffset + delta
  }
}

function handleMouseUp() {
  if (!isDragging) return
  isDragging = false

  // 只对齐当前拖拽的列
  if (dragType === 'hour') {
    hourOffset.value = Math.round(hourOffset.value / itemHeight) * itemHeight
  } else if (dragType === 'minute') {
    minuteOffset.value = Math.round(minuteOffset.value / itemHeight) * itemHeight
  } else if (dragType === 'second') {
    secondOffset.value = Math.round(secondOffset.value / itemHeight) * itemHeight
  }

  dragType = null
}

// 计算总秒数
const totalSeconds = computed(() => {
  return hours.value * 3600 + minutes.value * 60 + seconds.value
})

// 格式化显示
const formattedTime = computed(() => {
  const h = hours.value.toString().padStart(2, '0')
  const m = minutes.value.toString().padStart(2, '0')
  const s = seconds.value.toString().padStart(2, '0')
  return `${h}:${m}:${s}`
})

// 开始/暂停/继续
function handleAction() {
  if (appState.value === 'idle') {
    startCountdown()
  } else if (appState.value === 'running') {
    pauseTimer()
  } else if (appState.value === 'paused') {
    resumeTimer()
  }
}

function startCountdown() {
  if (totalSeconds.value === 0 && !initialTotalSeconds) return

  // 保存开始时的总秒数（如果是第一次启动）
  if (!initialTotalSeconds) {
    initialTotalSeconds = totalSeconds.value
  }

  appState.value = 'running'

  // 恢复转盘位置到初始设定的时间
  const initialHours = Math.floor(initialTotalSeconds / 3600)
  const initialMinutes = Math.floor((initialTotalSeconds % 3600) / 60)
  const initialSeconds = initialTotalSeconds % 60

  hourOffset.value = -initialHours * itemHeight
  minuteOffset.value = -initialMinutes * itemHeight
  secondOffset.value = -initialSeconds * itemHeight

  remainingSeconds.value = initialTotalSeconds

  timerInterval = setInterval(() => {
    if (isPaused) return

    remainingSeconds.value--
    updateDisplay()

    if (remainingSeconds.value <= 0) {
      timerComplete()
    }
  }, 1000)
}

function pauseTimer() {
  isPaused = true
  appState.value = 'paused'
}

function resumeTimer() {
  isPaused = false
  appState.value = 'running'
}

function stopTimer() {
  if (timerInterval) {
    clearInterval(timerInterval)
    timerInterval = null
  }
  isPaused = false
  appState.value = 'idle'
  remainingSeconds.value = 0
  initialTotalSeconds = 0  // 用户点击停止时重置
}

function timerComplete() {
  // 不调用 stopTimer，避免重置 initialTotalSeconds
  if (timerInterval) {
    clearInterval(timerInterval)
    timerInterval = null
  }
  isPaused = false
  remainingSeconds.value = 0

  // 创建全屏遮罩窗口
  createOverlayWindow()
}

let overlayWindow = null

async function createOverlayWindow() {
  const message = reminderText.value || settings.value.defaultReminder

  // 创建新窗口作为遮罩
  overlayWindow = new WebviewWindow('overlay', {
    url: 'overlay.html',
    title: '',
    width: 1,
    height: 1,
    x: 0,
    y: 0,
    fullscreen: true,
    decorations: false,
    transparent: true,
    alwaysOnTop: true,
    skipTaskbar: true,
    resizable: false,
    focus: true
  })

  // 等待窗口创建完成后发送消息
  overlayWindow.once('tauri://created', async () => {
    // 短暂延迟确保 listen 已注册
    setTimeout(async () => {
      await emit('overlay-message', {
        message: message,
        autoClose: settings.value.autoCloseTime,
        isLoop: isLoop.value
      })
    }, 100)
  })

  // 监听窗口真正关闭（而不是关闭请求）
  overlayWindow.once('tauri://destroyed', () => {
    overlayWindow = null
    if (isLoop.value) {
      startCountdown()
    } else {
      resetUI()
    }
  })
}

// 这个函数不再处理循环，只用于清除引用
async function closeOverlay() {
  if (overlayWindow) {
    try {
      await overlayWindow.close()
    } catch (e) {
      console.error('Close overlay error:', e)
    }
    overlayWindow = null
  }
  // 循环逻辑移到了 tauri://destroyed 中
}

function resetUI() {
  appState.value = 'idle'
}

function updateDisplay() {
  const total = remainingSeconds.value
  hours.value = Math.floor(total / 3600)
  minutes.value = Math.floor((total % 3600) / 60)
  seconds.value = total % 60

  // 如果正在运行，根据当前时间值直接设置转盘位置
  if (appState.value === 'running') {
    hourOffset.value = -hours.value * itemHeight
    minuteOffset.value = -minutes.value * itemHeight
    secondOffset.value = -seconds.value * itemHeight
  }
}

// 设置相关
function openSettings() {
  showSettings.value = true
}

function saveSettings() {
  settings.value.autoCloseTime = Math.min(Math.max(settings.value.autoCloseTime, 1), 60)
  localStorage.setItem('cue-settings', JSON.stringify(settings.value))
  showSettings.value = false
}

function cancelSettings() {
  showSettings.value = false
}

// 加载设置
onMounted(() => {
  const saved = localStorage.getItem('cue-settings')
  if (saved) {
    try {
      settings.value = { ...settings.value, ...JSON.parse(saved) }
    } catch (e) {
      console.error('Failed to load settings:', e)
    }
  }
})

// 按钮文字
const actionBtnText = computed(() => {
  switch (appState.value) {
    case 'idle': return '开始计时'
    case 'running': return '暂停'
    case 'paused': return '继续'
    default: return '开始计时'
  }
})
</script>

<template>
  <div class="app">
    <!-- 背景装饰 -->
    <div class="bg-gradient"></div>
    <div class="bg-grid"></div>

    <!-- 设置按钮 -->
    <button class="settings-btn" @click="openSettings" aria-label="设置">
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
        <circle cx="12" cy="12" r="3"></circle>
        <path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"></path>
      </svg>
    </button>

    <!-- 转盘选择器 -->
    <div class="picker-wrapper">
      <div class="picker-container" @mousemove="handleMouseMove" @mouseup="handleMouseUp" @mouseleave="handleMouseUp">
        <div class="picker-column" :class="{ disabled: appState !== 'idle' }" @wheel="(e) => handleWheel(e, 'hour')" @mousedown="(e) => handleMouseDown(e, 'hour')">
          <div class="picker-track"></div>
          <div class="picker-highlight"></div>
          <div class="picker-wheel" :style="{ transform: `translateY(${hourOffset}px)` }">
            <div
              v-for="(num, i) in [...hourOptions, ...hourOptions, ...hourOptions]"
              :key="`h-${i}`"
              class="picker-item"
              :class="{ active: i % 24 === hourIndex + 24 }"
            >
              {{ num }}
            </div>
          </div>
        </div>
        <span class="separator">:</span>
        <div class="picker-column" :class="{ disabled: appState !== 'idle' }" @wheel="(e) => handleWheel(e, 'minute')" @mousedown="(e) => handleMouseDown(e, 'minute')">
          <div class="picker-track"></div>
          <div class="picker-highlight"></div>
          <div class="picker-wheel" :style="{ transform: `translateY(${minuteOffset}px)` }">
            <div
              v-for="(num, i) in [...minuteOptions, ...minuteOptions, ...minuteOptions]"
              :key="`m-${i}`"
              class="picker-item"
              :class="{ active: i % 60 === minuteIndex + 60 }"
            >
              {{ num }}
            </div>
          </div>
        </div>
        <span class="separator">:</span>
        <div class="picker-column" :class="{ disabled: appState !== 'idle' }" @wheel="(e) => handleWheel(e, 'second')" @mousedown="(e) => handleMouseDown(e, 'second')">
          <div class="picker-track"></div>
          <div class="picker-highlight"></div>
          <div class="picker-wheel" :style="{ transform: `translateY(${secondOffset}px)` }">
            <div
              v-for="(num, i) in [...secondOptions, ...secondOptions, ...secondOptions]"
              :key="`s-${i}`"
              class="picker-item"
              :class="{ active: i % 60 === secondIndex + 60 }"
            >
              {{ num }}
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 提醒内容和循环开关 -->
    <div class="controls-row">
      <div class="input-wrapper">
        <input
          type="text"
          class="custom-input"
          v-model="reminderText"
          placeholder="提醒内容（可选）"
          maxlength="50"
          :disabled="appState !== 'idle'"
        >
      </div>

      <div class="switch-wrapper">
        <label class="switch">
          <input type="checkbox" v-model="isLoop" :disabled="appState !== 'idle'">
          <span class="slider"></span>
        </label>
        <span class="switch-label">循环</span>
      </div>
    </div>

    <!-- 主按钮组 -->
    <div class="button-group">
      <button
        class="cyber-btn primary"
        :class="{ running: appState !== 'idle' }"
        @click="handleAction"
      >
        <span class="btn-text">{{ actionBtnText }}</span>
      </button>

      <button v-if="appState !== 'idle'" class="cyber-btn danger" @click="stopTimer">
        <span class="btn-text">停止</span>
      </button>
    </div>

    <!-- 设置弹窗 -->
    <Transition name="modal">
      <div v-if="showSettings" class="modal-overlay" @click.self="cancelSettings">
        <div class="modal-content">
          <div class="modal-header">
            <h3>设置</h3>
            <button class="modal-close" @click="cancelSettings">
              <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <line x1="18" y1="6" x2="6" y2="18"></line>
                <line x1="6" y1="6" x2="18" y2="18"></line>
              </svg>
            </button>
          </div>
          <div class="setting-item">
            <label>自动关闭时间（秒）</label>
            <div class="input-number">
              <button class="num-btn" @click="settings.autoCloseTime = Math.max(1, settings.autoCloseTime - 1)">−</button>
              <input
                id="auto-close-input"
                type="number"
                v-model.number="settings.autoCloseTime"
                min="1"
                max="60"
              >
              <button class="num-btn" @click="settings.autoCloseTime = Math.min(60, settings.autoCloseTime + 1)">+</button>
            </div>
          </div>
          <div class="modal-actions">
            <button class="primary-btn" @click="saveSettings">保存</button>
            <button class="secondary-btn" @click="cancelSettings">取消</button>
          </div>
        </div>
      </div>
    </Transition>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;600&display=swap');

:root {
  --bg-primary: #0d0d12;
  --bg-secondary: #141419;
  --bg-tertiary: #1c1c24;
  --accent-primary: #4ade80;
  --accent-secondary: #22d3ee;
  --accent-glow: rgba(74, 222, 128, 0.25);
  --text-primary: #e2e8f0;
  --text-secondary: #94a3b8;
  --text-muted: #64748b;
  --border-subtle: rgba(74, 222, 128, 0.1);
  --border-active: #4ade80;
  --cyber-glow: rgba(74, 222, 128, 0.12);
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'JetBrains Mono', monospace;
  background: var(--bg-primary);
  min-height: 100vh;
  color: var(--text-primary);
  overflow: hidden;
}

.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px 20px;
  position: relative;
  gap: 28px;
}

/* 背景效果 */
.bg-gradient {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background:
    radial-gradient(ellipse 100% 80% at 20% 0%, rgba(74, 222, 128, 0.04) 0%, transparent 50%),
    radial-gradient(ellipse 80% 60% at 80% 100%, rgba(34, 211, 238, 0.03) 0%, transparent 50%);
  pointer-events: none;
  z-index: 0;
}

.bg-grid {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image:
    linear-gradient(rgba(74, 222, 128, 0.015) 1px, transparent 1px),
    linear-gradient(90deg, rgba(74, 222, 128, 0.015) 1px, transparent 1px);
  background-size: 30px 30px;
  pointer-events: none;
  z-index: 0;
}

/* 设置按钮 */
.settings-btn {
  position: absolute;
  top: 28px;
  right: 28px;
  background: rgba(20, 20, 25, 0.9);
  border: 1px solid var(--border-subtle);
  color: var(--text-secondary);
  cursor: pointer;
  padding: 10px;
  border-radius: 8px;
  transition: all 0.2s ease;
  z-index: 10;
}

.settings-btn:hover {
  color: var(--accent-primary);
  border-color: var(--accent-primary);
  box-shadow: 0 0 15px var(--cyber-glow);
}

/* 转盘选择器容器 */
.picker-wrapper {
  position: relative;
  z-index: 1;
}

.picker-container {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 22px 28px;
  background: rgba(20, 20, 25, 0.9);
  border: 1px solid var(--border-subtle);
  border-radius: 12px;
  box-shadow:
    0 4px 20px rgba(0, 0, 0, 0.3),
    0 0 0 1px rgba(74, 222, 128, 0.05);
  position: relative;
}

.picker-container::before {
  content: '';
  position: absolute;
  top: -1px;
  left: 15%;
  right: 15%;
  height: 1px;
  background: linear-gradient(90deg, transparent, var(--accent-primary), transparent);
  border-radius: 1px;
}

/* 转盘列 */
.picker-column {
  width: 72px;
  height: 200px;
  overflow: hidden;
  position: relative;
  border-radius: 8px;
  cursor: grab;
  user-select: none;
}

.picker-column.disabled {
  cursor: default;
  opacity: 0.6;
}

.picker-column:active {
  cursor: grabbing;
}

/* 转盘轨道 */
.picker-track {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 100%;
  height: 60px;
  background: linear-gradient(180deg,
    transparent 0%,
    rgba(74, 222, 128, 0.03) 20%,
    rgba(74, 222, 128, 0.08) 50%,
    rgba(74, 222, 128, 0.03) 80%,
    transparent 100%
  );
  border-radius: 6px;
  pointer-events: none;
}

/* 高亮框 */
.picker-highlight {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 100%;
  height: 60px;
  background: linear-gradient(135deg,
    rgba(74, 222, 128, 0.08),
    rgba(34, 211, 238, 0.05)
  );
  border-radius: 8px;
  border: 1px solid rgba(74, 222, 128, 0.2);
  box-shadow:
    0 0 15px var(--cyber-glow),
    inset 0 0 12px rgba(74, 222, 128, 0.03);
  pointer-events: none;
  z-index: 3;
}

.picker-wheel {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-top: 70px;
  transition: transform 0.1s linear;
  position: absolute;
  width: 100%;
  z-index: 2;
}

.picker-item {
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: 'JetBrains Mono', monospace;
  font-size: 28px;
  font-weight: 500;
  color: var(--text-muted);
  user-select: none;
  transition: all 0.15s ease;
}

.picker-item.active {
  color: var(--accent-primary);
  font-weight: 600;
  text-shadow: 0 0 15px var(--accent-glow);
}

.separator {
  font-family: 'JetBrains Mono', monospace;
  font-size: 32px;
  font-weight: 600;
  color: var(--accent-primary);
  align-self: center;
  margin-top: -8px;
  opacity: 1;
  text-shadow: 0 0 12px var(--accent-glow);
}

/* 输入框 */
.input-wrapper {
  width: 100%;
  z-index: 1;
}

.custom-input {
  width: 100%;
  padding: 14px 18px;
  background: rgba(20, 20, 25, 0.9);
  border: 1px solid var(--border-subtle);
  border-radius: 8px;
  color: var(--text-primary);
  font-family: 'JetBrains Mono', monospace;
  font-size: 14px;
  outline: none;
  transition: all 0.2s ease;
}

.custom-input::placeholder {
  color: var(--text-muted);
}

.custom-input:focus {
  border-color: var(--accent-primary);
  box-shadow: 0 0 15px var(--cyber-glow);
}

.custom-input:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* 控制行 - 输入框和开关 */
.controls-row {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 16px;
  width: 100%;
  max-width: 420px;
  z-index: 1;
}

.controls-row .input-wrapper {
  flex: 1;
  min-width: 220px;
}

/* 循环开关 */
.switch-wrapper {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  background: rgba(20, 20, 25, 0.9);
  border: 1px solid var(--border-subtle);
  border-radius: 8px;
}

.switch {
  position: relative;
  display: inline-block;
  width: 44px;
  height: 24px;
  flex-shrink: 0;
}

.switch input {
  opacity: 0;
  width: 0;
  height: 0;
}

.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: var(--bg-tertiary);
  border: 1px solid var(--border-subtle);
  transition: all 0.2s ease;
  border-radius: 24px;
}

.slider:before {
  position: absolute;
  content: "";
  height: 18px;
  width: 18px;
  left: 2px;
  bottom: 2px;
  background: var(--text-muted);
  transition: all 0.2s ease;
  border-radius: 50%;
}

input:checked + .slider {
  background: var(--accent-primary);
  border-color: var(--accent-primary);
  box-shadow: 0 0 12px var(--cyber-glow);
}

input:checked + .slider:before {
  transform: translateX(20px);
  background: white;
}

.switch input:disabled + .slider {
  opacity: 0.4;
  cursor: not-allowed;
}

.switch-label {
  font-size: 13px;
  color: var(--text-secondary);
  font-weight: 500;
  letter-spacing: 0.5px;
}

/* 按钮组 */
.button-group {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 16px;
  z-index: 1;
  width: 100%;
  max-width: 380px;
}

/* 按钮 */
.cyber-btn {
  position: relative;
  flex: 1;
  padding: 14px 28px;
  background: transparent;
  border: none;
  font-family: 'JetBrains Mono', monospace;
  font-size: 14px;
  font-weight: 600;
  letter-spacing: 1px;
  cursor: pointer;
  transition: all 0.2s ease;
  height: 50px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 10px;
}

/* 主按钮 */
.cyber-btn.primary {
  color: var(--bg-primary);
  background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
  box-shadow: 0 4px 15px var(--cyber-glow);
}

.cyber-btn.primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px var(--cyber-glow);
}

.cyber-btn.primary:active {
  transform: translateY(0);
}

.cyber-btn.primary.running {
  background: linear-gradient(135deg, var(--accent-secondary), var(--accent-primary));
}

.cyber-btn.primary.running:hover {
  box-shadow: 0 6px 20px rgba(34, 211, 238, 0.2);
}

/* 危险按钮 */
.cyber-btn.danger {
  color: #f87171;
  background: rgba(20, 20, 25, 0.9);
  border: 2px solid #f87171;
}

.cyber-btn.danger:hover {
  background: #f87171;
  color: var(--bg-primary);
  box-shadow: 0 4px 15px rgba(248, 113, 113, 0.2);
}

/* 设置弹窗 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(13, 13, 18, 0.8);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}

.modal-content {
  background: var(--bg-secondary);
  border: 1px solid var(--border-subtle);
  border-radius: 12px;
  padding: 24px;
  width: 90%;
  max-width: 340px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.4);
  position: relative;
}

.modal-content::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 2px;
  background: linear-gradient(90deg, var(--accent-primary), var(--accent-secondary));
  border-radius: 12px 12px 0 0;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.modal-header h3 {
  color: var(--text-primary);
  font-size: 18px;
  font-weight: 600;
}

.modal-close {
  background: transparent;
  border: none;
  color: var(--text-muted);
  cursor: pointer;
  padding: 4px;
  border-radius: 6px;
  transition: all 0.2s ease;
}

.modal-close:hover {
  color: #f87171;
}

.setting-item {
  margin-bottom: 20px;
}

.setting-item label {
  display: block;
  margin-bottom: 10px;
  color: var(--text-secondary);
  font-size: 13px;
  font-weight: 500;
}

.input-number {
  display: flex;
  align-items: center;
  gap: 8px;
}

.input-number input {
  flex: 1;
  padding: 12px;
  background: var(--bg-tertiary);
  border: 1px solid var(--border-subtle);
  border-radius: 6px;
  color: var(--text-primary);
  font-family: 'JetBrains Mono', monospace;
  font-size: 16px;
  text-align: center;
  outline: none;
}

.input-number input:focus {
  border-color: var(--accent-primary);
  box-shadow: 0 0 12px var(--cyber-glow);
}

.num-btn {
  width: 36px;
  height: 36px;
  background: var(--bg-tertiary);
  border: 1px solid var(--border-subtle);
  border-radius: 6px;
  color: var(--text-secondary);
  font-size: 18px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.num-btn:hover {
  border-color: var(--accent-primary);
  color: var(--accent-primary);
}

.modal-actions {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
}

.primary-btn {
  padding: 12px 24px;
  background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
  color: var(--bg-primary);
  border: none;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
}

.primary-btn:hover {
  box-shadow: 0 4px 15px var(--cyber-glow);
}

.secondary-btn {
  padding: 12px 24px;
  background: transparent;
  color: var(--text-secondary);
  border: 1px solid var(--border-subtle);
  border-radius: 8px;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.secondary-btn:hover {
  border-color: var(--accent-primary);
  color: var(--accent-primary);
}

/* 弹窗动画 */
.modal-enter-active,
.modal-leave-active {
  transition: all 0.3s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-from .modal-content,
.modal-leave-to .modal-content {
  transform: scale(0.95) translateY(20px);
}
</style>