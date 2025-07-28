<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed, watch } from 'vue'
import CalibrationPage from './components/CalibrationPage.vue'


// 游戏状态
const gameStarted = ref(false)
const score = ref(0)
const combo = ref(0)
const gameOver = ref(false)
const countdown = ref(0)
const isCountingDown = ref(false)
const clickAttempts = ref(0)
const maxClickAttempts = 2
const failureMessage = ref('')
const showCalibration = ref(false)

// 失败动画状态
const isFailureAnimating = ref(false)
const failedTileIndex = ref(-1)
const blinkCount = ref(0)
const isBlinking = ref(false)

// 一圈内无操作检测
const lastActionTime = ref(0)
const orbitDuration = ref(0) // 一圈的时间（毫秒）

// 粒子系统
interface Particle {
  id: number
  x: number
  y: number
  vx: number
  vy: number
  life: number
  maxLife: number
  size: number
  color: string
}

const particles = ref<Particle[]>([])
const isParticleAnimating = ref(false)

// 颜色选择
const firePlanetColor = ref(localStorage.getItem('firePlanetColor') || 'red')
const icePlanetColor = ref(localStorage.getItem('icePlanetColor') || 'blue')
const showColorModal = ref(false)
const currentEditingPlanet = ref<'fire' | 'ice' | ''>('') // 'fire' or 'ice'

// 颜色选项
const colorOptions = {
  fire: [
    { name: 'red', gradient: 'linear-gradient(45deg, #ff9a9e, #ff6b6b, #ff4757)', shadow: 'rgba(255,107,107,0.8)' },
    { name: 'orange', gradient: 'linear-gradient(45deg, #ffa726, #ff9800, #f57c00)', shadow: 'rgba(255,152,0,0.8)' },
    { name: 'purple', gradient: 'linear-gradient(45deg, #ba68c8, #9c27b0, #7b1fa2)', shadow: 'rgba(156,39,176,0.8)' },
    { name: 'pink', gradient: 'linear-gradient(45deg, #f48fb1, #e91e63, #c2185b)', shadow: 'rgba(233,30,99,0.8)' },
    { name: 'yellow', gradient: 'linear-gradient(45deg, #fff176, #ffeb3b, #fbc02d)', shadow: 'rgba(255,235,59,0.8)' },
    { name: 'crimson', gradient: 'linear-gradient(45deg, #ef5350, #f44336, #d32f2f)', shadow: 'rgba(244,67,54,0.8)' }
  ],
  ice: [
    { name: 'blue', gradient: 'linear-gradient(45deg, #a8edea, #4ecdc4, #45b7aa)', shadow: 'rgba(78,205,196,0.8)' },
    { name: 'cyan', gradient: 'linear-gradient(45deg, #81d4fa, #29b6f6, #0288d1)', shadow: 'rgba(41,182,246,0.8)' },
    { name: 'green', gradient: 'linear-gradient(45deg, #a5d6a7, #66bb6a, #43a047)', shadow: 'rgba(102,187,106,0.8)' },
    { name: 'teal', gradient: 'linear-gradient(45deg, #80cbc4, #26a69a, #00695c)', shadow: 'rgba(38,166,154,0.8)' },
    { name: 'indigo', gradient: 'linear-gradient(45deg, #9fa8da, #5c6bc0, #3949ab)', shadow: 'rgba(92,107,192,0.8)' },
    { name: 'mint', gradient: 'linear-gradient(45deg, #b2dfdb, #4db6ac, #00695c)', shadow: 'rgba(77,182,172,0.8)' }
  ]
}

// 监听颜色变化并保存到localStorage
watch(firePlanetColor, (newColor) => {
  localStorage.setItem('firePlanetColor', newColor)
})

watch(icePlanetColor, (newColor) => {
  localStorage.setItem('icePlanetColor', newColor)
})

// 打开颜色选择弹框
const openColorModal = (planetType: 'fire' | 'ice') => {
  currentEditingPlanet.value = planetType
  showColorModal.value = true
}

// 选择颜色
const selectColor = (colorName: string) => {
  if (currentEditingPlanet.value === 'fire') {
    firePlanetColor.value = colorName
  } else {
    icePlanetColor.value = colorName
  }
  showColorModal.value = false
}

// 关闭弹框
const closeColorModal = () => {
  showColorModal.value = false
  currentEditingPlanet.value = ''
}

// 校准延迟设置
const calibrationDelay = ref(0)
const loadCalibrationData = () => {
  const savedData = localStorage.getItem('audioCalibration')
  if (savedData) {
    try {
      const data = JSON.parse(savedData)
      calibrationDelay.value = data.averageDelay || 0
      console.log('已加载校准延迟:', calibrationDelay.value, 'ms')
    } catch (error) {
      console.error('加载校准数据失败:', error)
    }
  }
}


// 错误提示状态
interface ErrorIndicator {
  id: number
  x: number
  y: number
  visible: boolean
}

const errorIndicators = ref<ErrorIndicator[]>([])

// 轨道数据
interface TrackTile {
  id: number
  x: number
  y: number
  angle: number
  isActive: boolean
}

// 星球状态
interface Planet {
  x: number
  y: number
  isOrbiting: boolean
}

const trackTiles = ref<TrackTile[]>([])
const currentTileIndex = ref(0)
const planets = ref<Planet[]>([
  { x: 300, y: 300, isOrbiting: false }, // 火星球 (红色)
  { x: 350, y: 300, isOrbiting: true }   // 冰星球 (蓝色)
])

const orbitRadius = 60
const orbitAngle = ref(0)
const orbitSpeed = 0.05
let gameLoop: number | null = null
// let lastTapTime = 0
// const perfectTiming = 100 // 完美时机的毫秒范围

// 摄像机跟随
const cameraOffset = ref({ x: 0, y: 0 })
const cameraSmooth = 0.1

// 生成轨道
const generateTrack = () => {
  const tiles: TrackTile[] = []
  let x = 300
  let y = 300
  let angle = 0
  
  // 生成一条简单的轨道路径
  for (let i = 0; i < 20; i++) {
    tiles.push({
      id: i,
      x,
      y,
      angle,
      isActive: i === 0
    })
    
    // 随机改变方向
    if (i > 0 && Math.random() > 0.7) {
      angle += (Math.random() - 0.5) * Math.PI / 2
    }
    
    x += Math.cos(angle) * 50
    y += Math.sin(angle) * 50
  }
  
  trackTiles.value = tiles
}

