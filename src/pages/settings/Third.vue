// 第三方应用列表界面
<template>
  <div class="page" dark>
    <toolbar :title="$t('Title.ThirdApp')" :showbackicon="true"  @goback="back" :shadow="false" lockpass  ref="toolbar"/>
    <v-container fluid v-bind="{ [`grid-list-md`]: true }">
      <card padding="8px 8px" margin="8px 8px" v-if="working">
        <div class="mt-5 textcenter">
          <v-progress-circular indeterminate color="primary"></v-progress-circular>
        </div>
      </card>
      <card  padding="8px 8px" margin="8px 8px" v-else>
        <p v-if="err">
          {{$t(err)}}
        </p>
      </card>
      <v-layout row wrap  v-if="!working">
        <v-flex
          xs3
          v-for="(app,index) in apps"
          :key="index"
          @click="choose(app)"
        >
          <v-card flat tile class="pa-2 textcenter app-card" >
             <v-avatar  :size="`80px`">
               <img :src="app.image">
             </v-avatar>
             <v-card-title primary-title class="app-title">
               <div class="textcenter" style="width: 100%;">{{app.title}}</div>
             </v-card-title>
          </v-card>
        </v-flex>
      </v-layout>

    <v-dialog v-model="showConfirmDlg" max-width="95%" persistent>
      <div>
        <div class="card-content dlg-content">
          <div class="avatar-div textcenter">
            <v-avatar>
              <img src="../../assets/img/logo-red.png" />
            </v-avatar>
          </div>
          <div class="t2 skip-white pt-2 pb-4">{{$t('Third.OpenAppHint',[choosed.title])}}</div>
          <div class="btns flex-row">
            <div class="flex1 skip-red textcenter" @click="openApp">{{$t('Button.OK')}}</div>
            <div class="flex1 skip-red textcenter" @click="showConfirmDlg = false">{{$t('Button.Cancel')}}</div>
          </div>
        </div>
      </div>
    </v-dialog>
    </v-container>

    <send-asset v-if="showSendAsset" 
      :destination="sendTarget.destination"
      :appname="choosed.title"
      :asset_code="sendTarget.code"
      :asset_issuer="sendTarget.issuer"
      :memo_type="sendTarget.memo_type"
      :memo="sendTarget.memo"
      :amount="sendTarget.amount"
      :pathPayment="pathPayment"
      @exit="exitSendAsset"
      @sendsuccess="sendAssetSuccess"
       ></send-asset>
    <back-up-data v-if="appEventType === 'backup' && appEventData" 
      :appname="choosed.title" 
      @exit="exitBackUpEvent" @success="successBackUpEvent" />

    <recovery-data v-if="appEventType === 'recovery' && appEventData" 
      :appname="choosed.title" :encryptData="appEventData.data.data"
      @exit="exitRecoveryEvent" @success="successRecoveryEvent" />

    <trust-line v-if="appEventType === 'trust' && appEventData" 
      :appname="choosed.title" 
      :destination="appEventData.data.destination"
      :asset_code="appEventData.data.code"
      :asset_issuer="appEventData.data.issuer"
      :memo_type="sendTarget.memo_type"
      @exit="exitTrustEvent" @success="successTrustEvent" />
  </div>
</template>

<script>
const thridAppConfig = `https://raw.githubusercontent.com/StellarCN/firefly/docs/third.json`
import axios from 'axios'
import { mapState, mapActions} from 'vuex'
import Toolbar from '@/components/Toolbar'
import Card from '@/components/Card'
import Loading from '@/components/Loading'
import  defaultsDeep  from 'lodash/defaultsDeep'
import SendAsset from '@/components/third/SendAsset'
import RecoveryData from '@/components/third/RecoveryData'
import TrustLine from '@/components/third/TrustLine'
import BackUpData from '@/components/third/BackUpData'
import { FFWScript, FFW_EVENT_TYPE_PAY,FFW_EVENT_TYPE_PATHPAYMENT,FFW_EVENT_TYPE_SIGN
   ,FFW_EVENT_TYPE_BACKUP,FFW_EVENT_TYPE_RECOVERY,FFW_EVENT_TYPE_TRUST } from '@/api/ffw'
import { signToBase64, verifyByBase64 } from '@/api/keypair'
import debounce from 'lodash/debounce'

// export const FFW_EVENT_TYPE_PAY = 'pay'
// export const FFW_EVENT_TYPE_PATHPAYMENT = 'pathPayment'
// export const FFW_EVENT_TYPE_SIGN = 'sign'
// export const FFW_EVENT_TYPE_BACKUP = 'backup'
// export const FFW_EVENT_TYPE_RECOVERY = 'recovery'
// export const FFW_EVENT_TYPE_TRUST = 'trust'

