<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { WebviewWindow } from '@tauri-apps/api/webviewWindow'
import { emit } from '@tauri-apps/api/event'

// 定时器任务列表
const timers = ref([
  {
    id: 1,
    name: '任务 1',
    workMinutes: 25,
    restMinutes: 5,
    reminderText: '',
    autoCloseTime: 10,
    isLoop: false,
    state: 'idle', // idle, running, paused
    remainingSeconds: 0,
    isBreak: false, // false=工作, true=休息
    timerInterval: null
  }
])

let nextId = 2

// 当前激活的任务ID
const activeTimerId = ref(1)

// 格式化时间显示
function formatTime(seconds) {
  const m = Math.floor(seconds / 60)
  const s = seconds % 60
  return `${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`
}

// 获取当前任务
function getCurrentTimer() {
  return timers.value.find(t => t.id === activeTimerId.value)
}

// 添加新任务
function addTimer() {
  const newTimer = {
    id: nextId++,
    name: `任务 ${nextId - 1}`,
    workMinutes: 25,
    restMinutes: 5,
    reminderText: '',
    autoCloseTime: 10,
    isLoop: false,
    state: 'idle',
    remainingSeconds: 0,
    isBreak: false,
    timerInterval: null
  }
  timers.value.push(newTimer)
  activeTimerId.value = newTimer.id
}

// 删除任务
function deleteTimer(id) {
  const timer = timers.value.find(t => t.id === id)
  if (timer) {
    if (timer.state !== 'idle') {
      stopTimer(id)
    }
    timers.value = timers.value.filter(t => t.id !== id)
    if (activeTimerId.value === id && timers.value.length > 0) {
      activeTimerId.value = timers.value[0].id
    }
  }
}

// 开始/暂停/继续
function handleAction(id) {
  const timer = timers.value.find(t => t.id === id)
  if (!timer) return

  if (timer.state === 'idle') {
    startCountdown(id)
  } else if (timer.state === 'running') {
    pauseTimer(id)
  } else if (timer.state === 'paused') {
    resumeTimer(id)
  }
}

function startCountdown(id) {
  const timer = timers.value.find(t => t.id === id)
  if (!timer) return

  const totalSeconds = timer.isBreak
    ? timer.restMinutes * 60
    : timer.workMinutes * 60

  if (totalSeconds === 0) return

  timer.state = 'running'
  timer.remainingSeconds = totalSeconds

  timer.timerInterval = setInterval(() => {
    if (timer.state === 'paused') return

    timer.remainingSeconds--

    if (timer.remainingSeconds <= 0) {
      timerComplete(id)
    }
  }, 1000)
}

function pauseTimer(id) {
  const timer = timers.value.find(t => t.id === id)
  if (timer) {
    timer.state = 'paused'
  }
}

function resumeTimer(id) {
  const timer = timers.value.find(t => t.id === id)
  if (timer) {
    timer.state = 'running'
  }
}

function stopTimer(id) {
  const timer = timers.value.find(t => t.id === id)
  if (timer) {
    if (timer.timerInterval) {
      clearInterval(timer.timerInterval)
      timer.timerInterval = null
    }
    timer.state = 'idle'
    timer.remainingSeconds = 0
    timer.isBreak = false
  }
}

function timerComplete(id) {
  const timer = timers.value.find(t => t.id === id)
  if (!timer) return

  if (timer.timerInterval) {
    clearInterval(timer.timerInterval)
    timer.timerInterval = null
  }

  timer.remainingSeconds = 0
  createOverlayWindow(id)
}

let overlayWindow = null

async function createOverlayWindow(timerId) {
  const timer = timers.value.find(t => t.id === timerId)
  if (!timer) return

  const message = timer.reminderText || '时间到了！'
  const isLoop = timer.isLoop
  const currentIsBreak = timer.isBreak

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

  overlayWindow.once('tauri://created', async () => {
    setTimeout(async () => {
      await emit('overlay-message', {
        message: message,
        autoClose: timer.autoCloseTime,
        isLoop: isLoop
      })
    }, 100)
  })

  overlayWindow.once('tauri://destroyed', () => {
    overlayWindow = null
    if (isLoop) {
      timer.isBreak = !currentIsBreak
      startCountdown(timerId)
    } else {
      timer.state = 'idle'
    }
  })
}

// 清理
onUnmounted(() => {
  timers.value.forEach(timer => {
    if (timer.timerInterval) {
      clearInterval(timer.timerInterval)
    }
  })
})

// 按钮文字
function getActionBtnText(state) {
  switch (state) {
    case 'idle': return '开始'
    case 'running': return '暂停'
    case 'paused': return '继续'
    default: return '开始'
  }
}
</script>