// 计算环绕星球位置
const getOrbitingPlanet = computed(() => {
  const centerPlanet = planets.value.find(p => !p.isOrbiting)
  if (!centerPlanet) return { x: 0, y: 0 }
  
  return {
    x: centerPlanet.x + Math.cos(orbitAngle.value) * orbitRadius,
    y: centerPlanet.y + Math.sin(orbitAngle.value) * orbitRadius
  }
})

// 游戏控制
const startGame = () => {
  isCountingDown.value = true
  countdown.value = 3
  failureMessage.value = ''
  
  // 开始倒计时
  const countdownInterval = setInterval(() => {
    countdown.value--
    if (countdown.value <= 0) {
      clearInterval(countdownInterval)
      isCountingDown.value = false
      
      // 正式开始游戏
      gameStarted.value = true
      gameOver.value = false
      score.value = 0
      combo.value = 0
      currentTileIndex.value = 0
      clickAttempts.value = 0
      
      // 重置摄像机位置
      cameraOffset.value = { x: 0, y: 0 }
      
      generateTrack()
      
      // 设置小球初始位置在第一个瓦片上
      const firstTile = trackTiles.value[0]
      planets.value = [
        { x: firstTile.x, y: firstTile.y, isOrbiting: false }, // 火星球在第一个瓦片上
        { x: firstTile.x, y: firstTile.y, isOrbiting: true }   // 冰星球开始环绕
      ]
      
      // 设置初始环绕角度，根据第二个瓦片的方向
      const secondTile = trackTiles.value[1]
      if (secondTile) {
        const angle = Math.atan2(secondTile.y - firstTile.y, secondTile.x - firstTile.x)
        orbitAngle.value = angle - Math.PI / 2 // 让环绕星球朝向下一个瓦片方向
      } else {
        orbitAngle.value = 0
      }
      
      // 计算一圈的时间（2π / orbitSpeed 帧 * 16.67ms/帧）
      orbitDuration.value = (2 * Math.PI / orbitSpeed) * (1000 / 60)
      lastActionTime.value = Date.now()
      
      startGameLoop()
    }
  }, 1000)
}

const stopGame = () => {
  gameStarted.value = false
  gameOver.value = true
  isCountingDown.value = false
  clickAttempts.value = 0
  if (gameLoop) {
    cancelAnimationFrame(gameLoop)
    gameLoop = null
  }
}

// 失败动画处理
const startFailureAnimation = (tileIndex: number) => {
  isFailureAnimating.value = true
  failedTileIndex.value = tileIndex
  blinkCount.value = 0
  
  // 停止游戏循环但保持游戏状态
  if (gameLoop) {
    cancelAnimationFrame(gameLoop)
    gameLoop = null
  }
  
  // 开始闪烁动画
  startBlinkAnimation()
}

// 闪烁动画
const startBlinkAnimation = () => {
  if (blinkCount.value >= 6) { // 闪烁3次（每次包含开和关）
    // 闪烁结束，开始粒子粉碎效果
    startParticleExplosion()
    return
  }
  
  isBlinking.value = !isBlinking.value
  blinkCount.value++
  
  setTimeout(() => {
    startBlinkAnimation()
  }, 150) // 每300ms切换一次
}

// 粒子爆炸效果
const startParticleExplosion = () => {
  isParticleAnimating.value = true
  particles.value = []
  
  // 获取两个星球的位置
  const centerPlanet = planets.value.find(p => !p.isOrbiting)
  const orbitingPlanetPos = getOrbitingPlanet.value
  
  if (centerPlanet) {
    // 为中心星球创建粒子
    createParticlesForPlanet(centerPlanet.x, centerPlanet.y, firePlanetColor.value)
    // 为环绕星球创建粒子
    createParticlesForPlanet(orbitingPlanetPos.x, orbitingPlanetPos.y, icePlanetColor.value)
  }
  
  // 开始粒子动画循环
  startParticleAnimation()
}

// 为单个星球创建粒子
const createParticlesForPlanet = (x: number, y: number, colorName: string) => {
  const colorOptions = {
    fire: {
      red: ['#ff6b6b', '#ff4757', '#ff3742'],
      orange: ['#ffa726', '#ff9800', '#f57c00'],
      purple: ['#ba68c8', '#9c27b0', '#7b1fa2'],
      pink: ['#f48fb1', '#e91e63', '#c2185b'],
      yellow: ['#fff176', '#ffeb3b', '#fbc02d'],
      crimson: ['#ef5350', '#f44336', '#d32f2f']
    },
    ice: {
      blue: ['#4ecdc4', '#45b7aa', '#3ba99c'],
      cyan: ['#29b6f6', '#0288d1', '#0277bd'],
      green: ['#66bb6a', '#43a047', '#388e3c'],
      teal: ['#26a69a', '#00695c', '#004d40'],
      indigo: ['#5c6bc0', '#3949ab', '#303f9f'],
      mint: ['#4db6ac', '#00695c', '#004d40']
    }
  }
  
  // 确定颜色类型和颜色数组
  let colors: string[] = []
  if (colorName in colorOptions.fire) {
    colors = colorOptions.fire[colorName as keyof typeof colorOptions.fire]
  } else if (colorName in colorOptions.ice) {
    colors = colorOptions.ice[colorName as keyof typeof colorOptions.ice]
  } else {
    colors = ['#ff6b6b', '#ff4757', '#ff3742'] // 默认红色
  }
  
  // 创建30个粒子
  for (let i = 0; i < 30; i++) {
    const angle = (Math.PI * 2 * i) / 30 + Math.random() * 0.5
    const speed = 2 + Math.random() * 4
    const life = 60 + Math.random() * 40 // 60-100帧生命周期
    
    particles.value.push({
      id: Date.now() + Math.random(),
      x: x,
      y: y,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed,
      life: life,
      maxLife: life,
      size: 3 + Math.random() * 4,
      color: colors[Math.floor(Math.random() * colors.length)]
    })
  }
}

// 粒子动画循环
const startParticleAnimation = () => {
  const animateParticles = () => {
    if (!isParticleAnimating.value) return
    
    // 更新粒子
    particles.value = particles.value.filter(particle => {
      particle.x += particle.vx
      particle.y += particle.vy
      particle.vy += 0.1 // 重力
      particle.life--
      
      return particle.life > 0
    })
    
    // 如果还有粒子，继续动画
    if (particles.value.length > 0) {
      requestAnimationFrame(animateParticles)
    } else {
      // 粒子动画结束，显示失败框
      isParticleAnimating.value = false
      setTimeout(() => {
        isFailureAnimating.value = false
        gameStarted.value = false
        gameOver.value = true
      }, 300)
    }
  }
  
  requestAnimationFrame(animateParticles)
}

