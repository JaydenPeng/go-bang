<script setup lang="ts">
import { onMounted, ref } from 'vue'

// 胜利弹窗控制
const showWinModal = ref(false)
const winner = ref('')
const isReviewing = ref(false) // 复盘状态控制

// 玩家信息
const players = ref([
  { id: 1, name: '黑方', logo: '⚫', isCurrent: true },
  { id: 2, name: '白方', logo: '⚪', isCurrent: false }
])

const canvasRef = ref<HTMLCanvasElement | null>(null)
const boardSize = 15 // 15x15棋盘
const cellSize = ref(40) // 每个格子大小，使用响应式变量
const starPoints = [3, 7, 11] // 星位位置

// 棋盘状态，0表示空，1表示黑子，2表示白子
const boardState = ref<number[][]>(Array(boardSize).fill(0).map(() => Array(boardSize).fill(0)))
const currentPlayer = ref(1) // 1表示黑子，2表示白子

// 更新当前玩家指示
const updateCurrentPlayer = (playerId: number) => {
  players.value.forEach(player => {
    player.isCurrent = player.id === playerId
  })
}

/**
 * 检查落子是否有效
 * @param x 横坐标
 * @param y 纵坐标
 * @returns 返回是否有效落子
 */
const isValidMove = (x: number, y: number): boolean => {
  return boardState.value[x][y] === 0
}

/**
 * 检查是否获胜
 * @param x 最后落子的横坐标
 * @param y 最后落子的纵坐标
 * @returns 返回是否获胜
 */
const checkWin = (x: number, y: number): boolean => {
  const player = boardState.value[x][y]
  const directions = [
    [1, 0],   // 水平
    [0, 1],   // 垂直
    [1, 1],   // 对角线
    [1, -1]   // 反对角线
  ]
  
  for (const [dx, dy] of directions) {
    let count = 1
    
    // 正向检查
    for (let i = 1; i < 5; i++) {
      const nx = x + dx * i
      const ny = y + dy * i
      if (nx < 0 || nx >= boardSize || ny < 0 || ny >= boardSize || boardState.value[nx][ny] !== player) {
        break
      }
      count++
    }
    
    // 反向检查
    for (let i = 1; i < 5; i++) {
      const nx = x - dx * i
      const ny = y - dy * i
      if (nx < 0 || nx >= boardSize || ny < 0 || ny >= boardSize || boardState.value[nx][ny] !== player) {
        break
      }
      count++
    }
    
    if (count >= 5) {
      // 确保获胜提示正确显示
      winner.value = player === 1 ? '黑方' : '白方'
      showWinModal.value = true
      return true
    }
  }
  
  return false
}

/**
 * 重置游戏
 */
const resetGame = () => {
  boardState.value = Array(boardSize).fill(0).map(() => Array(boardSize).fill(0))
  currentPlayer.value = 1
  updateCurrentPlayer(1)
  drawPieces()
}

/**
 * 将屏幕坐标转换为棋盘坐标
 * @param clientX 鼠标X坐标
 * @param clientY 鼠标Y坐标
 * @returns 返回棋盘坐标[x, y]，如果不在棋盘内返回null
 */
const getBoardPosition = (clientX: number, clientY: number): [number, number] | null => {
  const canvas = canvasRef.value
  if (!canvas) return null
  
  const rect = canvas.getBoundingClientRect()
const x = Math.floor((clientX - rect.left) / cellSize.value)
const y = Math.floor((clientY - rect.top) / cellSize.value)
  
  if (x < 0 || x >= boardSize || y < 0 || y >= boardSize) return null
  
  return [x, y]
}

/**
 * 初始化棋盘
 */
const initBoard = () => {
  const canvas = canvasRef.value
  if (!canvas) return
  
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  
  // 设置画布大小
const totalSize = boardSize * cellSize.value
  canvas.width = totalSize
  canvas.height = totalSize
  
  // 绘制棋盘背景（木质纹理）
  ctx.fillStyle = '#DCB35C'
  ctx.fillRect(0, 0, totalSize, totalSize)
  
  // 添加木质纹理效果
  ctx.strokeStyle = 'rgba(0, 0, 0, 0.1)'
  ctx.lineWidth = 1
  for (let i = 0; i < totalSize; i += 5) {
    ctx.beginPath()
    ctx.moveTo(i, 0)
    ctx.lineTo(i, totalSize)
    ctx.stroke()
  }
  
  // 绘制网格线（更精致的样式）
  ctx.strokeStyle = '#5C4033'
  ctx.lineWidth = 1.5
  
  for (let i = 0; i < boardSize; i++) {
    // 横线
    ctx.beginPath()
    ctx.moveTo(cellSize.value / 2, i * cellSize.value + cellSize.value / 2)
    ctx.lineTo(totalSize - cellSize.value / 2, i * cellSize.value + cellSize.value / 2)
    ctx.stroke()
    
    // 竖线
    ctx.beginPath()
    ctx.moveTo(i * cellSize.value + cellSize.value / 2, cellSize.value / 2)
    ctx.lineTo(i * cellSize.value + cellSize.value / 2, totalSize - cellSize.value / 2)
    ctx.stroke()
  }
  
  // 绘制星位
  ctx.fillStyle = '#000'
  starPoints.forEach(x => {
    starPoints.forEach(y => {
      ctx.beginPath()
      ctx.arc(
        x * cellSize.value + cellSize.value / 2,
        y * cellSize.value + cellSize.value / 2,
        3, // 星位点半径
        0,
        Math.PI * 2
      )
      ctx.fill()
    })
  })
}

