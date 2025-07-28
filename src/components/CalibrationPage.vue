<template>
  <div class="calibration-container" @click="handleClick">
    <!-- 标题和倒计时 -->
    <div class="calibration-header">
      <h1>音频延迟校准</h1>
      <div class="timer">{{ Math.ceil(remainingTime / 1000) }}s</div>
    </div>

    <!-- 设置图标 - 右上角 -->
    <div class="settings-icon" @click.stop="openManualSettings" title="手动设置延迟">
      ⚙️
    </div>

    <!-- 校准说明 -->
    <div class="calibration-info">
      <p>按键击打节拍，可随时加入。</p>
      <div class="delay-display">{{ Math.round(averageDelay) }}ms</div>
    </div>

    <!-- 游戏区域 -->
    <div class="game-area">
      <!-- 圆环轨道 -->
      <div class="track-ring"></div>
      
      <!-- 水平虚线（校准线） -->
      <div 
        class="calibration-line" 
        :style="{ transform: `rotate(${calibrationAngle}deg)` }"
      ></div>
      
      <!-- 红色小球 -->
      <div 
        class="ball red-ball" 
        :style="{
          left: redBall.x + 'px',
          top: redBall.y + 'px'
        }"
      ></div>
      
      <!-- 蓝色小球 -->
      <div 
        class="ball blue-ball" 
        :style="{
          left: blueBall.x + 'px',
          top: blueBall.y + 'px'
        }"
      ></div>
      
      <!-- 叉叉标记 -->
      <div 
        v-for="mark in clickMarks"
        :key="mark.id"
        class="click-mark"
        :style="{
          left: mark.x + 'px',
          top: mark.y + 'px'
        }"
      >
        <div class="cross-line cross-line-1"></div>
        <div class="cross-line cross-line-2"></div>
      </div>
      
      <!-- 点击提示圆圈 -->
      <div 
        v-if="showClickIndicator"
        class="click-indicator"
        :style="{
          left: lastClickX + 'px',
          top: lastClickY + 'px'
        }"
      ></div>
    </div>

    <!-- 控制按钮 -->
    <div class="calibration-actions">
      <button @click.stop="resetCalibration" class="action-btn reset-btn">
        重置
      </button>
      <button @click.stop="saveAndExit" class="action-btn save-btn">
        保存并退出
      </button>
      <button @click.stop="emit('close')" class="action-btn close-btn">
        关闭
      </button>
    </div>

    <!-- 校准统计 -->
    <div class="calibration-stats">
      <div class="stat-item">
        <span class="stat-label">点击次数:</span>
        <span class="stat-value">{{ clickCount }}</span>
      </div>
      <div class="stat-item">
        <span class="stat-label">平均延迟:</span>
        <span class="stat-value">{{ Math.round(averageDelay) }}ms</span>
      </div>
    </div>

    <!-- 手动设置延迟弹窗 -->
    <div v-if="showManualSettings" class="manual-settings-overlay" @click="closeManualSettings">
      <div class="manual-settings-modal" @click.stop>
        <div class="modal-header">
          <h3>手动设置延迟</h3>
          <button class="close-btn" @click="closeManualSettings">×</button>
        </div>
        <div class="modal-content">
          <div class="input-group">
            <label for="delay-input">延迟时间 (ms):</label>
            <input 
              id="delay-input"
              type="number" 
              v-model.number="manualDelay" 
              min="0" 
              max="1000" 
              step="1"
              placeholder="请输入延迟时间"
            />
          </div>
          <div class="modal-actions">
            <button @click="applyManualDelay" class="apply-btn">应用</button>
            <button @click="closeManualSettings" class="cancel-btn">取消</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, defineEmits } from 'vue'

const emit = defineEmits(['close'])

// 校准状态
const isCalibrating = ref(true)
const remainingTime = ref(15000) // 15秒
const clickCount = ref(0)
const clickTimes = ref<number[]>([])
const clickAngles = ref<number[]>([])
const averageDelay = ref(0)
const calibrationAngle = ref(0)

// 手动设置延迟
const showManualSettings = ref(false)
const manualDelay = ref(0)

// 小球状态
interface Ball {
  x: number
  y: number
  angle: number
  speed: number
}

const redBall = ref<Ball>({
  x: 300,
  y: 200,
  angle: 0,
  speed: 3
})