// 游戏主循环
const startGameLoop = () => {
  const loop = () => {
    if (!gameStarted.value) return
    
    // 失败动画期间停止旋转
    if (!isFailureAnimating.value) {
      // 更新环绕角度
      orbitAngle.value += orbitSpeed
      
      // 检测一圈内无操作超时
      const currentTime = Date.now()
      if (currentTime - lastActionTime.value > orbitDuration.value) {
        // 一圈内没有操作，算作失败
        clickAttempts.value++
        
        if (clickAttempts.value >= maxClickAttempts) {
          // 游戏结束
          failureMessage.value = '一圈内无操作！游戏结束！'
          combo.value = 0
          startFailureAnimation(currentTileIndex.value)
        } else {
          // 还有机会，显示提示并重置计时器
          failureMessage.value = `一圈内无操作！还有 ${maxClickAttempts - clickAttempts.value} 次机会`
          combo.value = 0
          lastActionTime.value = currentTime
          
          // 显示错误指示器
          const orbitingPlanetPos = getOrbitingPlanet.value
          showErrorIndicator(orbitingPlanetPos.x, orbitingPlanetPos.y)
        }
      }
    }
    
    // 更新摄像机跟随
    const centerPlanet = planets.value.find(p => !p.isOrbiting)
    if (centerPlanet) {
      const targetX = 400 - centerPlanet.x // 屏幕中心为400px
      const targetY = 300 - centerPlanet.y // 屏幕中心为300px
      
      cameraOffset.value.x += (targetX - cameraOffset.value.x) * cameraSmooth
      cameraOffset.value.y += (targetY - cameraOffset.value.y) * cameraSmooth
    }
    
    gameLoop = requestAnimationFrame(loop)
  }
  
  gameLoop = requestAnimationFrame(loop)
}

// 按键处理 - 单按键游戏
const handleKeyDown = (event: KeyboardEvent) => {
  if (!gameStarted.value || gameOver.value) return
  
  // 任意键都可以触发
  if (event.code === 'Space' || event.code === 'Enter' || 
      event.code === 'KeyX' || event.code === 'KeyZ') {
    event.preventDefault()
    handleTap()
  }
}

const handleTap = () => {
  // 更新最后操作时间
  lastActionTime.value = Date.now()
  
  // 检查是否在正确的时机点击
  const currentTile = trackTiles.value[currentTileIndex.value]
  const nextTile = trackTiles.value[currentTileIndex.value + 1]
  
  if (!currentTile || !nextTile) {
    // 游戏结束
    stopGame()
    return
  }
  
  const centerPlanet = planets.value.find(p => !p.isOrbiting)
  if (!centerPlanet) return
  
  // 计算从中心星球到下一个瓦片的目标角度
  const targetAngle = Math.atan2(nextTile.y - centerPlanet.y, nextTile.x - centerPlanet.x)
  
  // 应用延迟补偿：如果有延迟，需要预测小球在延迟时间后的位置
  // 延迟补偿应该是减去延迟时间内小球会移动的角度
  const delayCompensationAngle = (calibrationDelay.value / 1000) * orbitSpeed * 60
  const compensatedCurrentAngle = orbitAngle.value - delayCompensationAngle
  
  // 标准化角度到 [0, 2π] 范围
  const normalizeAngle = (angle: number) => {
    while (angle < 0) angle += 2 * Math.PI
    while (angle >= 2 * Math.PI) angle -= 2 * Math.PI
    return angle
  }
  
  const normalizedTarget = normalizeAngle(targetAngle)
  const normalizedCurrent = normalizeAngle(compensatedCurrentAngle)
  
  // 计算角度差（考虑环形特性）
  let angleDiff = Math.abs(normalizedTarget - normalizedCurrent)
  if (angleDiff > Math.PI) {
    angleDiff = 2 * Math.PI - angleDiff
  }
  
  // 计算环绕星球的实际位置
  const orbitingPlanetPos = getOrbitingPlanet.value
  const distanceToTarget = Math.sqrt(
    Math.pow(orbitingPlanetPos.x - nextTile.x, 2) + 
    Math.pow(orbitingPlanetPos.y - nextTile.y, 2)
  )
  
  // 调试信息
  console.log('目标角度:', (normalizedTarget * 180 / Math.PI).toFixed(1), '°')
  console.log('当前角度:', (normalizedCurrent * 180 / Math.PI).toFixed(1), '°')
  console.log('角度差:', (angleDiff * 180 / Math.PI).toFixed(1), '°')
  console.log('距离目标:', distanceToTarget.toFixed(1), 'px')
  console.log('点击次数:', clickAttempts.value + 1)
  
  // 更严格的击中判断：既要角度接近，也要距离接近
  const hitAngleRange = Math.PI / 12 // 15度
  const hitDistanceRange = 40 // 40像素内
  
  const isAngleGood = angleDiff < hitAngleRange
  const isDistanceGood = distanceToTarget < hitDistanceRange
  
  if (isAngleGood && isDistanceGood) {
    // 成功击中
    currentTileIndex.value++
    combo.value++
    score.value += 100 + combo.value * 10
    clickAttempts.value = 0 // 重置点击次数
    failureMessage.value = '' // 清除失败消息
    
    console.log('击中成功！角度差:', (angleDiff * 180 / Math.PI).toFixed(1), '°', '距离:', distanceToTarget.toFixed(1), 'px')
    
    // 切换星球角色
    planets.value.forEach(planet => {
      planet.isOrbiting = !planet.isOrbiting
    })
    
    // 更新星球位置到新的瓦片
    const newCenterPlanet = planets.value.find(p => !p.isOrbiting)
    if (newCenterPlanet) {
      newCenterPlanet.x = nextTile.x
      newCenterPlanet.y = nextTile.y
    }
    
    // 重置环绕角度 - 根据下一个瓦片的方向设置初始角度
    const nextNextTile = trackTiles.value[currentTileIndex.value + 1]
    if (nextNextTile) {
      const angle = Math.atan2(nextNextTile.y - nextTile.y, nextNextTile.x - nextTile.x)
      orbitAngle.value = angle - Math.PI / 2 // 让环绕星球朝向下一个瓦片方向
    } else {
      orbitAngle.value = 0
    }
    
    // 更新活跃瓦片
    trackTiles.value.forEach((tile, index) => {
      tile.isActive = index === currentTileIndex.value
    })
    
    // 检查是否完成所有瓦片
    if (currentTileIndex.value >= trackTiles.value.length - 1) {
      stopGame()
    }
  } else {
    // 错过了，增加点击次数
    clickAttempts.value++
    
    // 计算考虑延迟补偿后的星球位置用于显示错误指示器
    const compensatedPos = {
      x: centerPlanet.x + Math.cos(compensatedCurrentAngle) * orbitRadius,
      y: centerPlanet.y + Math.sin(compensatedCurrentAngle) * orbitRadius
    }
    showErrorIndicator(compensatedPos.x, compensatedPos.y)
    
    // 更精确的失败原因判断
    let feedbackMessage = ''
    if (!isAngleGood && !isDistanceGood) {
      const isTooEarly = normalizedCurrent < normalizedTarget
      feedbackMessage = isTooEarly ? '点击太快了！' : '点击太慢了！'
    } else if (!isAngleGood) {
      const isTooEarly = normalizedCurrent < normalizedTarget
      feedbackMessage = isTooEarly ? '点击太快了！' : '点击太慢了！'
    } else if (!isDistanceGood) {
      feedbackMessage = '时机不对！'
    }
    
    console.log('击中失败，角度差:', (angleDiff * 180 / Math.PI).toFixed(1), '°', '距离:', distanceToTarget.toFixed(1), 'px')
    console.log('需要角度小于:', (hitAngleRange * 180 / Math.PI).toFixed(1), '°', '距离小于:', hitDistanceRange, 'px')
    console.log(feedbackMessage)
    
    if (clickAttempts.value >= maxClickAttempts) {
      // 两次机会都用完了，游戏结束
      failureMessage.value = `${feedbackMessage} 游戏结束！`
      combo.value = 0
      startFailureAnimation(currentTileIndex.value)
    } else {
      // 还有机会，显示提示
      failureMessage.value = `${feedbackMessage} 还有 ${maxClickAttempts - clickAttempts.value} 次机会`
      combo.value = 0
    }
  }
}

