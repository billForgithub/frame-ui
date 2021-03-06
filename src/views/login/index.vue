<template>
  <div class="login-container">
    <div class="login-info">
      <div class="title">zuihou-admin-ui</div>
      <div class="sub-title">超级管理后台</div>
      <div class="desc">1. {{ $t('common.desc.a') }}</div>
      <div class="desc">2. {{ $t('common.desc.b') }}</div>
      <div class="desc">3. {{ $t('common.desc.c') }}</div>
      <div class="desc">4. {{ $t('common.desc.d') }}</div>
      <div class="desc">5. {{ $t('common.desc.e') }}</div>
      <div class="desc">6. {{ $t('common.desc.f') }}</div>
    </div>
    <el-form
      ref="loginForm"
      :model="loginForm"
      :rules="rules"
      class="login-form"
      autocomplete="off"
      label-position="left"
    >
      <div class="title-container">
        <h3 class="title">{{ $t('login.title') }}平台</h3>
        <lang-select class="set-language" />
      </div>
      <span v-if="login.type === &quot;up&quot;">
        <el-form-item prop="account">
          <el-input
            ref="account"
            v-model="loginForm.account"
            :placeholder="$t(&quot;login.username&quot;)"
            prefix-icon="el-icon-user"
            name="account"
            type="text"
            autocomplete="off"
            @keyup.enter.native="handleLogin"
          />
        </el-form-item>
        <el-form-item prop="password">
          <el-input
            ref="password"
            v-model="loginForm.password"
            prefix-icon="el-icon-key"
            type="password"
            :placeholder="$t(&quot;login.password&quot;)"
            name="password"
            autocomplete="off"
            :show-password="true"
            @keyup.enter.native="handleLogin"
          />
        </el-form-item>
        <el-form-item prop="code" class="code-input">
          <el-input
            ref="code"
            v-model="loginForm.code"
            prefix-icon="el-icon-lock"
            :placeholder="$t(&quot;login.code&quot;)"
            name="code"
            type="text"
            autocomplete="off"
            style="width: 70%"
            @keyup.enter.native="handleLogin"
          />
        </el-form-item>
        <img :src="imageCode" alt="codeImage" class="code-image" @click="getCodeImage">
        <el-button
          :loading="loading"
          type="primary"
          style="width:100%;margin-bottom:14px;"
          @click.native.prevent="handleLogin"
        >{{ $t('login.logIn') }}</el-button>
      </span>
    </el-form>
    <span class="login-footer">
      © 2019
      <a target="_blank" href="https://github.com/zuihou">zuihou</a> - zuihou-admin-cloud
    </span>
  </div>
</template>

<script>
import LangSelect from '@/components/LangSelect'
import db from '@/utils/localstorage'
import { randomNum } from '@/utils'
import loginApi from '@/api/Login.js'
import commonApi from '@/api/Common.js'

