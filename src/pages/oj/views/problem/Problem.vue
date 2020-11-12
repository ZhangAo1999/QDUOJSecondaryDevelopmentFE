<template>
  <div class="flex-container">
    <div id="problem-main">
      <!--problem main-->
      <Panel :padding="40" shadow>
        <div slot="title">{{problem.title}}</div>
        <div id="problem-content" class="markdown-body" v-katex @dblclick="editProblem">
          <p class="title">{{$t('m.Description')}}</p>
          <p class="content" v-html=problem.description></p>
          <!-- {{$t('m.music')}} -->
          <p class="title">{{$t('m.Input')}} <span v-if="problem.io_mode.io_mode=='File IO'">({{$t('m.FromFile')}}: {{ problem.io_mode.input }})</span></p>
          <p class="content" v-html=problem.input_description></p>

          <p class="title">{{$t('m.Output')}} <span v-if="problem.io_mode.io_mode=='File IO'">({{$t('m.ToFile')}}: {{ problem.io_mode.output }})</span></p>
          <p class="content" v-html=problem.output_description></p>

          <div v-for="(sample, index) of problem.samples" :key="index">
            <div class="flex-container sample">
              <div class="sample-input">
                <p class="title">{{$t('m.Sample_Input')}} {{index + 1}}
                  <a class="copy"
                     v-clipboard:copy="sample.input"
                     v-clipboard:success="onCopy"
                     v-clipboard:error="onCopyError">
                    <Icon type="clipboard"></Icon>
                  </a>
                </p>
                <pre>{{sample.input}}</pre>
              </div>
              <div class="sample-output">
                <p class="title">{{$t('m.Sample_Output')}} {{index + 1}}</p>
                <pre>{{sample.output}}</pre>
              </div>
            </div>
          </div>

          <div v-if="problem.hint">
            <p class="title">{{$t('m.Hint')}}</p>
            <Card :class="'ivu-card-' + theme" dis-hover>
              <div class="content" v-html=problem.hint></div>
            </Card>
          </div>

          <div v-if="problem.source">
            <p class="title">{{$t('m.Source')}}</p>
            <p class="content">{{problem.source}}</p>
          </div>

        </div>
        <el-dialog title="Edit Problem" :visible.sync="editing" :before-close="cancelEditProblem"
                  :close-on-click-modal="false">
          <el-form ref="form" :model="problem" :rules="rules" label-position="top" label-width="70px">
            <el-row :gutter="20">
              <el-col :span="6">
                <el-form-item prop="_id" :label="$t('m.Display_ID')"
                              :required="this.routeName === 'create-contest-problem' || this.routeName === 'edit-contest-problem'">
                  <el-input :placeholder="$t('m.Display_ID')" v-model="problem._id"></el-input>
                </el-form-item>
              </el-col>
              <el-col :span="18">
                <el-form-item prop="title" :label="$t('m.Title')" required>
                  <el-input :placeholder="$t('m.Title')" v-model="problem.title"></el-input>
                </el-form-item>
              </el-col>
            </el-row>
            <el-row :gutter="20">
              <el-col :span="24">
                <el-form-item prop="description" :label="$t('m.Description')" required>
                  <Simditor v-model="problem.description"></Simditor>
                </el-form-item>
              </el-col>
            </el-row>
            <el-row :gutter="20">
              <el-col :span="24">
                <el-form-item prop="input_description" :label="$t('m.Input_Description')" required>
                  <Simditor v-model="problem.input_description"></Simditor>
                </el-form-item>
              </el-col>
              <el-col :span="24">
                <el-form-item prop="output_description" :label="$t('m.Output_Description')" required>
                  <Simditor v-model="problem.output_description"></Simditor>
                </el-form-item>
              </el-col>
            </el-row>
            <el-row :gutter="20">
              <el-col :span="8">
                <el-form-item :label="$t('m.Time_Limit') + ' (ms)' " required>
                  <el-input type="Number" :placeholder="$t('m.Time_Limit')" v-model="problem.time_limit"></el-input>
                </el-form-item>
              </el-col>
              <el-col :span="8">
                <el-form-item :label="$t('m.Memory_limit') + ' (MB)' " required>
                  <el-input type="Number" :placeholder="$t('m.Memory_limit')" v-model="problem.memory_limit"></el-input>
                </el-form-item>
              </el-col>
              <el-col :span="8">
                <el-form-item :label="$t('m.Difficulty')">
                  <el-select class="difficulty-select" size="small" :placeholder="$t('m.Difficulty')" v-model="problem.difficulty">
                    <el-option :label="$t('m.Low')" value="Low"></el-option>
                    <el-option :label="$t('m.Mid')" value="Mid"></el-option>
                    <el-option :label="$t('m.High')" value="High"></el-option>
                  </el-select>
                </el-form-item>
              </el-col>
            </el-row>
            <el-row :gutter="20">
              <el-col :span="4">
                <el-form-item :label="$t('m.Visible')">
                  <el-switch
                    v-model="problem.visible"
                    active-text=""
                    inactive-text="">
                  </el-switch>
                </el-form-item>
              </el-col>
              <el-col :span="4">
                <el-form-item :label="$t('m.ShareSubmission')">
                  <el-switch
                    v-model="problem.share_submission"
                    active-text=""
                    inactive-text="">
                  </el-switch>
                </el-form-item>
              </el-col>
              <el-col :span="8">
                <el-form-item :label="$t('m.Tag')" :error="error.tags" required>
                  <span class="tags">
                    <el-tag
                      v-for="tag in problem.tags"
                      :closable="true"
                      :close-transition="false"
                      :key="tag"
                      type="success"
                      @close="closeTag(tag)"
                    >{{tag}}</el-tag>
                  </span>
                  <el-autocomplete
                    v-if="inputVisible"
                    size="mini"
                    class="input-new-tag"
                    v-model="tagInput"
                    :trigger-on-focus="false"
                    @keyup.enter.native="addTag"
                    @blur="addTag"
                    @select="addTag"
                    :fetch-suggestions="querySearch">
                  </el-autocomplete>
                  <el-button class="button-new-tag" v-else size="small" @click="inputVisible = true">+ {{$t('m.New_Tag')}}</el-button>
                </el-form-item>
              </el-col>
              <el-col :span="8">
                <el-form-item :label="$t('m.Languages')" :error="error.languages" required>
                  <el-checkbox-group v-model="problem.languages">
                    <el-tooltip class="spj-radio" v-for="lang in allLanguage.languages" :key="'spj'+lang.name" effect="dark"
                                :content="lang.description" placement="top-start">
                      <el-checkbox :label="lang.name"></el-checkbox>
                    </el-tooltip>
                  </el-checkbox-group>
                </el-form-item>
              </el-col>
            </el-row>
            <div>
              <el-form-item v-for="(sample, index) in problem.samples" :key="'sample'+index">
                <Accordion :title="'Sample' + (index + 1)">
                  <el-button type="warning" size="small" icon="el-icon-delete" slot="header" @click="deleteSample(index)">
                    Delete
                  </el-button>
                  <el-row :gutter="20">
                    <el-col :span="12">
                      <el-form-item :label="$t('m.Input_Samples')" required>
                        <el-input
                          :rows="5"
                          type="textarea"
                          :placeholder="$t('m.Input_Samples')"
                          v-model="sample.input">
                        </el-input>
                      </el-form-item>
                    </el-col>
                    <el-col :span="12">
                      <el-form-item :label="$t('m.Output_Samples')" required>
                        <el-input
                          :rows="5"
                          type="textarea"
                          :placeholder="$t('m.Output_Samples')"
                          v-model="sample.output">
                        </el-input>
                      </el-form-item>
                    </el-col>
                  </el-row>
                </Accordion>
              </el-form-item>
            </div>
            <div class="add-sample-btn">
              <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" type="button" class="add-samples" @click="addSample()"><i class="el-icon-plus"></i>{{$t('m.Add_Sample')}}
              </button>
            </div>
            <el-form-item style="margin-top: 20px" :label="$t('m.Hint')">
              <Simditor v-model="problem.hint" placeholder=""></Simditor>
            </el-form-item>
            <el-form-item :label="$t('m.Source')">
              <el-input :placeholder="$t('m.Source')" v-model="problem.source"></el-input>
            </el-form-item>
            <el-form-item>
              <span style="color:red;">{{errorMsg}}</span>
            </el-form-item>
          </el-form>
          <span slot="footer" class="dialog-footer">
            <el-button plain type="primary" @click.native="cancelEditProblem">{{$t('m.Cancel')}}</el-button>
            <el-button type="primary" @click.native="finishEditProblem">{{$t('m.Save')}}</el-button>
          </span>
        </el-dialog>
      </Panel>
      <!--problem main end-->
      <Card :class="'ivu-card-' + theme" :padding="20" id="submit-code" dis-hover>
        <CodeMirror :value.sync="code"
                    :languages="problem.languages"
                    :language="language"
                    :theme="theme"
                    @resetCode="onResetToTemplate"
                    @changeTheme="onChangeTheme"
                    @changeLang="onChangeLang"></CodeMirror>
        <Row type="flex" justify="space-between">
          <Col :span="10">
            <div class="status" v-if="statusVisible">
              <template v-if="!this.contestID || (this.contestID && OIContestRealTimePermission)">
                <span>{{$t('m.Status')}}</span>
                <Tag type="dot" :color="submissionStatus.color" @click.native="handleRoute('/status/'+submissionId)">
                  {{$t('m.' + submissionStatus.text.replace(/ /g, "_"))}}
                </Tag>
              </template>
              <template v-else-if="this.contestID && !OIContestRealTimePermission">
                <Alert type="success" show-icon>{{$t('m.Submitted_successfully')}}</Alert>
              </template>
            </div>
            <div v-else-if="problem.my_status === 0">
              <Alert type="success" show-icon>{{$t('m.You_have_solved_the_problem')}}</Alert>
            </div>
            <div v-else-if="this.contestID && !OIContestRealTimePermission && submissionExists">
              <Alert type="success" show-icon>{{$t('m.You_have_submitted_a_solution')}}</Alert>
            </div>
            <div v-if="contestEnded">
              <Alert type="warning" show-icon>{{$t('m.Contest_has_ended')}}</Alert>
            </div>
          </Col>

          <Col :span="12">
            <template v-if="captchaRequired">
              <div class="captcha-container">
                <Tooltip v-if="captchaRequired" content="Click to refresh" placement="top">
                  <img :src="captchaSrc" @click="getCaptchaSrc"/>
                </Tooltip>
                <Input v-model="captchaCode" class="captcha-code"/>
              </div>
            </template>
            <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" type="warning" icon="edit" :loading="submitting" @click="submitCode"
                    :disabled="problemSubmitDisabled || submitted"
                    class="fl-right">
              <span v-if="submitting">{{$t('m.Submitting')}}</span>
              <span v-else>{{$t('m.Submit')}}</span>
            </Button>
          </Col>
        </Row>
      </Card>
    </div>

    <div id="right-column">
      <VerticalMenu @on-click="handleRoute">
        <template v-if="this.contestID">
          <VerticalMenu-item :route="{name: 'contest-problem-list', params: {contestID: contestID}}">
            <Icon type="ios-photos"></Icon>
            {{$t('m.Problems')}}
          </VerticalMenu-item>

          <VerticalMenu-item :route="{name: 'contest-announcement-list', params: {contestID: contestID}}">
            <Icon type="chatbubble-working"></Icon>
            {{$t('m.Announcements')}}
          </VerticalMenu-item>
        </template>

        <VerticalMenu-item v-if="!this.contestID || OIContestRealTimePermission" :route="submissionRoute">
          <Icon type="navicon-round"></Icon>
           {{$t('m.Submissions')}}
        </VerticalMenu-item>

        <template v-if="this.contestID">
          <VerticalMenu-item v-if="!this.contestID || OIContestRealTimePermission"
                             :route="{name: 'contest-rank', params: {contestID: contestID}}">
            <Icon type="stats-bars"></Icon>
            {{$t('m.Rankings')}}
          </VerticalMenu-item>
          <VerticalMenu-item :route="{name: 'contest-details', params: {contestID: contestID}}">
            <Icon type="home"></Icon>
            {{$t('m.View_Contest')}}
          </VerticalMenu-item>
        </template>
      </VerticalMenu>

      <Card :class="'ivu-card-' + theme" id="info">
        <div slot="title" class="header">
          <Icon type="information-circled"></Icon>
          <span class="card-title">{{$t('m.Information')}}</span>
        </div>
        <ul>
          <li><p>ID</p>
            <p>{{problem._id}}</p></li>
          <li>
            <p>{{$t('m.Time_Limit')}}</p>
            <p>{{problem.time_limit}}MS</p></li>
          <li>
            <p>{{$t('m.Memory_Limit')}}</p>
            <p>{{problem.memory_limit}}MB</p></li>
          <li>
          <li>
            <p>{{$t('m.IOMode')}}</p>
            <p>{{problem.io_mode.io_mode}}</p>
          </li>
          <li>
            <p>{{$t('m.Created')}}</p>
            <p>{{problem.created_by.username}}</p></li>
          <li v-if="problem.difficulty">
            <p>{{$t('m.Level')}}</p>
            <p>{{$t('m.' + problem.difficulty)}}</p></li>
          <li v-if="problem.total_score">
            <p>{{$t('m.Score')}}</p>
            <p>{{problem.total_score}}</p>
          </li>
          <li>
            <p>{{$t('m.Tags')}}</p>
            <p>
              <Poptip trigger="hover" placement="left-end">
                <a>{{$t('m.Show')}}</a>
                <div slot="content">
                  <Tag v-for="tag in problem.tags" :key="tag">{{tag}}</Tag>
                </div>
              </Poptip>
            </p>
          </li>
        </ul>
      </Card>

      <Card :class="'ivu-card-' + theme" id="pieChart" :padding="0" v-if="!this.contestID || OIContestRealTimePermission">
        <div slot="title">
          <Icon type="ios-analytics"></Icon>
          <span class="card-title">{{$t('m.Statistic')}}</span>
          <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" type="ghost" size="small" id="detail" @click="graphVisible = !graphVisible">Details</Button>
        </div>
        <div class="echarts">
          <ECharts :options="pie"></ECharts>
        </div>
      </Card>
    </div>

    <Modal v-model="graphVisible">
      <div id="pieChart-detail">
        <ECharts :options="largePie" :initOptions="largePieInitOpts"></ECharts>
      </div>
      <div slot="footer">
        <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" type="ghost" @click="graphVisible=false">{{$t('m.Close')}}</Button>
      </div>
    </Modal>
  </div>
