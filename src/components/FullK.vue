/**
 * 全屏K线图相关
 * @Date: 2018-01-25 11:53:34 
 * @Last Modified time: 2018-03-14 16:44:48
 * @License: MIT 
 */
<template>
<card class="k-card" padding="5px 5px">
  <div slot="card-content" class="k">
      <div class="flex-row atitle" v-if="titleData && titleData.price !== null && lastTradeAggregation">
          <div class="flex1 title-btn-div">
              <v-btn class="btn-back k-icon" icon @click="back">
                <i class="material-icons  k-icon">keyboard_arrow_left</i>
            </v-btn>
          </div>
          <div :class="'flex3 textcenter ' + ( titleData.change >=0 ? 'up':'down') ">
              <div class="price textcenter">
                  <span class="price">{{titleData.price}}</span>
                  <span class="code">{{counter.code}}</span>
              </div>
              <div class="flex-row">
                  <div class="flex1 textright">
                        <span v-if="titleData.change>0">+</span>
                        <span>{{titleData.change}}&nbsp;&nbsp;</span>
                  </div>
                  <div class="flex1 textleft">
                        <span v-if="titleData.rate>0"> +</span>
                        <span>{{titleData.rate}}%</span>
                  </div>
              </div>
          </div>
          <div class="flex3 values">
              <div class=""><span class="label">24H {{$t('high')}} </span><span>{{Number(lastTradeAggregation.high).toFixed(4)}}</span></div>
              <div class=""><span class="label">24H {{$t('low')}} </span><span>{{Number(lastTradeAggregation.low).toFixed(4)}}</span></div>
              <div class=""><span class="label">24H {{$t('volume')}} </span><span>{{Number(lastTradeAggregation.base_volume).toFixed(4)}}</span></div>
          </div>
          <div class="flex1 title-btn-div">
             
          </div>
         
      </div>
             
      <div class="kgraph" :id="id" v-bind:style="{height: height+'px', width: width+'px'}"></div>
      <div class="flex-row textcenter chgresolution">
          <div :class="'flex1 ' + (resolution_key === 'week' ? 'active' : '')" @click="chgResolutionKey('week')">{{$t('week')}}</div>
          <div :class="'flex1 ' + (resolution_key === 'day' ? 'active' : '')" @click="chgResolutionKey('day')">{{$t('day')}}</div>
          <div :class="'flex1 ' + (resolution_key === 'hour' ? 'active' : '')" @click="chgResolutionKey('hour')">{{$t('hour')}}</div>
          <div :class="'flex1 ' + (resolution_key === '15min' ? 'active' : '')" @click="chgResolutionKey('15min')">15{{$t('minute')}}</div>
          <div :class="'flex1 ' + (resolution_key === '1min' ? 'active' : '')" @click="chgResolutionKey('1min')">1{{$t('minute')}}</div>
      </div>
  </div>
</card>
</template>

<script>
import kmixins from '@/mixins/k-mixins'
import orderbookMixins from '@/mixins/orderbook-mixins'
import { mapState, mapActions, mapGetters} from 'vue'
import Card from '@/components/Card'
//var echarts = require('echarts')
import echarts from '@/libs/pkgs/initEcharts'


import NP from 'number-precision'
import { getTradeAggregation } from '@/api/tradeAggregation'
import { getAsset } from '@/api/assets'
var moment = require('moment')
import {Decimal} from 'decimal.js'

