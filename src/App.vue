<script setup lang="ts">
import { ref, onUnmounted, nextTick } from 'vue'
import dayjs from 'dayjs'

interface OptionType {
  id: number
  text: string
}
interface StayTimeItem extends OptionType {
  interTime: number[] // 每次进入的时间
  outTime: number[] // 每次离开的时间
  totalTime: number // 总停留时长
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

const correctAnswerStayTimer = ref<number | null>(null) // 用于存储 停留在正确答案的 setTimeout 的 ID
const CORRECT_ANSWER_THRESHOLD = 2000 // 单次停留在正确答案超过2s则结束答题
const isQuizStarted = ref(false) // 是否开启答题
const isQuizEnded = ref(false) // 是否结束答题

let currentHoverIndex = -1 // 记录当前正在停留的选项索引
let enterTimestamp: number | null = null // 记录进入当前选项的时间

// 开始答题
const startQuiz = async () => {
  isQuizStarted.value = true
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
    })
  })
}

// 眼动时间统计
const eyesPointHandle = (data: { x: number; y: number }) => {
  const { x, y } = data
  const now = dayjs().valueOf()

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
    // 如果之前有停留的区域，先记录离开
    if (currentHoverIndex !== -1 && enterTimestamp !== null) {
      stayTime.value[currentHoverIndex].outTime.push(now)
      const duration = now - enterTimestamp
      stayTime.value[currentHoverIndex].totalTime += duration
    }

    // 清除之前的正确答案计时器（如果有）
    if (correctAnswerStayTimer.value) {
      clearTimeout(correctAnswerStayTimer.value)
      correctAnswerStayTimer.value = null
    }

    // 开始进入新区域
    currentHoverIndex = hoveredIndex
    stayItem.index = hoveredIndex
    stayItem.data = hoveredData
    stayTime.value[hoveredIndex].interTime.push(now)
    enterTimestamp = now

    // 判断是否进入正确答案区域，如果是则设置2秒计时器
    if (hoveredData!.id === answerId) {
      correctAnswerStayTimer.value = window.setTimeout(() => {
        console.log('【自动结束】在正确答案停留超过2秒')
        endQuiz()
      }, CORRECT_ANSWER_THRESHOLD)
    }
  } else if (hoveredIndex === -1 && currentHoverIndex !== -1 && enterTimestamp !== null) {
    // 如果移出了所有选项区域
    stayTime.value[currentHoverIndex].outTime.push(now)
    const duration = now - enterTimestamp
    stayTime.value[currentHoverIndex].totalTime += duration

    // 清除正确答案计时器
    if (correctAnswerStayTimer.value) {
      clearTimeout(correctAnswerStayTimer.value)
      correctAnswerStayTimer.value = null
    }

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

  // 如果当前还在某个选项上，需要记录离开时间
  if (currentHoverIndex !== -1 && enterTimestamp !== null) {
    const now = dayjs().valueOf()
    stayTime.value[currentHoverIndex].outTime.push(now)
    const duration = now - enterTimestamp
    stayTime.value[currentHoverIndex].totalTime += duration
    currentHoverIndex = -1
    enterTimestamp = null
    stayItem.index = -1
    stayItem.data = null
  }

  // 清除正确答案计时器
  if (correctAnswerStayTimer.value) {
    clearTimeout(correctAnswerStayTimer.value)
    correctAnswerStayTimer.value = null
  }

  isQuizEnded.value = true

  console.log(stayTime.value)
}

const formatTime = (milliseconds: number) => {
  const seconds = Math.floor(milliseconds / 1000)
  const ms = milliseconds % 1000
  return `${seconds}.${ms.toString().padStart(3, '0')}s`
}

onUnmounted(() => {
  document.onmousemove = null
  if (correctAnswerStayTimer.value) {
    clearTimeout(correctAnswerStayTimer.value)
    correctAnswerStayTimer.value = null
  }
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
          <button v-for="(option, index) in options" :key="option.id" class="option-button">
            <p>{{ option.text }}</p>
            <div class="option-info">{{ formatTime(stayTime[index]?.totalTime) }}</div>
          </button>
        </div>
        <button class="end-button" @click="endQuiz">结束答题</button>
      </div>

      <!-- 结果页面 -->
      <div v-else class="result-card">
        <h2 class="result-title">答题结果</h2>
        <div class="stay-time-list">
          <div
            v-for="item in stayTime"
            :key="item.id"
            class="stay-time-item"
            :class="{ 'correct-answer': item.id === answerId }"
          >
            <span class="option-text">{{ item.text }}</span>
            <span class="stay-time">{{ formatTime(item.totalTime) }}</span>
          </div>
        </div>
        <button
          class="restart-button"
          @click="
            () => {
              isQuizStarted = false
              isQuizEnded = false
              stayTime.length = 0
            }
          "
        >
          重新开始
        </button>
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
    max-width: 600px;
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
      }

      &:hover {
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
    text-align: center;

    .result-title {
      font-size: 24px;
      color: #333;
      margin-bottom: 30px;
      font-weight: 600;
    }

    .stay-time-list {
      margin-bottom: 30px;

      .stay-time-item {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 16px;
        border-bottom: 1px solid #eee;
        font-size: 18px;
        color: #333;

        &:last-child {
          border-bottom: none;
        }

        &.correct-answer {
          background-color: #f0f9ff;
          border-left: 4px solid #409eff;
        }

        .option-text {
          font-weight: 500;
        }

        .stay-time {
          font-weight: 600;
          color: #409eff;
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
