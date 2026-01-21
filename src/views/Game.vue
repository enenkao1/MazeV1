<script setup lang="ts">
import { reactive, ref, computed } from 'vue'

// 游戏状态
const gameStarted = ref(false)
const currentSceneId = ref(1)
const moveCount = ref(0)
const lastSceneId = ref(0)
const deadEndProb = ref(0.15) // 死胡同初始概率 15%

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
  }
}

const currentScene = computed(() => scenes[currentSceneId.value])

// 开始游戏
const startGame = () => {
  gameStarted.value = true
  currentSceneId.value = 1
  moveCount.value = 0
  lastSceneId.value = 0
  deadEndProb.value = 0.15
}

// 移动逻辑
const handleMove = (direction: string) => {
  lastSceneId.value = currentSceneId.value
  moveCount.value++
  
  let nextSceneId = 1
  
  // 判定是否进入死胡同（场景 7）
  // 逻辑：移动次数 >= 3 且 上一个场景不是死胡同
  if (moveCount.value >= 3 && lastSceneId.value !== 7) {
    const roll = Math.random()
    if (roll < deadEndProb.value) {
      nextSceneId = 7
      deadEndProb.value = 0.15 // 出现死胡同，概率恢复初始 15%
    } else {
      // 没中死胡同，从 1-6 中选一个
      const possibleScenes = [1, 2, 3, 4, 5, 6].filter(id => id !== currentSceneId.value)
      const randomIndex = Math.floor(Math.random() * possibleScenes.length)
      nextSceneId = possibleScenes[randomIndex]
      
      deadEndProb.value += 0.1 // 没中死胡同，下次概率增加 10%
    }
  } else {
    // 移动次数少于 3 次，或者上一个是死胡同，则只从 1-6 随机
    const possibleScenes = [1, 2, 3, 4, 5, 6].filter(id => id !== currentSceneId.value)
    const randomIndex = Math.floor(Math.random() * possibleScenes.length)
    nextSceneId = possibleScenes[randomIndex]
    
    // 如果上一个是死胡同，此时不增加概率，保持初始
    if (lastSceneId.value === 7) {
      deadEndProb.value = 0.15
    }
  }

  currentSceneId.value = nextSceneId
  
  // 如果是移动，可以考虑增加楼层或触发事件
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
            <div class="scene-text">
              {{ currentScene.text }}
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

        <!-- 场景标记 -->
        <div class="scene-tag">
          场景 {{ currentSceneId }}
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

.scene-text {
  font-size: 22px;
  font-weight: bold;
  width: auto;
  max-width: 600px;
  line-height: 1.6;
  padding: 0 30px;
  white-space: nowrap;
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

.scene-tag {
  position: absolute;
  bottom: 40px;
  right: 40px;
  font-size: 16px;
  font-weight: bold;
  color: #999;
}
</style>