<template>
  <div class="app">
    <!-- 背景装饰 -->
    <div class="bg-gradient"></div>
    <div class="bg-grid"></div>

    <!-- 左侧任务列表 -->
    <div class="sidebar">
      <div class="sidebar-header">任务列表</div>
      <div class="task-list">
        <div
          v-for="timer in timers"
          :key="timer.id"
          class="task-item"
          :class="{ active: timer.id === activeTimerId, running: timer.state === 'running' }"
          @click="activeTimerId = timer.id"
        >
          <div class="task-status" :class="timer.state"></div>
          <input
            v-if="timer.state === 'idle' && timer.id === activeTimerId"
            type="text"
            class="task-name-input"
            v-model="timer.name"
            maxlength="20"
            @click.stop
          >
          <span v-else class="task-name">{{ timer.name }}</span>
          <button
            v-if="timers.length > 1 && timer.state === 'idle'"
            class="delete-btn"
            @click.stop="deleteTimer(timer.id)"
          >
            <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="18" y1="6" x2="6" y2="18"></line>
              <line x1="6" y1="6" x2="18" y2="18"></line>
            </svg>
          </button>
        </div>
      </div>
      <button class="add-task-btn" @click="addTimer">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <line x1="12" y1="5" x2="12" y2="19"></line>
          <line x1="5" y1="12" x2="19" y2="12"></line>
        </svg>
        添加任务
      </button>
    </div>

    <!-- 右侧详情区 -->
    <div class="detail-panel">
      <template v-if="getCurrentTimer()">
        <!-- 时间显示 -->
        <div class="timer-display">
          <span class="time-value">{{ formatTime(getCurrentTimer().remainingSeconds) }}</span>
          <span class="time-label">{{ getCurrentTimer().isBreak ? '休息' : '工作' }}</span>
        </div>

        <!-- 设置区 -->
        <div v-if="getCurrentTimer().state === 'idle'" class="settings-area">
          <div class="input-row">
            <div class="input-group">
              <label>工作(分钟)</label>
              <input
                type="number"
                v-model.number="getCurrentTimer().workMinutes"
                min="1"
                max="999"
              >
            </div>
            <div class="input-group">
              <label>休息(分钟)</label>
              <input
                type="number"
                v-model.number="getCurrentTimer().restMinutes"
                min="0"
                max="999"
              >
            </div>
          </div>

          <div class="input-row">
            <div class="input-group">
              <label>提醒内容</label>
              <input
                type="text"
                v-model="getCurrentTimer().reminderText"
                placeholder="时间到了！"
                maxlength="50"
              >
            </div>
          </div>

          <div class="input-row">
            <div class="input-group small">
              <label>自动关闭(秒)</label>
              <input
                type="number"
                v-model.number="getCurrentTimer().autoCloseTime"
                min="1"
                max="60"
              >
            </div>
            <div class="loop-switch">
              <input type="checkbox" id="loop-check" v-model="getCurrentTimer().isLoop">
              <label for="loop-check" class="slider"></label>
              <span class="switch-label">循环</span>
            </div>
          </div>
        </div>

        <!-- 操作按钮 -->
        <div class="timer-actions">
          <button
            class="action-btn primary"
            @click="handleAction(getCurrentTimer().id)"
          >
            {{ getActionBtnText(getCurrentTimer().state) }}
          </button>
          <button
            v-if="getCurrentTimer().state !== 'idle'"
            class="action-btn danger"
            @click="stopTimer(getCurrentTimer().id)"
          >
            停止
          </button>
        </div>
      </template>
    </div>
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
  overflow: auto;
}

