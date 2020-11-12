<template>
  <div class="flex-container">
    <div id="contest-main">
      <!--children-->
      <transition name="fadeInUp">
        <router-view></router-view>
      </transition>
      <!--children end-->
      <div class="flex-container" v-if="route_name === 'contest-details'">
        <template>
          <div id="contest-desc" @dblclick="editContest">
            <Panel :padding="20" shadow>
              <div slot="title">
                {{contest.title}}
              </div>
              <div slot="extra">
                <Tag type="dot" :color="countdownColor">
                  <span id="countdown">{{countdown}}</span>
                </Tag>
              </div>
              <div v-html="contest.description" class="markdown-body"></div>
              <div v-if="passwordFormVisible" class="contest-password">
                <Input v-model="contestPassword" type="password"
                       placeholder="contest password" class="contest-password-input"
                       @on-enter="checkPassword"/>
                <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" type="info" @click="checkPassword">Enter</Button>
              </div>
            </Panel>
            <Table :class="'ivu-table-' + theme" :columns="columns" :data="contest_table" disabled-hover style="margin-bottom: 40px;"></Table>
            <el-dialog title="Edit Contest" :visible.sync="editing" :before-close="cancelEditContest"
                  :close-on-click-modal="false">
              <el-form label-position="top">
                <el-row :gutter="20">
                  <el-col :span="24">
                    <el-form-item :label="$t('m.ContestTitle')" required>
                      <el-input v-model="contest.title" :placeholder="$t('m.ContestTitle')"></el-input>
                    </el-form-item>
                  </el-col>
                  <el-col :span="24">
                    <el-form-item :label="$t('m.ContestDescription')" required>
                      <Simditor v-model="contest.description"></Simditor>
                    </el-form-item>
                  </el-col>
                  <el-col :span="8">
                    <el-form-item :label="$t('m.Contest_Start_Time')" required>
                      <el-date-picker
                        v-model="contest.start_time"
                        type="datetime"
                        :placeholder="$t('m.Contest_Start_Time')">
                      </el-date-picker>
                    </el-form-item>
                  </el-col>
                  <el-col :span="8">
                    <el-form-item :label="$t('m.Contest_End_Time')" required>
                      <el-date-picker
                        v-model="contest.end_time"
                        type="datetime"
                        :placeholder="$t('m.Contest_End_Time')">
                      </el-date-picker>
                    </el-form-item>
                  </el-col>
                  <el-col :span="8">
                    <el-form-item :label="$t('m.Contest_Password')">
                      <el-input v-model="contest.password" :placeholder="$t('m.Contest_Password')"></el-input>
                    </el-form-item>
                  </el-col>
                  <el-col :span="8">
                    <el-form-item :label="$t('m.Contest_Rule_Type')">
                      <el-radio class="radio" v-model="contest.rule_type" label="ACM" :disabled="disableRuleType">ACM</el-radio>
                      <el-radio class="radio" v-model="contest.rule_type" label="OI" :disabled="disableRuleType">OI</el-radio>
                    </el-form-item>
                  </el-col>
                  <el-col :span="8">
                    <el-form-item :label="$t('m.Real_Time_Rank')">
                      <el-switch
                        v-model="contest.real_time_rank"
                        active-color="#13ce66"
                        inactive-color="#ff4949">
                      </el-switch>
                    </el-form-item>
                  </el-col>
                  <el-col :span="8">
                    <el-form-item :label="$t('m.Contest_Status')">
                      <el-switch
                        v-model="contest.visible"
                        active-text=""
                        inactive-text="">
                      </el-switch>
                    </el-form-item>
                  </el-col>
                  <el-col :span="24">
                    <el-form-item :label="$t('m.Allowed_IP_Ranges')">
                      <div v-for="(range, index) in contest.allowed_ip_ranges" :key="index">
                        <el-row :gutter="20" style="margin-bottom: 15px">
                          <el-col :span="8">
                            <el-input v-model="range.value" :placeholder="$t('m.CIDR_Network')"></el-input>
                          </el-col>
                          <el-col :span="10">
                            <el-button plain icon="el-icon-fa-plus" @click="addIPRange"></el-button>
                            <el-button plain icon="el-icon-fa-trash" @click="removeIPRange(range)"></el-button>
                          </el-col>
                        </el-row>
                      </div>
                    </el-form-item>
                  </el-col>
                </el-row>
                <el-form-item>
                  <span style="color:red;">{{errorMsg}}</span>
                </el-form-item>
              </el-form>
              <span slot="footer" class="dialog-footer">
                <el-button plain type="primary" @click.native="cancelEditContest">{{$t('m.Cancel')}}</el-button>
                <el-button type="primary" @click.native="finishEditContest">{{$t('m.Save')}}</el-button>
              </span>
            </el-dialog>
          </div>
        </template>
      </div>

    </div>
    <div v-show="showMenu" id="contest-menu">
      <VerticalMenu @on-click="handleRoute">
        <VerticalMenu-item :route="{name: 'contest-details', params: {contestID: contestID}}">
          <Icon type="home"></Icon>
          {{$t('m.Overview')}}
        </VerticalMenu-item>

        <VerticalMenu-item :disabled="contestMenuDisabled"
                           :route="{name: 'contest-announcement-list', params: {contestID: contestID}}">
          <Icon type="chatbubble-working"></Icon>
          {{$t('m.Announcements')}}
        </VerticalMenu-item>

        <VerticalMenu-item :disabled="contestMenuDisabled"
                           :route="{name: 'contest-problem-list', params: {contestID: contestID}}">
          <Icon type="ios-photos"></Icon>
          {{$t('m.Problems')}}
        </VerticalMenu-item>

        <VerticalMenu-item v-if="OIContestRealTimePermission"
                           :disabled="contestMenuDisabled"
                           :route="{name: 'contest-submission-list'}">
          <Icon type="navicon-round"></Icon>
          {{$t('m.Submissions')}}
        </VerticalMenu-item>

        <VerticalMenu-item v-if="OIContestRealTimePermission"
                           :disabled="contestMenuDisabled"
                           :route="{name: 'contest-rank', params: {contestID: contestID}}">
          <Icon type="stats-bars"></Icon>
          {{$t('m.Rankings')}}
        </VerticalMenu-item>

        <VerticalMenu-item v-if="showAdminHelper"
                           :route="{name: 'acm-helper', params: {contestID: contestID}}">
          <Icon type="ios-paw"></Icon>
          {{$t('m.Admin_Helper')}}
        </VerticalMenu-item>
      </VerticalMenu>
    </div>
  </div>
