<template>
  <div>
    <div v-if="breakShow" class="ongoing-wrapper">
      <el-table :data="ongoingExams" style="width: 100%" size="mini" border>
        <el-table-column prop="title" label="考试名称" min-width="220" show-overflow-tooltip />
        <el-table-column label="剩余时间" align="center" min-width="180">
          <template v-slot="scope">
            <span class="ongoing-timer">{{ scope.row.remainingText }}</span>
          </template>
        </el-table-column>
        <el-table-column label="操作" align="center" width="160">
          <template v-slot="scope">
            <el-button type="danger" size="mini" icon="el-icon-caret-right" @click="continueExam(scope.row.id)">继续考试</el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <data-table ref="pagingTable" :options="options" :list-query="listQuery">
      <template #filter-content>
        <el-select v-model="listQuery.params.openType" class="filter-item" placeholder="开放类型" clearable>
          <el-option v-for="item in openTypes" :key="item.value" :label="item.label" :value="item.value" />
        </el-select>
        <el-input v-model="listQuery.params.title" placeholder="搜索考试名称" style="width: 200px;" class="filter-item" />
      </template>
      <template #data-columns>
        <el-table-column label="考试名称" prop="title" show-overflow-tooltip />
        <el-table-column label="考试类型" align="center">
          <template v-slot="scope">
            {{ scope.row.openType | examOpenType }}
          </template>
        </el-table-column>
        <el-table-column label="考试时间" width="220px" align="center">
          <template v-slot="scope">
            <span v-if="scope.row.timeLimit">
              {{ scope.row.startTime }} ~ {{ scope.row.endTime }}
            </span>
            <span v-else>不限时</span>
          </template>
        </el-table-column>
        <el-table-column label="考试时长" align="center">
          <template v-slot="scope">
            {{ scope.row.totalTime }}分钟
          </template>
        </el-table-column>
        <el-table-column label="考试总分" prop="totalScore" align="center" />
        <el-table-column label="及格线" prop="qualifyScore" align="center" />
        <el-table-column label="操作" align="center">
          <template v-slot="scope">
            <el-button v-if="scope.row.state===0" icon="el-icon-caret-right" type="primary" size="mini" @click="handlePre(scope.row.id)">开始考试</el-button>
            <el-button v-if="scope.row.state===1" icon="el-icon-s-release" size="mini" disabled>已禁用</el-button>
            <el-button v-if="scope.row.state===2" icon="el-icon-s-fold" size="mini" disabled>待开始</el-button>
            <el-button v-if="scope.row.state===3" icon="el-icon-s-unfold" size="mini" disabled>已结束</el-button>
          </template>
        </el-table-column>
      </template>
    </data-table>
  </div>
</template>
<script>
import DataTable from '@/components/DataTable'
import { checkProcess } from '@/api/paper/exam'

export default {
  components: { DataTable },
  data() {
    return {
      breakShow: false,
      breakId: '',
      ongoingExams: [],
      countdownTimers: {},
      openTypes: [
        {
          value: 1,
          label: '完全开放'
        },
        {
          value: 2,
          label: '定向考试'
        }
      ],
      listQuery: {
        current: 1,
        size: 10,
        params: {}
      },
      options: {
        multi: false,
        listUrl: '/exam/api/exam/exam/online-paging'
      }
    }
  },
  created() {
    this.check()
  },
  beforeDestroy() {
    this.clearCountdowns()
  },
  methods: {
    handlePre(examId) {
      this.$router.push({ name: 'PreExam', params: { examId: examId }})
    },
    continueExam(examId) {
      this.$router.push({ name: 'StartExam', params: { id: examId }})
    },
    check() {
      checkProcess().then(res => {
        this.clearCountdowns()
        if (res.data && res.data.id) {
          this.breakShow = true
          this.breakId = res.data.id
          const examRow = this.normalizeOngoingExam(res.data)
          this.ongoingExams = [examRow]
          this.setupCountdown(examRow)
        } else {
          this.breakShow = false
          this.ongoingExams = []
        }
      })
    },
    normalizeOngoingExam(examData) {
      const remainingSeconds = this.calcRemainingSeconds(examData)
      return {
        id: examData.id,
        title: examData.title || examData.examName || '正在进行的考试',
        remainingSeconds: remainingSeconds,
        remainingText: this.formatRemaining(remainingSeconds)
      }
    },
    calcRemainingSeconds(examData) {
      if (examData && examData.limitTime) {
        const timestamp = new Date(examData.limitTime).getTime()
        const diff = Math.floor((timestamp - Date.now()) / 1000)
        return diff
      }
      const total = Number(examData.totalTime || 0) * 60
      const used = Number(examData.userTime || 0) * 60
      return total - used
    },
    formatRemaining(seconds) {
      if (seconds === undefined || seconds === null || seconds <= 0) {
        return '00分钟00秒'
      }
      const min = parseInt(seconds / 60)
      const sec = parseInt(seconds % 60)
      const minText = min > 9 ? min : '0' + min
      const secText = sec > 9 ? sec : '0' + sec
      return `${minText}分钟${secText}秒`
    },
    setupCountdown(exam) {
      if (!exam || exam.remainingSeconds === undefined) {
        return
      }
      this.clearTimer(exam.id)
      if (exam.remainingSeconds <= 0) {
        exam.remainingText = this.formatRemaining(0)
        return
      }
      this.countdownTimers[exam.id] = setInterval(() => {
        exam.remainingSeconds -= 1
        if (exam.remainingSeconds <= 0) {
          exam.remainingSeconds = 0
          exam.remainingText = this.formatRemaining(0)
          this.clearTimer(exam.id)
        } else {
          exam.remainingText = this.formatRemaining(exam.remainingSeconds)
        }
      }, 1000)
    },
    clearTimer(id) {
      if (this.countdownTimers[id]) {
        clearInterval(this.countdownTimers[id])
        delete this.countdownTimers[id]
      }
    },
    clearCountdowns() {
      Object.keys(this.countdownTimers).forEach(id => this.clearTimer(id))
    }
  }
}
</script>

<style scoped>
.ongoing-wrapper {
  padding: 20px;
}

.ongoing-timer {
  color: #ff0000;
  font-weight: 700;
}
</style>