</template>

<script>
  import {mapGetters, mapActions} from 'vuex'
  import {types} from '../../../../store'
  import Simditor from '@admin/components/Simditor.vue'
  import Accordion from '@admin/components/Accordion.vue'
  import CodeMirror from '@oj/components/CodeMirror.vue'
  import storage from '@/utils/storage'
  import {FormMixin} from '@oj/components/mixins'
  import {JUDGE_STATUS, CONTEST_STATUS, buildProblemCodeKey, USER_TYPE} from '@/utils/constants'
  import apiAdmin from '@admin/api'
  import api from '@oj/api'
  import {pie, largePie} from './chartData'

  // 只显示这些状态的图形占用
  const filtedStatus = ['-1', '-2', '0', '1', '2', '3', '4', '8']

  export default {
    name: 'Problem',
    components: {
      CodeMirror,
      Simditor,
      Accordion
    },
    mixins: [FormMixin],
    data () {
      return {
        rules: {
          _id: {required: true, message: 'Display ID is required', trigger: 'blur'},
          title: {required: true, message: 'Title is required', trigger: 'blur'},
          input_description: {required: true, message: 'Input Description is required', trigger: 'blur'},
          output_description: {required: true, message: 'Output Description is required', trigger: 'blur'}
        },
        routeName: '',
        error: {
          tags: '',
          spj: '',
          languages: '',
          testCase: ''
        },
        errorMsg: '',
        inputVisible: false,
        allLanguage: {},
        tagInput: '',
        editing: false,
        proProblem: {},
        statusVisible: false,
        captchaRequired: false,
        graphVisible: false,
        submissionExists: false,
        captchaCode: '',
        captchaSrc: '',
        contestID: '',
        problemID: '',
        submitting: false,
        code: '',
        language: 'C++',
        theme: 'solarized',
        submissionId: '',
        submitted: false,
        result: {
          result: 9
        },
        problem: {
          title: '',
          description: '',
          hint: '',
          my_status: '',
          template: {},
          languages: [],
          created_by: {
            username: ''
          },
          tags: [],
          io_mode: {'io_mode': 'Standard IO'}
        },
        pie: pie,
        largePie: largePie,
        // echarts 无法获取隐藏dom的大小，需手动指定
        largePieInitOpts: {
          width: '500',
          height: '480'
        }
      }
    },
    beforeRouteEnter (to, from, next) {
      let problemCode = storage.get(buildProblemCodeKey(to.params.problemID, to.params.contestID))
      if (problemCode) {
        next(vm => {
          vm.language = problemCode.language
          vm.code = problemCode.code
          vm.theme = problemCode.theme
        })
      } else {
        next()
      }
    },
    mounted () {
      this.routeName = this.$route.name === 'problem-details' ? 'edit-problem' : 'edit-contest-problem'
      this.$store.commit(types.CHANGE_CONTEST_ITEM_VISIBLE, {menu: false})
      this.init()
      apiAdmin.getLanguages().then(res => {
        let contestID = this.$route.params.contestId
        if (contestID) {
          this.problem.contest_id = this.reProblem.contest_id = contestID
          this.disableRuleType = true
          apiAdmin.getContest(contestID).then(res => {
            this.problem.rule_type = this.reProblem.rule_type = res.data.data.rule_type
            this.contest = res.data.data
          })
        }

        this.problem.spj_language = 'C'

        let allLanguage = res.data.data
        this.allLanguage = allLanguage

        // get problem after getting languages list to avoid find undefined value in `watch problem.languages`
        if (this.mode === 'edit') {
          this.title = this.$i18n.t('m.Edit_Problem')
          let funcName = {'edit-problem': 'getProblem', 'edit-contest-problem': 'getContestProblem'}[this.routeName]
          apiAdmin[funcName](this.$route.params.problemId).then(problemRes => {
            let data = problemRes.data.data
            if (!data.spj_code) {
              data.spj_code = ''
            }
            data.spj_language = data.spj_language || 'C'
            this.problem = data
            this.testCaseUploaded = true
          })
        }
      })
    },
    methods: {
      ...mapActions(['changeDomTitle']),
      editProblem () {
        if (!this.isAdmin) { return }
        this.editing = true
        this.proProblem = JSON.parse(JSON.stringify(this.problem))
        this.errorMsg = ''
      },
      cancelEditProblem () {
        this.editing = false
        this.problem = this.proProblem
      },
      finishEditProblem () {
        if (!this.problem.samples.length) {
          this.$error('Sample is required')
          this.errorMsg = 'Sample is required'
          return
        }
        for (let sample of this.problem.samples) {
          if (!sample.input || !sample.output) {
            this.$error('Sample input and output is required')
            this.errorMsg = 'Sample input and output is required'
            return
          }
        }
        if (!this.problem.tags.length) {
          this.error.tags = 'Please add at least one tag'
          this.$error(this.error.tags)
          this.errorMsg = 'Please add at least one tag'
          return
        }
        if (!this.problem.languages.length) {
          this.error.languages = 'Please choose at least one language for problem'
          this.$error(this.error.languages)
          this.errorMsg = 'Please choose at least one language for problem'
          return
        }
        apiAdmin['getProblem'](this.problem.id).then(problemRes => {
          let problem = problemRes.data.data
          let problemProps = Object.keys(this.problem)
          for (let k = 0; k < problemProps.length; ++k) {
            let propName = problemProps[k]
            if (this.problem[propName] !== problem[propName]) {
              problem[propName] = this.problem[propName]
            }
          }
          apiAdmin['editProblem'](problem).then(() => {
            this.editing = false
            this.init()
          }).catch(res => {
            this.errorMsg = res.data.data
            this.editing = true
          })
        })
      },
      addTag () {
        let inputValue = this.tagInput
        if (inputValue) {
          this.problem.tags.push(inputValue)
        }
        this.inputVisible = false
        this.tagInput = ''
      },
      closeTag (tag) {
        this.problem.tags.splice(this.problem.tags.indexOf(tag), 1)
      },
      addSample () {
        this.problem.samples.push({input: '', output: ''})
      },
      deleteSample (index) {
        this.problem.samples.splice(index, 1)
      },
      querySearch (queryString, cb) {
        api.getProblemTagList().then(res => {
          let tagList = []
          for (let tag of res.data.data) {
            tagList.push({value: tag.name})
          }
          cb(tagList)
        }).catch(() => {
        })
      },
      init () {
        this.$Loading.start()
        this.contestID = this.$route.params.contestID
        this.problemID = this.$route.params.problemID
        let func = this.$route.name === 'problem-details' ? 'getProblem' : 'getContestProblem'
        api[func](this.problemID, this.contestID).then(res => {
          this.$Loading.finish()
          let problem = res.data.data
          this.changeDomTitle({title: problem.title})
          api.submissionExists(problem.id).then(res => {
            this.submissionExists = res.data.data
          })
          problem.languages = problem.languages.sort()
          this.problem = problem
          this.changePie(problem)

          // 在beforeRouteEnter中修改了, 说明本地有code，无需加载template
          if (this.code !== '') {
            return
          }
          // try to load problem template
          this.language = this.problem.languages[0]
          let template = this.problem.template
          if (template && template[this.language]) {
            this.code = template[this.language]
          }
        }, () => {
          this.$Loading.error()
        })
      },
      changePie (problemData) {
        // 只显示特定的一些状态
        for (let k in problemData.statistic_info) {
          if (filtedStatus.indexOf(k) === -1) {
            delete problemData.statistic_info[k]
          }
        }
        let acNum = problemData.accepted_number
        let data = [
          {name: 'WA', value: problemData.submission_number - acNum},
          {name: 'AC', value: acNum}
        ]
        this.pie.series[0].data = data
        // 只把大图的AC selected下，这里需要做一下deepcopy
        let data2 = JSON.parse(JSON.stringify(data))
        data2[1].selected = true
        this.largePie.series[1].data = data2

        // 根据结果设置legend,没有提交过的legend不显示
        let legend = Object.keys(problemData.statistic_info).map(ele => JUDGE_STATUS[ele].short)
        if (legend.length === 0) {
          legend.push('AC', 'WA')
        }
        this.largePie.legend.data = legend

        // 把ac的数据提取出来放在最后
        let acCount = problemData.statistic_info['0']
        delete problemData.statistic_info['0']

        let largePieData = []
        Object.keys(problemData.statistic_info).forEach(ele => {
          largePieData.push({name: JUDGE_STATUS[ele].short, value: problemData.statistic_info[ele]})
        })
        largePieData.push({name: 'AC', value: acCount})
        this.largePie.series[0].data = largePieData
      },
      handleRoute (route) {
        this.$router.push(route)
      },
      onChangeLang (newLang) {
        if (this.problem.template[newLang]) {
          if (this.code.trim() === '') {
            this.code = this.problem.template[newLang]
          }
        }
        this.language = newLang
      },
      onChangeTheme (newTheme) {
        this.theme = newTheme
      },
      onResetToTemplate () {
        this.$Modal.confirm({
          content: this.$i18n.t('m.Are_you_sure_you_want_to_reset_your_code'),
          onOk: () => {
            let template = this.problem.template
            if (template && template[this.language]) {
              this.code = template[this.language]
            } else {
              this.code = ''
            }
          }
        })
      },
      checkSubmissionStatus () {
        // 使用setTimeout避免一些问题
        if (this.refreshStatus) {
          // 如果之前的提交状态检查还没有停止,则停止,否则将会失去timeout的引用造成无限请求
          clearTimeout(this.refreshStatus)
        }
        const checkStatus = () => {
          let id = this.submissionId
          api.getSubmission(id).then(res => {
            this.result = res.data.data
            if (Object.keys(res.data.data.statistic_info).length !== 0) {
              this.submitting = false
              this.submitted = false
              clearTimeout(this.refreshStatus)
              this.init()
            } else {
              this.refreshStatus = setTimeout(checkStatus, 2000)
            }
          }, res => {
            this.submitting = false
            clearTimeout(this.refreshStatus)
          })
        }
        this.refreshStatus = setTimeout(checkStatus, 2000)
      },
      submitCode () {
        if (this.code.trim() === '') {
          this.$error(this.$i18n.t('m.Code_can_not_be_empty'))
          return
        }
        this.submissionId = ''
        this.result = {result: 9}
        this.submitting = true
        let data = {
          problem_id: this.problem.id,
          language: this.language,
          code: this.code,
          contest_id: this.contestID
        }
        if (this.captchaRequired) {
          data.captcha = this.captchaCode
        }
        const submitFunc = (data, detailsVisible) => {
          this.statusVisible = true
          api.submitCode(data).then(res => {
            this.submissionId = res.data.data && res.data.data.submission_id
            // 定时检查状态
            this.submitting = false
            this.submissionExists = true
            if (!detailsVisible) {
              this.$Modal.success({
                title: this.$i18n.t('m.Success'),
                content: this.$i18n.t('m.Submit_code_successfully')
              })
              return
            }
            this.submitted = true
            this.checkSubmissionStatus()
          }, res => {
            this.getCaptchaSrc()
            if (res.data.data.startsWith('Captcha is required')) {
              this.captchaRequired = true
            }
            this.submitting = false
            this.statusVisible = false
          })
        }

        if (this.contestRuleType === 'OI' && !this.OIContestRealTimePermission) {
          if (this.submissionExists) {
            this.$Modal.confirm({
              title: '',
              content: '<h3>' + this.$i18n.t('m.You_have_submission_in_this_problem_sure_to_cover_it') + '<h3>',
              onOk: () => {
                // 暂时解决对话框与后面提示对话框冲突的问题(否则一闪而过）
                setTimeout(() => {
                  submitFunc(data, false)
                }, 1000)
              },
              onCancel: () => {
                this.submitting = false
              }
            })
          } else {
            submitFunc(data, false)
          }
        } else {
          submitFunc(data, true)
        }
      },
      onCopy (event) {
        this.$success('Code copied')
      },
      onCopyError (e) {
        this.$error('Failed to copy code')
      }
    },
    computed: {
      ...mapGetters(['problemSubmitDisabled', 'contestRuleType', 'OIContestRealTimePermission',
        'contestStatus', 'user', 'theme']),
      isAdmin () {
        return this.user.admin_type === USER_TYPE.SUPER_ADMIN
      },
      contest () {
        return this.$store.state.contest.contest
      },
      contestEnded () {
        return this.contestStatus === CONTEST_STATUS.ENDED
      },
      submissionStatus () {
        return {
          text: JUDGE_STATUS[this.result.result]['name'],
          color: JUDGE_STATUS[this.result.result]['color']
        }
      },
      submissionRoute () {
        if (this.contestID) {
          return {name: 'contest-submission-list', query: {problemID: this.problemID}}
        } else {
          return {name: 'submission-list', query: {problemID: this.problemID}}
        }
      }
    },
    beforeRouteLeave (to, from, next) {
      // 防止切换组件后仍然不断请求
      clearInterval(this.refreshStatus)

      this.$store.commit(types.CHANGE_CONTEST_ITEM_VISIBLE, {menu: true})
      storage.set(buildProblemCodeKey(this.problem._id, from.params.contestID), {
        code: this.code,
        language: this.language,
        theme: this.theme
      })
      next()
    },
    watch: {
      '$route' () {
        this.init()
      }
    }
  }