.app {
  min-height: 100vh;
  display: flex;
  position: relative;
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

/* 左侧任务列表 */
.sidebar {
  width: 240px;
  min-height: 100vh;
  background: rgba(20, 20, 25, 0.95);
  border-right: 1px solid var(--border-subtle);
  padding: 24px 16px;
  display: flex;
  flex-direction: column;
  z-index: 1;
}

.sidebar-header {
  font-size: 12px;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 2px;
  margin-bottom: 20px;
  padding-left: 8px;
}

.task-list {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.task-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 14px;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.task-item:hover {
  background: rgba(74, 222, 128, 0.05);
}

.task-item.active {
  background: rgba(74, 222, 128, 0.1);
  border: 1px solid var(--border-subtle);
}

.task-item.running .task-status {
  background: var(--accent-secondary);
  box-shadow: 0 0 8px var(--accent-secondary);
}

.task-status {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--text-muted);
  flex-shrink: 0;
}

.task-status.idle {
  background: var(--text-muted);
}

.task-status.running {
  background: var(--accent-primary);
  box-shadow: 0 0 8px var(--accent-primary);
}

.task-status.paused {
  background: #fbbf24;
  box-shadow: 0 0 8px #fbbf24;
}

.task-name {
  flex: 1;
  font-size: 14px;
  color: var(--text-primary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.task-name-input {
  flex: 1;
  background: transparent;
  border: none;
  color: var(--text-primary);
  font-family: 'JetBrains Mono', monospace;
  font-size: 14px;
  outline: none;
  border-bottom: 1px solid var(--border-subtle);
  padding: 2px 0;
}

.task-name-input:focus {
  border-color: var(--accent-primary);
}

.delete-btn {
  background: transparent;
  border: none;
  color: var(--text-muted);
  cursor: pointer;
  padding: 4px;
  border-radius: 4px;
  opacity: 0;
  transition: all 0.2s ease;
}

.task-item:hover .delete-btn {
  opacity: 1;
}

.delete-btn:hover {
  color: #f87171;
  background: rgba(248, 113, 113, 0.1);
}

.add-task-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 12px;
  margin-top: 16px;
  background: transparent;
  border: 2px dashed var(--border-subtle);
  border-radius: 8px;
  color: var(--text-secondary);
  font-family: 'JetBrains Mono', monospace;
  font-size: 13px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.add-task-btn:hover {
  border-color: var(--accent-primary);
  color: var(--accent-primary);
}

/* 右侧详情区 */
.detail-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px;
  z-index: 1;
}

/* 时间显示 */
.timer-display {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  margin-bottom: 40px;
}

.time-value {
  font-size: 72px;
  font-weight: 600;
  color: var(--accent-primary);
  text-shadow: 0 0 30px var(--accent-glow);
  font-variant-numeric: tabular-nums;
}

.time-label {
  font-size: 14px;
  color: var(--text-secondary);
  text-transform: uppercase;
  letter-spacing: 3px;
}

/* 设置区 */
.settings-area {
  width: 100%;
  max-width: 400px;
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin-bottom: 40px;
}

.input-row {
  display: flex;
  gap: 16px;
}

.input-group {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.input-group.small {
  flex: 0 0 120px;
}

.input-group label {
  font-size: 12px;
  color: var(--text-secondary);
}

.input-group input {
  padding: 14px 16px;
  background: var(--bg-tertiary);
  border: 1px solid var(--border-subtle);
  border-radius: 8px;
  color: var(--text-primary);
  font-family: 'JetBrains Mono', monospace;
  font-size: 15px;
  outline: none;
  transition: all 0.2s ease;
}

.input-group input:focus {
  border-color: var(--accent-primary);
  box-shadow: 0 0 12px var(--cyber-glow);
}

/* 循环开关 */
.loop-switch {
  display: flex;
  align-items: center;
  gap: 12px;
  padding-top: 8px;
}

.loop-switch input {
  display: none;
}

.loop-switch .slider {
  width: 44px;
  height: 24px;
  background: var(--bg-tertiary);
  border: 1px solid var(--border-subtle);
  border-radius: 24px;
  position: relative;
  cursor: pointer;
  transition: all 0.2s ease;
}

.loop-switch .slider::before {
  content: '';
  position: absolute;
  width: 18px;
  height: 18px;
  left: 2px;
  top: 2px;
  background: var(--text-muted);
  border-radius: 50%;
  transition: all 0.2s ease;
}

.loop-switch input:checked + .slider {
  background: var(--accent-primary);
  border-color: var(--accent-primary);
}

.loop-switch input:checked + .slider::before {
  transform: translateX(20px);
  background: white;
}

.switch-label {
  font-size: 13px;
  color: var(--text-secondary);
}

/* 操作按钮 */
.timer-actions {
  display: flex;
  gap: 16px;
}

.action-btn {
  padding: 16px 40px;
  background: transparent;
  border: none;
  font-family: 'JetBrains Mono', monospace;
  font-size: 14px;
  font-weight: 600;
  letter-spacing: 1px;
  cursor: pointer;
  transition: all 0.2s ease;
  border-radius: 10px;
}

.action-btn.primary {
  background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
  color: var(--bg-primary);
}

.action-btn.primary:hover {
  box-shadow: 0 4px 20px var(--cyber-glow);
}

.action-btn.danger {
  background: transparent;
  border: 2px solid #f87171;
  color: #f87171;
}

.action-btn.danger:hover {
  background: #f87171;
  color: var(--bg-primary);
}
</style>