const blueBall = ref<Ball>({
  x: 500,
  y: 200,
  angle: 180,
  speed: 3
})

// 点击指示器
const showClickIndicator = ref(false)
const lastClickX = ref(0)
const lastClickY = ref(0)

// 落点标记
interface ClickMark {
  x: number
  y: number
  id: number
}
const clickMarks = ref<ClickMark[]>([])
let markIdCounter = 0

// 游戏区域中心
const centerX = 400
const centerY = 300
const radius = 100

// 定时器
let gameLoop: number | null = null
let calibrationTimer: number | null = null

// 计算小球位置
const updateBallPosition = (ball: Ball) => {
  ball.angle += ball.speed
  if (ball.angle >= 360) ball.angle -= 360
  
  const radian = (ball.angle * Math.PI) / 180
  ball.x = centerX + Math.cos(radian) * radius - 15 // 减去小球半径
  ball.y = centerY + Math.sin(radian) * radius - 15
}

// 计算理论点击时间
const calculateTheoreticalTime = (clickX: number, clickY: number): number => {
  // 计算点击位置相对于中心的角度
  const dx = clickX - centerX
  const dy = clickY - centerY
  let clickAngle = Math.atan2(dy, dx) * 180 / Math.PI
  if (clickAngle < 0) clickAngle += 360
  
  // 找到最近的小球
  const redAngleDiff = Math.abs(redBall.value.angle - clickAngle)
  const blueAngleDiff = Math.abs(blueBall.value.angle - clickAngle)
  
  const nearestBall = redAngleDiff < blueAngleDiff ? redBall.value : blueBall.value
  
  // 计算小球到达点击位置需要的时间
  let angleDiff = clickAngle - nearestBall.angle
  if (angleDiff < 0) angleDiff += 360
  if (angleDiff > 180) angleDiff = 360 - angleDiff
  
  return (angleDiff / nearestBall.speed) * 16.67 // 假设60fps，每帧16.67ms
}

// 处理点击事件
const handleClick = (event: MouseEvent) => {
  if (!isCalibrating.value) return
  
  const rect = (event.currentTarget as HTMLElement).getBoundingClientRect()
  const clickX = event.clientX - rect.left
  const clickY = event.clientY - rect.top
  
  // 找到最近的小球
  const redDistance = Math.sqrt((redBall.value.x + 15 - clickX) ** 2 + (redBall.value.y + 15 - clickY) ** 2)
  const blueDistance = Math.sqrt((blueBall.value.x + 15 - clickX) ** 2 + (blueBall.value.y + 15 - clickY) ** 2)
  
  const nearestBall = redDistance < blueDistance ? redBall.value : blueBall.value
  
  // 在小球当前位置添加叉叉标记
  clickMarks.value.push({
    x: nearestBall.x,
    y: nearestBall.y,
    id: markIdCounter++
  })
  
  // 显示点击指示器
  lastClickX.value = clickX - 10
  lastClickY.value = clickY - 10
  showClickIndicator.value = true
  setTimeout(() => {
    showClickIndicator.value = false
  }, 300)
  
  const currentTime = Date.now()
  clickCount.value++
  
  // 使用小球位置计算角度
  const ballAngle = nearestBall.angle
  
  if (clickCount.value === 1) {
    // 第一次点击：不记录延迟，调整虚线到落点角度
    calibrationAngle.value = ballAngle
    clickAngles.value.push(ballAngle)
  } else {
    // 第二次及后续点击：记录延迟时间
    const theoreticalTime = calculateTheoreticalTime(clickX, clickY)
    const actualTime = currentTime
    const delay = actualTime - (currentTime - theoreticalTime)
    
    clickTimes.value.push(delay)
    clickAngles.value.push(ballAngle)
    
    // 计算平均延迟
    averageDelay.value = clickTimes.value.reduce((sum, time) => sum + time, 0) / clickTimes.value.length
    
    // 更新校准线角度为所有点击的平均角度
    calibrationAngle.value = clickAngles.value.reduce((sum, angle) => sum + angle, 0) / clickAngles.value.length
  }
}