/**
 * 绘制所有棋子
 * @param callback 绘制完成后执行的回调函数
 */
const drawPieces = (callback?: () => void) => {
  const canvas = canvasRef.value
  if (!canvas) return
  
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  
  // 清空之前绘制的棋子
  ctx.clearRect(0, 0, canvas.width, canvas.height)
  
  // 重新绘制棋盘
  initBoard()
  
  // 绘制所有棋子
  for (let x = 0; x < boardSize; x++) {
    for (let y = 0; y < boardSize; y++) {
      if (boardState.value[x][y] !== 0) {
        ctx.beginPath()
        ctx.fillStyle = boardState.value[x][y] === 1 ? '#000' : '#fff'
        ctx.arc(
          x * cellSize.value + cellSize.value / 2,
          y * cellSize.value + cellSize.value / 2,
          cellSize.value * 0.4, // 棋子半径
          0,
          Math.PI * 2
        )
        ctx.fill()
        
        // 为白子添加边框
        if (boardState.value[x][y] === 2) {
          ctx.strokeStyle = '#000'
          ctx.lineWidth = 1
          ctx.stroke()
        }
      }
    }
  }
  
  // 执行回调函数
  if (callback) {
    callback();
  }
}

const clickHandler = ref<((e: MouseEvent) => void) | null>(null)

onMounted(() => {
  initBoard()
  
  const canvas = canvasRef.value
  if (canvas) {
    clickHandler.value = (e: MouseEvent) => {
      const position = getBoardPosition(e.clientX, e.clientY)
      if (position) {
        const [x, y] = position
        if (isValidMove(x, y)) {
          boardState.value[x][y] = currentPlayer.value
          const lastPlayer = currentPlayer.value
          
          drawPieces(() => {
            if (checkWin(x, y)) {
              // 锁定棋盘操作
              showWinModal.value = true
              // 保持事件监听用于后续操作
            } else {
              currentPlayer.value = currentPlayer.value === 1 ? 2 : 1
              updateCurrentPlayer(currentPlayer.value)
            }
          })
        }
      }
    }
    
    canvas.addEventListener('click', clickHandler.value)
  }
  
  window.addEventListener('resize', handleResize)
  handleResize()
})

// 窗口大小变化处理函数
const handleResize = () => {
  const container = document.querySelector('.board-container')
  if (container) {
    const containerWidth = container.clientWidth
    const containerHeight = container.clientHeight
    const minDimension = Math.min(containerWidth, containerHeight) - 40 // 留出边距
    
    // 计算合适的格子大小
    const newCellSize = Math.floor(minDimension / boardSize)
    cellSize.value = newCellSize
    
    // 重新初始化棋盘并绘制棋子
    initBoard()
    drawPieces()
  }
}

/**
 * 再来一局
 */
const playAgain = () => {
  resetGame()
  showWinModal.value = false
  isReviewing.value = false
  // 重新绑定点击事件
  const canvas = canvasRef.value
  if (canvas && clickHandler.value) {
    canvas.addEventListener('click', clickHandler.value)
  }
}

/**
 * 复盘（目前简单隐藏弹窗，可根据需求扩展）
 */
const reviewGame = () => {
  showWinModal.value = false
  isReviewing.value = true
  // 移除点击事件
  const canvas = canvasRef.value
  if (canvas && clickHandler.value) {
    canvas.removeEventListener('click', clickHandler.value)
  }
}
</script>

<template>
  <div class="players-info">
    <div 
      v-for="player in players" 
      :key="player.id" 
      class="player"
      :class="{ 'current-player': player.isCurrent }"
    >
      <span class="player-logo">{{ player.logo }}</span>
      <span class="player-name">{{ player.name }}</span>
      <span v-if="player.isCurrent" class="arrow">→</span>
    </div>
  </div>
  <div class="board-container">
    <canvas ref="canvasRef" class="board"></canvas>
  </div>

  <div v-if="showWinModal" class="win-modal">
    <div class="modal-content">
      <h3>{{ winner }} 获胜！</h3>
      <button @click="playAgain">再来一局</button>
      <button @click="reviewGame">复盘</button>
    </div>
    </div>
<!-- 由于不清楚多余的 </div> 对应的开始标签，这里暂时移除该结束标签 -->

  <div v-if="isReviewing" class="review-control">
    <button @click="playAgain">再来一局</button>
  </div>
</template>

<style scoped>
.board-container {
  display: flex;
  justify-content: center;
  margin: 20px;
  margin-top: 0;
}

.players-info {
  display: flex;
  justify-content: space-around;
  font-size: 18px;
}

.players-board-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.board-container {
  margin-top: 20px;
}

.player {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 20px;
  border-radius: 8px;
  transition: all 0.3s;
}

.current-player {
  background-color: #f0f0f0;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

.player-logo {
  font-size: 24px;
}

.arrow {
  margin-left: 10px;
  color: #f56c6c;
}

.review-control {
  margin-top: 20px;
  text-align: center;
}

.review-control button {
  padding: 12px 24px;
  font-size: 16px;
  background-color: #409eff;
  color: white;
  border-radius: 4px;
}

.win-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background-color: white;
  padding: 20px;
  border-radius: 8px;
  text-align: center;
}

.modal-content button {
  margin: 10px;
  padding: 10px 20px;
  cursor: pointer;
}
</style>