export default {
  name: 'Login',
  components: { LangSelect },
  data () {
    return {
      tabActiveName: 'bindLogin',
      login: {
        type: 'up'
      },
      logo: [
        { img: 'gitee.png', name: 'gitee', radius: true },
        { img: 'github.png', name: 'github', radius: true },
        { img: 'tencent_cloud.png', name: 'tencent_cloud', radius: true },
        { img: 'qq.png', name: 'qq', radius: false },
        { img: 'dingtalk.png', name: 'dingtalk', radius: true },
        { img: 'microsoft.png', name: 'microsoft', radius: false }
      ],
      loginForm: {
        account: 'demoAdmin',
        password: 'zuihou',
        bindAccount: '',
        bindPassword: '',
        signAccount: '',
        signPassword: ''
      },
      rules: {
        account: { required: true, message: this.$t('rules.require'), trigger: 'blur' },
        password: { required: true, message: this.$t('rules.require'), trigger: 'blur' },
        code: { required: true, message: this.$t('rules.require'), trigger: 'blur' },
        bindAccount: { required: true, message: this.$t('rules.require'), trigger: 'blur' },
        bindPassword: { required: true, message: this.$t('rules.require'), trigger: 'blur' },
        signAccount: [
          { required: true, message: this.$t('rules.require'), trigger: 'blur' },
          { min: 4, max: 10, message: this.$t('rules.range4to10'), trigger: 'blur' }
        ],
        signPassword: [
          { required: true, message: this.$t('rules.require'), trigger: 'blur' },
          { min: 6, max: 20, message: this.$t('rules.range6to20'), trigger: 'blur' }
        ]
      },
      authUser: null,
      loading: false,
      showDialog: false,
      redirect: undefined,
      otherQuery: {},
      randomId: randomNum(24, 16),
      imageCode: '',
      page: {
        width: window.screen.width * 0.5,
        height: window.screen.height * 0.5
      }
    }
  },
  created () {

  },
  mounted () {
    db.clear()
    this.getCodeImage()
  },
  destroyed () {
    window.removeEventListener('message', this.resolveSocialLogin)
  },
  methods: {
    getCodeImage () {
      loginApi.getCaptcha(this.randomId)
        .then(res => {
          if (res.byteLength <= 100) {
            this.$message({
              message: this.$t('tips.systemError'),
              type: 'error'
            })
          }
          return 'data:image/png;base64,' + btoa(
            new Uint8Array(res)
              .reduce((data, byte) => data + String.fromCharCode(byte), '')
          )
        }).then((res) => {
          this.imageCode = res
        }).catch((e) => {
          if (e.toString().indexOf('429') !== -1) {
            this.$message({
              message: this.$t('tips.tooManyRequest'),
              type: 'error'
            })
          } else {
            this.$message({
              message: this.$t('tips.getCodeImageFailed'),
              type: 'error'
            })
          }
        })
    },

    bindLogin () {
      let account_c = false
      let password_c = false
      this.$refs.loginForm.validateField('bindAccount', e => { if (!e) { account_c = true } })
      this.$refs.loginForm.validateField('bindPassword', e => { if (!e) { password_c = true } })
      if (account_c && password_c) {
        this.loading = true
        const that = this
        const params = {
          bindAccount: that.loginForm.bindAccount,
          bindPassword: that.loginForm.bindPassword,
          ...that.authUser
        }
        params.token = null
        that.$post('auth/social/bind/login', params).then((r) => {
          const data = r.data.data
          this.saveLoginData(data)
          this.getUserDetailInfo()
          this.loginSuccessCallback(that.loginForm.bindAccount)
        }).catch((error) => {
          console.error(error)
          that.loading = false
        })
      }
    },

    handleLogin () {
      let account_c = false
      let password_c = false
      let code_c = false
      this.$refs.loginForm.validateField('account', e => { if (!e) { account_c = true } })
      this.$refs.loginForm.validateField('password', e => { if (!e) { password_c = true } })
      this.$refs.loginForm.validateField('code', e => { if (!e) { code_c = true } })
      if (account_c && password_c && code_c) {
        this.loading = true
        const that = this
        that.loginForm['key'] = that.randomId
        loginApi.login(this.loginForm)
          .then(res => {
            if (res.isSuccess) {
              that.saveLoginData(res.data.token)
              that.saveUserInfo(res.data.user)
              that.loginSuccessCallback(res.data.user)
              that.$message({
                message: this.$t('tips.loginSuccess'),
                type: 'success'
              })
              that.loading = false
              that.$router.push('/')
            } else {
              that.loading = false
              that.getCodeImage()
            }
          })
      }
    },
    saveLoginData (token) {
      this.$store.commit('account/setToken', token.token)
      const current = new Date()
      const expireTime = current.setTime(current.getTime() + 1000 * token.expire)
      this.$store.commit('account/setExpireTime', expireTime)
    },
    saveUserInfo (user) {
      this.$store.commit('account/setUser', user)
      // TODO 从后台拉取权限列表
      const permissions = [
        "user:view",
        "user:export",
        "role:view",
        "role:add",
        "role:export",
        "menu:view",
        "menu:add",
        "menu:export",
        "dept:view",
        "dept:add",
        "dept:export",
        "client:view",
        "client:add",
        "client:decrypt",
        "log:view",
        "log:export",
        "monitor:loginlog",
        "loginlog:export",
        "monitor:register",
        "monitor:zipkin",
        "monitor:kibana",
        "mobitor:admin",
        "monitor:swagger",
        "grafana:view",
        "gen:config",
        "gen:generate",
        "gen:generate:gen",
        "others:eximport"
      ]
      this.$store.commit('account/setPermissions', permissions)
    },
    loginSuccessCallback (user) {
      commonApi.enums()
        .then(res => {
          if (res.isSuccess) {
            this.$store.commit('common/setEnums', res.data)
          }
        })
    }
  }
}
</script>