// 显示错误提示
const showErrorIndicator = (x: number, y: number) => {
  const id = Date.now() + Math.random()
  const indicator: ErrorIndicator = {
    id,
    x,
    y,
    visible: true
  }
  
  errorIndicators.value.push(indicator)
  
  // 0.5秒后移除
  setTimeout(() => {
    const index = errorIndicators.value.findIndex(item => item.id === id)
    if (index !== -1) {
      errorIndicators.value.splice(index, 1)
    }
  }, 500)
}

// 游戏结束后的操作
const restartGame = () => {
  gameOver.value = false
  // 重置失败动画状态
  isFailureAnimating.value = false
  failedTileIndex.value = -1
  blinkCount.value = 0
  isBlinking.value = false
  // 重置计时器状态
  lastActionTime.value = 0
  orbitDuration.value = 0
  // 重置粒子状态
  particles.value = []
  isParticleAnimating.value = false
  startGame()
}

const backToHome = () => {
  gameOver.value = false
  gameStarted.value = false
  isCountingDown.value = false
  score.value = 0
  combo.value = 0
  currentTileIndex.value = 0
  clickAttempts.value = 0
  failureMessage.value = ''
  errorIndicators.value = []
  // 重置失败动画状态
  isFailureAnimating.value = false
  failedTileIndex.value = -1
  blinkCount.value = 0
  isBlinking.value = false
  // 重置计时器状态
  lastActionTime.value = 0
  orbitDuration.value = 0
  // 重置粒子状态
  particles.value = []
  isParticleAnimating.value = false
  if (gameLoop) {
    cancelAnimationFrame(gameLoop)
    gameLoop = null
  }
}

// 校准功能
const calibrate = () => {
  showCalibration.value = true
}

const closeCalibration = () => {
  showCalibration.value = false
  loadCalibrationData() // 重新加载校准数据
}



// 生成轨道路径SVG
const getTrackPath = () => {
  if (trackTiles.value.length === 0) return ''
  
  let path = `M ${trackTiles.value[0].x} ${trackTiles.value[0].y}`
  
  for (let i = 1; i < trackTiles.value.length; i++) {
    const tile = trackTiles.value[i]
    path += ` L ${tile.x} ${tile.y}`
  }
  
  return path
}

// 生命周期
onMounted(() => {
  window.addEventListener('keydown', handleKeyDown)
  generateTrack()
  loadCalibrationData() // 加载校准数据
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
  if (gameLoop) {
    cancelAnimationFrame(gameLoop)
  }
})
</script>

