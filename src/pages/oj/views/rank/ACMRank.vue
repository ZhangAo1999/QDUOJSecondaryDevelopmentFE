<template>
  <Row type="flex" justify="space-around">
    <Col :span="22">
    <Panel :padding="10">
      <div slot="title">{{$t('m.ACM_Ranklist')}}</div>
      <div class="echarts">
        <ECharts :options="options" ref="chart" auto-resize></ECharts>
      </div>
    </Panel>
    <Table :class="'ivu-table-' + theme" :data="dataRank" :columns="columns" :loading="loadingTable" size="large"></Table>
    <Pagination :total="total" :page-size.sync="limit" :current.sync="page"
                @on-change="getRankData" show-sizer
                @on-page-size-change="getRankData(1)"></Pagination>
    </Col>
  </Row>
</template>

<script>
  import api from '@oj/api'
  import { mapGetters } from 'vuex'
  import Pagination from '@oj/components/Pagination'
  import utils from '@/utils/utils'
  import {types} from '@/store'
  import { RULE_TYPE } from '@/utils/constants'

  export default {
    name: 'acm-rank',
    components: {
      Pagination
    },
    data () {
      return {
        page: 1,
        limit: 30,
        total: 0,
        loadingTable: false,
        dataRank: [],
        currentRank: '',
        columns: [
          {
            align: 'center',
            width: 60,
            render: (h, params) => {
              return h('span', {}, params.index + (this.page - 1) * this.limit + 1)
            }
          },
          {
            title: this.$i18n.t('m.User_User'),
            align: 'center',
            render: (h, params) => {
              return h('a', {
                style: {
                  'display': 'inline-block',
                  'max-width': '200px'
                },
                on: {
                  click: () => {
                    this.$router.push(
                      {
                        name: 'user-home',
                        query: {username: params.row.user.username}
                      })
                  }
                }
              }, params.row.user.username)
            }
          },
          {
            title: this.$i18n.t('m.mood'),
            align: 'center',
            render: (h, params) => {
              return h('div', {
                on: {
                  dblclick: () => {
                    if (!this.isMe(params.row.id)) { return }
                    this.currentRank = params.index + (this.page - 1) * this.limit + 1
                  }
                }
              }, [
                h('span', {
                  style: {
                    'display': this.currentRank === params.index + (this.page - 1) * this.limit + 1
                    ? 'none' : 'inline-block'
                  }
                }, params.row.mood),
                h('input', {
                  class: 'ivu-input',
                  attrs: {
                    type: 'text',
                    size: 'large',
                    value: params.row.mood
                  },
                  on: {
                    keyup: (event) => {
                      if (event.key === 'Enter') {
                        this.currentRank = ''
                        const mood = event.target.value.trim()
                        if (mood === params.row.mood) { return }
                        params.row.mood = mood
                        this.editMood(mood)
                      } else if (event.key === 'Escape') {
                        this.currentRank = ''
                      }
                    },
                    blur: (event) => {
                      this.currentRank = ''
                      const mood = event.target.value.trim()
                      if (mood === params.row.mood) { return }
                      params.row.mood = mood
                      this.editMood(mood)
                    }
                  },
                  style: {
                    'padding': '2px 0',
                    'font-size': '14px',
                    'display': this.currentRank === params.index + (this.page - 1) * this.limit + 1
                    ? 'inline-block' : 'none'
                  }
                })
              ])
            }
          },
          {
            title: this.$i18n.t('m.AC'),
            align: 'center',
            key: 'accepted_number'
          },
          {
            title: this.$i18n.t('m.Total'),
            align: 'center',
            key: 'submission_number'
          },
          {
            title: this.$i18n.t('m.Rating'),
            align: 'center',
            render: (h, params) => {
              return h('span', utils.getACRate(params.row.accepted_number, params.row.submission_number))
            }
          }
        ],
        options: {
          tooltip: {
            trigger: 'axis'
          },
          legend: {
            data: [this.$i18n.t('m.AC'), this.$i18n.t('m.Total')]
          },
          grid: {
            x: '3%',
            x2: '3%'
          },
          toolbox: {
            show: true,
            feature: {
              dataView: {show: true, readOnly: true},
              magicType: {show: true, type: ['line', 'bar', 'stack']},
              saveAsImage: {show: true}
            },
            right: '10%'
          },
          calculable: true,
          xAxis: [
            {
              type: 'category',
              data: ['root'],
              axisLabel: {
                interval: 0,
                showMinLabel: true,
                showMaxLabel: true,
                align: 'center',
                formatter: (value, index) => {
                  return utils.breakLongWords(value, 10)
                }
              }
            }
          ],
          yAxis: [
            {
              type: 'value'
            }
          ],
          series: [
            {
              name: this.$i18n.t('m.AC'),
              type: 'bar',
              data: [0],
              markPoint: {
                data: [
                  {type: 'max', name: 'max'}
                ]
              }
            },
            {
              name: this.$i18n.t('m.Total'),
              type: 'bar',
              data: [0],
              markPoint: {
                data: [
                  {type: 'max', name: 'max'}
                ]
              }
            }
          ]
        }
      }
    },
    mounted () {
      this.getRankData(1)
    },
    methods: {
      isMe (id) {
        return id === this.user.id
      },
      editMood (mood) {
        let profile = this.$store.state.user.profile
        let formProfile = {
          real_name: '',
          mood: '',
          major: '',
          blog: '',
          school: '',
          github: '',
          language: ''
        }
        Object.keys(formProfile).forEach(element => {
          if (profile[element] !== undefined) {
            formProfile[element] = profile[element]
          }
        })
        formProfile.mood = mood
        let updateData = utils.filterEmptyValue(Object.assign({}, formProfile))
        api.updateProfile(updateData).then(res => {
          this.$success('Success')
          this.$store.commit(types.CHANGE_PROFILE, {profile: res.data.data})
        })
      },
      getRankData (page) {
        let offset = (page - 1) * this.limit
        let bar = this.$refs.chart
        bar.showLoading({maskColor: 'rgba(250, 250, 250, 0.8)'})
        this.loadingTable = true
        api.getUserRank(offset, this.limit, RULE_TYPE.ACM).then(res => {
          this.loadingTable = false
          if (page === 1) {
            this.changeCharts(res.data.data.results.slice(0, 10))
          }
          this.total = res.data.data.total
          this.dataRank = res.data.data.results
          bar.hideLoading()
        }).catch(() => {
          this.loadingTable = false
          bar.hideLoading()
        })
      },
      changeCharts (rankData) {
        let [usernames, acData, totalData] = [[], [], []]
        rankData.forEach(ele => {
          usernames.push(ele.user.username)
          acData.push(ele.accepted_number)
          totalData.push(ele.submission_number)
        })
        this.options.xAxis[0].data = usernames
        this.options.series[0].data = acData
        this.options.series[1].data = totalData
      }
    },
    computed: {
      ...mapGetters(['isAuthenticated', 'user', 'theme'])
    }
  }
</script>

<style scoped lang="less">
  .echarts {
    margin: 0 auto;
    width: 95%;
    height: 400px;
  }
</style>
