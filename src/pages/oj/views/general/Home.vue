<template>
  <Row type="flex" justify="space-around">
    <Col :span="22">
    <panel shadow v-if="contests.length" class="contest">
      <div slot="title">
        <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" type="text"  class="contest-title" @click="goContest">{{title_status}}</Button>
      </div>
      <Carousel v-model="index" trigger="hover" autoplay :autoplay-speed="6000" class="contest">
        <CarouselItem v-for="(contest, index) of contests" :key="index">
          <div class="contest-content">
            <div class="contest-content-tags">
              <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" type="info" shape="circle" size="small" icon="calendar">
                {{contest.start_time | localtime('YYYY-M-D HH:mm') }}
              </Button>
              <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" type="success" shape="circle" size="small" icon="android-time">
                {{getDuration(contest.start_time, contest.end_time)}}
              </Button>
              <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" type="warning" shape="circle" size="small" icon="trophy">
                {{contest.rule_type}}
              </Button>
            </div>
            <div class="contest-content-description">
              <blockquote v-html="contest.description"></blockquote>
            </div>
          </div>
        </CarouselItem>
      </Carousel>
    </panel>
    <Announcements class="announcement"></Announcements>
    </Col>
  </Row>
</template>

<script>
  import Announcements from './Announcements.vue'
  import api from '@oj/api'
  import time from '@/utils/time'
  import { CONTEST_STATUS } from '@/utils/constants'
  import { mapGetters } from 'vuex'

  export default {
    name: 'home',
    components: {
      Announcements
    },
    data () {
      return {
        contests: [],
        index: 0
      }
    },
    mounted () {
      let params = { status: CONTEST_STATUS.UNDERWAY + ' ' + CONTEST_STATUS.NOT_START } // 成功修改后端做了个可以展示多种类型比赛的api
      api.getContestList(0, 5, params).then(res => {
        this.contests = res.data.data.results
      })
    },
    methods: {
      getDuration (startTime, endTime) {
        return time.duration(startTime, endTime)
      },
      goContest () {
        this.$router.push({
          name: 'contest-details',
          params: {contestID: this.contests[this.index].id}
        })
      }
    },
    computed: {
      ...mapGetters(['theme']),
      title_status () {
        return this.contests[this.index].title + '（' +
        (this.contests[this.index].status === CONTEST_STATUS.NOT_START
        ? this.$i18n.t('m.Not_Started') : this.$i18n.t('m.Underway')) + '）'
      }
    }
  }
</script>

<style lang="less" scoped>
  .contest {
    &-title {
      font-style: italic;
      font-size: 21px;
    }
    &-content {
      padding: 0 70px 40px 70px;
      &-description {
        margin-top: 25px;
      }
    }
  }

  .announcement {
    margin-top: 20px;
  }
</style>
