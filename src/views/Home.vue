<template lang="pug">
  v-app
    v-content
      v-container(fill-height)
        v-row
          v-col
            v-alert(v-if='color=="error"', dense, outlined, type='error')
              | 警告：你正在透支明天！
            template(v-if='circular')
              .text-center
                v-progress-circular(
                  :size='280'
                  :width='40'
                  :value='totalProgress'
                  :color='color'
                )
                  v-progress-circular(
                    :size='180'
                    :width='20'
                    :value='workHourProgressDisplay'
                    :color='color'
                    :rotate='workHourRotation'
                  )
                    strong.headline
                      template(v-if='workHourProgressDisplay > 0') {{ workHourProgressDisplay }}%
                      p {{ totalProgress }}%
                      v-btn(text, icon, color='grey', @click='dialog=true')
                        v-icon {{ mdiCog }}
                template(v-if='workTimeLeft > 0')
                  p.mt-3 工作剩余 {{ totalWorkTimeLeft }}
                template(v-if='isNowSleepTime')
                  p.mt-3 睡眠经过 {{ totalTimeToTargetStr }}
                template(v-else)
                  p.mt-3 全天剩余 {{ totalTimeToTargetStr }}

            template(v-else)
              v-progress-linear(
                :reverse='true'
                :value="totalProgress"
                :color='color'
                height="65")
              v-row
                v-col
                  strong {{ totalProgress }}%
                  p {{ totalTimeToTargetStr }}
                  //- p now: {{ now.calendar() }}
                  //- p 开始时间: {{ startRaw.calendar() }}
                  //- p 结束时间: {{ endDayRaw.calendar() }}
                  //- p 一天开始: {{ newDayBeginsRaw.calendar() }}
                v-col
                  v-btn(text, icon, color='grey', @click='dialog=true').float-right
                    v-icon {{ mdiCog }}
        v-dialog(v-model='dialog', max-width='800', persistent)
          v-card
            v-card-title 设置
              v-spacer
              v-btn(text ,icon, @click='save')
                v-icon {{ mdiClose }}
            v-card-text
              v-form
                v-row
                  v-col
                    v-switch(label='暗黑模式', v-model='$vuetify.theme.dark')
                    v-switch(label='环形进度条', v-model='circular')
                    //- v-text-field(label='起床时间')
                    TimePicker(v-model='start', label='起床时间')
                    TimePicker(v-model='end', label='睡觉时间')
                    v-divider.my-2
                    .subheader 高级选项
                    TimePicker(v-model='newDayBegins', label='新一天开始时间')
                    TimePicker(v-model='offWork', label='下班时间')
                    v-switch(label='工作时间进度同步日进度', v-model='workProessSyncDay')                    
            v-card-actions
              v-spacer
              v-btn(color='primary', @click='save') 保存
                      
</template>

