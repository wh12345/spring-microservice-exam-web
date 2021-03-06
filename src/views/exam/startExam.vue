<template>
  <div>
    <el-row :gutter="30">
      <el-col :span="18" :offset="2">
        <el-card class="subject-box-card" v-loading="loading">
          <div class="subject-exam-title" v-if="!loading && tempSubject.id !== ''">{{exam.examinationName}}（共{{exam.subjectCount}}题，合计{{exam.totalScore}}分）</div>
          <div class="subject-content" v-if="!loading && tempSubject.id !== ''">
            <div class="subject-title">
              <span class="subject-title-number">{{tempSubject.serialNumber}}.&nbsp;</span>
              <span class="subject-title-content" v-html="tempSubject.subjectName"></span>
              <span class="subject-title-content">&nbsp;({{tempSubject.score}})分</span>
            </div>
            <ul v-if="tempSubject.type === 0" class="subject-options">
              <li class="subject-option">
                <input :checked="answer === 'A'" class="toggle" type="checkbox" @change="toggleOption('A')">
                <label @click="toggleOption('A')"><span class="subject-option-prefix">A.&nbsp;</span><span v-html="tempSubject.options[0].optionContent" class="subject-option-prefix"></span></label>
              </li>
              <li class="subject-option">
                <input :checked="answer === 'B'" class="toggle" type="checkbox" @change="toggleOption('B')">
                <label @click="toggleOption('B')"><span class="subject-option-prefix">B.&nbsp;</span><span v-html="tempSubject.options[1].optionContent" class="subject-option-prefix"></span></label>
              </li>
              <li class="subject-option">
                <input :checked="answer === 'C'" class="toggle" type="checkbox" @change="toggleOption('C')">
                <label @click="toggleOption('C')"><span class="subject-option-prefix">C.&nbsp;</span><span v-html="tempSubject.options[2].optionContent" class="subject-option-prefix"></span></label>
              </li>
              <li class="subject-option">
                <input :checked="answer === 'D'" class="toggle" type="checkbox" @change="toggleOption('D')">
                <label @click="toggleOption('D')"><span class="subject-option-prefix">D.&nbsp;</span><span v-html="tempSubject.options[3].optionContent" class="subject-option-prefix"></span></label>
              </li>
            </ul>
            <div v-if="tempSubject.type === 1" class="subject-answer">
              <tinymce ref="answerEditor" :height="300" v-model="answer"/>
            </div>
          </div>
          <div class="subject-buttons" v-if="!loading && tempSubject.id !== ''">
            <el-button plain @click="last">上一题</el-button>
            <el-button v-if="tempSubject.serialNumber !== subjectCount" plain @click="next">下一题</el-button>
            <el-button v-if="tempSubject.serialNumber !== 0 && tempSubject.serialNumber === subjectCount" type="success" @click="submitExam" v-bind:disabled="disableSubmit">提交</el-button>
          </div>
        </el-card>
      </el-col>
      <el-col :span="2">
        <div class="time-remain">
          剩余时间:
          <div class="time">
            <count-down v-on:start_callback="countDownS_cb(1)" v-on:end_callback="countDownE_cb(1)" :current-time="currentTime" :start-time="startTime" :end-time="endTime" :tip-text="'距离考试开始'" :tip-text-end="'距离考试结束'" :end-text="'考试结束'" :hourTxt="':'" :minutesTxt="':'" :secondsTxt="''">
            </count-down>
          </div>
        </div>
        <div class="current-progress">
          当前进度: {{tempSubject.serialNumber}}/{{subjectCount}}
        </div>
        <div class="answer-card">
          <el-button type="text" icon="el-icon-date" @click="answerCard">答题卡</el-button>
        </div>
        <el-button type="success" icon="el-icon-date" @click="submitExam" v-bind:disabled="disableSubmit">提交</el-button>
      </el-col>
    </el-row>
    <el-dialog title="答题卡" :visible.sync="dialogVisible" width="50%" top="10vh" center>
      <div class="answer-card-title" >{{exam.examinationName}}（共{{subjectCount}}题，合计{{exam.totalScore}}分）</div>
      <div class="answer-card-split"></div>
      <el-row class="answer-card-content">
        <el-button circle v-for="index in subjectCount" :key="index" @click="toSubject(index)" >&nbsp;{{index}}&nbsp;</el-button>
      </el-row>
    </el-dialog>
  </div>
</template>
<script>
import { mapState, mapGetters } from 'vuex'
import CountDown from 'vue2-countdown'
import { saveAndNext } from '@/api/exam/answer'
import { getSubjectCount } from '@/api/exam/exam'
import { getCurrentTime } from '@/api/exam/examRecord'
import store from '@/store'
import moment from 'moment'
import { notifySuccess, notifyFail, isNotEmpty } from '@/utils/util'
import Tinymce from '@/components/Tinymce'

