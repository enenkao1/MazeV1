<script setup lang="ts">
import { reactive, ref, computed } from 'vue'

// 游戏状态
const gameStarted = ref(false)
const gameFinished = ref(false)
const currentSceneId = ref(1)
const moveCount = ref(0)
const lastSceneId = ref(0)
const deadEndProb = ref(0.4) // 死胡同初始概率 40%

// 节点记录：记录所有生成过的节点场景 ID
// Key 为路径指纹，例如 "main_0" 或 "main_2_left_forward"
const nodeCache = ref<Record<string, number>>({})
const nodeStates = ref<Record<string, any>>({}) // 记录特定节点的额外状态，如宝箱是否打开
const currentPath = ref('main_0')

// 提示框开关
const showHintBox = ref(false)
const hintText = ref('这里是文本')
const onHintClose = ref<(() => void) | null>(null) // 提示框关闭后的回调

// 主路相关状态
const mainPath = ref<string[]>([])
const mainPathScenes = ref<number[]>([])
const mainPathIndex = ref(0)
const isOnMainPath = ref(true)
const lastMainPathIndex = ref(0) // 记录偏离主路时的位置
const scene9Count = ref(0) // 当前层生成的场景 9 数量
const chestProb = ref(0.4) // 宝箱初始概率 40%

const gameState = reactive({
  floor: 0,
  hp: 100,
  maxHp: 100,
  weapon: '无',
  weaponLevel: 0,
  armor: '无',
  armorLevel: 0,
  keys: 0
})

// 场景定义
interface Scene {
  id: number
  text: string
  allowedMoves: ('forward' | 'backward' | 'left' | 'right')[]
}

const scenes: Record<number, Scene> = {
  1: {
    id: 1,
    text: '昏暗的前方有三条岔路，向前、向左，向右',
    allowedMoves: ['forward', 'backward', 'left', 'right']
  },
  2: {
    id: 2,
    text: '昏暗的前方道路上出现了向左的道路',
    allowedMoves: ['forward', 'backward', 'left']
  },
  3: {
    id: 3,
    text: '昏暗的前方道路上出现了向右的道路',
    allowedMoves: ['forward', 'backward', 'right']
  },
  4: {
    id: 4,
    text: '前方出现了向左的拐角',
    allowedMoves: ['backward', 'left']
  },
  5: {
    id: 5,
    text: '前方出现了向右的拐角',
    allowedMoves: ['backward', 'right']
  },
  6: {
    id: 6,
    text: '前方走到了头，出现了向左和向右的道路',
    allowedMoves: ['backward', 'left', 'right']
  },
  7: {
    id: 7,
    text: '这里是死胡同',
    allowedMoves: ['backward']
  },
  8: {
    id: 8,
    text: '你发现了通往下一层的楼梯！',
    allowedMoves: ['forward', 'backward']
  },
  9: {
    id: 9,
    text: '你发现了一个宝箱！',
    allowedMoves: ['forward', 'backward']
  }
}

const currentScene = computed(() => {
  const scene = { ...scenes[currentSceneId.value] }
  // 如果是场景 9 且宝箱已打开，修改文字
  if (currentSceneId.value === 9 && nodeStates.value[currentPath.value]?.opened) {
    scene.text = '一个打开过的宝箱'
  }
  return scene
})

// 获取场景图片路径
const getSceneImage = (id: number) => {
  if (id === 9) {
    const state = nodeStates.value[currentPath.value]
    if (state?.chestOpenedFinal) return new URL(`../assets/scene9_3.png`, import.meta.url).href
    if (state?.opened) return new URL(`../assets/scene9_2.png`, import.meta.url).href
    return new URL(`../assets/scene9.png`, import.meta.url).href
  }
  // 根据实际文件列表，scene7 是 jpg，其余是 png
  const ext = id === 7 ? 'jpg' : 'png'
  return new URL(`../assets/scene${id}.${ext}`, import.meta.url).href
}

// 生成主路逻辑
const generateMaze = () => {
  // 初始化缓存和路径
  nodeCache.value = {}
  nodeStates.value = {}
  currentPath.value = 'main_0'
  scene9Count.value = 0
  chestProb.value = 0.4 // 新的一层重置宝箱概率
  
  // 生成主路
  mainPathIndex.value = 0
  isOnMainPath.value = true
  const path: string[] = []
  const pathScenes: number[] = [1]
  
  nodeCache.value['main_0'] = 1 // 起始点固定为场景 1
  
  for (let i = 0; i < 4; i++) {
    const currentSceneIdForPath = pathScenes[i]
    const allowed = scenes[currentSceneIdForPath].allowedMoves.filter(m => m !== 'backward')
    const dir = allowed[Math.floor(Math.random() * allowed.length)]
    path.push(dir)
    
    if (i < 3) {
      const nextSceneId = Math.floor(Math.random() * 6) + 1
      pathScenes.push(nextSceneId)
      nodeCache.value[`main_${i+1}`] = nextSceneId // 记录主路固定节点
    }
  }

  // 场景 8 固定在主路终点
  nodeCache.value['main_4'] = 8
  
  mainPath.value = path
  mainPathScenes.value = pathScenes
}

