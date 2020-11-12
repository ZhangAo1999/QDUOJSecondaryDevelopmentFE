<template>
  <Row type="flex">
    <Col :span="24">
    <Panel id="contest-card" shadow>
      <div slot="title">{{query.rule_type === '' ? this.$i18n.t('m.All') : query.rule_type}} {{$t('m.Contests')}}</div>
      <div slot="extra">
        <ul class="filter">
          <li>
            <Dropdown @on-click="onRuleChange">
              <span>{{query.rule_type === '' ? this.$i18n.t('m.Rule') : this.$i18n.t('m.' + query.rule_type)}}
                <Icon type="arrow-down-b"></Icon>
              </span>
              <Dropdown-menu slot="list">
                <Dropdown-item name="">{{$t('m.All')}}</Dropdown-item>
                <Dropdown-item name="OI">{{$t('m.OI')}}</Dropdown-item>
                <Dropdown-item name="ACM">{{$t('m.ACM')}}</Dropdown-item>
              </Dropdown-menu>
            </Dropdown>
          </li>
          <li>
            <Dropdown @on-click="onStatusChange">
              <span>{{query.status === '' ? this.$i18n.t('m.Status') : this.$i18n.t('m.' + CONTEST_STATUS_REVERSE[query.status].name.replace(/ /g,"_"))}}
                <Icon type="arrow-down-b"></Icon>
              </span>
              <Dropdown-menu slot="list">
                <Dropdown-item name="">{{$t('m.All')}}</Dropdown-item>
                <Dropdown-item :name="CONTEST_STATUS.UNDERWAY">{{$t('m.Underway')}}</Dropdown-item>
                <Dropdown-item :name="CONTEST_STATUS.NOT_START">{{$t('m.Not_Started')}}</Dropdown-item>
                <Dropdown-item :name="CONTEST_STATUS.ENDED">{{$t('m.Ended')}}</Dropdown-item>
              </Dropdown-menu>
            </Dropdown>
          </li>
          <li>
            <Input id="keyword" @on-enter="changeRoute" @on-click="changeRoute" v-model="query.keyword"
                   icon="ios-search-strong" placeholder="Keyword"/>
          </li>
        </ul>
      </div>
      <p id="no-contest" v-if="contests.length == 0">{{$t('m.No_contest')}}</p>
      <ol id="contest-list">
        <li v-for="contest in contests" :key="contest.title" @dblclick="editContest(contest)">
          <Row type="flex" justify="space-between" align="middle">
            <img class="trophy" src="../../../../assets/Cup.png"/>
            <Col :span="18" class="contest-main">
            <p v-show="currentContest.id !== contest.id" class="title">
              <a class="entry" :class="'entry-' + theme" @click.stop="goContest(contest)">
                {{contest.title}}
              </a>
              <template v-if="contest.contest_type != 'Public'">
                <Icon type="ios-locked-outline" size="20"></Icon>
              </template>
            </p>
            <div v-show="currentContest.id === contest.id" class="ivu-input-wrapper ivu-input-type ivu-input-group ivu-input-group-with-append ivu-input-hide-icon" style="width: 60%">
              <i class="ivu-icon ivu-icon-load-c ivu-load-loop ivu-input-icon ivu-input-icon-validate"></i>
              <input :value="currentContest.title" autocomplete="off" spellcheck="false" type="text" placeholder="" class="ivu-input" style="font-size:18px"
                     v-input-focus="contest === currentContest || !editPassword"
                     @keyup.esc="cancelEditTitle()"
                     @keyup.enter="finishEditTitle(contest, $event)"
                      >
              <div class="ivu-input-group-append" style="width:180px">
                <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" v-show="currentContest.contest_type !== 'Public'" type="button" class="ivu-btn ivu-btn-ghost ivu-btn-icon-only" @click="unlock(contest)" style="padding-left:4px">
                  <i class="ivu-icon ivu-icon-ios-locked-outline" style="font-size:20px"></i>
                </button>
                <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" v-show="currentContest.contest_type === 'Public'" type="button" class="ivu-btn ivu-btn-ghost ivu-btn-icon-only" @click="lock(contest)" style="padding-left:4px">
                  <i class="ivu-icon ivu-icon-ios-unlocked-outline" style="font-size:20px"></i>
                </button>
                <input :disabled="currentContest.contest_type === 'Public'" :value="currentContest.password" 
                     autocomplete="off" spellcheck="false" type="text" placeholder="" class="ivu-input" 
                     style="font-size: 14px;width:140px;float:right;height:24px"
                     v-input-focus="editPassword" @focus="editPassword = true"
                     @keyup.esc="cancelEditPassword()"
                     @keyup.enter="finishEditPassword($event)"
                      >
              </div>
            </div>
            <ul class="detail">
              <li>
                <Icon type="calendar" color="#3091f2"></Icon>
                {{contest.start_time | localtime('YYYY-M-D HH:mm') }}
              </li>
              <li>
                <Icon type="android-time" color="#3091f2"></Icon>
                {{getDuration(contest.start_time, contest.end_time)}}
              </li>
              <li>
                <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" size="small" shape="circle" @click="onRuleChange(contest.rule_type)">
                  {{contest.rule_type}}
                </Button>
              </li>
            </ul>
            </Col>
            <Col :span="4" style="text-align: center">
            <Tag type="dot" :color="CONTEST_STATUS_REVERSE[contest.status].color">{{$t('m.' + CONTEST_STATUS_REVERSE[contest.status].name.replace(/ /g, "_"))}}</Tag>
            </Col>
          </Row>
        </li>
      </ol>
    </Panel>
    <Pagination :total="total" :pageSize="limit" @on-change="getContestList" :current.sync="page"></Pagination>
    </Col>
  </Row>