<template>
  <div class="game-container">
    <!-- 游戏标题 -->
    <div class="game-title">
      <h1>🔥 冰与火之舞 ❄️</h1>
      <p>按 空格键/X/Z/回车 来切换星球轨道</p>
    </div>

    <!-- 主页预览区域 -->
    <div v-if="!gameStarted && !showCalibration" class="home-preview">
      <div class="preview-container">
        <div class="preview-orbit"></div>
        <div class="preview-planet fire-planet" :data-color="firePlanetColor"></div>
        <div class="preview-planet ice-planet" :data-color="icePlanetColor"></div>
      </div>
      <p class="preview-text">当前延迟设置: {{ Math.floor(calibrationDelay) }}ms</p>
      
      <!-- 颜色选择按钮 -->
       <div class="color-selector">
         <button class="color-btn fire-btn" @click="openColorModal('fire')">
           <span class="btn-icon fire-planet" :data-color="firePlanetColor"></span>
           更换火星球颜色
         </button>
         <button class="color-btn ice-btn" @click="openColorModal('ice')">
           <span class="btn-icon ice-planet" :data-color="icePlanetColor"></span>
           更换冰星球颜色
         </button>
       </div>
    </div>
    
    <!-- 倒计时显示 -->
    <div v-if="isCountingDown" class="countdown-overlay">
      <div class="countdown-number">{{ countdown > 0 ? countdown : 'GO!' }}</div>
      <p>准备开始游戏...</p>
    </div>
    
    <!-- 游戏信息面板 -->
    <div class="game-info" v-if="gameStarted">
      <div class="info-item">
        <span>得分: {{ score }}</span>
      </div>
      <div class="info-item">
        <span>连击: {{ combo }}</span>
      </div>
      <div class="info-item">
        <span>进度: {{ currentTileIndex }}/{{ trackTiles.length - 1 }}</span>
      </div>
      <div class="info-item" v-if="clickAttempts > 0">
        <span>剩余机会: {{ maxClickAttempts - clickAttempts }}</span>
      </div>
      <div class="info-item">
        <span>延迟: {{ Math.floor(calibrationDelay) }}ms</span>
      </div>

    </div>
    
    <!-- 失败消息提示 -->
    <div v-if="failureMessage" class="failure-message">
      {{ failureMessage }}
    </div>
    
    <!-- 游戏区域 -->
    <div class="game-area" v-if="gameStarted" :style="{ transform: `translate(${cameraOffset.x}px, ${cameraOffset.y}px)` }">
      <!-- 轨道瓦片 -->
      <div 
        v-for="tile in trackTiles" 
        :key="tile.id"
        class="track-tile"
        :class="{ 
          'active-tile': tile.isActive, 
          'completed-tile': tile.id < currentTileIndex,
          'failed-tile': tile.id === failedTileIndex && isFailureAnimating
        }"
        :style="{ 
          left: tile.x + 'px', 
          top: tile.y + 'px' 
        }"
      ></div>
      
      <!-- 共享圆环轨道 -->
      <div 
        class="shared-orbit-container"
        :style="{ 
          left: (trackTiles[currentTileIndex] ? trackTiles[currentTileIndex].x : planets[0].x ) + 'px', 
          top: (trackTiles[currentTileIndex] ? trackTiles[currentTileIndex].y : planets[0].y) + 'px' 
        }"
      >
        <div class="shared-orbit" :class="{ 
          'orbiting': planets.some(p => p.isOrbiting),
          'blinking': isFailureAnimating && isBlinking
        }"></div>
      </div>
      
      <!-- 星球 -->
      <div 
        v-for="(planet, index) in planets" 
        :key="index"
        v-show="!isParticleAnimating"
        class="planet"
        :class="{ 
          'fire-planet': index === 0, 
          'ice-planet': index === 1,
          'orbiting': planet.isOrbiting,
          'blinking': isFailureAnimating && isBlinking
        }"
        :data-color="index === 0 ? firePlanetColor : icePlanetColor"
        :style="{ 
          left: planet.isOrbiting ? getOrbitingPlanet.x + 'px' : planet.x + 'px', 
          top: planet.isOrbiting ? getOrbitingPlanet.y + 'px' : planet.y + 'px' 
        }"
      ></div>
      
      <!-- 轨道路径线 -->
      <svg class="track-path" width="1600" height="1200">
        <path 
          :d="getTrackPath()" 
          stroke="rgba(255,255,255,0.3)" 
          stroke-width="2" 
          fill="none"
          stroke-dasharray="5,5"
        />
      </svg>
      
      <!-- 错误提示指示器 -->
      <div 
        v-for="indicator in errorIndicators" 
        :key="indicator.id"
        class="error-indicator"
        :style="{ 
          left: indicator.x + 'px', 
          top: indicator.y + 'px' 
        }"
      >
        <div class="error-ball">
          <div class="error-cross">
            <div class="cross-line cross-line-1"></div>
            <div class="cross-line cross-line-2"></div>
          </div>
        </div>
      </div>
      
      <!-- 粒子效果 -->
      <div 
        v-for="particle in particles" 
        :key="particle.id"
        class="particle"
        :style="{ 
          left: particle.x + 'px', 
          top: particle.y + 'px',
          width: particle.size + 'px',
          height: particle.size + 'px',
          backgroundColor: particle.color,
          opacity: particle.life / particle.maxLife
        }"
      ></div>
    </div>
    
    <!-- 开始/结束按钮 -->
    <div class="game-controls">
      <button v-if="!gameStarted && !isCountingDown" @click="startGame" class="start-btn">
        开始游戏
      </button>
      <button v-if="!gameStarted && !isCountingDown" @click="calibrate" class="calibrate-btn">
        校准
      </button>
      <button v-if="gameStarted" @click="stopGame" class="stop-btn">
        结束游戏
      </button>
    </div>
    
    <!-- 游戏结束提示 -->
    <div v-if="gameOver && !isFailureAnimating" class="game-over-overlay">
      <div class="game-over-content">
        <h2>{{ currentTileIndex >= trackTiles.length - 1 ? '恭喜完成!' : '游戏结束!' }}</h2>
        <p>最终得分: {{ score }}</p>
        <p>最高连击: {{ combo }}</p>
        <p>完成进度: {{ currentTileIndex }}/{{ trackTiles.length - 1 }}</p>
        
        <div class="game-over-buttons">
          <button @click="restartGame" class="game-over-btn restart-btn">
            重新开始
          </button>
          <button @click="backToHome" class="game-over-btn home-btn">
            回到主页
          </button>
          <button @click="calibrate" class="game-over-btn calibrate-btn">
            校准
          </button>
        </div>
      </div>
    </div>
      
      <!-- 校准页面 -->
      <CalibrationPage v-if="showCalibration" @close="closeCalibration" />
      
    <!-- 颜色选择弹框 -->
    <div v-if="showColorModal" class="color-modal-overlay" @click="closeColorModal">
      <div class="color-modal" @click.stop>
        <div class="modal-header">
          <h3>选择{{ currentEditingPlanet === 'fire' ? '火星球' : '冰星球' }}颜色</h3>
          <button class="close-btn" @click="closeColorModal">×</button>
        </div>
        <div class="modal-content">
          <div class="color-grid">
            <div 
              v-for="color in colorOptions[currentEditingPlanet as 'fire' | 'ice']"
              :key="color.name"
              class="modal-color-option"
              :class="{ 
                active: (currentEditingPlanet === 'fire' ? firePlanetColor : icePlanetColor) === color.name 
              }"
              :style="{ background: color.gradient }"
              @click="selectColor(color.name)"
            >
              <div class="color-name">{{ color.name }}</div>
            </div>
          </div>
        </div>
      </div>
    </div>
      
    </div>
  </template>

<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
  color: white;
  font-family: 'Arial', sans-serif;
  padding: 20px;
  overflow: hidden;
  position: relative;
  z-index: 1;
}

/* 主页预览区域样式 */
.home-preview {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 40px 0;
  z-index: 10;
}

.preview-container {
  position: relative;
  width: 200px;
  height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 20px;
}

.preview-orbit {
  position: absolute;
  width: 120px;
  height: 120px;
  border: 2px dashed rgba(255,182,193,0.6);
  border-radius: 50%;
  animation: previewOrbitRotate 4s linear infinite;
}

