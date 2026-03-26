<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { getCurrentWindow } from '@tauri-apps/api/window'

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
let remainingSeconds = ref(0)

// 生成数字数组
const generateNumbers = (start, end, step = 1) => {
  const nums = []
  for (let i = start; i <= end; i += step) {
    nums.push(i.toString().padStart(2, '0'))
  }
  return nums
}

const hourOptions = generateNumbers(0, 23)
const minuteOptions = generateNumbers(0, 59, 5)
const secondOptions = generateNumbers(0, 59, 5)

// 转盘索引
const hourIndex = ref(0)
const minuteIndex = ref(1)
const secondIndex = ref(0)

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
  if (totalSeconds.value === 0) return

  appState.value = 'running'
  remainingSeconds.value = totalSeconds.value

  timerInterval = setInterval(() => {
    remainingSeconds.value--
    updateDisplay()

    if (remainingSeconds.value <= 0) {
      timerComplete()
    }
  }, 1000)
}

function pauseTimer() {
  appState.value = 'paused'
}

function resumeTimer() {
  appState.value = 'running'
}

function stopTimer() {
  if (timerInterval) {
    clearInterval(timerInterval)
    timerInterval = null
  }
  appState.value = 'idle'
  remainingSeconds.value = 0
}

async function timerComplete() {
  stopTimer()

  // 全屏显示
  const appWindow = getCurrentWindow()
  await appWindow.setFullscreen(true)

  showOverlay.value = true

  // 自动关闭
  setTimeout(() => {
    closeOverlay()
  }, settings.value.autoCloseTime * 1000)
}

async function closeOverlay() {
  showOverlay.value = false

  // 退出全屏
  const appWindow = getCurrentWindow()
  await appWindow.setFullscreen(false)

  if (isLoop.value) {
    startCountdown()
  } else {
    resetUI()
  }
}

function resetUI() {
  appState.value = 'idle'
}

function updateDisplay() {
  const total = remainingSeconds.value
  hours.value = Math.floor(total / 3600)
  minutes.value = Math.floor((total % 3600) / 60)
  seconds.value = total % 60
}

// 设置相关
function openSettings() {
  showSettings.value = true
}

