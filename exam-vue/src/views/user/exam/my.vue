<template>

  <div>
    <data-table
      ref="pagingTable"
      :options="options"
      :list-query="listQuery"
    >
      <template #filter-content>

        <el-input v-model="listQuery.params.title" placeholder="搜索考试名称" style="width: 200px;" class="filter-item" />

      </template>

      <template #data-columns>

        <el-table-column
          label="考试名称"
          prop="title"
          show-overflow-tooltip
        />

        <el-table-column
          label="考试次数"
          prop="tryCount"
          align="center"
        />

        <el-table-column
          label="最高分"
          prop="maxScore"
          align="center"
        />
        <!-- [新增] 最低分：展示后端计算的历史最低成绩 -->
<!--        <el-table-column-->
<!--          label="最低分"-->
<!--          prop="minScore"-->
<!--          align="center"-->
<!--        />-->

        <!-- [新增] 及格分：展示该考试的及格标准 -->
<!--        <el-table-column-->
<!--          label="及格分"-->
<!--          prop="qualifyScore"-->
<!--          align="center"-->
<!--        />-->

        <el-table-column
          label="是否通过"
          align="center"
        >

          <template v-slot="scope">
            <span v-if="scope.row.passed" style="color: #00ff00;">通过</span>
            <span v-else style="color: #ff0000;">未通过</span>
          </template>

        </el-table-column>

        <el-table-column
          label="最后考试时间"
          prop="updateTime"
          align="center"
        />

        <el-table-column
          label="操作"
          align="center"
          min-width="100"
        >
          <template v-slot="scope">
            <el-button type="primary" size="mini" icon="el-icon-view" @click="handleExamDetail(scope.row.examId)">详情</el-button>
            <el-button type="warning" size="mini" icon="el-icon-close" @click="handlerExamBook(scope.row.examId)">错题</el-button>
            <!--趋势图按钮：点击触发 handleTrend 方法 -->
            <el-button type="success" size="mini" icon="el-icon-data-line" @click="handleTrend(scope.row)">成绩分析</el-button>
          </template>

        </el-table-column>

      </template>

    </data-table>

    <el-dialog :visible.sync="dialogVisible" title="考试明细" width="60%">

      <div class="el-dialog-div">
        <my-paper-list :exam-id="examId" :user-id="userId" />
      </div>

    </el-dialog>
    <!-- 成绩分析弹窗 -->
    <el-dialog :visible.sync="trendVisible" title="成绩全维分析" width="900px">
      <el-row :gutter="20">
        <!-- 折线图容器 -->
        <el-col :span="14">
          <div id="trendChart" style="width: 100%; height: 400px;"></div>
        </el-col>
        <!-- 雷达图容器 -->
        <el-col :span="10">
          <div id="radarChart" style="width: 100%; height: 400px;"></div>
        </el-col>
      </el-row>
    </el-dialog>

  </div>

</template>

<script>
import DataTable from '@/components/DataTable'
import MyPaperList from './paper'
import { mapGetters } from 'vuex'

import * as echarts from 'echarts'
import { listPaper } from '@/api/paper/paper'