// 开始游戏
const startGame = () => {
  gameStarted.value = true
  gameFinished.value = false
  currentSceneId.value = 1
  moveCount.value = 0
  lastSceneId.value = 0
  deadEndProb.value = 0.4
  gameState.floor = 0
  
  generateMaze()
}

// 辅助函数：获取已记录节点或生成新节点
const getOrCreateNode = (pathKey: string): number => {
  if (nodeCache.value[pathKey]) {
    return nodeCache.value[pathKey]
  }

  // 生成新节点逻辑（包含死胡同概率）
  let nextId: number
  const roll = Math.random()
  if (moveCount.value >= 3 && lastSceneId.value !== 7 && roll < deadEndProb.value) {
    // 判定是否将死胡同替换为场景 9（宝箱）
    // 逻辑：初始 40%，每遇到一个死胡同（场景 7）增加 15%
    if (Math.random() < chestProb.value && scene9Count.value < 2) {
      nextId = 9
      scene9Count.value++
      nodeStates.value[pathKey] = { opened: false, chestOpenedFinal: false }
    } else {
      nextId = 7
      chestProb.value += 0.15 // 遇到死胡同，增加下次出现宝箱的概率
    }
    deadEndProb.value = 0.4 // 命中后重置为 40%
  } else {
    // 排除掉当前的场景，增加变化
    const possibleScenes = [1, 2, 3, 4, 5, 6].filter(id => id !== currentSceneId.value)
    nextId = possibleScenes[Math.floor(Math.random() * possibleScenes.length)]
    deadEndProb.value += 0.2 // 未命中则累加 20%
  }

  nodeCache.value[pathKey] = nextId
  return nextId
}

// 提示框关闭逻辑
const handleCloseHint = () => {
  showHintBox.value = false
  if (onHintClose.value) {
    onHintClose.value()
    onHintClose.value = null
  }
}

// 移动逻辑
const handleMove = (direction: string) => {
  if (gameFinished.value && currentSceneId.value !== 8) return

  // 特殊处理场景 8 的“通往下一层”
  if (currentSceneId.value === 8 && direction === 'forward') {
    gameState.floor--
    moveCount.value = 0
    lastSceneId.value = 0
    deadEndProb.value = 0.4
    generateMaze()
    currentSceneId.value = 1
    return
  }

  // 特殊处理场景 9 的“打开宝箱”
  if (currentSceneId.value === 9 && direction === 'forward') {
    const state = nodeStates.value[currentPath.value]
    if (state && !state.opened) {
      state.opened = true
      const rand = Math.random()
      let item = ''
      if (rand < 0.33) {
        item = '铁剑'
        if (gameState.weapon === '无') {
          gameState.weapon = '铁剑'
          gameState.weaponLevel = 1
        }
      } else if (rand < 0.66) {
        item = '铠甲'
        if (gameState.armor === '无') {
          gameState.armor = '铠甲'
          gameState.armorLevel = 1
        }
      } else {
        item = '钥匙'
        gameState.keys++
      }
      
      hintText.value = `你获得了${item}`
      showHintBox.value = true
      onHintClose.value = () => {
        state.chestOpenedFinal = true
      }
    }
    return
  }

  lastSceneId.value = currentSceneId.value
  moveCount.value++
  
  let nextSceneId = 1
  let newPath = ''

  if (isOnMainPath.value) {
    // 在主路上
    if (direction === mainPath.value[mainPathIndex.value]) {
      // 走对了
      mainPathIndex.value++
      newPath = `main_${mainPathIndex.value}`
      nextSceneId = nodeCache.value[newPath]
    } else if (direction === 'backward' && mainPathIndex.value > 0) {
      // 主路向后
      mainPathIndex.value--
      newPath = `main_${mainPathIndex.value}`
      nextSceneId = nodeCache.value[newPath]
    } else {
      // 走错了，偏离主路
      isOnMainPath.value = false
      lastMainPathIndex.value = mainPathIndex.value
      newPath = `main_${mainPathIndex.value}_${direction}`
      nextSceneId = getOrCreateNode(newPath)
    }
  } else {
    // 不在主路上
    if (direction === 'backward') {
      // 回到进入错误路径的主路节点
      isOnMainPath.value = true
      mainPathIndex.value = lastMainPathIndex.value
      newPath = `main_${mainPathIndex.value}`
      nextSceneId = nodeCache.value[newPath]
    } else {
      // 继续在错误道路上移动
      newPath = `${currentPath.value}_${direction}`
      nextSceneId = getOrCreateNode(newPath)
    }
  }

  currentPath.value = newPath
  currentSceneId.value = nextSceneId
}

