<script setup lang="ts">
import { ref, onUnmounted, nextTick, computed } from 'vue'

interface OptionType {
  id: number
  text: string
}
interface StayTimeItem extends OptionType {
  interTime: number[] // 每次进入的时间
  outTime: number[] // 每次离开的时间
  totalTime: number // 总停留时长
  currentTime: number // 当前这次停留的时间
}

const question = ref('以下哪个城市是法国的首都？')
const answerId = 2
const options = [
  { id: 1, text: '伦敦' },
  { id: 2, text: '巴黎' },
  { id: 3, text: '柏林' },
  { id: 4, text: '罗马' },
]

const stayTime = ref<StayTimeItem[]>([]) // 最终统计的计时数据
const answerPoint: { minY: number; minX: number; maxX: number; maxY: number }[] = []
const stayItem: { index: number; data: OptionType | null } = {
  index: -1,
  data: null,
}

const animationFrameId = ref<number | null>(null) // RAF ID
const CORRECT_ANSWER_THRESHOLD = 2000 // 停留在正确答案超过2s则结束答题
const isQuizStarted = ref(false) // 是否开启答题
const isQuizEnded = ref(false) // 是否结束答题
const quizStartTime = ref<number>(0) // 答题开始时间

let currentHoverIndex = -1 // 记录当前正在停留的选项索引
let enterTimestamp: number | null = null // 记录进入当前选项的时间

// 计算前5秒停留时间排名
const first5SecondsRanking = computed(() => {
  if (!isQuizEnded.value || quizStartTime.value === 0) return []

  const fiveSecondsLimit = quizStartTime.value + 5000 // 前5秒的时间界限

  const rankings = stayTime.value
    .map((item) => {
      let timeInFirst5Seconds = 0

      // 遍历所有进入和离开的时间对
      for (let i = 0; i < item.interTime.length; i++) {
        const enterTime = item.interTime[i]
        const exitTime = item.outTime[i] || Date.now() // 如果没有离开时间，使用当前时间

        // 计算在前5秒内的停留时间
        if (enterTime < fiveSecondsLimit) {
          const effectiveExitTime = Math.min(exitTime, fiveSecondsLimit)
          if (effectiveExitTime > enterTime) {
            timeInFirst5Seconds += effectiveExitTime - enterTime
          }
        }
      }

      return {
        ...item,
        first5SecondsTime: timeInFirst5Seconds,
      }
    })
    .sort((a, b) => b.first5SecondsTime - a.first5SecondsTime)

  return rankings
})

// 计算总停留时间排名
const totalTimeRanking = computed(() => {
  if (!isQuizEnded.value) return []

  return [...stayTime.value].sort((a, b) => b.totalTime - a.totalTime)
})

// 实时更新计时的函数
const updateStayTime = () => {
  if (currentHoverIndex !== -1 && enterTimestamp !== null) {
    const now = Date.now()
    const currentDuration = now - enterTimestamp

    // 更新当前停留时间
    stayTime.value[currentHoverIndex].currentTime = currentDuration

    // 检查是否在正确答案区域停留超过2秒
    const currentOption = options[currentHoverIndex]
    if (currentOption.id === answerId && currentDuration >= CORRECT_ANSWER_THRESHOLD) {
      console.log('【自动结束】在正确答案停留超过2秒')
      endQuiz()
      return
    }
  }

  if (!isQuizEnded.value) {
    animationFrameId.value = requestAnimationFrame(updateStayTime)
  }
}

// 开始实时计时
const startRealTimeTimer = () => {
  if (animationFrameId.value) {
    cancelAnimationFrame(animationFrameId.value)
  }
  animationFrameId.value = requestAnimationFrame(updateStayTime)
}

// 停止实时计时
const stopRealTimeTimer = () => {
  if (animationFrameId.value) {
    cancelAnimationFrame(animationFrameId.value)
    animationFrameId.value = null
  }
}

// 开始答题
const startQuiz = async () => {
  isQuizStarted.value = true
  quizStartTime.value = Date.now() // 记录答题开始时间
  await nextTick()
  document.querySelectorAll('.option-button').forEach((i) => {
    const style = i.getBoundingClientRect()
    answerPoint.push({
      minY: style.y,
      minX: style.x,
      maxX: style.x + style.width,
      maxY: style.y + style.height,
    })
  })

  // 鼠标模拟眼动
  document.onmousemove = (e) => {
    eyesPointHandle({ y: e.clientY, x: e.clientX })
  }

  // 初始化 stayTime 数据
  options.forEach((opt) => {
    stayTime.value.push({
      ...opt,
      interTime: [],
      outTime: [],
      totalTime: 0,
      currentTime: 0,
    })
  })

  // 开始实时计时
  startRealTimeTimer()
}