export default {
  name: 'MyExamList',
  components: { MyPaperList, DataTable },
  data() {
    return {

      dialogVisible: false,
      examId: '',
      //趋势图相关变量
      trendVisible: false,
      chartInstance: null,
      radarInstance: null,

      listQuery: {
        current: 1,
        size: 10,
        params: {
          title: ''
        }
      },

      options: {
        // 可批量操作
        multi: false,
        // 列表请求URL
        listUrl: '/exam/api/user/exam/my-paging'
      }
    }
  },
  computed: {
    ...mapGetters([
      'userId'
    ])
  },
  methods: {

    // 开始考试
    handleExamDetail(examId) {
      this.examId = examId
      this.dialogVisible = true
    },

    handlerExamBook(examId) {
      this.$router.push({ name: 'BookList', params: { examId: examId }})
    },

    //查看趋势图核心逻辑
    handleTrend(row) {
      this.trendVisible = true // 打开弹窗

      this.$nextTick(() => {
        // 1. 初始化图表实例
        if (!this.chartInstance) this.chartInstance = echarts.init(document.getElementById('trendChart'))
        if (!this.radarInstance) this.radarInstance = echarts.init(document.getElementById('radarChart'))


        // 2. 查数据
        listPaper(this.userId, row.examId).then(res => {

          //数据反转：按时间从旧到新排列
          const records = res.data.records.reverse()

          //画左边的折线图
          const xData = records.map((item, index) => `第${index + 1}次`)
          const yData = records.map(item => item.userScore)

          this.chartInstance.setOption({
            title: { text: `${row.title} - 走势`, left: 'center' },
            tooltip: { trigger: 'axis' },
            xAxis: { type: 'category', data: xData },
            yAxis: { type: 'value' },
            series: [{
              data: yData, type: 'line', smooth: true,
            }]
          })

          //画右边的雷达图
          if (records.length > 0) {
            //计算单次考试的三个维度的函数
            const calcDimensions = (exam) => {
              const totalScore = exam.totalScore || 100;
              const myScore = exam.userScore || 0;
              let acc = (myScore / totalScore * 100); // 得分率

              const totalTime = exam.totalTime || 60;
              const userTime = (exam.userTime === undefined || exam.userTime === null) ? (totalTime/2) : exam.
                userTime;
              let eff = 0;
              if (totalTime > 0) eff = (totalTime - userTime) / totalTime * 100; // 效率

              let objTotal;//总分
              if (exam.hasSaq === false) {
                // 没简答题
                objTotal = totalScore;
              } else {
                // 有简答题，但是又不知道没一张卷子的客观题比例，所以就估计为80%为客观题
                objTotal = totalScore * 0.8;
              }

              //得分
              let objScore = exam.objScore;
              if (objScore === undefined || objScore === null) {
                objScore = (myScore / totalScore * objTotal);
              }

              // 算出百分比
              let solid = 0;
              if (objTotal > 0) {
                solid = (objScore / objTotal * 100);
              }

              return {
                acc: Number(acc.toFixed(1)), // 保留一位小数
                eff: Number(eff.toFixed(1)),//由于toFixed返回的是字符串，所以得转化为数字，不然雷达图显示不出来
                solid: Number(solid.toFixed(1))
              };
            };

            //计算历史平均
            let sumAcc = 0, sumEff = 0, sumSolid = 0;
            records.forEach(r => {
              const dims = calcDimensions(r);
              sumAcc += dims.acc;
              sumEff += dims.eff;
              sumSolid += dims.solid;
            });
            const count = records.length;
            const avgData = [
              (sumAcc / count).toFixed(1),
              (sumEff / count).toFixed(1),
              (sumSolid / count).toFixed(1)
            ];

            //计算最近一次
            const lastExam = records[records.length - 1]; //因为之前reverse过
            const lastDims = calcDimensions(lastExam);
            const lastData = [
              lastDims.acc.toFixed(1),//不需要转化为数字是因为echarts会自动把字符串先尝试转化为数字，之所以之前需要加，是因为中间还有一个运算过程，运算过程如果按照字符串计算就会出错
              lastDims.eff.toFixed(1),
              lastDims.solid.toFixed(1)
            ];

            const radarOption = {
              title: { text: '能力雷达图', left: 'center', top: '20px' },
              tooltip: {},
              legend: { bottom: '5px', data: ['最近一次', '历史平均'] },
              radar: {
                indicator: [
                  { name: '得分能力', max: 100 },
                  { name: '答题速率', max: 100 },
                  { name: '基础功底', max: 100 }
                ],
                center: ['50%', '55%'],
                radius: '60%'
              },
              series: [{
                name: '能力雷达图',
                type: 'radar',
                data: [
                  {
                    value: lastData,
                    name: '最近一次',
                    areaStyle: { color: 'rgba(255, 165, 0, 0.5)' },
                    itemStyle: { color: 'orange' }
                  },
                  {
                    value: avgData,
                    name: '历史平均',
                    lineStyle: { type: 'dashed' }, // 虚线表示平均
                    itemStyle: { color: '#409EFF' }
                  }
                ]
              }]
            };

            this.radarInstance.setOption(radarOption);
          }
        })
      })
    }
  }
}
</script>

<style scoped>

.el-dialog-div{
  height: 60vh;
  overflow: auto;
  padding: 10px;
}

</style>