// 重置校准
const resetCalibration = () => {
  clickCount.value = 0
  clickTimes.value = []
  clickAngles.value = []
  averageDelay.value = 0
  calibrationAngle.value = 0
  remainingTime.value = 15000
  isCalibrating.value = true
  clickMarks.value = []
  markIdCounter = 0
  
  // 重置小球位置
  redBall.value.angle = 0
  blueBall.value.angle = 180
  
  startCalibration()
}

// 保存并退出
const saveAndExit = () => {
  // 触发自定义事件，传递校准结果
  const calibrationData = {
    averageDelay: averageDelay.value,
    clickCount: clickCount.value,
    calibrationAngle: calibrationAngle.value
  }
  
  // 保存到localStorage
  localStorage.setItem('audioCalibration', JSON.stringify(calibrationData))
  
  emit('close')
}

// 手动设置延迟相关函数
const openManualSettings = () => {
  manualDelay.value = Math.round(averageDelay.value) || 0
  showManualSettings.value = true
}

const closeManualSettings = () => {
  showManualSettings.value = false
}

const applyManualDelay = () => {
  if (manualDelay.value >= 0 && manualDelay.value <= 1000) {
    averageDelay.value = manualDelay.value
    
    // 保存手动设置的延迟到localStorage
    const calibrationData = {
      averageDelay: averageDelay.value,
      clickCount: clickCount.value,
      calibrationAngle: calibrationAngle.value,
      isManuallySet: true
    }
    localStorage.setItem('audioCalibration', JSON.stringify(calibrationData))
    
    closeManualSettings()
  }
}

// 开始校准
const startCalibration = () => {
  if (gameLoop) cancelAnimationFrame(gameLoop)
  if (calibrationTimer) clearInterval(calibrationTimer)
  
  // 游戏循环
  const animate = () => {
    if (!isCalibrating.value) return
    
    updateBallPosition(redBall.value)
    updateBallPosition(blueBall.value)
    
    gameLoop = requestAnimationFrame(animate)
  }
  
  animate()
  
  // 倒计时
  calibrationTimer = setInterval(() => {
    remainingTime.value -= 100
    if (remainingTime.value <= 0) {
      isCalibrating.value = false
      if (gameLoop) cancelAnimationFrame(gameLoop)
      if (calibrationTimer) clearInterval(calibrationTimer)
    }
  }, 100)
}

onMounted(() => {
  startCalibration()
})

onUnmounted(() => {
  if (gameLoop) cancelAnimationFrame(gameLoop)
  if (calibrationTimer) clearInterval(calibrationTimer)
})
</script>

<style scoped>
.calibration-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: linear-gradient(135deg, #e8d5ff 0%, #d4b5ff 100%);
  color: #333;
  font-family: 'Arial', sans-serif;
  z-index: 1000;
  overflow: hidden;
  cursor: crosshair;
}

.calibration-header {
  position: absolute;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
}

.calibration-header h1 {
  margin: 0;
  font-size: 24px;
  color: #6b46c1;
}

.timer {
  font-size: 32px;
  font-weight: bold;
  color: #7c3aed;
  margin-top: 10px;
}

.settings-icon {
  position: fixed;
  top: 20px;
  right: 20px;
  font-size: 28px;
  cursor: pointer;
  padding: 8px;
  transition: all 0.3s ease;
  user-select: none;
  z-index: 1500;
  color: rgba(255, 255, 255, 0.7);
}

.settings-icon:hover {
  color: rgba(255, 255, 255, 1);
  transform: rotate(90deg);
}

.calibration-info {
  position: absolute;
  top: 120px;
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
}

.calibration-info p {
  margin: 0;
  font-size: 16px;
  color: #6b46c1;
}

.delay-display {
  font-size: 28px;
  font-weight: bold;
  color: #7c3aed;
  margin-top: 10px;
}

.game-area {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 800px;
  height: 600px;
}

.track-ring {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 200px;
  height: 200px;
  border: 3px dashed #7c3aed;
  border-radius: 50%;
  opacity: 0.6;
}

.calibration-line {
  position: absolute;
  top: 50%;
  left: 0;
  width: 100%;
  height: 2px;
  background: repeating-linear-gradient(
    to right,
    #7c3aed 0px,
    #7c3aed 10px,
    transparent 10px,
    transparent 20px
  );
  transform-origin: center;
  transition: transform 0.3s ease;
}

.ball {
  position: absolute;
  width: 30px;
  height: 30px;
  border-radius: 50%;
  transition: all 0.1s ease;
}