// 眼动时间统计
const eyesPointHandle = (data: { x: number; y: number }) => {
  const { x, y } = data
  const now = Date.now()

  let hoveredIndex = -1
  let hoveredData: OptionType | null = null

  // 先判断停留在哪个答案点
  for (let i = 0; i < answerPoint.length; i++) {
    const point = answerPoint[i]
    const option = options[i]
    if (x >= point.minX && x <= point.maxX && y >= point.minY && y <= point.maxY) {
      hoveredIndex = i
      hoveredData = option
      break
    }
  }

  // 如果进入新的选项区域
  if (hoveredIndex !== -1 && hoveredIndex !== currentHoverIndex) {
    // 如果之前有停留的区域，先记录离开并累计时间
    if (currentHoverIndex !== -1 && enterTimestamp !== null) {
      const duration = now - enterTimestamp
      stayTime.value[currentHoverIndex].outTime.push(now)
      stayTime.value[currentHoverIndex].totalTime += duration
      stayTime.value[currentHoverIndex].currentTime = 0
    }

    // 开始进入新区域
    currentHoverIndex = hoveredIndex
    stayItem.index = hoveredIndex
    stayItem.data = hoveredData
    stayTime.value[hoveredIndex].interTime.push(now)
    enterTimestamp = now
  } else if (hoveredIndex === -1 && currentHoverIndex !== -1 && enterTimestamp !== null) {
    // 如果移出了所有选项区域
    const duration = now - enterTimestamp
    stayTime.value[currentHoverIndex].outTime.push(now)
    stayTime.value[currentHoverIndex].totalTime += duration
    stayTime.value[currentHoverIndex].currentTime = 0

    currentHoverIndex = -1
    enterTimestamp = null
    stayItem.index = -1
    stayItem.data = null
  }
}

// 停止答题
const endQuiz = () => {
  // 停止监听鼠标移动
  document.onmousemove = null

  // 停止实时计时
  stopRealTimeTimer()

  // 如果当前还在某个选项上，需要记录离开时间
  if (currentHoverIndex !== -1 && enterTimestamp !== null) {
    const now = Date.now()
    const duration = now - enterTimestamp
    stayTime.value[currentHoverIndex].outTime.push(now)
    stayTime.value[currentHoverIndex].totalTime += duration
    stayTime.value[currentHoverIndex].currentTime = 0

    currentHoverIndex = -1
    enterTimestamp = null
    stayItem.index = -1
    stayItem.data = null
  }

  isQuizEnded.value = true
  console.log(stayTime.value)
}

const formatTime = (milliseconds: number) => {
  const seconds = Math.floor(milliseconds / 1000)
  const ms = milliseconds % 1000
  return `${seconds}.${ms.toString().padStart(3, '0')}s`
}

// 获取显示时间（总时间 + 当前时间）
const getDisplayTime = (index: number) => {
  if (stayTime.value[index]) {
    return stayTime.value[index].totalTime + stayTime.value[index].currentTime
  }
  return 0
}

// 获取排名标识
const getRankBadge = (index: number) => {
  const badges = ['🥇', '🥈', '🥉', '4th', '5th']
  return badges[index] || `${index + 1}th`
}

// 重新开始
const restartQuiz = () => {
  isQuizStarted.value = false
  isQuizEnded.value = false
  stayTime.value.length = 0
  answerPoint.length = 0
  currentHoverIndex = -1
  enterTimestamp = null
  quizStartTime.value = 0
}

onUnmounted(() => {
  document.onmousemove = null
  stopRealTimeTimer()
})
</script>

