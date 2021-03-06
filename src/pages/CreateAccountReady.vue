/**
 * 准备创建钱包
 */
<template>
<div class="page">

    <toolbar :title="$t(title)" :showbackicon="showbackicon">
      <v-btn icon style="visibility: hidden;" slot="left-tool">
        <v-icon class="back-icon"/>
      </v-btn>
    </toolbar>
    
    <div class="content">
      <div class="label">{{$t('Account.AccountName')}}</div>
      <div class="value" >{{name}}</div>
      <div class="label">{{$t('Account.Password')}}</div>
      <div class="value">{{password}}</div>
      <div class="label">{{$t('Account.AccountAddress')}}</div>
      <div class="value" @click="copy(address)">{{address}}</div>
      <div class="label">{{$t('SecretKey')}}</div>
      <div class="value" @click="copy(seed)">{{seed}}</div>
      <div class="qrcode">
        <qrcode :text="qrtext" :size="200" color="red"/>
      </div>
      <div class="hint">{{$t('Account.CreateAccountReadyHint')}}</div>
    </div>
    <div class="footer">
      <v-layout row wrap>
        <v-flex xs6 @click="goback">
          <span>{{$t('Return')}}</span>
        </v-flex>
        <v-flex xs6 @click="save">
          <span>{{$t('BackedUp')}}</span>
        </v-flex>
       </v-layout>  
    </div>

    <v-dialog v-model="coveringDlg" persistent max-width="95%">
      <v-card>
        <v-card-title class="headline">{{$t('Account.WhetherCoverAccount')}}</v-card-title>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="green" flat @click.native="coveringDlg = false">{{$t('Button.Cancel')}}</v-btn>
          <v-btn color="red" flat @click.native="doCoverAccount">{{$t('Button.OK')}}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- 密钥输入窗口 -->
    <v-dialog class="si-dlg" v-model="seedInputDlgShow" persistent max-width="95%"  fullscreen transition="dialog-bottom-transition" :overlay=false>
      <toolbar :title="$t(title)" :showbackicon="showbackicon">
        <div class="si-text" slot="right-tool"  @click="showSkipDlg = true">
          {{$t('Account.Skip')}}
        </div>
      </toolbar>
      
      <div class="si-bg">

        <div class="si-card">
          <div class="headline">{{$t('Account.InputSecretKey')}}</div>
          <v-layout wrap>
            <v-flex xs12 sm6 md4>
              <secret-key-input ref="secretkeyRef" disabled></secret-key-input>
            </v-flex>
          </v-layout>
          
        </div>
        <div class="btn-group flex-row textcenter">
          <div class="flex1 btn-cancel" @click="seedInputDlgShow = false">{{$t('Return')}}</div>
          <div class="flex1 btn-ok" @click="btnOKSeedInput">{{$t('Validate')}}</div>
        </div>
      </div>

    </v-dialog>

    <!-- 是否跳过验证窗口 -->
    <v-dialog v-model="showSkipDlg" max-width="95%" persistent>
      <div>
        <div class="card-content dlg-content">
          <div class="avatar-div textcenter">
            <v-avatar>
              <img src="../assets/img/logo-red.png" />
            </v-avatar>
          </div>
          <div class="t1 skip-red textcenter">{{$t('Account.ConfirmSkip')}}</div>
          <div class="t2 skip-white pt-2 pb-4">{{$t('Account.SkipValidHint')}}</div>
          <div class="btns flex-row">
            <div class="flex1 skip-red textcenter" @click="doSave">{{$t('Button.OK')}}</div>
            <div class="flex1 skip-red textcenter" @click="showSkipDlg = false">{{$t('Account.Later')}}</div>
          </div>
        </div>
      </div>
    </v-dialog>

    <!-- 验证通过窗口 -->
    <v-dialog v-model="showSeedValidDlg" max-width="95%" persistent>
      <div>
        <div class="card-content dlg-content">
          <div class="avatar-div textcenter">
            <v-avatar>
              <img src="../assets/img/logo-green.png" />
            </v-avatar>
          </div>
          <div class="t1 dlg-green textcenter">{{$t('Congratulations')}}!</div>
          <div class="t2 skip-white pt-2 pb-4">{{$t('Account.SecretKeyValidHint')}}</div>
          <div class="btns dlg-green textcenter" @click="toNextPage">
            {{$t('Account.Start')}}
          </div>
        </div>
      </div>
    </v-dialog>

    <!-- 验证失败窗口 -->
    <v-dialog v-model="showSeedInValidDlg" max-width="95%" persistent>
      <div>
        <div class="card-content dlg-content">
          <div class="avatar-div textcenter">
            <v-avatar>
              <img src="../assets/img/logo-red.png" />
            </v-avatar>
          </div>
          <div class="t1 skip-red textcenter">{{$t('Error.SeedWrong')}}!</div>
          <div class="t2 skip-white pt-2 pb-4">{{$t('Account.SecretKeyUnValidHint')}}</div>
          <div class="btns textcenter skip-red" @click="showSeedInValidDlg = false">
            {{$t('Account.ReVerify')}}
          </div>
        </div>
      </div>
    </v-dialog>

  <loading :show="showLoading" :loading="working" :success="dealok" :fail='dealfail' 
      :title="loadingTitle" :msg="loadingMsg" :closeable="dealfail" @close="hiddenLoadingView"/>