// 检查动作是否可用
const isActionDisabled = (action: 'forward' | 'backward' | 'left' | 'right') => {
  // 每一层的第一个主路节点（main_0）且在主路上时，向后不可用
  if (isOnMainPath.value && mainPathIndex.value === 0 && action === 'backward') return true
  
  // 场景 9 宝箱打开后向前不可用
  if (currentSceneId.value === 9 && action === 'forward' && nodeStates.value[currentPath.value]?.opened) {
    return true
  }
  
  return !currentScene.value.allowedMoves.includes(action)
}
</script>

<template>
  <div class="game-page">
    <div class="main-content">
      <!-- 第一幕：序章 -->
      <div v-if="!gameStarted" class="prologue">
        <p>你是一名探险者，你在探索一个地牢迷宫</p>
        <button class="game-btn" @click="startGame">进入迷宫</button>
      </div>

        <!-- 第二幕：迷宫探索 -->
        <div v-else class="exploration">
          <div class="control-pad">
            <!-- 向上按钮 -->
            <div class="pad-row top-row">
              <button 
                class="game-btn" 
                :disabled="isActionDisabled('forward')"
                @click="handleMove('forward')"
              >
                {{ currentSceneId === 8 ? '通往下一层' : (currentSceneId === 9 ? '打开宝箱' : '向前') }}
              </button>
            </div>
            
            <div class="pad-row middle">
              <div class="side-status-wrapper">
                <!-- 状态栏：使用绝对定位，不占据文档流空间 -->
                <div class="side-status-bar">
                  <div class="side-status-item">
                    <span class="side-label">楼层</span>
                    <span class="side-value">{{ gameState.floor }}</span>
                  </div>
                  <div class="side-status-item">
                    <span class="side-label">HP</span>
                    <span class="side-value">{{ gameState.hp }}/{{ gameState.maxHp }}</span>
                  </div>
                  <div class="side-status-item">
                    <span class="side-label">武器</span>
                    <span class="side-value">{{ gameState.weapon }}</span>
                  </div>
                  <div class="side-status-item">
                    <span class="side-label">护甲</span>
                    <span class="side-value">{{ gameState.armor }}</span>
                  </div>
                  <div class="side-status-item">
                    <span class="side-label">钥匙</span>
                    <span class="side-value">{{ gameState.keys }}</span>
                  </div>
                </div>

                <!-- 向左按钮 -->
                <button 
                  class="game-btn" 
                  :disabled="isActionDisabled('left')"
                  @click="handleMove('left')"
                >向左</button>
              </div>

              <!-- 中央画面文本 -->
              <div class="scene-container">
                <div class="scene-image-wrapper">
                  <img :src="getSceneImage(currentSceneId)" :alt="'Scene ' + currentSceneId" class="scene-image" />
                </div>
                <div class="scene-text">
                  {{ currentScene.text }}
                </div>
                <div v-if="isOnMainPath && currentSceneId !== 8" class="main-path-hint">
                  你已经在主路上
                </div>
              </div>

              <!-- 向右按钮 -->
              <button 
                class="game-btn right-btn-offset" 
                :disabled="isActionDisabled('right')"
                @click="handleMove('right')"
              >向右</button>
            </div>

            <!-- 向下按钮 -->
            <div class="pad-row bottom-row">
              <button 
                class="game-btn" 
                :disabled="isActionDisabled('backward')"
                @click="handleMove('backward')"
              >向后</button>
            </div>
          </div>

          <!-- 场景标记与调试信息 -->
          <div class="debug-info">
            <div class="debug-item">
              <button class="game-btn toggle-btn" @click="showHintBox = !showHintBox">
                {{ showHintBox ? '关闭提示框' : '打开提示框' }}
              </button>
            </div>
            <div class="debug-item">主路: {{ mainPath.join(' -> ') }}</div>
            <div class="debug-item">当前路径 ID: {{ currentPath }}</div>
            <div class="debug-item">场景 {{ currentSceneId }} ({{ isOnMainPath ? '主路' : '支路' }})</div>
          </div>

          <!-- 提示框内容 -->
          <div v-if="showHintBox" class="hint-box-overlay">
            <div class="hint-box-container">
              <div class="hint-box">
                <div class="hint-content">
                  {{ hintText }}
                </div>
                <button class="game-btn continue-btn" @click="handleCloseHint">继续</button>
              </div>
            </div>
          </div>
        </div>
    </div>
  </div>
</template>

<style scoped>
.game-page {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  width: 100vw;
  background-color: #fff;
  color: #000;
  overflow: hidden;
}