function saveSettings() {
  const autoClose = parseInt(document.getElementById('auto-close-input')?.value) || 10
  settings.value.autoCloseTime = Math.min(Math.max(autoClose, 1), 60)
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
    <!-- 设置按钮 -->
    <button class="settings-btn" @click="openSettings" aria-label="设置">
      <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <circle cx="12" cy="12" r="3"></circle>
        <path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"></path>
      </svg>
    </button>

    <!-- 转盘选择器 -->
    <div class="picker-container">
      <div class="picker-column">
        <div class="picker-wheel" :style="{ transform: `translateY(${hourIndex * 60}px)` }">
          <div
            v-for="(num, i) in [...hourOptions, ...hourOptions, ...hourOptions]"
            :key="`h-${i}`"
            class="picker-item"
            :class="{ active: i % 24 === hourIndex + 24 }"
          >
            {{ num }}
          </div>
        </div>
        <input type="range" min="0" max="23" v-model="hours" class="picker-input" @change="hourIndex = hours">
      </div>
      <span class="separator">:</span>
      <div class="picker-column">
        <div class="picker-wheel" :style="{ transform: `translateY(${minuteIndex * 60}px)` }">
          <div
            v-for="(num, i) in [...minuteOptions, ...minuteOptions, ...minuteOptions]"
            :key="`m-${i}`"
            class="picker-item"
            :class="{ active: i % 12 === minuteIndex + 12 }"
          >
            {{ num }}
          </div>
        </div>
        <input type="range" min="0" max="11" v-model="minutes" class="picker-input" @change="minuteIndex = minutes / 5">
      </div>
      <span class="separator">:</span>
      <div class="picker-column">
        <div class="picker-wheel" :style="{ transform: `translateY(${secondIndex * 60}px)` }">
          <div
            v-for="(num, i) in [...secondOptions, ...secondOptions, ...secondOptions]"
            :key="`s-${i}`"
            class="picker-item"
            :class="{ active: i % 12 === secondIndex + 12 }"
          >
            {{ num }}
          </div>
        </div>
        <input type="range" min="0" max="11" v-model="seconds" class="picker-input" @change="secondIndex = seconds / 5">
      </div>
    </div>

    <!-- 计时显示（运行时） -->
    <div v-if="appState !== 'idle'" class="countdown-display">
      {{ formattedTime }}
    </div>

    <!-- 提醒内容输入 -->
    <div class="input-group">
      <input
        type="text"
        v-model="reminderText"
        placeholder="提醒内容（可选）"
        maxlength="50"
        :disabled="appState !== 'idle'"
      >
    </div>

    <!-- 循环开关 -->
    <div class="switch-group">
      <label class="switch">
        <input type="checkbox" v-model="isLoop" :disabled="appState !== 'idle'">
        <span class="slider"></span>
      </label>
      <span class="switch-label">循环</span>
    </div>

    <!-- 主按钮 -->
    <button
      class="action-btn"
      :class="{ stop: appState !== 'idle' }"
      @click="handleAction"
    >
      {{ actionBtnText }}
    </button>

    <!-- 停止按钮（运行时显示） -->
    <button v-if="appState !== 'idle'" class="stop-btn" @click="stopTimer">
      停止
    </button>

    <!-- 设置弹窗 -->
    <div v-if="showSettings" class="modal-overlay" @click.self="cancelSettings">
      <div class="modal-content">
        <h3>设置</h3>
        <div class="setting-item">
          <label>自动关闭时间（秒）</label>
          <input
            id="auto-close-input"
            type="number"
            :value="settings.autoCloseTime"
            min="1"
            max="60"
          >
        </div>
        <div class="modal-actions">
          <button class="primary-btn" @click="saveSettings">保存</button>
          <button class="secondary-btn" @click="cancelSettings">取消</button>
        </div>
      </div>
    </div>

    <!-- 提醒遮罩 -->
    <div v-if="showOverlay" class="overlay">
      <div class="overlay-content">
        <p>{{ reminderText || settings.defaultReminder }}</p>
      </div>
      <button class="close-btn" @click="closeOverlay">关闭</button>
    </div>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'SF Pro Display', -apple-system, BlinkMacSystemFont, sans-serif;
  background: #1a1a2e;
  min-height: 100vh;
  color: #eee;
  overflow: hidden;
}

.app {
  padding: 20px;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px;
  position: relative;
}

/* 设置按钮 */
.settings-btn {
  position: absolute;
  top: 20px;
  right: 20px;
  background: transparent;
  border: none;
  color: #4a4a6a;
  cursor: pointer;
  padding: 8px;
  border-radius: 8px;
  transition: background 0.2s;
}

.settings-btn:hover {
  background: rgba(255,255,255,0.1);
}

/* 转盘选择器 */
.picker-container {
  background: #16213e;
  border-radius: 16px;
  padding: 30px 20px;
  display: flex;
  gap: 4px;
  margin-top: 40px;
}

.picker-column {
  width: 70px;
  height: 180px;
  overflow: hidden;
  position: relative;
  border-radius: 12px;
}

.picker-wheel {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-top: 60px;
  transition: transform 0.1s ease-out;
  position: absolute;
  width: 100%;
}

.picker-item {
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 26px;
  color: #4a4a6a;
  font-weight: 400;
  user-select: none;
}

.picker-item.active {
  color: #00d4ff;
  font-weight: 500;
  text-shadow: 0 0 20px rgba(0,212,255,0.5);
}

.picker-input {
  position: absolute;
  width: 100%;
  height: 100%;
  opacity: 0;
  cursor: pointer;
  z-index: 10;
}

.separator {
  font-size: 32px;
  color: #4a4a6a;
  align-self: center;
  margin-top: -10px;
}

/* 计时显示 */
.countdown-display {
  font-size: 48px;
  font-weight: 300;
  color: #00d4ff;
  font-variant-numeric: tabular-nums;
  text-shadow: 0 0 30px rgba(0,212,255,0.5);
}