</template>

<script>
  import moment from 'moment'
  import apiAdmin from '@admin/api'
  import api from '@oj/api'
  import { mapState, mapGetters, mapActions } from 'vuex'
  import Simditor from '@admin/components/Simditor.vue'
  import { types } from '@/store'
  import { CONTEST_STATUS_REVERSE, CONTEST_STATUS, USER_TYPE } from '@/utils/constants'
  import time from '@/utils/time'

  export default {
    name: 'ContestDetail',
    components: {
      Simditor
    },
    data () {
      return {
        disableRuleType: false,
        errorMsg: '',
        editing: false,
        CONTEST_STATUS: CONTEST_STATUS,
        route_name: '',
        btnLoading: false,
        contest: {},
        contestID: '',
        contestPassword: '',
        columns: [
          {
            title: this.$i18n.t('m.StartAt'),
            render: (h, params) => {
              return h('span', time.utcToLocal(params.row.start_time))
            }
          },
          {
            title: this.$i18n.t('m.EndAt'),
            render: (h, params) => {
              return h('span', time.utcToLocal(params.row.end_time))
            }
          },
          {
            title: this.$i18n.t('m.ContestType'),
            render: (h, params) => {
              return h('span', this.$i18n.t('m.' + (params.row.contest_type === 'Public'
              ? 'Public' : params.row.contest_type.replace(' ', '_'))))
            }
          },
          {
            title: this.$i18n.t('m.Rule'),
            render: (h, params) => {
              return h('span', this.$i18n.t('m.' + params.row.rule_type))
            }
          },
          {
            title: this.$i18n.t('m.Creator'),
            render: (h, data) => {
              return h('span', data.row.created_by.username)
            }
          }
        ]
      }
    },
    mounted () {
      this.contestID = this.$route.params.contestID
      this.route_name = this.$route.name
      this.disableRuleType = true
      this.$store.dispatch('getContest').then(res => {
        this.changeDomTitle({title: res.data.data.title})
        let data = res.data.data
        let endTime = moment(data.end_time)
        if (endTime.isAfter(moment(data.now))) {
          this.timer = setInterval(() => {
            this.$store.commit(types.NOW_ADD_1S)
          }, 1000)
        }
      })
      apiAdmin.getContest(this.contestID).then(res => {
        let data = res.data.data
        let ranges = []
        for (let v of data.allowed_ip_ranges) {
          ranges.push({value: v})
        }
        if (ranges.length === 0) {
          ranges.push({value: ''})
        }
        data.allowed_ip_ranges = ranges
        this.contest = data
      })
    },
    methods: {
      ...mapActions(['changeDomTitle']),
      handleRoute (route) {
        this.$router.push(route)
      },
      editContest () {
        if (!this.isAdmin) { return }
        this.editing = true
        this.proContest = JSON.parse(JSON.stringify(this.contest))
        this.errorMsg = ''
      },
      cancelEditContest () {
        this.editing = false
        this.contest = this.proContest
      },
      finishEditContest () {
        let ranges = []
        for (let v of this.contest.allowed_ip_ranges) {
          if (v.value !== '') {
            ranges.push(v.value)
          }
        }
        this.contest.allowed_ip_ranges = ranges
        apiAdmin['editContest'](this.contest).then(res => {
          this.editing = false
          this.init()
        }).catch(res => {
          this.errorMsg = res.data.data
          this.editing = true
        })
      },
      init () {
        this.contestID = this.$route.params.contestID
        this.route_name = this.$route.name
        this.disableRuleType = true
        this.$store.dispatch('getContest').then(res => {
          this.changeDomTitle({title: res.data.data.title})
          let data = res.data.data
          let endTime = moment(data.end_time)
          if (endTime.isAfter(moment(data.now))) {
            this.timer = setInterval(() => {
              this.$store.commit(types.NOW_ADD_1S)
            }, 1000)
          }
        })
        apiAdmin.getContest(this.contestID).then(res => {
          let data = res.data.data
          let ranges = []
          for (let v of data.allowed_ip_ranges) {
            ranges.push({value: v})
          }
          if (ranges.length === 0) {
            ranges.push({value: ''})
          }
          data.allowed_ip_ranges = ranges
          this.contest = data
        })
      },
      checkPassword () {
        if (this.contestPassword === '') {
          this.$error('Password can\'t be empty')
          return
        }
        this.btnLoading = true
        api.checkContestPassword(this.contestID, this.contestPassword).then((res) => {
          this.$success('Succeeded')
          this.$store.commit(types.CONTEST_ACCESS, {access: true})
          this.btnLoading = false
        }, (res) => {
          this.btnLoading = false
        })
      },
      addIPRange () {
        this.contest.allowed_ip_ranges.push({value: ''})
      },
      removeIPRange (range) {
        let index = this.contest.allowed_ip_ranges.indexOf(range)
        if (index !== -1) {
          this.contest.allowed_ip_ranges.splice(index, 1)
        }
      }
    },
    computed: {
      ...mapState({
        showMenu: state => state.contest.itemVisible.menu,
        // contest: state => state.contest.contest,
        contest_table: state => [state.contest.contest],
        now: state => state.contest.now
      }),
      ...mapGetters(
        ['contestMenuDisabled', 'contestRuleType', 'contestStatus', 'countdown', 'isContestAdmin',
          'OIContestRealTimePermission', 'passwordFormVisible', 'user', 'theme']
      ),
      isAdmin () {
        return this.user.admin_type === USER_TYPE.SUPER_ADMIN
      },
      countdownColor () {
        if (this.contestStatus) {
          return CONTEST_STATUS_REVERSE[this.contestStatus].color
        }
      },
      showAdminHelper () {
        return this.isContestAdmin && this.contestRuleType === 'ACM'
      }
    },
    watch: {
      '$route' (newVal) {
        this.route_name = newVal.name
        this.contestID = newVal.params.contestID
        this.changeDomTitle({title: this.contest.title})
      }
    },
    beforeDestroy () {
      clearInterval(this.timer)
      this.$store.commit(types.CLEAR_CONTEST)
    }
  }
</script>

<style scoped lang="less">
  pre {
    display: inline-block;
  }

  #countdown {
    font-size: 16px;
  }

  .flex-container {
    #contest-main {
      flex: 1 1;
      width: 0;
      #contest-desc {
        flex: auto;
      }
    }
    #contest-menu {
      flex: none;
      width: 210px;
      margin-left: 20px;
    }
    .contest-password {
      margin-top: 20px;
      margin-bottom: -10px;
      &-input {
        width: 200px;
        margin-right: 10px;
      }
    }
  }
</style>