</template>

<script>
  import apiAdmin from '@admin/api'
  import api from '@oj/api'
  import { mapGetters } from 'vuex'
  import utils from '@/utils/utils'
  import Pagination from '@/pages/oj/components/Pagination'
  import time from '@/utils/time'
  import { CONTEST_STATUS, CONTEST_STATUS_REVERSE, CONTEST_TYPE, USER_TYPE } from '@/utils/constants'

  const limit = 8

  export default {
    name: 'contest-list',
    components: {
      Pagination
    },
    data () {
      return {
        page: 1,
        query: {
          status: '',
          keyword: '',
          rule_type: ''
        },
        limit: limit,
        total: 0,
        rows: '',
        contests: [],
        currentContest: '',
        editPassword: false,
        CONTEST_STATUS_REVERSE: CONTEST_STATUS_REVERSE,
        CONTEST_STATUS: CONTEST_STATUS,
//      for password modal use
        cur_contest_id: ''
      }
    },
    beforeRouteEnter (to, from, next) {
      api.getContestList(0, limit).then((res) => {
        next((vm) => {
          vm.contests = res.data.data.results
          vm.total = res.data.data.total
        })
      }, (res) => {
        next()
      })
    },
    methods: {
      editContest (contest) {
        if (!this.isAdmin) { return }
        apiAdmin['getContest'](contest.id).then(res => {
          this.currentContest = res.data.data
        })
      },
      cancelEditTitle () {
        this.currentContest = ''
      },
      finishEditTitle (contest, event) {
        this.currentContest = ''
        const title = event.target.value.trim()
        if (contest.title === title) { return }
        apiAdmin['getContest'](contest.id).then(res => {
          contest = res.data.data
          contest.title = title
          apiAdmin['editContest'](contest).then(() => {
            this.init()
          })
        })
      },
      cancelEditPassword () {
        this.editPassword = false
      },
      finishEditPassword (event) {
        this.editPassword = false
        const password = event.target.value.trim()
        if (this.currentContest.password === password) { return }
        this.currentContest.password = password
        apiAdmin['editContest'](this.currentContest).then(() => {
          this.init()
        })
      },
      unlock (contest) {
        apiAdmin['getContest'](contest.id).then(res => {
          contest = res.data.data
          contest.contest_type = 'Public'
          contest.password = null
          apiAdmin['editContest'](contest).then(res => {
            this.currentContest.contest_type = 'Public'
            this.init()
          })
        })
      },
      lock (contest) {
        this.currentContest.contest_type = 'Password Protected'
        this.editPassword = true
      },
      init () {
        let route = this.$route.query
        this.query.status = route.status || ''
        this.query.rule_type = route.rule_type || ''
        this.query.keyword = route.keyword || ''
        this.page = parseInt(route.page) || 1
        this.getContestList()
      },
      getContestList (page = 1) {
        let offset = (page - 1) * this.limit
        api.getContestList(offset, this.limit, this.query).then((res) => {
          this.contests = res.data.data.results
          this.total = res.data.data.total
        })
      },
      changeRoute () {
        let query = Object.assign({}, this.query)
        query.page = this.page
        this.$router.push({
          name: 'contest-list',
          query: utils.filterEmptyValue(query)
        })
      },
      onRuleChange (rule) {
        this.query.rule_type = rule
        this.page = 1
        this.changeRoute()
      },
      onStatusChange (status) {
        this.query.status = status
        this.page = 1
        this.changeRoute()
      },
      goContest (contest) {
        this.cur_contest_id = contest.id
        if (contest.contest_type !== CONTEST_TYPE.PUBLIC && !this.isAuthenticated) {
          this.$error(this.$i18n.t('m.Please_login_first'))
          this.$store.dispatch('changeModalStatus', {visible: true})
        } else {
          this.$router.push({name: 'contest-details', params: {contestID: contest.id}})
        }
      },

      getDuration (startTime, endTime) {
        return time.duration(startTime, endTime)
      }
    },
    computed: {
      ...mapGetters(['isAuthenticated', 'user', 'theme']),
      isAdmin () {
        return this.user.admin_type === USER_TYPE.SUPER_ADMIN
      }
    },
    watch: {
      '$route' (newVal, oldVal) {
        if (newVal !== oldVal) {
          this.init()
        }
      }
    }

  }
</script>
<style lang="less" scoped>
  #contest-card {
    #keyword {
      width: 80%;
      margin-right: 30px;
    }
    #no-contest {
      text-align: center;
      font-size: 16px;
      padding: 20px;
    }
    #contest-list {
      > li {
        padding: 20px;
        border-bottom: 1px solid rgba(187, 187, 187, 0.5);
        list-style: none;

        .trophy {
          height: 40px;
          margin-left: 10px;
          margin-right: -20px;
        }
        .contest-main {
          .title {
            font-size: 18px;
            a.entry {
              color: #495060;
              &:hover {
                color: #2d8cf0;
                border-bottom: 1px solid #2d8cf0;
              }
            }
            a.entry-dark {
              color: #fff;
              &:hover {
                color: #2d8cf0;
                border-bottom: 1px solid #2d8cf0;
              }
            }
            a.entry-primary {
              color: #fff;
              &:hover {
                color: #2d8cf0;
                border-bottom: 1px solid #2d8cf0;
              }
            }
          }
          li {
            display: inline-block;
            padding: 10px 0 0 10px;
            &:first-child {
              padding: 10px 0 0 0;
            }
          }
        }
      }
    }
  }
</style>