@keyframes previewOrbitRotate {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.preview-planet {
  position: absolute;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  border: 3px solid white;
  box-shadow: 0 0 15px rgba(255,255,255,0.5);
}

.preview-planet.fire-planet {
  background: radial-gradient(circle at 30% 30%, #ff9a9e, #ff6b6b, #ff4757);
  box-shadow: 0 0 20px rgba(255,107,107,0.8);
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.preview-planet.fire-planet[data-color="orange"] {
   background: radial-gradient(circle at 30% 30%, #ffa726, #ff9800, #f57c00);
   box-shadow: 0 0 20px rgba(255,152,0,0.8);
 }
 
 .preview-planet.fire-planet[data-color="purple"] {
   background: radial-gradient(circle at 30% 30%, #ba68c8, #9c27b0, #7b1fa2);
   box-shadow: 0 0 20px rgba(156,39,176,0.8);
 }

 .preview-planet.fire-planet[data-color="pink"] {
   background: radial-gradient(circle at 30% 30%, #f48fb1, #e91e63, #c2185b);
   box-shadow: 0 0 20px rgba(233,30,99,0.8);
 }

 .preview-planet.fire-planet[data-color="yellow"] {
   background: radial-gradient(circle at 30% 30%, #fff176, #ffeb3b, #fbc02d);
   box-shadow: 0 0 20px rgba(255,235,59,0.8);
 }

 .preview-planet.fire-planet[data-color="crimson"] {
   background: radial-gradient(circle at 30% 30%, #ef5350, #f44336, #d32f2f);
   box-shadow: 0 0 20px rgba(244,67,54,0.8);
 }

.preview-planet.ice-planet {
  background: radial-gradient(circle at 30% 30%, #a8edea, #4ecdc4, #45b7aa);
  box-shadow: 0 0 20px rgba(78,205,196,0.8);
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  animation: previewOrbitMove 4s linear infinite;
}

.preview-planet.ice-planet[data-color="cyan"] {
   background: radial-gradient(circle at 30% 30%, #81d4fa, #29b6f6, #0288d1);
   box-shadow: 0 0 20px rgba(41,182,246,0.8);
 }
 
 .preview-planet.ice-planet[data-color="green"] {
   background: radial-gradient(circle at 30% 30%, #a5d6a7, #66bb6a, #43a047);
   box-shadow: 0 0 20px rgba(102,187,106,0.8);
 }

 .preview-planet.ice-planet[data-color="teal"] {
   background: radial-gradient(circle at 30% 30%, #80cbc4, #26a69a, #00695c);
   box-shadow: 0 0 20px rgba(38,166,154,0.8);
 }

 .preview-planet.ice-planet[data-color="indigo"] {
   background: radial-gradient(circle at 30% 30%, #9fa8da, #5c6bc0, #3949ab);
   box-shadow: 0 0 20px rgba(92,107,192,0.8);
 }

 .preview-planet.ice-planet[data-color="mint"] {
   background: radial-gradient(circle at 30% 30%, #b2dfdb, #4db6ac, #00695c);
   box-shadow: 0 0 20px rgba(77,182,172,0.8);
 }

@keyframes previewOrbitMove {
  from {
    transform: translate(-50%, -50%) rotate(0deg) translateX(60px) rotate(0deg);
  }
  to {
    transform: translate(-50%, -50%) rotate(360deg) translateX(60px) rotate(-360deg);
  }
}

.preview-text {
  font-size: 1.1rem;
  color: rgba(255,255,255,0.8);
  text-align: center;
  margin: 0;
  padding: 10px 20px;
  background: rgba(255,255,255,0.1);
  border-radius: 10px;
  backdrop-filter: blur(10px);
}

.game-title {
  text-align: center;
  margin-bottom: 20px;
  z-index: 10;
}

.game-title h1 {
  font-size: 2.5rem;
  margin: 0;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.5);
  background: linear-gradient(45deg, #ff6b6b, #4ecdc4);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.game-title p {
  margin: 10px 0;
  opacity: 0.8;
  font-size: 1.1rem;
}

.game-info {
  display: flex;
  gap: 30px;
  margin-bottom: 20px;
  background: rgba(255,255,255,0.1);
  padding: 15px 30px;
  border-radius: 10px;
  backdrop-filter: blur(10px);
  z-index: 10;
}

.info-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
  font-weight: bold;
}

.game-area {
  position: relative;
  width: 1200px;
  height: 900px;
  background: radial-gradient(circle at center, rgba(255,255,255,0.05) 0%, transparent 70%);
  border-radius: 15px;
  overflow: visible;
  transition: transform 0.1s ease-out;
}

.track-path {
  position: absolute;
  top: -300px;
  left: -400px;
  z-index: 1;
  pointer-events: none;
  overflow: visible;
}

.track-tile {
  position: absolute;
  width: 20px;
  height: 20px;
  background: rgba(255,255,255,0.3);
  border: 2px solid rgba(255,255,255,0.5);
  border-radius: 3px;
  transform: translate(-50%, -50%);
  transition: all 0.3s ease;
  z-index: 2;
}

.track-tile.active-tile {
  background: linear-gradient(45deg, #ff6b6b, #4ecdc4);
  border-color: white;
  box-shadow: 0 0 20px rgba(255,255,255,0.8);
  animation: activePulse 1s ease-in-out infinite;
}

.track-tile.completed-tile {
  background: rgba(100,255,100,0.6);
  border-color: rgba(100,255,100,0.8);
}

.track-tile.failed-tile {
  background: rgba(255,50,50,0.8) !important;
  border-color: rgba(255,50,50,1) !important;
  box-shadow: 0 0 20px rgba(255,50,50,0.8) !important;
}

@keyframes activePulse {
  0%, 100% { transform: translate(-50%, -50%) scale(1); }
  50% { transform: translate(-50%, -50%) scale(1.2); }
}

.shared-orbit-container {
  position: absolute;
  transform: translate(-50%, -50%);
  z-index: 3;
}

.shared-orbit {
  position: absolute;
  width: 95px;
  height: 95px;
  border: 2px dashed rgba(255,182,193,0.6);
  border-radius: 50%;
  transform: translate(-50%, -50%);
  background: transparent;
}

.shared-orbit.orbiting {
  border-color: rgba(255,182,193,0.8);
}

.planet {
  position: absolute;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  transform: translate(-50%, -50%);
  z-index: 5;
  transition: all 0.2s ease;
  border: 3px solid white;
  box-shadow: 0 0 15px rgba(255,255,255,0.5);
}

@keyframes orbitRotate {
  from { transform: translate(-50%, -50%) rotate(0deg); }
  to { transform: translate(-50%, -50%) rotate(360deg); }
}

.fire-planet {
  background: radial-gradient(circle at 30% 30%, #ff9a9e, #ff6b6b, #ff4757);
  box-shadow: 0 0 20px rgba(255,107,107,0.8);
}

.fire-planet[data-color="orange"] {
   background: radial-gradient(circle at 30% 30%, #ffa726, #ff9800, #f57c00);
   box-shadow: 0 0 20px rgba(255,152,0,0.8);
 }
 
 .fire-planet[data-color="purple"] {
   background: radial-gradient(circle at 30% 30%, #ba68c8, #9c27b0, #7b1fa2);
   box-shadow: 0 0 20px rgba(156,39,176,0.8);
 }

 .fire-planet[data-color="pink"] {
   background: radial-gradient(circle at 30% 30%, #f48fb1, #e91e63, #c2185b);
   box-shadow: 0 0 20px rgba(233,30,99,0.8);
 }

 .fire-planet[data-color="yellow"] {
   background: radial-gradient(circle at 30% 30%, #fff176, #ffeb3b, #fbc02d);
   box-shadow: 0 0 20px rgba(255,235,59,0.8);
 }

 .fire-planet[data-color="crimson"] {
   background: radial-gradient(circle at 30% 30%, #ef5350, #f44336, #d32f2f);
   box-shadow: 0 0 20px rgba(244,67,54,0.8);
 }

.ice-planet {
  background: radial-gradient(circle at 30% 30%, #a8edea, #4ecdc4, #45b7aa);
  box-shadow: 0 0 20px rgba(78,205,196,0.8);
}

.ice-planet[data-color="cyan"] {
   background: radial-gradient(circle at 30% 30%, #81d4fa, #29b6f6, #0288d1);
   box-shadow: 0 0 20px rgba(41,182,246,0.8);
 }
 
 .ice-planet[data-color="green"] {
   background: radial-gradient(circle at 30% 30%, #a5d6a7, #66bb6a, #43a047);
   box-shadow: 0 0 20px rgba(102,187,106,0.8);
 }

 .ice-planet[data-color="teal"] {
   background: radial-gradient(circle at 30% 30%, #80cbc4, #26a69a, #00695c);
   box-shadow: 0 0 20px rgba(38,166,154,0.8);
 }

 .ice-planet[data-color="indigo"] {
   background: radial-gradient(circle at 30% 30%, #9fa8da, #5c6bc0, #3949ab);
   box-shadow: 0 0 20px rgba(92,107,192,0.8);
 }

 .ice-planet[data-color="mint"] {
   background: radial-gradient(circle at 30% 30%, #b2dfdb, #4db6ac, #00695c);
   box-shadow: 0 0 20px rgba(77,182,172,0.8);
 }

.planet.orbiting {
  animation: orbitGlow 0.5s ease-in-out infinite alternate;
}

@keyframes orbitGlow {
  from { 
    box-shadow: 0 0 15px rgba(255,255,255,0.5);
    transform: translate(-50%, -50%) scale(1);
  }
  to { 
    box-shadow: 0 0 25px rgba(255,255,255,0.9);
    transform: translate(-50%, -50%) scale(1.1);
  }
}

/* 闪烁效果 */
.blinking {
  opacity: 0 !important;
}

.shared-orbit.blinking {
  border-color: transparent !important;
}

.planet.blinking {
  box-shadow: none !important;
}

/* 错误提示样式 */
.error-indicator {
  position: absolute;
  transform: translate(-50%, -50%);
  z-index: 5;
  animation: errorFadeOut 0.5s ease-out forwards;
}

.error-ball {
  width: 16px;
  height: 16px;
  background: rgba(255, 255, 255, 0.7);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 0 8px rgba(255, 255, 255, 0.5);
}

.error-cross {
  position: relative;
  width: 10px;
  height: 10px;
}

.cross-line {
  position: absolute;
  background: #ff4757;
  border-radius: 1px;
}

.cross-line-1 {
  width: 10px;
  height: 2px;
  top: 50%;
  left: 0;
  transform: translateY(-50%) rotate(45deg);
}

.cross-line-2 {
  width: 10px;
  height: 2px;
  top: 50%;
  left: 0;
  transform: translateY(-50%) rotate(-45deg);
}

@keyframes errorFadeOut {
  0% {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1);
  }
  70% {
    opacity: 0.8;
    transform: translate(-50%, -50%) scale(1.1);
  }
  100% {
    opacity: 0;
    transform: translate(-50%, -50%) scale(0.8);
  }
}

.game-controls {
  margin-top: 20px;
  z-index: 10;
  display: flex;
  gap: 15px;
  justify-content: center;
}

.start-btn, .stop-btn, .game-controls .calibrate-btn {
  padding: 15px 30px;
  font-size: 1.2rem;
  border: none;
  border-radius: 25px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: bold;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.start-btn {
  background: linear-gradient(45deg, #56ab2f, #a8e6cf);
  color: white;
}

.start-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 15px rgba(0,0,0,0.3);
}

.stop-btn {
  background: linear-gradient(45deg, #ff416c, #ff4b2b);
  color: white;
}

.stop-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 15px rgba(0,0,0,0.3);
}

.game-controls .calibrate-btn {
  background: linear-gradient(45deg, #f093fb, #f5576c);
  color: white;
}

.game-controls .calibrate-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 15px rgba(0,0,0,0.3);
}

/* 粒子效果样式 */
.particle {
  position: absolute;
  border-radius: 50%;
  pointer-events: none;
  z-index: 6;
  transform: translate(-50%, -50%);
  box-shadow: 0 0 4px currentColor;
  animation: particleGlow 0.1s ease-in-out infinite alternate;
}

@keyframes particleGlow {
  from { 
    box-shadow: 0 0 2px currentColor;
  }
  to { 
    box-shadow: 0 0 6px currentColor;
  }
}

.game-over-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  backdrop-filter: blur(5px);
}

.game-over-content {
  background: rgba(26,26,46,0.95);
  color: white;
  padding: 40px;
  border-radius: 20px;
  text-align: center;
  border: 2px solid rgba(255,255,255,0.2);
  box-shadow: 0 10px 30px rgba(0,0,0,0.5);
}

.game-over-content h2 {
  margin: 0 0 20px 0;
  font-size: 2rem;
  background: linear-gradient(45deg, #ff6b6b, #4ecdc4);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.game-over-content p {
  margin: 10px 0;
  font-size: 1.1rem;
  opacity: 0.9;
}

.game-over-buttons {
  display: flex;
  gap: 15px;
  margin-top: 30px;
  justify-content: center;
}

.game-over-btn {
  padding: 12px 24px;
  font-size: 1rem;
  font-weight: bold;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  color: white;
}

.restart-btn {
  background: linear-gradient(45deg, #4ecdc4, #44a08d);
  box-shadow: 0 4px 15px rgba(78,205,196,0.4);
}

.restart-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(78,205,196,0.6);
}

.home-btn {
  background: linear-gradient(45deg, #667eea, #764ba2);
  box-shadow: 0 4px 15px rgba(102,126,234,0.4);
}

.home-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102,126,234,0.6);
}

.calibrate-btn {
  background: linear-gradient(45deg, #f093fb, #f5576c);
  box-shadow: 0 4px 15px rgba(240,147,251,0.4);
}

.calibrate-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(240,147,251,0.6);
}

/* 添加一些星空背景效果 */
.game-container::before {
  content: '';
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: 
    radial-gradient(2px 2px at 20px 30px, #eee, transparent),
    radial-gradient(2px 2px at 40px 70px, rgba(255,255,255,0.8), transparent),
    radial-gradient(1px 1px at 90px 40px, #fff, transparent),
    radial-gradient(1px 1px at 130px 80px, rgba(255,255,255,0.6), transparent),
    radial-gradient(2px 2px at 160px 30px, #ddd, transparent);
  background-repeat: repeat;
  background-size: 200px 100px;
  animation: sparkle 20s linear infinite;
  z-index: 0;
  pointer-events: none;
}

@keyframes sparkle {
  from { transform: translateX(0); }
  to { transform: translateX(-200px); }
}

/* 倒计时样式 */
.countdown-overlay {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  z-index: 30;
  background: rgba(0,0,0,0.8);
  padding: 60px;
  border-radius: 20px;
  border: 3px solid #4ecdc4;
  box-shadow: 0 0 40px rgba(78,205,196,0.6);
}

.countdown-number {
  font-size: 6rem;
  font-weight: bold;
  color: #4ecdc4;
  text-shadow: 0 0 20px rgba(78,205,196,0.8);
  margin-bottom: 20px;
  animation: countdownPulse 1s ease-in-out;
}

@keyframes countdownPulse {
  0% { transform: scale(0.8); opacity: 0; }
  50% { transform: scale(1.2); opacity: 1; }
  100% { transform: scale(1); opacity: 1; }
}

.countdown-overlay p {
  font-size: 1.5rem;
  color: white;
  margin: 0;
}

/* 失败消息样式 */
.failure-message {
  position: fixed;
  top: 20%;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(255, 65, 108, 0.9);
  color: white;
  padding: 20px 40px;
  border-radius: 15px;
  font-size: 1.3rem;
  font-weight: bold;
  text-align: center;
  z-index: 25;
  border: 2px solid #ff416c;
  box-shadow: 0 0 20px rgba(255, 65, 108, 0.5);
  animation: messageSlideIn 0.5s ease-out;
}

@keyframes messageSlideIn {
  from {
    transform: translateX(-50%) translateY(-20px);
    opacity: 0;
  }
  to {
    transform: translateX(-50%) translateY(0);
    opacity: 1;
  }
}

/* 颜色选择器样式 */
 .color-selector {
   margin-top: 20px;
   display: flex;
   gap: 15px;
   justify-content: center;
 }
 
 .color-btn {
   display: flex;
   align-items: center;
   gap: 10px;
   padding: 12px 20px;
   background: rgba(255,255,255,0.1);
   border: 2px solid rgba(255,255,255,0.3);
   border-radius: 25px;
   color: white;
   font-weight: bold;
   cursor: pointer;
   transition: all 0.3s ease;
   backdrop-filter: blur(10px);
 }
 
 .color-btn:hover {
   background: rgba(255,255,255,0.2);
   border-color: rgba(255,255,255,0.5);
   transform: translateY(-2px);
   box-shadow: 0 8px 15px rgba(0,0,0,0.3);
 }
 
 .btn-icon {
    width: 20px;
    height: 20px;
    border-radius: 50%;
    border: 2px solid white;
    box-shadow: 0 0 8px rgba(255,255,255,0.3);
  }

  /* 颜色选择弹框样式 */
  .color-modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0,0,0,0.8);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 2000;
    backdrop-filter: blur(5px);
  }

  .color-modal {
    background: rgba(26,26,46,0.95);
    border-radius: 20px;
    padding: 0;
    max-width: 500px;
    width: 90%;
    border: 2px solid rgba(255,255,255,0.2);
    box-shadow: 0 20px 40px rgba(0,0,0,0.5);
    animation: modalSlideIn 0.3s ease-out;
  }

  @keyframes modalSlideIn {
    from {
      transform: scale(0.8);
      opacity: 0;
    }
    to {
      transform: scale(1);
      opacity: 1;
    }
  }

  .modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 20px 25px;
    border-bottom: 1px solid rgba(255,255,255,0.1);
  }

  .modal-header h3 {
    margin: 0;
    color: white;
    font-size: 1.3rem;
    background: linear-gradient(45deg, #ff6b6b, #4ecdc4);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  .close-btn {
    background: none;
    border: none;
    color: rgba(255,255,255,0.7);
    font-size: 2rem;
    cursor: pointer;
    padding: 0;
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    transition: all 0.3s ease;
  }

  .close-btn:hover {
    background: rgba(255,255,255,0.1);
    color: white;
  }

  .modal-content {
    padding: 25px;
  }

  .color-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 15px;
  }

  .modal-color-option {
    aspect-ratio: 1;
    border-radius: 15px;
    cursor: pointer;
    border: 3px solid transparent;
    transition: all 0.3s ease;
    position: relative;
    overflow: hidden;
    display: flex;
    align-items: end;
    justify-content: center;
    padding: 10px;
    box-shadow: 0 4px 15px rgba(0,0,0,0.3);
  }

  .modal-color-option:hover {
    transform: translateY(-3px);
    box-shadow: 0 8px 25px rgba(0,0,0,0.4);
  }

  .modal-color-option.active {
    border-color: white;
    transform: translateY(-3px) scale(1.05);
    box-shadow: 0 0 20px rgba(255,255,255,0.6);
  }

  .color-name {
    background: rgba(0,0,0,0.7);
    color: white;
    padding: 4px 8px;
    border-radius: 8px;
    font-size: 0.8rem;
    font-weight: bold;
    text-transform: capitalize;
    backdrop-filter: blur(5px);
  }
 </style>