export default {
    mixins: [orderbookMixins, kmixins],
    data(){
        return {
           height: 340,//默认的K张图完整高度
           width: 100,//K张图宽度
           topH: 71,//底部工具条的高度
           blank: 30,//图中的空白
           miniBlank: 10,//
           kH: 120, //K线图高度
           dH: 20,//内容选择框高度
           vH: 40,//交易量高度
           mH: 40,//macd图的高度
           toolH: 37,//底部切换区间的工具条的高度
        }
    },
    computed: {
    },
    beforeDestroy () {
      screen.orientation.lock('portrait');        
    },
    beforeCreate () {
      screen.orientation.lock('landscape');  
    },
    created () {
      let clientH = document.documentElement.clientWidth
      this.width = document.documentElement.clientHeight - 10
      let allkH = new Decimal(clientH).minus(this.topH).minus(this.toolH)
      let h = this.height - this.blank*3 - this.miniBlank - this.toolH
      let allkHB = allkH.minus(this.blank*3).minus(this.miniBlank)
      this.kH = allkHB.times(this.kH).div(h).toNumber()
      this.vH = allkHB.times(this.vH).div(h).toNumber()
      this.mH = allkHB.times(this.mH).div(h).toNumber()
      this.dH = allkHB.times(this.dH).div(h).toNumber()
      this.height = allkH.toNumber()
    },
    mounted () {

        this.$nextTick(()=>{
           //加载界面
          this.initView()
        })
    },
    watch:{
      data(){
        this.resetViewData()
      }
    },
    methods: {
        initView() {
            this.ele = echarts.init(document.getElementById(this.id))
            this.opt = this.koption()
            this.ele.setOption(this.opt)
        },
        resetViewData(){
          if(!this.opt)return
          this.opt.xAxis[0].data = this.dates
          this.opt.xAxis[1].data = this.dates
          this.opt.series[0].data = this.data
          this.opt.series[1].data = this.calculateMA(5)
          this.opt.series[2].data = this.calculateMA(10)
          this.opt.series[3].data = this.boll.upper
          this.opt.series[4].data = this.boll.lower
          this.opt.series[5].data = this.boll.mid
          this.opt.series[6].data = this.volumes
          this.opt.series[7].data = this.macd.bars
          this.opt.series[8].data = this.macd.diffs
          this.opt.series[9].data = this.macd.deas


          this.ele.setOption(this.opt)
        },
        koption() {

            return {
                animation: true,
                color: this.colors,
                backgroundColor: "#212122",
              //  title: {left: 'center', text: this.base.code + '/' + this.counter.code },
                legend: { show: true, top: 2,data: ['MA5', 'MA10','UB','MB','LB'], textStyle: {
                  color: "#777"
                }, inactiveColor: '#555'},
                tooltip: {
                    trigger: 'axis',
                    axisPointer: { type: 'cross' },
                    position:  (pos, params, dom, rect, size)=>{
                        return pos[0] < size.viewSize[0] / 2 ? {top: 10, right: 10}:{top: 10, left: 10}
                    },
                    confine: true,
                    formatter: (params, tick)=>{
                        let series = params.sort((a,b)=>a.seriesIndex>b.seriesIndex);
                        let result = `${series[0].name}<br/>`;
                        let openlabel = this.$t('open_price')
                        let closelabel = this.$t('close_price')
                        let highlabel = this.$t('high_price')
                        let lowlabel = this.$t('low_price')
                        let volumelabel = this.$t('volumes')
                        if(series[0].data){
                            result += `${openlabel}: ${series[0].data[1]}<br/>`
                            + `${closelabel}: ${series[0].data[2]}<br/>`
                            + `${highlabel}: ${series[0].data[3]}<br/>`
                            + `${lowlabel}: ${series[0].data[4]}<br/>`
                            
                        }
                        try{
                        result += 
                             `MA5: ${series[1].data}<br/>`
                            + `MA10: ${series[2].data}<br/>`
                            + `UB: ${series[3].data.toFixed(7)}<br/>`
                            + `MB: ${series[5].data.toFixed(7)}<br/>`
                            + `LB: ${series[4].data.toFixed(7)}<br/>`
                            + `${volumelabel}: ${series[6].data.toFixed(7)}<br/>`
                        }catch(err){
                            console.error(err)
                        }

                        return result
                    }

                },
                axisPointer: { link: [{xAxisIndex: [0, 1, 2]}]
                },
                dataZoom: [{
                    type: 'slider',
                    xAxisIndex: [0, 1, 2],
                    realtime: false,
                    start: 50,
                    end: 100,
                    top: this.blank*2+this.kH,//65,
                    height: this.dH,
                    handleIcon: 'M10.7,11.9H9.3c-4.9,0.3-8.8,4.4-8.8,9.4c0,5,3.9,9.1,8.8,9.4h1.3c4.9-0.3,8.8-4.4,8.8-9.4C19.5,16.3,15.6,12.2,10.7,11.9z M13.3,24.4H6.7V23h6.6V24.4z M13.3,19.6H6.7v-1.4h6.6V19.6z',
                    handleSize: '120%'
                },{
                    type: 'inside',
                    xAxisIndex: [0, 1, 2],
                    start: 50,
                    end: 100
                }],
                xAxis: [{
                    type: 'category',
                    data: this.dates,
                    boundaryGap : false,
                    axisLine: { lineStyle: { color: '#777' } },
                    axisLabel: {
                        formatter: function (value) {
                            return value
                        }
                    },
                    min: 'dataMin',
                    max: 'dataMax',
                    axisPointer: {show: true}
                }, {
                    type: 'category',
                    gridIndex: 1,
                    data: this.dates,
                    scale: true,
                    boundaryGap : false,
                    splitLine: {show: false},
                    axisLabel: {show: false},
                    axisTick: {show: false},
                    axisLine: { lineStyle: { color: '#777' } },
                    splitNumber: 20,
                    min: 'dataMin',
                    max: 'dataMax',
                    axisPointer: {
                        type: 'shadow',
                        label: {show: false},
                        triggerTooltip: true,
                        handle: {show: false,margin: 30,color: '#B80C00'}
                    }
                }, {
                    type: 'category',
                    gridIndex: 2,
                    data: this.dates,
                    scale: true,
                    boundaryGap : false,
                    splitLine: {show: false},
                    axisLabel: {show: false},
                    axisTick: {show: false},
                    axisLine: { lineStyle: { color: '#777' } },
                    splitNumber: 20,
                    min: 'dataMin',
                    max: 'dataMax',
                    axisPointer: {
                        type: 'shadow',
                        label: {show: false},
                        triggerTooltip: true,
                        handle: {show: false,margin: 30,color: '#B80C00'}
                    }
                }],
                yAxis: [{
                    scale: true,
                    splitNumber: 2,
                    axisLine: { lineStyle: { color: '#777' } },
                    splitLine: { show: false },
                    axisTick: { show: false },
                    axisLabel: {
                        inside: true,
                        formatter: '{value}\n'
                    }
                }, {
                    scale: true,
                    gridIndex: 1,
                    splitNumber: 2,
                    axisLabel: { inside: true,
                        formatter: '{value}\n'},
                    axisLine: {show: true,lineStyle: { color: '#777' }},
                    axisTick: {show: false},
                    splitLine: {show: false}
                }, {
                    scale: true,
                    gridIndex: 2,
                    splitNumber: 4,
                    axisLine: {onZero: true,lineStyle: { color: '#777' }},
                    axisTick: {show: false},
                    splitLine: {show: false},
                    axisLabel: { inside: true,
                        formatter: '{value}\n'}
                }],
                grid: [{
                    left: 5,
                    right: 5,
                    top: this.blank,
                    height: this.kH,
                    bottom: 0
                }, {
                    left: 5,
                    right: 5,
                    height: this.vH,
                    top: this.blank*2 + this.miniBlank+this.kH + this.dH,
                    bottom: 0
                }, {
                    left: 5,
                    right: 5,
                    height: this.mH,
                    top: this.blank*3+ this.miniBlank+this.kH+this.dh + this.vH,
                    bottom: 0
                }],
                series: [{
                    type: 'candlestick',
                    name: '分时',
                    data: this.data,
                    itemStyle: {
                        normal: {
                            color: '#14b143',
                            color0: '#ef232a',
                            borderColor: '#14b143',
                            borderColor0: '#ef232a'
                        },
                        emphasis: {
                            color: '#444',
                            color0: 'black',
                            borderColor: '#444',
                            borderColor0: 'black'
                        }
                    }
                }, {
                    name: 'MA5',
                    type: 'line',
                    data: this.calculateMA(5),
                    smooth: true,
                    showSymbol: false,
                    lineStyle: {
                        normal: {
                            width: 1
                        }
                    }
                }, {
                    name: 'MA10',
                    type: 'line',
                    data: this.calculateMA(10),
                    smooth: true,
                    showSymbol: false,
                    lineStyle: {
                        normal: {
                            width: 1
                        }
                    }
                },
                // boll start
                {
                    name: 'UB',
                    type: 'line',
                    data: this.boll.upper,
                    smooth: true,
                    showSymbol: false,
                    lineStyle: {
                        normal: {
                            width: 1,
                            color:'#9c27b0',
                        }
                    }
                },
               {
                    name: 'LB',
                    type: 'line',
                    data: this.boll.lower,
                    smooth: true,
                    showSymbol: false,
                    lineStyle: {
                        normal: {
                            width: 1,
                            color:'#ffeb3b',
                        }
                    }
                }, {
                    name: 'MB',
                    type: 'line',
                    data: this.boll.mid,
                    smooth: true,
                    showSymbol: false,
                    lineStyle: {
                        normal: {
                            width: 1,
                            color:'#f5f5f5',
                        }
                    }
                },

                {
                    name: 'Volume',
                    type: 'bar',
                    xAxisIndex: 1,
                    yAxisIndex: 1,
                    itemStyle: {
                        normal: {
                          color: (params)=>{
                            let obj = this.subVolumes[params.dataIndex]
                            if(obj && obj[0] > obj[1]){
                                return "#733520"
                            }
                            return '#216549'
                          }
                        }
                    },
                    data: this.volumes
                }, 

                // boll end --
                // macd --start 
                {
                  name: 'MACD',
                  type: 'bar',
                  xAxisIndex: 2,
                  yAxisIndex: 2,
                  data: this.macd.bars,
                  itemStyle: {
                  normal: {
                        color: function(params) {
                            var colorList;
                            if (params.data >= 0) {
                                colorList = '#ef232a';
                            } else {
                                colorList = '#14b143';
                            }
                            return colorList;
                        },
                    }
                  }
                },{
                    name: 'DIF',
                    type: 'line',
                    xAxisIndex: 2,
                    yAxisIndex: 2,
                    data: this.macd.diffs
                },{
                    name: 'DEA',
                    type: 'line',
                    xAxisIndex: 2,
                    yAxisIndex: 2,
                    data: this.macd.deas
                }
                // macd end===
                ]
            }
        }, // end of koption
        chgResolutionKey(key){
            this.chgResolution(key)
            this.reloadAll()
        },
        reloadAll(){
            this.cleanData()
            this.initView();
            this.fetch();
            this.fetchLastTradeAggregation()
            this.fetchLastTrade()
        }

    },
    components: {
        Card
    }

}
</script>

<style lang="stylus" scoped>
@require './K.styl'
</style>