/* 主内容区域 */
.main-content {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
}

.prologue, .exploration {
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.scene-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
  /* 移除可能导致偏移的 margin */
}

.scene-image-wrapper {
  width: 720px;
  height: 540px;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  border: 2px solid #000;
  background-color: #f0f0f0;
  /* 移除 margin-bottom，改用 gap 控制 */
}

.scene-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.scene-text {
  font-size: 22px;
  font-weight: bold;
  width: auto;
  max-width: 600px;
  line-height: 1.6;
  padding: 0 30px;
  white-space: nowrap;
}

.main-path-hint {
  font-size: 14px;
  color: #42b883; /* 绿色提示在主路上 */
  font-weight: bold;
}

.game-over-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  margin-top: 10px;
}

.game-over-hint {
  font-size: 18px;
  color: #ff4d4f; /* 红色提示游戏结束 */
  font-weight: bold;
}

.restart-btn {
  margin-top: 10px;
  border-color: #ff4d4f;
  color: #ff4d4f;
}

.restart-btn:hover {
  background-color: #ff4d4f;
  color: #fff;
}

/* 按钮通用样式 */
.game-btn {
  padding: 10px 25px;
  font-size: 18px;
  cursor: pointer;
  background-color: #fff;
  color: #000;
  border: 2px solid #000;
  font-family: 'PingFang SC', sans-serif;
  font-weight: bold;
  transition: all 0.2s;
  min-width: 100px;
}

.game-btn:hover:not(:disabled) {
  background-color: #000;
  color: #fff;
}

.game-btn:disabled {
  border-color: #ccc;
  color: #ccc;
  cursor: not-allowed;
}

.right-btn-offset {
  transform: translateY(-30px); /* 向上提 30px */
}

/* 控制盘布局 */
.control-pad {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.top-row {
  margin-bottom: 20px;
}

.bottom-row {
  margin-top: 20px;
}

.pad-row.middle {
  align-items: center; /* 确保中间一排元素垂直居中 */
}

.pad-row {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 15px;
}

/* 侧边状态栏样式 */
.side-status-wrapper {
  position: relative;
  display: flex;
  align-items: center;
  transform: translateY(-30px); /* 向上提 30px */
}

.side-status-bar {
  position: absolute;
  right: calc(100% + 100px); /* 原来是 50px，再往左移 50px 变为 100px */
  display: flex;
  flex-direction: column;
  gap: 30px; /* 间距调大为 30px */
  text-align: right;
  border-right: 2px solid #000;
  padding-right: 15px;
  min-width: 120px;
  white-space: nowrap;
  top: 50%;
  transform: translateY(-50%); /* 保持相对于按钮居中 */
}

.side-status-item {
  display: flex;
  flex-direction: row; /* 调整为水平排列 */
  align-items: baseline;
  justify-content: flex-end;
  gap: 10px; /* 标签与数值之间的间距 */
}

.side-label {
  font-size: 16px; /* 稍微调大标签，使与20px数值更协调 */
  color: #666;
  font-weight: bold;
}

.side-value {
  font-size: 20px; /* 字体调整为 20px */
  font-weight: bold;
}

.debug-info {
  position: absolute;
  bottom: 40px;
  right: 40px;
  text-align: right;
  font-family: monospace;
  background: rgba(255, 255, 255, 0.8);
  padding: 10px;
  border: 1px dashed #ccc;
  z-index: 10;
}

.debug-item {
  font-size: 12px;
  color: #999;
  margin-bottom: 2px;
}

.toggle-btn {
  padding: 5px 10px;
  font-size: 12px;
  min-width: auto;
  margin-bottom: 10px;
}

.hint-box-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  pointer-events: none; /* 允许点击下方的按钮，除非你想让它遮挡 */
  z-index: 100;
}

.hint-box-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  transform: translateY(-30px); /* 原来 -20px，再往上移 50px，总共 -70px */
}

.hint-box {
  width: calc(720px * 0.8); /* 图片宽度的 80% */
  height: 400px;/* 图片高度的 80% */
  background-color: rgba(255, 255, 255, 0.95);
  border: 5px solid #000;
  border-radius: 20px;
  padding: 50px 50px 20px 50px;
  box-sizing: border-box;
  text-align: center;
  font-size: 24px;
  font-weight: bold;
  pointer-events: auto; /* 提示框本身可以接收点击 */
  box-shadow: 0 10px 30px rgba(0,0,0,0.2);
  
  /* 内部布局：垂直排列，文本在上，按钮在下 */
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: center;
}

.hint-content {
  width: 100%;
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
}

.continue-btn {
  pointer-events: auto;
  background-color: #000;
  color: #fff;
  margin-top: 20px; /* 按钮与上方文本的间距 */
}

.continue-btn:hover {
  background-color: #333;
}
</style>