</script>

<style lang="less" scoped>
  .card-title {
    margin-left: 8px;
  }

  .input-new-tag {
    width: 78px;
  }
  
  .button-new-tag {
    height: 24px;
    line-height: 22px;
    padding-top: 0;
    padding-bottom: 0;
  }

  .tags {
    .el-tag {
      margin-right: 10px;
    }
  }

  .add-samples {
    width: 100%;
    background-color: #fff;
    border: 1px dashed #aaa;
    outline: none;
    cursor: pointer;
    color: #666;
    height: 35px;
    font-size: 14px;
    &:hover {
      background-color: #f9fafc;
    }
    i {
      margin-right: 10px;
    }
  }
  .add-sample-btn {
    margin-bottom: 10px;
  }

  .flex-container {
    #problem-main {
      flex: auto;
      margin-right: 18px;
    }
    #right-column {
      flex: none;
      width: 220px;
    }
  }

  #problem-content {
    margin-top: -50px;
    .title {
      font-size: 20px;
      font-weight: 400;
      margin: 25px 0 8px 0;
      color: #3091f2;
      .copy {
        padding-left: 8px;
      }
    }
    p.content {
      margin-left: 25px;
      margin-right: 20px;
      font-size: 15px
    }
    .sample {
      align-items: stretch;
      &-input, &-output {
        width: 50%;
        flex: 1 1 auto;
        display: flex;
        flex-direction: column;
        margin-right: 5%;
      }
      pre {
        flex: 1 1 auto;
        align-self: stretch;
        border-style: solid;
        background: transparent;
      }
    }
  }

  #submit-code {
    margin-top: 20px;
    margin-bottom: 20px;
    .status {
      float: left;
      span {
        margin-right: 10px;
        margin-left: 10px;
      }
    }
    .captcha-container {
      display: inline-block;
      .captcha-code {
        width: auto;
        margin-top: -20px;
        margin-left: 20px;
      }
    }
  }

  #info {
    margin-bottom: 20px;
    margin-top: 20px;
    ul {
      list-style-type: none;
      li {
        border-bottom: 1px dotted #e9eaec;
        margin-bottom: 10px;
        p {
          display: inline-block;
        }
        p:first-child {
          width: 90px;
        }
        p:last-child {
          float: right;
        }
      }
    }
  }

  .fl-right {
    float: right;
  }

  #pieChart {
    .echarts {
      height: 250px;
      width: 210px;
    }
    #detail {
      position: absolute;
      right: 10px;
      top: 10px;
    }
  }

  #pieChart-detail {
    margin-top: 20px;
    width: 500px;
    height: 480px;
  }
</style>