/* 输入框 */
.input-group {
  width: 100%;
  max-width: 300px;
}

.input-group input {
  width: 100%;
  padding: 12px 16px;
  border: 1px solid #4a4a6a;
  border-radius: 8px;
  background: #16213e;
  color: #eee;
  font-size: 16px;
  outline: none;
  transition: border-color 0.2s;
}

.input-group input:focus {
  border-color: #00d4ff;
}

.input-group input::placeholder {
  color: #4a4a6a;
}

.input-group input:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* 循环开关 */
.switch-group {
  display: flex;
  align-items: center;
  gap: 12px;
}

.switch {
  position: relative;
  display: inline-block;
  width: 50px;
  height: 28px;
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
  background-color: #4a4a6a;
  transition: 0.3s;
  border-radius: 28px;
}

.slider:before {
  position: absolute;
  content: "";
  height: 22px;
  width: 22px;
  left: 3px;
  bottom: 3px;
  background-color: #eee;
  transition: 0.3s;
  border-radius: 50%;
}

input:checked + .slider {
  background-color: #00d4ff;
}

input:checked + .slider:before {
  transform: translateX(22px);
}

.switch input:disabled + .slider {
  opacity: 0.5;
  cursor: not-allowed;
}

.switch-label {
  font-size: 16px;
  color: #4a4a6a;
}

/* 按钮 */
.action-btn {
  padding: 14px 48px;
  background: #00d4ff;
  color: #1a1a2e;
  border: none;
  border-radius: 12px;
  font-size: 18px;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.1s, box-shadow 0.2s;
}

.action-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 20px rgba(0,212,255,0.4);
}

.action-btn:active {
  transform: translateY(0);
}

.action-btn.stop {
  background: #4a4a6a;
  color: #eee;
}

.action-btn.stop:hover {
  box-shadow: 0 4px 20px rgba(74,74,106,0.4);
}

.stop-btn {
  padding: 10px 24px;
  background: transparent;
  color: #4a4a6a;
  border: 1px solid #4a4a6a;
  border-radius: 8px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s;
}

.stop-btn:hover {
  border-color: #eee;
  color: #eee;
}

/* 设置弹窗 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}

.modal-content {
  background: #16213e;
  border-radius: 16px;
  padding: 24px;
  width: 90%;
  max-width: 320px;
}

.modal-content h3 {
  margin-bottom: 20px;
  color: #eee;
}

.setting-item {
  margin-bottom: 20px;
}

.setting-item label {
  display: block;
  margin-bottom: 8px;
  color: #4a4a6a;
  font-size: 14px;
}

.setting-item input {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid #4a4a6a;
  border-radius: 8px;
  background: #1a1a2e;
  color: #eee;
  font-size: 16px;
  outline: none;
}

.setting-item input:focus {
  border-color: #00d4ff;
}

.modal-actions {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
}

.primary-btn {
  padding: 14px 32px;
  background: #00d4ff;
  color: #1a1a2e;
  border: none;
  border-radius: 12px;
  font-size: 18px;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.1s;
}

.primary-btn:hover {
  transform: translateY(-2px);
}

.secondary-btn {
  padding: 14px 32px;
  background: transparent;
  color: #4a4a6a;
  border: 1px solid #4a4a6a;
  border-radius: 12px;
  font-size: 18px;
  cursor: pointer;
  transition: border-color 0.2s;
}

.secondary-btn:hover {
  border-color: #eee;
  color: #eee;
}

/* 提醒遮罩 */
.overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.9);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 200;
  animation: fadeIn 0.3s ease-out;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.overlay-content {
  text-align: center;
  margin-bottom: 60px;
}

.overlay-content p {
  font-size: 32px;
  color: #00d4ff;
  text-shadow: 0 0 30px rgba(0,212,255,0.6);
}

.close-btn {
  position: absolute;
  bottom: 40px;
  padding: 14px 48px;
  background: #00d4ff;
  color: #1a1a2e;
  border: none;
  border-radius: 12px;
  font-size: 18px;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.1s;
}

.close-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 20px rgba(0,212,255,0.4);
}
</style>