</div>
</template>

<script>
import Toolbar from '@/components/Toolbar'
import { mapState, mapActions} from 'vuex'
import {address as genAddress} from '@/api/account'
import QRCode from '@/components/QRCode'
import Card from '@/components/Card'
import { exportAccount } from '@/api/qr'
import Loading from '@/components/Loading'
import SecretKeyInput from '@/components/SecretKeyInput'
//import StellarHDWallet from 'stellar-hd-wallet'
import { closeStreams, initStreams,cleanStreamData } from '@/streams'
import  defaultsDeep  from 'lodash/defaultsDeep'
export default {
  data(){
    return {
      title: 'CreateAccount',
      showbackicon: false,
      qrsize:200,
      logo: require("../assets/img/logo.png"),
      dialog: false,
      guide:0,
      primary: 'primary',
      secondary: 'blue',

      coveringDlg: false,
      working: false,
      dealok: false,
      dealfail: false,
      showLoading: false,
      loadingTitle: null,
      loadingMsg: null,
      
      //新增账户的情况下，要求用户输入密钥
      seedInputDlgShow: false,
      seedInput: null,

      showSkipDlg: false,
      showSeedValidDlg: false,
      showSeedInValidDlg: false,


    
    }
  },
  computed:{
    ...mapState({
      seed: state => state.seed,
      name: state => state.accountname,
      password: state => state.accountpassword,
      extdata: state => state.seedExtData,
      accounts: state => state.accounts.data || [],
      isImportAccount: state => state.isImportAccount,
      isCreateAccount: state => state.isCreateAccount,
    }),
    address(){
      if(this.seed){
        return genAddress(this.seed)
      }
      return ''
    },
    qrtext(){
      if(!this.seed)return ''
      //类似于stargaza的格式
      var data = {stellar:{name:this.name,key:this.seed}}
      let account = defaultsDeep({
        name: this.name,
        address: genAddress(this.seed)
      },this.extdata || {})
      let accountData = {
        seed: this.seed
      }
      return exportAccount(account,accountData)
      // return JSON.stringify(exportAccount(account,accountData))
    }
  
  },
  beforeMount () {
    // const mnemonic = StellarHDWallet.generateMnemonic({
    //   language: 'chinese_simplified',
    // })
    // console.log(mnemonic)
    // console.log(typeof mnemonic)
    // const wallet = StellarHDWallet.fromMnemonic(mnemonic)

    // console.log(wallet.getPublicKey(0)) // => GDKYMXOAJ5MK4EVIHHNWRGAAOUZMNZYAETMHFCD6JCVBPZ77TUAZFPKT
    //  console.log(wallet.getSecret(0)) // => SCVVKNLBHOWBNJYHD3CNROOA2P3K35I5GNTYUHLLMUHMHWQYNEI7LVED
    //  console.log(wallet.getKeypair(0) )// => StellarBase.Keypair for account 0
    window.localStorage.setItem('login_flag','1')
    console.log(window.localStorage.getItem('login_flag'))
  },
  mounted(){
    this.dialog = true;
    
  },
  methods: {
    ...mapActions(['createAccount','cleanGlobalState','coverAccount']),
    goback(){
      this.$router.back()
      //this.$router.push({name:'CreateAccount'})
    },
    copy(value){
      if(cordova.plugins.clipboard){
        cordova.plugins.clipboard.copy(value)
        this.$toasted.show(this.$t('CopySuccess'))
      }
    },
    swipe(direction){
      if(direction==='Left'  && this.guide <3){
          this.guide ++
      }
      if(direction==='Right' && this.guide >1){
          this.guide --
      }
    },
    btnOKSeedInput(){
      
      this.seedInput = this.$refs.secretkeyRef.getSeed()
      console.log(`------${this.seedInput}---`)
      console.log(`---${this.seed}---`)
      if(this.seed != this.seedInput){
        //this.seedInputErr = 'Error.SeedWrong'
        this.showSeedInValidDlg = true
        return;
      }
      this.seedInputDlgShow = false
      this.doSave();
    },
    save(){
      //要求用户输入密钥，只有密钥输入正确才能继续进行
      if(this.isCreateAccount){
        this.seedInputDlgShow = true;
      }else{
        this.doSave();
      }
    },
    doSave(){
      //保存，并跳转到main界面
      //校验地址是否冲突
      var index = this.accounts.map(ele=>ele.address).indexOf(this.address)
      if(index>=0){
        this.$toasted.error(this.$t('MyAddress.DuplicateAddress'))
        this.coveringDlg = true
        return 
      }
      let account = {name: this.name,address: this.address,seed: this.seed,password: this.password}
      this.working = true
      this.dealok = false
      this.dealfail = false
      this.showLoading = true
      this.loadingTitle = null
      this.loadingMsg = null
      this.createAccount(account)
        .then(data=>{
         this.ok()
         //this.$toasted.show(this.$t('Account.CreateAccountSuccess'));
         this.loadingTitle = this.$t('Account.CreateAccountSuccess')
          this.cleanGlobalState()
          this.showSeedValidDlg = true
        }).catch(err=>{
          //this.$toasted.error(this.$t('Account.CreateAccountError'))
          this.loadingTitle = this.$t('Account.CreateAccountError')
          this.fail()
          setTimeout(()=>{
            this.showLoading = false
            this.showSeedInValidDlg = true
          },1000)
          

        })
      },

    doCoverAccount(){
      if(this.working)return
      this.working = true
      this.showLoading = true
      this.dealok = false
      this.dealfail = false
      this.loadingTitle = null
      this.loadingMsg = null
      this.coverAccount({name: this.name,address: this.address,seed: this.seed,password: this.password})
        .then(data=>{
            this.ok()
            //this.$toasted.show(this.$t('Account.CreateAccountSuccess'));
            this.loadingTitle = this.$t('Account.CreateAccountSuccess')
            this.cleanGlobalState()
            setTimeout(()=>{
              this.$router.push({name: 'MyAssets'})
            },1500)
        })
        .catch(err=>{
          //this.$toasted.error(this.$t('Account.CreateAccountError'))
          this.loadingTitle = this.$t('Account.CreateAccountError')
          this.fail()
        })
    },
    ok(){
      this.dealok = true
      this.dealfail = false
      this.working = false
      setTimeout(()=>{
        this.showLoading = false
      },1000)
    },
    fail(){
      this.dealok = false
      this.dealfail = true
      this.working = false
     
    },
    
    hiddenLoadingView(){
      this.loadingTitle = null
      this.loadingMsg = null
      this.dealfail = false
      this.showLoading = false
    },
    toNextPage(){
      this.showSeedValidDlg = false
      cleanStreamData()
      closeStreams()
      initStreams(this.address)
      this.$router.push({name:'MyAssets'})
    },
    skipValidSecretKey(){
      this.showSkipDlg  = false
      this.doSave()
    }

  },
  
  components: {
    Toolbar,
    qrcode: QRCode,
    Loading,
    SecretKeyInput,
    Card,
  }
}
</script>