export default {
  data(){
    return {
      apps: [],
      working: false,
      err: null,
      showConfirmDlg: false,
      choosed: {}, //当前选中的app
      showSendAsset: false,
      sendTarget:{},
      pathPayment: true,//发送功能是否支持pathPayment
      appInstance: null,

      appEventType: null,//接收到的appevent事件
      appEventData: null,//接收的appevent的data
    }
  },
   computed:{
    ...mapState({
      account: state => state.accounts.selectedAccount,
      accountData: state => state.accounts.accountData,
      islogin: state => (state.accounts.accountData.seed ? true : false),
      allcontacts: state => state.app.contacts||[],
      myaddresses: state => state.app.myaddresses||[]
    }),
  },
  beforeMount () {
    this.working = true
    this.err = null
    axios.get(thridAppConfig)
      .then(response=>{
        this.working = false
        //console.log(response)
        this.apps = response.data.apps
      })
      .catch(err=>{
        this.working = false
        console.error(err)
        this.err = 'Error.AjaxTimeout'
      })
  },
  mounted () {
    if(!this.islogin){
      this.$refs.toolbar.showPasswordLogin()
    }
  },
  methods: {
    back(){
      this.$router.back()
    },
    choose(app){
      this.choosed = app
      let val = localStorage.getItem(app.site)
      if(val){
        this.openApp()
        return
      }
      this.showConfirmDlg = true

    },
    openApp(){
      localStorage.setItem(this.choosed.site, "confirm")
      this.showConfirmDlg = false
      if(cordova.platformId === 'browser'){
        this.appInstance = cordova.InAppBrowser.open(this.choosed.site, '_blank', 'location=yes,toolbar=yes,toolbarcolor=#21ce90');
      }else{
        this.appInstance = cordova.ThemeableBrowser.open(this.choosed.site, '_blank', {
              statusbar: {
                  color: '#21ce90'
              },
              toolbar: {
                  height: 44,
                  color: '#21ce90'
              },
              browserProgress: {
                showProgress: true,
                progressBgColor: "#007e9c",
                progressColor: "#FF5E00"
              },
              title: {
                  color: '#FFFFFF',
                  showPageTitle: true,
                  staticText: this.choosed.title 
              },
              backButton: {
                  image: 'back',
                  imagePressed: 'back_pressed',
                  align: 'left',
                  event: 'backPressed'
              },
              forwardButton: {
                  image: 'forward',
                  imagePressed: 'forward_pressed',
                  align: 'left',
                  event: 'forwardPressed'
              },
              closeButton: {
                  image: 'close',
                  imagePressed: 'close_pressed',
                  align: 'left',
                  event: 'closePressed'
              },
              reloadButton: {
                  image: 'reload',
                  imagePressed: 'reload_pressed',
                  align: 'right',
                  event: 'reloadPressed'
              },
              customButtons: [
                  {
                      image: 'share',
                      imagePressed: 'share_pressed',
                      align: 'right',
                      event: 'sharePressed'
                  }
              ],
              backButtonCanClose: true
          })
          
      }
      this.appInstance.addEventListener('reloadPressed', e => {
        this.appInstance.reload()
      })
      this.appInstance.addEventListener('sharePressed', e => {
          this.shareCB(e.url)
      })
      // this.appInstance.removeEventListener('closePressed')
      this.appInstance.addEventListener('closePressed',()=>{
        this.appInstance.close()
        this.appInstance = undefined
      })
      this.appInstance.addEventListener('loadstop',() => {
        //let script = `if(!window.FFW){window.FFW = {};FFW.address = "${this.account.address}";FFW.pay = function(destination,code,issuer,amount,memo_type,memo){ var params = { type:'pay',destination: destination, code: code, issuer: issuer, amount: amount, memo_type: memo_type, memo: memo };cordova_iab.postMessage(JSON.stringify(params));};};`
        //let scriptEle = `if(!window.FFW){var script = document.createElement('script');script.setAttribute('type', 'text/javascript');script.text = "${script}";document.body.appendChild(script);}`
        //alert(scriptEle)
        let contacts = this.allcontacts
        let myaddresses = this.myaddresses
        let script = FFWScript(this.account.address, {contacts,myaddresses} )
        this.appInstance.executeScript({ code: script },params => {
          //console.log(params)
          //alert('after script insert')
          //alert(params)
        })

      })
      let that = this
      this.appInstance.addEventListener('message', debounce(function (e){
        console.log('-----------get message ---- ')
        console.log(JSON.stringify(e))
        let type = e.data.type
        if(type === FFW_EVENT_TYPE_PAY){
          this.doPayEvent(e)
        }else if(type === FFW_EVENT_TYPE_PATHPAYMENT){
          that.doPathPaymentEvent(e)
        }else if(type === FFW_EVENT_TYPE_SIGN){
          that.appEventType = e.data.type
          that.appEventData = e.data
          that.doSign(e)
        }else{  
          that.appEventType = e.data.type
          that.appEventData = e.data
          that.appInstance.hide()
        }
      },300))
    },
    doPayEvent(e){
      try{
        this.showSendAsset = true
        this.sendTarget = e.data
        this.pathPayment = false
        this.appInstance.hide()
      }catch(err){
        console.error(err)
        //alert('error:'+err.message)
      }
    },
    doSign(e){
      //签名
      let data = e.data.data
      if(data){
        let cdata = signToBase64(this.accountData.seed, data)
        console.log('---------------encrypt data---' + cdata)
        this.doCallbackEvent(this.callbackData('success', 'success', cdata))
      }else{
        this.doCallbackEvent(this.callbackData('fail','no data to sign'))
      }
    },
    doPathPaymentEvent(e){
      try{
        this.showSendAsset = true
        this.sendTarget = e.data
        this.pathPayment = true
        this.appInstance.hide()
      }catch(err){
        console.error(err)
        //alert('error:'+err.message)
      }
    },
    doCallbackEvent(data){
      console.log('-----------docallback event---' + JSON.stringify(this.appEventData))
      if(this.appEventData && this.appEventData.callback){
        try{
          let cb = this.appEventData.callback
          let code = `${cb}({code: "${data.code}",message:"${data.message}",data:"${data.data}"})`
          console.log('===============callback------event---')
          console.log(code)
          this.appInstance.executeScript({
            code: code }, 
            params=>{})
        }catch(err){
          console.error(err)
        }
      }
    },
    callbackData(code,message,data){
      // return JSON.stringify({code,message,data})
      return {code,message,data}
    },
    exitSendAsset(){
      this.showSendAsset = false
      this.appInstance.show()
      this.doCallbackEvent(this.callbackData('fail','cancel payment'))
    },
    sendAssetSuccess(){
      this.showSendAsset = false
      this.appInstance.show()
      //TODO 怎么通知应用
      this.doCallbackEvent(this.callbackData('success','success'))
    },
    shareCB(url){
      let options = {
        subject: this.choosed.title,
        url: url,
        chooserTitle: this.$t('Share')
      }
      window.plugins.socialsharing.shareWithOptions(options, result=>{
        console.log("Share completed? " + result.completed); // On Android apps mostly return false even while it's true
        console.log("Shared to app: " + result.app); // On Android result.app is currently empty. On iOS it's empty when sharing is cancelled (result.completed=false)
      }, msg=>{
        console.log("Sharing failed with message: " + msg);
      });
    },
    exitEvent(msg){
      this.appInstance.show()
      this.doCallbackEvent(this.callbackData('fail',msg))
      this.$nextTick(()=>{
        this.appEventType = null
        this.appEventData = null
      })
    },
    successEvent(msg='success',data){
      this.appInstance.show()
      this.doCallbackEvent(this.callbackData('success',msg, data))
      this.$nextTick(()=>{
        this.appEventType = null
        this.appEventData = null
      })
    },
    exitBackUpEvent(){
      this.exitEvent('cancel back up')
    },
    successBackUpEvent(data){
      this.successEvent('success',data)
    },
    exitRecoveryEvent(){
      this.exitEvent('cancel recovery')
    },
    successRecoveryEvent(){
      this.successEvent()
    },
    exitTrustEvent(){
      this.exitEvent('cancel trust')
    },
    successTrustEvent(){
      this.successEvent()
    }


  },
  components: {
    Toolbar,
    Loading,
    Card,
    SendAsset,
    TrustLine,
    RecoveryData,
    BackUpData,
  }
}
</script>


<style lang="stylus" scoped>
@require '../../stylus/color.styl'
.app-card
  background: $primarycolor.gray
.app-title
  padding: .1rem .1rem
  overflow: hidden
  white-space: nowrap
.card-content
  padding: 20px 10px
.t2
  font-size: 16px
.btns
  font-size: 16px
.dlg-green
  color: $primarycolor.green
.dlg-content
  background: $secondarycolor.gray
  color: $primarycolor.red
</style>
