<template>
  <div>
    <NavBar :theme="theme"></NavBar>
    <div class="content-app">
      <transition name="fadeInUp" mode="out-in">
        <router-view></router-view>
      </transition>
      <div class="footer">
        <p v-html="website.website_footer"></p>
        <p>Powered by <a href="https://github.com/ZhangAo1999/QDUOJSecondaryDevelopment">OnlineJudge</a>
          <span v-if="version">&nbsp; Version: {{ version }}</span>
        </p>
      </div>
    </div>
    <BackTop></BackTop>
  </div>
</template>

<script>
  import { mapActions, mapState, mapGetters } from 'vuex'
  import NavBar from '@oj/components/NavBar.vue'
  export default {
    name: 'app',
    components: {
      NavBar
    },
    data () {
      return {
        version: process.env.VERSION
      }
    },
    created () {
      try {
        document.body.removeChild(document.getElementById('app-loader'))
      } catch (e) {
      }
    },
    mounted () {
      this.getWebsiteConfig()
      this.changeBackgroundColor()
    },
    methods: {
      ...mapActions(['getWebsiteConfig', 'changeDomTitle']),
      changeBackgroundColor () {
        console.log(this.theme)
        if (this.theme === 'primary') {
          document.querySelector('body').setAttribute('style', 'background-color: #6af;')
          document.querySelector('html').setAttribute('style', 'background-color: #6af;')
        } else if (this.theme === 'dark') {
          document.querySelector('body').setAttribute('style', 'background-color: #345;')
          document.querySelector('html').setAttribute('style', 'background-color: #345;')
        } else {
          document.querySelector('body').setAttribute('style', 'background-color: #eee;')
          document.querySelector('html').setAttribute('style', 'background-color: #eee;')
        }
      }
    },
    computed: {
      ...mapState(['website']),
      ...mapGetters(['theme'])
    },
    watch: {
      'website' () {
        this.changeDomTitle()
      },
      '$route' () {
        this.changeDomTitle()
      },
      'theme' () {
        this.changeBackgroundColor()
      }
    }
  }
</script>

<style lang="less">
  * {
    -webkit-box-sizing: border-box;
    -moz-box-sizing: border-box;
    box-sizing: border-box;
  }
  a {
    text-decoration: none;
// background-color: transparent;
    &:active, &:hover {
      outline-width: 0;
    }
  }
  @media screen and (max-width: 1200px) {
  .content-app {
    margin-top: 160px;
    padding: 0 2%;
  }
}
@media screen and (min-width: 1200px) {
  .content-app {
    margin-top: 80px;
    padding: 0 2%;
  }
}
  .footer {
    margin-top: 20px;
    margin-bottom: 10px;
    text-align: center;
    font-size: small;
  }
  .fadeInUp-enter-active {
    animation: fadeInUp .8s;
  }
</style>