export default {
  components: {
    CountDown,
    Tinymce
  },
  data () {
    return {
      loading: false,
      currentTime: 0,
      startTime: 0,
      endTime: 0,
      disableSubmit: true,
      tempSubject: {
        id: '',
        serialNumber: 1,
        examinationId: '',
        categoryId: 0,
        subjectName: '',
        type: 0,
        choicesType: 0,
        options: [
          { subjectChoicesId: '', optionName: 'A', optionContent: '' },
          { subjectChoicesId: '', optionName: 'B', optionContent: '' },
          { subjectChoicesId: '', optionName: 'C', optionContent: '' },
          { subjectChoicesId: '', optionName: 'D', optionContent: '' }
        ],
        answer: {
          subjectId: '',
          answer: '',
          answerType: '',
          score: '',
          type: 0
        },
        score: 5,
        analysis: '',
        level: 2
      },
      query: {
        serialNumber: 1,
        examRecordId: '',
        userId: ''
      },
      option: '',
      answer: '',
      dialogVisible: false,
      tempAnswer: {
        id: '',
        userId: '',
        examinationId: '',
        courseId: '',
        subjectId: '',
        answer: '',
        type: 0
      },
      subjectCount: 0
    }
  },
  computed: {
    // 获取用户信息
    ...mapState({
      userInfo: state => state.user.userInfo,
      examRecord: state => state.exam.examRecord
    }),
    ...mapGetters([
      'exam', 'subject'
    ])
  },
  created () {
    // 考试ID
    this.query.examinationId = this.exam.id
    // 考试记录ID
    this.query.examRecordId = this.examRecord.id
    // 用户ID
    this.query.userId = this.userInfo.id
    // 开始考试
    this.startExam()
  },
  methods: {
    countDownS_cb: function (x) {
      this.$notify({
        title: '提示',
        message: '考试开始',
        type: 'warn',
        duration: 2000
      })
    },
    // 开始考试
    startExam () {
      // 校验考试时间
      getCurrentTime().then(response => {
        const currentTime = moment(response.data.data)
        if (currentTime.isAfter(this.exam.endTime)) {
          this.$notify({
            title: '提示',
            message: '考试已结束',
            type: 'warn',
            duration: 2000
          })
        } else if (currentTime.isBefore(this.exam.startTime)) {
          // 考试未开始
          this.$notify({
            title: '提示',
            message: '考试未开始',
            type: 'warn',
            duration: 2000
          })
        } else {
          // 获取考试的题目数量
          getSubjectCount(this.exam.id).then(response => {
            // 题目数
            this.subjectCount = response.data.data
          })
          // 获取服务器的当前时间
          this.currentTime = currentTime.valueOf()
          // 考试开始时间
          this.startTime = this.currentTime
          // 考试结束时间
          this.endTime = moment(this.exam.endTime).valueOf()
          this.disableSubmit = false
          // 初始化题目和答题
          this.tempSubject = this.subject
          // 答题
          this.tempAnswer = this.tempSubject.answer
          // 选项
          this.answer = this.tempAnswer.answer || ''
          // 题号
          this.query.serialNumber = this.subject.serialNumber
        }
      }).catch(() => {
        notifyFail(this, '开始考试失败！')
      })
    },
    // 考试结束
    countDownE_cb: function (x) {
      this.$notify({
        title: '提示',
        message: '考试结束',
        type: 'warn',
        duration: 2000
      })
      this.disableSubmit = true
      this.loading = false
    },
    // 上一题
    last () {
      if (this.query.serialNumber === 1) {
        this.$notify({
          title: '提示',
          message: '已经是第一题了',
          type: 'warn',
          duration: 2000
        })
      } else {
        // 题目序号减一
        this.query.serialNumber--
        // 保存当前题目，同时加载下一题
        this.saveCurrentSubjectAndGetNextSubject()
      }
    },
    // 下一题
    next () {
      // 增加序号
      this.query.serialNumber++
      // 保存当前题目，同时加载下一题
      this.saveCurrentSubjectAndGetNextSubject()
    },
    // 保存当前题目，同时根据序号加载下一题
    saveCurrentSubjectAndGetNextSubject () {
      this.loading = true
      let answerId = isNotEmpty(this.tempAnswer) ? this.tempAnswer.id : ''
      // 构造答案
      let answer = this.getAnswer(answerId)
      // 提交到后台
      saveAndNext(answer).then(response => {
        if (response.data.data === null) {
          notifySuccess(this, '已经是最后一题了')
          this.query.serialNumber--
        } else {
          // 题目内容
          this.tempSubject = response.data.data
          // 答题
          this.tempAnswer = response.data.data.answer
          this.answer = isNotEmpty(this.tempAnswer) ? this.tempAnswer.answer : ''
          // 保存题目答案到localStorage
          this.subject.answer = this.tempAnswer
          store.dispatch('SetSubjectInfo', this.tempSubject).then(() => {})
        }
        this.loading = false
      }).catch(() => {
        notifyFail(this, '加载题目失败')
        this.loading = false
      })
    },
    // 答题卡
    answerCard () {
      this.dialogVisible = true
    },
    // 跳转题目
    toSubject (index) {
      this.query.serialNumber = index
      // 保存当前题目，同时加载下一题
      this.saveCurrentSubjectAndGetNextSubject()
      this.dialogVisible = false
    },
    // 提交
    submitExam () {
      this.$confirm('确定要提交吗?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        let answerId = isNotEmpty(this.tempAnswer) ? this.tempAnswer.id : ''
        // 构造答案
        let answer = this.getAnswer(answerId)
        saveAndNext(answer).then(response => {
          // 提交到后台
          store.dispatch('SubmitExam', { examinationId: this.exam.id, examRecordId: this.examRecord.id, userId: this.userInfo.id }).then(() => {
            notifySuccess(this, '提交成功')
            // 禁用提交按钮
            this.disableSubmit = true
            this.$router.push({name: 'exam-record'})
          }).catch(() => {
            notifyFail(this, '提交失败')
          })
        }).catch(() => {
          notifyFail(this, '提交题目失败')
        })
      })
    },
    // 选中选项
    toggleOption (answer) {
      this.answer = answer
    },
    // 根据题目类型返回填写的答案
    getAnswer (answerId) {
      return {
        id: answerId,
        userId: this.userInfo.id,
        examinationId: this.exam.id,
        examRecordId: this.examRecord.id,
        subjectId: this.tempSubject.id,
        answer: this.answer,
        type: this.tempSubject.type,
        serialNumber: this.query.serialNumber
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss" rel="stylesheet/scss" scoped>
  @import "../../assets/css/common.scss";
  .start-exam-msg {
    @extend %message-common;
  }
  .subject-box-card {
    margin-bottom: 30px;
    min-height: 400px;
    .subject-exam-title{
      font-size: 18px;
      line-height: 25px;
      padding: 18px 20px;
      border-bottom: 1px solid #DEDEDE;
      margin-bottom: 12px;
    }
    .subject {
      padding-left: 30px;
      padding-right: 75px;
    }
    .subject-content{
      margin: 0 auto ;
      -webkit-font-smoothing: antialiased;
      -moz-osx-font-smoothing: grayscale;
      font-weight: 300;
      background: #fff;
      z-index: 1;
      position: relative;
    }
    /* 题目 */
    .subject-title {
      font-size: 18px;
      line-height: 22px;
      .subject-title-number {
        display: inline-block;
        line-height: 22px;
      }
      .subject-title-content {
        display: inline-block;
      }
    }
    .subject-options {
      margin: 0;
      padding: 0;
      list-style: none;
      > li {
        position: relative;
        font-size: 24px;
        .toggle {
          opacity: 0;
          text-align: center;
          width: 35px;
          /* auto, since non-WebKit browsers doesn't support input styling */
          height: auto;
          position: absolute;
          top: 0;
          bottom: 0;
          margin: auto 0;
          border: none;
          /* Mobile Safari */
          -webkit-appearance: none;
          appearance: none;
        }
        .toggle+label {
          background-image: url('data:image/svg+xml;utf8,%3Csvg%20xmlns%3D%22http%3A//www.w3.org/2000/svg%22%20width%3D%2240%22%20height%3D%2240%22%20viewBox%3D%22-10%20-18%20100%20135%22%3E%3Ccircle%20cx%3D%2250%22%20cy%3D%2250%22%20r%3D%2250%22%20fill%3D%22none%22%20stroke%3D%22%23ededed%22%20stroke-width%3D%223%22/%3E%3C/svg%3E');
          background-repeat: no-repeat;
          background-position: center left;
          background-size: 30px;
        }
        .toggle:checked+label {
          background-size: 30px;
          background-image: url('data:image/svg+xml;utf8,%3Csvg%20xmlns%3D%22http%3A//www.w3.org/2000/svg%22%20width%3D%2240%22%20height%3D%2240%22%20viewBox%3D%22-10%20-18%20100%20135%22%3E%3Ccircle%20cx%3D%2250%22%20cy%3D%2250%22%20r%3D%2250%22%20fill%3D%22none%22%20stroke%3D%22%23bddad5%22%20stroke-width%3D%223%22/%3E%3Cpath%20fill%3D%22%235dc2af%22%20d%3D%22M72%2025L42%2071%2027%2056l-4%204%2020%2020%2034-52z%22/%3E%3C/svg%3E');
        }
        label {
          word-break: break-all;
          padding: 10px 10px 10px 45px;
          display: block;
          line-height: 1.0;
          transition: color 0.4s;
        }
        /* 选项名称 */
        .subject-option-prefix {
          font-size: 16px;
          display: inline-block
        }
      }
    }
    .subject-answer {
      padding: 16px;
    }
  }
  .subject-buttons {
    text-align: center;
  }
  .time-remain .time {
    font-size: 18px;
    line-height: 22px;
    color: #FF0000;
    font-weight: 400;
  }

  /* 答题卡 */
  .answer-card-title {
    font-size: 13px;
    color: #3A3E51;
    line-height: 17px;
    padding: 10px 0;
  }
  .answer-card-split {
    width: 100%;
    border-bottom: 1px solid #E6E6E6;
  }
  .answer-card-content {
    padding-bottom: 10px;
    font-size: 0;
    margin-right: -15px;
    > button {
      margin-top: 5px;
    }
  }
</style>