<style lang="stylus" scoped>
@require '../stylus/color.styl'
.content
  position: fixed
  overflow-y: auto
  top: 48px
  left: 0
  right: 0
  bottom: 0
  padding: 20px 20px
  background: $secondarycolor.gray
  border-radius:10px
  margin:7px 7px 50px 7px
  .label
    font-size: 14px
    color: $primarycolor.green
    padding-top: 2px
    padding-bottom: 2px
  .value
    display: block
    font-size: 16px
    color: $primarycolor.font
    width: 100%
    white-space:normal
    word-wrap: break-word
    word-break: break-all
  .qrcode
    text-align: center

.footer
  position:fixed
  bottom:0
  left:0
  right:0
  z-index:99
  background:$primarycolor.gray
  height:42px
  line-height:42px
  font-size:16px
  text-align:center
  color:$primarycolor.green
.btn-available
  color:$primarycolor.green
.btn-unavailable
  color:$secondarycolor.green
.hint
  color:$primarycolor.red
  font-size: 15px
.headline
  color: $primarycolor.green
  font-size: 16px !important 
  padding-bottom: 10px
.notice
  font-size:14px
  color: $primarycolor.font
  min-height: 6em
/*.stepper__header
  box-shadow 0*/
.stepper-content
  display flex
  flex-direction column
.guide-img-wrapper
  height: 150px
  width: 100%
.guide-img
  height:150px
  max-width:320px  
  min-height: 150px
.guide-header
  box-shadow 0 0 0 0
.si-text
  font-size: 14px
.si-bg
  position: absolute
  top: 48px
  left: 0
  right: 0
  bottom: 0
  background: $primarycolor.gray
  padding: 10px 10px
  padding-bottom: 0px
.si-card
  background: $secondarycolor.gray
  border-radius: 5px
  padding: 1rem 5px

  // height: 30%
.btn-group
  position: fixed
  left: 0
  right: 0
  bottom: 10px
  .btn-cancel
  .btn-ok
    font-size: 16px
    color: $primarycolor.green
.card-content
  padding: 20px 10px
.t1
  font-size: 20px
  padding-top: 5px
  padding-bottom: 5px
.t2
  font-size: 16px
.skip-red
  color: $primarycolor.red
.btns
  font-size: 16px
.dlg-green
  color: $primarycolor.green
.dlg-content
  background: $secondarycolor.gray
</style>