<template>
  <div class="quiz-container">
    <div class="quiz-content">
      <!-- 开始页面 -->
      <div v-if="!isQuizStarted" class="start-screen">
        <h1 class="title">答题系统</h1>
        <p class="description">点击开始按钮开始答题</p>
        <button class="start-button" @click="startQuiz">开始答题</button>
      </div>

      <!-- 答题页面 -->
      <div v-else-if="!isQuizEnded" class="question-card">
        <h2 class="question">{{ question }}</h2>
        <div class="options-container">
          <button
            v-for="(option, index) in options"
            :key="option.id"
            class="option-button"
            :class="{ hovering: currentHoverIndex === index }"
          >
            <p>{{ option.text }}</p>
            <div class="option-info">{{ formatTime(getDisplayTime(index)) }}</div>
          </button>
        </div>
        <button class="end-button" @click="endQuiz">结束答题</button>
      </div>

      <!-- 结果页面 -->
      <div v-else class="result-card">
        <h2 class="result-title">答题结果</h2>

        <!-- 总体数据统计 -->
        <div class="stats-section">
          <h3 class="section-title">📊 总体数据统计</h3>
          <div class="stats-grid">
            <div
              v-for="item in stayTime"
              :key="`stats-${item.id}`"
              class="stats-item"
              :class="{ 'correct-answer': item.id === answerId }"
            >
              <div class="stats-header">
                <span class="option-text">{{ item.text }}</span>
                <span v-if="item.id === answerId" class="correct-badge">✓ 正确答案</span>
              </div>
              <div class="stats-details">
                <div class="stat-row">
                  <span class="stat-label">总停留时长:</span>
                  <span class="stat-value time-value">{{ formatTime(item.totalTime) }}</span>
                </div>
                <div class="stat-row">
                  <span class="stat-label">总进入次数:</span>
                  <span class="stat-value count-value">{{ item.interTime.length }} 次</span>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 前5秒停留排名 -->
        <div class="ranking-section">
          <h3 class="section-title">🏆 前5秒停留时间排名</h3>
          <div class="ranking-list">
            <div
              v-for="(item, index) in first5SecondsRanking"
              :key="`rank-${item.id}`"
              class="ranking-item"
              :class="{
                'correct-answer': item.id === answerId,
                'top-3': index < 3,
              }"
            >
              <div class="rank-badge">{{ getRankBadge(index) }}</div>
              <div class="rank-content">
                <div class="rank-option">
                  {{ item.text }}
                  <span v-if="item.id === answerId" class="correct-mini-badge">正确</span>
                </div>
                <div class="rank-time">{{ formatTime(item.first5SecondsTime) }}</div>
              </div>
              <div class="rank-percentage">
                <div
                  class="percentage-bar"
                  :style="{
                    width: `${first5SecondsRanking.length > 0 ? (item.first5SecondsTime / first5SecondsRanking[0].first5SecondsTime) * 100 : 0}%`,
                  }"
                ></div>
              </div>
            </div>
          </div>
        </div>

        <!-- 总停留时间排名 -->
        <div class="ranking-section">
          <h3 class="section-title">⏱️ 总停留时间排名</h3>
          <div class="ranking-list">
            <div
              v-for="(item, index) in totalTimeRanking"
              :key="`total-rank-${item.id}`"
              class="ranking-item"
              :class="{
                'correct-answer': item.id === answerId,
                'top-3': index < 3,
              }"
            >
              <div class="rank-badge">{{ getRankBadge(index) }}</div>
              <div class="rank-content">
                <div class="rank-option">
                  {{ item.text }}
                  <span v-if="item.id === answerId" class="correct-mini-badge">正确</span>
                </div>
                <div class="rank-time">{{ formatTime(item.totalTime) }}</div>
              </div>
              <div class="rank-percentage">
                <div
                  class="percentage-bar total-time-bar"
                  :style="{
                    width: `${totalTimeRanking.length > 0 ? (item.totalTime / totalTimeRanking[0].totalTime) * 100 : 0}%`,
                  }"
                ></div>
              </div>
            </div>
          </div>
        </div>

        <button class="restart-button" @click="restartQuiz">重新开始</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.quiz-container {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #f5f7fa;
  padding: 20px;

  .quiz-content {
    width: 100%;
    max-width: 800px;
  }

  /* 开始页面样式 */
  .start-screen {
    background: white;
    border-radius: 16px;
    padding: 50px 30px;
    text-align: center;
    box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);

    .title {
      font-size: 28px;
      color: #333;
      margin-bottom: 16px;
      font-weight: 600;
    }

    .description {
      color: #666;
      margin-bottom: 30px;
      font-size: 16px;
    }

    .start-button {
      padding: 16px 32px;
      font-size: 18px;
      background-color: #409eff;
      color: white;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      transition: all 0.3s ease;
      font-weight: 500;

      &:hover {
        background-color: #337ecc;
        transform: translateY(-2px);
      }
    }
  }

  /* 答题页面样式 */
  .question-card {
    background: white;
    border-radius: 16px;
    padding: 30px;
    box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
    text-align: center;

    .question {
      font-size: 24px;
      color: #333;
      margin-bottom: 30px;
      line-height: 1.5;
      font-weight: 600;
    }

    .options-container {
      display: grid;
      gap: 16px;
      margin-bottom: 30px;
    }

    .option-button {
      padding: 18px 24px;
      font-size: 18px;
      border: 2px solid #e1e5eb;
      border-radius: 14px;
      background-color: white;
      color: #333;
      cursor: pointer;
      transition: all 0.3s ease;
      text-align: center;
      font-weight: 500;

      position: relative;
      .option-info {
        position: absolute;
        right: 2%;
        bottom: 5%;
        color: #409eff;
        font-weight: 600;
      }

      &:hover,
      &.hovering {
        border-color: #409eff;
        transform: translateY(-2px);
        .option-info {
          color: #d14b4b;
        }
      }
    }

    .end-button {
      padding: 14px 28px;
      font-size: 16px;
      background-color: #f56c6c;
      color: white;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      transition: all 0.3s ease;
      font-weight: 500;

      &:hover {
        background-color: #d14b4b;
        transform: translateY(-2px);
      }
    }
  }

  /* 结果页面样式 */
  .result-card {
    background: white;
    border-radius: 16px;
    padding: 30px;
    box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
    max-height: 80vh;
    overflow-y: auto;

    .result-title {
      font-size: 28px;
      color: #333;
      margin-bottom: 30px;
      font-weight: 600;
      text-align: center;
    }

    .section-title {
      font-size: 20px;
      color: #333;
      margin-bottom: 20px;
      font-weight: 600;
      display: flex;
      align-items: center;
      gap: 8px;
    }

    /* 统计区域样式 */
    .stats-section {
      margin-bottom: 40px;

      .stats-grid {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
        gap: 16px;

        .stats-item {
          background: #f8f9fa;
          border-radius: 12px;
          padding: 20px;
          border-left: 4px solid #e1e5eb;
          transition: all 0.3s ease;

          &.correct-answer {
            background: #f0f9ff;
            border-left-color: #409eff;
          }

          .stats-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 12px;

            .option-text {
              font-size: 18px;
              font-weight: 600;
              color: #333;
            }

            .correct-badge {
              background: #67c23a;
              color: white;
              padding: 4px 8px;
              border-radius: 4px;
              font-size: 12px;
              font-weight: 600;
            }
          }

          .stats-details {
            .stat-row {
              display: flex;
              justify-content: space-between;
              align-items: center;
              margin-bottom: 8px;

              &:last-child {
                margin-bottom: 0;
              }

              .stat-label {
                color: #666;
                font-size: 14px;
              }

              .stat-value {
                font-weight: 600;

                &.time-value {
                  color: #409eff;
                }

                &.count-value {
                  color: #67c23a;
                }
              }
            }
          }
        }
      }
    }

    /* 排名区域样式 */
    .ranking-section {
      margin-bottom: 30px;

      .ranking-list {
        .ranking-item {
          display: flex;
          align-items: center;
          padding: 16px;
          border-radius: 12px;
          margin-bottom: 12px;
          background: #f8f9fa;
          transition: all 0.3s ease;

          &.correct-answer {
            background: #f0f9ff;
          }

          &.top-3 {
            background: linear-gradient(135deg, #fff7ed, #fef3c7);
          }

          .rank-badge {
            font-size: 20px;
            font-weight: 700;
            min-width: 60px;
            text-align: center;
            color: #333;
          }

          .rank-content {
            flex: 1;
            margin: 0 20px;

            .rank-option {
              font-size: 18px;
              font-weight: 600;
              color: #333;
              display: flex;
              align-items: center;
              gap: 8px;

              .correct-mini-badge {
                background: #67c23a;
                color: white;
                padding: 2px 6px;
                border-radius: 3px;
                font-size: 10px;
              }
            }

            .rank-time {
              font-size: 16px;
              color: #409eff;
              font-weight: 600;
              margin-top: 4px;
            }
          }

          .rank-percentage {
            width: 100px;
            height: 8px;
            background: #e1e5eb;
            border-radius: 4px;
            overflow: hidden;

            .percentage-bar {
              height: 100%;
              background: linear-gradient(90deg, #409eff, #67c23a);
              transition: width 0.6s ease;

              &.total-time-bar {
                background: linear-gradient(90deg, #e6a23c, #f56c6c);
              }
            }
          }
        }
      }
    }

    .restart-button {
      padding: 14px 28px;
      font-size: 16px;
      background-color: #909399;
      color: white;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      transition: all 0.3s ease;
      font-weight: 500;
      display: block;
      margin: 0 auto;

      &:hover {
        background-color: #7a7d82;
        transform: translateY(-2px);
      }
    }
  }
}
</style>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html,
body,
#app {
  width: 100%;
  height: 100%;
}
</style>