<script>
// @ is an alias to /src
import HelloWorld from '@/components/HelloWorld.vue'
import TimePicker from '@/components/TimePicker.vue'
import dayjs from 'dayjs'
import { mdiCog, mdiClock, mdiClose } from '@mdi/js'
export default {
  name: 'Home',
  data(){
    return {
      dialog: false, //设置弹窗
      start: localStorage.getItem('settings.startDay') || '9:00', //早上9点
      end: localStorage.getItem('settings.end') || '22:00', //晚上10点
      color: 'primary', // 进度条颜色
      circular: localStorage.getItem('settings.circular') == 'true', //圆环进度条
      now: dayjs(), //dayjs().set('hour', 1).set('minute',0).set('second', 45), 
      newDayBegins: localStorage.getItem('settings.newDayBegins') || '0:00', //两天的分界线
      offWork: localStorage.getItem('settings.offWork') || '19:00', //下班时间
      workProessSyncDay: localStorage.getItem('settings.workProessSyncDay') == null ? true : localStorage.getItem('settings.workProessSyncDay'), //上班进度是否同步一天进度显示
      mdiCog,
      mdiClock,
      mdiClose
    }
  },
  created(){
    setInterval(()=>this.now = dayjs(),1000)
  },
  computed:{
    newDayBeginsRaw(){
      let newDayBeginsStrArr = this.newDayBegins.split(':')
      let newDayBegins = this.now.set('hour', newDayBeginsStrArr[0]).set('minute', newDayBeginsStrArr[1]).set('second', 0)
      return newDayBegins
    },
    startRaw(){
      let startStrArr = this.start.split(':').map(i=>parseInt(i))
      let startDate = this.now.set('hour', startStrArr[0]).set('minute', startStrArr[1]).set('second', 0)
      return startDate
    },
    endRaw(){
      let endStrArr = this.end.split(':').map(i=>parseInt(i))
      let endDate = this.now.set('hour', endStrArr[0]).set('minute', endStrArr[1]).set('second', 0)
      return endDate
    },
    lastEndRaw(){
      return this.endRaw.subtract(1, 'day')
    },
    trueEnd(){
      if (this.isEndBeforeStart && !this.now.isBefore(this.endRaw)) {
        return this.endRaw.add(1, 'day')
      }
      return this.endRaw
    },
    trueLastEnd(){
      return this.trueEnd.subtract(1, 'day')
    },
    isEndBeforeStart(){
      return this.endRaw.isBefore(this.startRaw)
    },
    timePastEnd(){
      let lastEnd
      if (this.now.isBefore(this.endRaw)) {
        lastEnd = this.endRaw.subtract(1, 'day')
      } else {
        lastEnd = this.endRaw
      }
      return this.now.unix() - lastEnd.unix()
    },
    dayDuration(){
      return this.trueEnd.unix() - this.startRaw.unix()
    },
    isNowSleepTime(){
      let end = this.endRaw
      let start
      if (this.isEndBeforeStart) {
        start = this.startRaw
      } else {
        start = this.startRaw.add(1, 'day')
      }
      if (this.now.isBefore(this.endRaw)) {
        end = end.subtract(1, 'day')
        start = start.subtract(1, 'day')
      }
      return !this.now.isBefore(end) && this.now.isBefore(start)
    },
    totalTimeToTargetStr(){
      let timeLeft
      if(this.isNowSleepTime) {
        let sleepTimePoint = this.now.subtract(this.timePastEnd, 'second')
        timeLeft = this.now.to(sleepTimePoint)
      } else {
        timeLeft = this.now.to(this.trueEnd)
      }
      return timeLeft.substr(0, timeLeft.length - 1)
    },
    totalProgress(){
      let dayBegin = this.newDayBeginsRaw.isBefore(this.startRaw) ? this.newDayBeginsRaw : this.newDayBeginsRaw.add(1, 'day')
      // 更新颜色
      if (this.isNowSleepTime && this.now.isBefore(dayBegin)) {
        this.color = 'error'
      } else {
        this.color = 'primary'
      }

      // 更新进度
      let progress
      if (this.isNowSleepTime) {
        progress = Math.abs(this.timePastEnd/(this.trueLastEnd.unix() - this.startRaw.unix()))
      } else {
        let deltaEnd = this.now.unix() - this.trueEnd.unix()
        let totalDuration = this.startRaw.unix() - this.trueEnd.unix()
        progress = Math.abs(deltaEnd/totalDuration)
      }
      progress = progress * 100
      progress = progress.toFixed(2)
      return progress
    },
    endWorkRaw(){
      let offWork = this.offWork.split(':').map(i=>parseInt(i))
      offWork = this.now.set('hour', offWork[0]).set('minute', offWork[1]).set('second', 0)
      return offWork
    },
    isWorkEndBeforeStart(){
      return this.endWorkRaw.isBefore(this.startRaw)
    },
    trueWorkEnd(){
      if (this.isWorkEndBeforeStart && !this.now.isBefore(this.endWorkRaw)) {
        return this.endWorkRaw.add(1, 'day')
      }
      return this.endWorkRaw
    },
    trueLastWorkEnd(){
      return this.trueWorkEnd.subtract(1, 'day')
    },
    workTimeLeft(){
      if (this.now.isBefore(this.trueLastWorkEnd)) {
        return Math.max(this.trueLastWorkEnd.unix() - this.now.unix(), 0)
      }else if (this.now.isBefore(this.startRaw)) {
        return 0
      } else {
        return Math.max(this.trueWorkEnd.unix() - this.now.unix(), 0)
      }
    },
    workHourDuration(){
      let workHourDuration = Math.max(this.trueWorkEnd.unix() - this.startRaw.unix(), 0)
      return workHourDuration
    },
    workHourPercentage(){
      let dayDuration = Math.max(this.trueEnd.unix() - this.startRaw.unix(), 1)
      return this.workHourDuration / dayDuration
    },
    workHourRotation(){
      if (this.workProessSyncDay) {
        return 360 * (1 - this.workHourPercentage)
      } else {
        return 0
      }
    },
    workHourProgress() {
      let progress = this.workTimeLeft / this.workHourDuration
      progress = progress * 100
      return progress
    },
    workHourProgressDisplay(){
      let progress = this.workHourProgress
      if(this.workProessSyncDay) {
        progress = progress * this.workHourPercentage
      }
      progress = progress.toFixed(2)
      return progress
    },
    totalWorkTimeLeft(){
      let timeLeft = this.now.to(this.trueWorkEnd)
      return timeLeft.substr(0, timeLeft.length - 1)
    }
  },
  methods:{
    save(){
      localStorage.setItem('settings.startDay', this.start)
      localStorage.setItem('settings.end', this.end)
      localStorage.setItem('settings.dark', this.$vuetify.theme.dark)
      localStorage.setItem('settings.newDayBegins', this.newDayBegins)
      localStorage.setItem('settings.circular', this.circular)
      localStorage.setItem('settings.workProessSyncDay', this.workProessSyncDay)
      this.dialog=false
    }
  },
  components: {
    HelloWorld,
    TimePicker
  }
}
</script>