<style lang="scss">
$bg: #283443;
$light_gray: #fff;
$cursor: #555;

@supports (-webkit-mask: none) and (not (cater-color: $cursor)) {
  .login-container .el-input input {
    color: $cursor;
  }
}
/* reset element-ui css */
.login-container {
  .el-input {
    display: inline-block;
    input {
      background: transparent;
      border: 0;
      -webkit-appearance: none;
      border-radius: 0;
      color: #000000;
      height: 28px;
      caret-color: $cursor;

      &:-webkit-autofill {
        box-shadow: 0 0 0 1000px $bg inset !important;
        -webkit-text-fill-color: $cursor !important;
      }
    }
  }

  .el-form-item {
    border: 1px solid #dedede;
    border-radius: 2px;
    color: #454545;
    transition: all 0.3s;
    &:hover {
      border-color: #57a3f3;
    }
  }
}
</style>
<style lang="scss" scoped>
$bg: #2d3a4b;
$dark_gray: #aaa;
$light_gray: #eee;

.login-container {
  background: url(../../assets/background.jpg) 50% no-repeat;
  background-size: cover;
  width: 100%;
  height: 100vh;
  .login-info {
    position: absolute;
    left: 15%;
    top: 44%;
    margin-top: -100px;
    color: #fff;
    .title {
      font-size: 1.8rem;
      font-weight: 600;
    }
    .sub-title {
      font-size: 1.5rem;
      margin: 0.3rem 0 0.7rem 1rem;
    }
    .desc {
      font-size: 0.96rem;
      line-height: 1.9rem;
    }
  }
  .login-form {
    position: absolute;
    top: 50%;
    left: 70%;
    margin: -180px 0 0 -160px;
    width: 350px;
    height: 380px;
    padding: 36px;
    background: #fff;
    border-radius: 3px;
    .code-input {
      width: 50%;
      display: inline-block;
      vertical-align: middle;
    }
    .code-image {
      display: inline-block;
      vertical-align: top;
      cursor: pointer;
    }
    .login-type {
      text-align: right;
      display: inline-block;
      width: 100%;
    }
    .logo-wrapper {
      display: inline-block;
      margin: 10px 0;
      img {
        width: 1.9rem;
        display: inline-block;
        margin: 0.8rem 0.8rem -0.8rem 0.8rem;
        cursor: pointer;
        &.radius {
          border-radius: 50%;
        }
      }
    }
  }
  .login-footer {
    position: fixed;
    bottom: 1rem;
    width: 100%;
    text-align: center;
    color: white;
    font-size: 0.85rem;
    line-height: 1rem;
    height: 1rem;
  }
  .tips {
    font-size: 14px;
    color: #fff;
    margin-bottom: 10px;

    span {
      &:first-of-type {
        margin-right: 16px;
      }
    }
  }

  .title-container {
    position: relative;

    .title {
      font-size: 20px;
      color: rgba(0, 0, 0, 0.85);
      margin: 0 auto 40px auto;
      text-align: center;
      font-weight: bold;
    }

    .set-language {
      color: #aaa;
      position: absolute;
      top: 3px;
      font-size: 18px;
      right: 0;
      cursor: pointer;
    }
  }

  .thirdparty-button {
    position: absolute;
    right: 0;
    bottom: 6px;
  }

  @media only screen and (max-width: 470px) {
    .thirdparty-button {
      display: none;
    }
  }

  @media screen and (max-width: 1100px) {
    .login-info {
      left: 8%;
    }
  }

  @media screen and (max-width: 970px) {
    .login-form {
      left: 50%;
    }
    .login-info {
      display: none;
    }
  }
}
</style>
