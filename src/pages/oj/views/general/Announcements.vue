<template>
  <Panel shadow :padding="10">
    <div slot="title" @dblclick="createAnnocement">
      {{title}}
    </div>
    <div slot="extra">
      <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" v-if="listVisible" type="info" @click="init" :loading="btnLoading">{{$t('m.Refresh')}}</Button>
      <Button :class="['ivu-btn-theme-' + theme, 'ivu-btn-ghost']" v-else type="ghost" icon="ios-undo" @click="goBack">{{$t('m.Back')}}</Button>
    </div>

    <transition-group name="announcement-animate" mode="in-out">
      <div class="no-announcement" v-if="!announcements.length" key="no-announcement">
        <p>{{$t('m.No_Announcements')}}</p>
      </div>
      <template v-if="listVisible">
        <ul class="announcements-container" key="list">
          <li v-for="announcement in announcements" :key="announcement.title">
            <div class="flex-container">
              <div v-show="announcement!==currentAnnouncement" class="title" @dblclick="toEditTitle(announcement)"><a class="entry" :class="'entry-' + theme" @click="goAnnouncement(announcement)">
                {{announcement.title}}</a></div>
              <input v-show="announcement===currentAnnouncement" class="ivu-input" :value="announcement.title"
                  @keyup.esc="cancelEditTitle()"
                  @keyup.enter="finishEditTitle(announcement, $event)"
                  @blur="finishEditTitle(announcement, $event)"
                  v-input-focus="announcement === currentAnnouncement">
              <div class="date">{{announcement.create_time | localtime }}</div>
              <div class="creator"> {{$t('m.By')}} {{announcement.created_by.username}}</div>
            </div>
          </li>
        </ul>
        <Pagination v-if="!isContest"
                    key="page"
                    :total="total"
                    :page-size="limit"
                    @on-change="getAnnouncementList">
        </Pagination>
      </template>

      <template v-else>
        <div @dblclick="toEditContent()" v-katex v-html="announcement.content" key="content" class="content-container markdown-body"></div>
      </template>
    </transition-group>
    <div>
      <el-dialog title="Edit Announcement" :visible.sync="editing"
                  :close-on-click-modal="false">
        <el-form label-position="top">
          <el-form-item :label="$t('m.Announcement_Content')" required>
            <Simditor v-model="currentAnnouncement.content"></Simditor>
          </el-form-item>
          <div class="visible-box">
            <span>{{$t('m.Announcement_visible')}}</span>
            <el-switch
              v-model="currentAnnouncement.visible"
              active-text=""
              inactive-text="">
            </el-switch>
          </div>
        </el-form>
        <span slot="footer" class="dialog-footer">
          <el-button plain type="primary" @click.native="cancelEditContent">{{$t('m.Cancel')}}</el-button>
          <el-button type="primary" @click.native="finishEditContent">{{$t('m.Save')}}</el-button>
        </span>
      </el-dialog>
      <el-dialog title="Create Announcement" :visible.sync="creating"
                  :close-on-click-modal="false">
        <el-form label-position="top">
          <el-form-item :label="$t('m.Announcement_Title')" required>
            <el-input
              v-model="currentAnnouncement.title"
              :placeholder="$t('m.Announcement_Title')" class="title-input">
            </el-input>
          </el-form-item>
          <el-form-item :label="$t('m.Announcement_Content')" required>
            <Simditor v-model="currentAnnouncement.content"></Simditor>
          </el-form-item>
          <div class="visible-box">
            <span>{{$t('m.Announcement_visible')}}</span>
            <el-switch
              v-model="currentAnnouncement.visible"
              active-text=""
              inactive-text="">
            </el-switch>
          </div>
        </el-form>
        <span slot="footer" class="dialog-footer">
          <el-button plain type="primary" @click.native="cancelCreateContent">{{$t('m.Cancel')}}</el-button>
          <el-button type="primary" @click.native="finishCreateContent">{{$t('m.Save')}}</el-button>
        </span>
      </el-dialog>
      <el-dialog :title="$t('m.Are_you_sure_you_want_to_delete_this_announcement')" 
        :visible.sync="deleting" 
        :close-on-click-modal="false">
        <span slot="footer" class="dialog-footer">
          <el-button plain type="primary" @click.native="deleting = false">{{$t('m.Cancel')}}</el-button>
          <el-button type="primary" @click.native="finishDelete">{{$t('m.Delete')}}</el-button>
        </span>
      </el-dialog>
      </div>
  </Panel>
</template>