.red-ball {
  background: radial-gradient(circle at 30% 30%, #ff6b6b, #e74c3c);
  box-shadow: 0 4px 8px rgba(231, 76, 60, 0.3);
}

.blue-ball {
  background: radial-gradient(circle at 30% 30%, #4dabf7, #3498db);
  box-shadow: 0 4px 8px rgba(52, 152, 219, 0.3);
}

.click-indicator {
  position: absolute;
  width: 20px;
  height: 20px;
  border: 3px solid #7c3aed;
  border-radius: 50%;
  background: rgba(124, 58, 237, 0.2);
  animation: clickPulse 0.3s ease-out;
}

.click-mark {
  position: absolute;
  width: 30px;
  height: 30px;
  pointer-events: none;
  animation: markAppear 0.5s ease-out;
}

.cross-line {
  position: absolute;
  width: 16px;
  height: 2px;
  background: #4a4a4a;
  top: 50%;
  left: 50%;
  transform-origin: center;
}

.cross-line-1 {
  transform: translate(-50%, -50%) rotate(45deg);
}

.cross-line-2 {
  transform: translate(-50%, -50%) rotate(-45deg);
}

@keyframes markAppear {
  0% {
    opacity: 0;
    transform: scale(2);
  }
  100% {
    opacity: 1;
    transform: scale(1);
  }
}

@keyframes clickPulse {
  0% {
    transform: scale(0.5);
    opacity: 1;
  }
  100% {
    transform: scale(2);
    opacity: 0;
  }
}

.calibration-actions {
  position: absolute;
  bottom: 80px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 15px;
}

.action-btn {
  padding: 12px 24px;
  border: none;
  border-radius: 25px;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.reset-btn {
  background: linear-gradient(45deg, #ff9a9e, #fecfef);
  color: #333;
}

.save-btn {
  background: linear-gradient(45deg, #a8edea, #fed6e3);
  color: #333;
}

.close-btn {
  background: linear-gradient(45deg, #d299c2, #fef9d7);
  color: #333;
}

.action-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.15);
}

.calibration-stats {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 30px;
}

.stat-item {
  text-align: center;
}

.stat-label {
  display: block;
  font-size: 14px;
  color: #6b46c1;
  margin-bottom: 5px;
}

.stat-value {
  display: block;
  font-size: 18px;
  font-weight: bold;
  color: #7c3aed;
}

/* 手动设置弹窗样式 */
.manual-settings-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
}

.manual-settings-modal {
  background: white;
  border-radius: 15px;
  padding: 0;
  min-width: 400px;
  max-width: 500px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  animation: modalAppear 0.3s ease-out;
}

@keyframes modalAppear {
  0% {
    opacity: 0;
    transform: scale(0.8);
  }
  100% {
    opacity: 1;
    transform: scale(1);
  }
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 25px;
  border-bottom: 1px solid #e5e7eb;
  background: linear-gradient(45deg, #f3e8ff, #e0e7ff);
  border-radius: 15px 15px 0 0;
}

.modal-header h3 {
  margin: 0;
  color: #6b46c1;
  font-size: 18px;
}

.modal-header .close-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #6b46c1;
  padding: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  transition: background 0.2s ease;
}

.modal-header .close-btn:hover {
  background: rgba(107, 70, 193, 0.1);
}

.modal-content {
  padding: 25px;
}

.input-group {
  margin-bottom: 20px;
}

.input-group label {
  display: block;
  margin-bottom: 8px;
  color: #374151;
  font-weight: 500;
}

.input-group input {
  width: 100%;
  padding: 12px 15px;
  border: 2px solid #e5e7eb;
  border-radius: 8px;
  font-size: 16px;
  transition: border-color 0.2s ease;
  box-sizing: border-box;
}

.input-group input:focus {
  outline: none;
  border-color: #7c3aed;
}

.modal-actions {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.modal-actions button {
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.apply-btn {
  background: linear-gradient(45deg, #7c3aed, #a855f7);
  color: white;
}

.apply-btn:hover {
  background: linear-gradient(45deg, #6d28d9, #9333ea);
  transform: translateY(-1px);
}

.cancel-btn {
  background: #f3f4f6;
  color: #374151;
}

.cancel-btn:hover {
  background: #e5e7eb;
  transform: translateY(-1px);
}
</style>