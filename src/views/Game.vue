<script setup lang="ts">
import { reactive, ref, computed } from 'vue'

// 游戏状态
const gameStarted = ref(false)
const gameFinished = ref(false)
const currentSceneId = ref(1)
const moveCount = ref(0)
const lastSceneId = ref(0)
const deadEndProb = ref(0.15) // 死胡同初始概率 15%

// 节点记录：记录所有生成过的节点场景 ID
// Key 为路径指纹，例如 "main_0" 或 "main_2_left_forward"
const nodeCache = ref<Record<string, number>>({})
const currentPath = ref('main_0')

// 主路相关状态
const mainPath = ref<string[]>([])
const mainPathScenes = ref<number[]>([])
const mainPathIndex = ref(0)
const isOnMainPath = ref(true)
const lastMainPathIndex = ref(0) // 记录偏离主路时的位置

const gameState = reactive({
  floor: 1,
  hp: 100,
  maxHp: 100,
  weapon: '无',
  armor: '无'
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
    text: '你到达了终点，游戏结束',
    allowedMoves: []
  }
}

const currentScene = computed(() => scenes[currentSceneId.value])

// 开始游戏
const startGame = () => {
  gameStarted.value = true
  gameFinished.value = false
  currentSceneId.value = 1
  moveCount.value = 0
  lastSceneId.value = 0
  deadEndProb.value = 0.15
  
  // 初始化缓存和路径
  nodeCache.value = {}
  currentPath.value = 'main_0'
  
  // 生成主路
  mainPathIndex.value = 0
  isOnMainPath.value = true
  const directions = ['forward', 'left', 'right'] // 主路不包含 backward
  const path: string[] = []
  const pathScenes: number[] = [1]
  
  nodeCache.value['main_0'] = 1 // 起始点固定为场景 1
  
  for (let i = 0; i < 5; i++) {
    const currentSceneIdForPath = pathScenes[i]
    const allowed = scenes[currentSceneIdForPath].allowedMoves.filter(m => m !== 'backward')
    const dir = allowed[Math.floor(Math.random() * allowed.length)]
    path.push(dir)
    
    if (i < 4) {
      const nextSceneId = Math.floor(Math.random() * 6) + 1
      pathScenes.push(nextSceneId)
      nodeCache.value[`main_${i+1}`] = nextSceneId // 记录主路固定节点
    }
  }
  
  mainPath.value = path
  mainPathScenes.value = pathScenes
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
    nextId = 7
    deadEndProb.value = 0.15 // 命中后重置
  } else {
    // 排除掉当前的场景，增加变化
    const possibleScenes = [1, 2, 3, 4, 5, 6].filter(id => id !== currentSceneId.value)
    nextId = possibleScenes[Math.floor(Math.random() * possibleScenes.length)]
    deadEndProb.value += 0.1 // 未命中则累加 10%
  }

  nodeCache.value[pathKey] = nextId
  return nextId
}

// 移动逻辑
const handleMove = (direction: string) => {
  if (gameFinished.value) return

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
      if (mainPathIndex.value === 5) {
        nextSceneId = 8
        gameFinished.value = true
      } else {
        nextSceneId = nodeCache.value[newPath]
      }
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
      // 回到主路偏离点
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
  
  if (moveCount.value % 10 === 0) {
    gameState.floor++
  }
}

// 检查动作是否可用
const isActionDisabled = (action: 'forward' | 'backward' | 'left' | 'right') => {
  // 第一个场景且向后不可用
  if (moveCount.value === 0 && action === 'backward') return true
  
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
              >向前</button>
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
                <div class="scene-text">
                  {{ currentScene.text }}
                </div>
                <div v-if="isOnMainPath && !gameFinished" class="main-path-hint">
                  你已经在主路上
                </div>
                <div v-if="gameFinished" class="game-over-container">
                  <div class="game-over-hint">你到达终点，游戏结束</div>
                  <button class="game-btn restart-btn" @click="startGame">重新开始</button>
                </div>
              </div>

              <!-- 向右按钮 -->
              <button 
                class="game-btn" 
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
            <div class="debug-item">主路: {{ mainPath.join(' -> ') }}</div>
            <div class="debug-item">当前路径 ID: {{ currentPath }}</div>
            <div class="debug-item">场景 {{ currentSceneId }} ({{ isOnMainPath ? '主路' : '支路' }})</div>
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
  gap: 10px;
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

/* 控制盘布局 */
.control-pad {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.top-row {
  margin-bottom: 15vh;
}

.bottom-row {
  margin-top: 15vh;
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
</style>