<script>
  import apiAdmin from '@admin/api'
  import api from '@oj/api'
  import { mapGetters } from 'vuex'
  import { USER_TYPE } from '@/utils/constants'
  import Pagination from '@oj/components/Pagination'
  import Simditor from '@admin/components/Simditor.vue'

  export default {
    name: 'Announcement',
    components: {
      Pagination,
      Simditor
    },
    data () {
      return {
        limit: 10,
        total: 10,
        btnLoading: false,
        announcements: [],
        announcement: '',
        listVisible: true,
        currentAnnouncement: '',
        editing: false,
        creating: false,
        deleting: false
      }
    },
    mounted () {
      this.init()
    },
    methods: {
      createAnnocement () {
        if (!this.listVisible) { return }
        this.creating = true
        this.currentAnnouncement = {}
        this.currentAnnouncement.title = ''
        this.currentAnnouncement.visible = true
        this.currentAnnouncement.content = ''
      },
      toEditTitle (announcement) {
        if (!this.isAdmin) { return }
        this.currentAnnouncement = announcement
      },
      toEditContent () {
        if (!this.isAdmin) { return }
        this.currentAnnouncement = JSON.parse(JSON.stringify(this.announcement))
        this.editing = true
      },
      cancelEditTitle () {
        this.currentAnnouncement = ''
      },
      cancelEditContent () {
        this.editing = false
      },
      cancelCreateContent () {
        this.creating = false
      },
      finishEditTitle (announcement, event) {
        if (!this.isAdmin) { return }
        const title = event.target.value.trim()
        this.currentAnnouncement = ''
        if (announcement.title === title) { return }
        let funcName = ''
        if (this.isContest) {
          announcement.contest_id = this.$route.params.contestID
          funcName = 'updateContestAnnouncement'
        } else {
          funcName = 'updateAnnouncement'
        }
        if (title) {
          announcement.title = title
          apiAdmin[funcName](announcement).then(res => {
            this.init()
          }).catch()
        } else {
          this.currentAnnouncement = announcement
          this.deleting = true
        }
      },
      finishEditContent () {
        if (!this.isAdmin) { return }
        if (this.announcement.content === this.currentAnnouncement.content &&
          this.announcement.visible === this.currentAnnouncement.visible) { return }
        let funcName = ''
        if (this.isContest) {
          this.currentAnnouncement.contest_id = this.$route.params.contestID
          funcName = 'updateContestAnnouncement'
        } else {
          funcName = 'updateAnnouncement'
        }
        apiAdmin[funcName](this.currentAnnouncement).then(res => {
          this.announcement = this.currentAnnouncement
          this.init()
        }).catch()
        this.editing = false
      },
      finishCreateContent () {
        if (!this.currentAnnouncement.title) { return }
        let funcName = ''
        if (this.isContest) {
          this.currentAnnouncement.contest_id = this.$route.params.contestID
          funcName = 'createContestAnnouncement'
        } else {
          funcName = 'createAnnouncement'
        }
        apiAdmin[funcName](this.currentAnnouncement).then(res => {
          this.init()
          this.creating = false
        }).catch()
      },
      init () {
        if (this.isContest) {
          this.getContestAnnouncementList()
        } else {
          this.getAnnouncementList()
        }
      },
      finishDelete () {
        this.deleting = false
        let funcName = this.isContest ? 'deleteContestAnnouncement' : 'deleteAnnouncement'
        apiAdmin[funcName](this.currentAnnouncement.id).then(res => {
          this.init()
        })
      },
      getAnnouncementList (page = 1) {
        this.btnLoading = true
        api.getAnnouncementList((page - 1) * this.limit, this.limit).then(res => {
          this.btnLoading = false
          this.announcements = res.data.data.results
          this.total = res.data.data.total
        }, () => {
          this.btnLoading = false
        })
      },
      getContestAnnouncementList () {
        this.btnLoading = true
        api.getContestAnnouncementList(this.$route.params.contestID).then(res => {
          this.btnLoading = false
          this.announcements = res.data.data
        }, () => {
          this.btnLoading = false
        })
      },
      goAnnouncement (announcement) {
        this.announcement = announcement
        this.listVisible = false
      },
      goBack () {
        this.listVisible = true
        this.announcement = ''
      }
    },
    computed: {
      ...mapGetters(['user', 'theme']),
      isAdmin () {
        return this.user.admin_type === USER_TYPE.SUPER_ADMIN
      },
      title () {
        if (this.listVisible) {
          return this.isContest ? this.$i18n.t('m.Contest_Announcements') : this.$i18n.t('m.Announcements')
        } else {
          return this.announcement.title
        }
      },
      isContest () {
        return !!this.$route.params.contestID
      }
    }
  }
</script>

<style scoped lang="less">
  .announcements-container {
    margin-top: -10px;
    margin-bottom: 10px;
    li {
      padding-top: 15px;
      list-style: none;
      padding-bottom: 15px;
      margin-left: 20px;
      font-size: 16px;
      border-bottom: 1px solid rgba(187, 187, 187, 0.5);
      &:last-child {
        border-bottom: none;
      }
      .flex-container {
        .title {
          flex: 1 1;
          text-align: left;
          padding-left: 10px;
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
        .creator {
          flex: none;
          width: 200px;
          text-align: center;
        }
        .date {
          flex: none;
          width: 200px;
          text-align: center;
        }
      }
      .ivu-input {
        margin-top: -5px;
        height: 40px;
        font-size: 18px;
      }
    }
  }

  .content-container {
    padding: 0 20px 20px 20px;
  }

  .no-announcement {
    text-align: center;
    font-size: 16px;
  }changeLocale

  .announcement-animate-enter-active {
    animation: fadeIn 1s;
  }
</style>
