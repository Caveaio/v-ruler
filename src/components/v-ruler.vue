<template>
  <div :style="wrapperStyle" class="vue-ruler-wrapper" onselectstart="return false;">
    <section v-show="rulerToggle">
      <div ref="horizontalRuler" class="vue-ruler-h" @mousedown.stop="horizontalDragRuler" style="background-color: #000000">
        <span v-for="(item,index) in xScale" :key="index" :style="{left:index * 50 + 2 + 'px'}" class="n">{{ item.id * scale }}</span>
      </div>
      <div ref="verticalRuler" class="vue-ruler-v" @mousedown.stop="verticalDragRuler" style="background-color: #000000">
        <span v-for="(item,index) in yScale" :key="index" :style="{top:index * 50 + 2 + 'px'}" class="n">{{ item.id * scale }}</span>
      </div>
      <div :style="{top:verticalDottedTop + 'px'}" class="vue-ruler-ref-dot-h" />
      <div :style="{left:horizontalDottedLeft + 'px'}" class="vue-ruler-ref-dot-v" />
      <div
        v-for="item in lineList"
        :title="item.title"
        :style="getLineStyle(item)"
        :key="item.id"
        :class="`vue-ruler-ref-line-${item.type}`"
        @mousedown="handleDragLine(item)"></div>
    </section>
    <div ref="content" class="vue-ruler-content" :style="contentStyle">
      <slot />
    </div>
    <div v-show="isDrag" class="vue-ruler-content-mask"></div>
  </div>
</template>

<script>
import { on, off } from './event.js'
export default {
  name: 'VRuler',
  components: {},
  props: {
    position: {
      type: String,
      default: 'relative',
      validator: function (val) {
        return ['absolute', 'fixed', 'relative', 'static', 'inherit'].indexOf(val) !== -1
      }
    }, // 规定元素的定位类型
    isHotKey: {
      type: Boolean, default: true
    }, // 热键开关
    isScaleRevise: {
      type: Boolean, default: false
    }, // 刻度修正(根据content进行刻度重置)
    value: {
      type: Array,
      default: () => {
        return [] // { type: 'h', site: 50 }, { type: 'v', site: 180 }
      }
    }, // 预置参考线
    contentLayout: {
      type: Object,
      default: () => {
        return { top: 0, left: 0 }
      }
    }, // 内容部分布局
    parent: {
      type: Boolean,
      default: false
    },
    visible: {
      type: Boolean,
      default: true
    },
    scale:{
      type:Number,
      default:1,
    }
  },
  data () {
    return {
      size: 17,
      left_top: 18, // 内容左上填充
      windowWidth: 0, // 窗口宽度
      windowHeight: 0, // 窗口高度
      xScale: [], // 水平刻度
      yScale: [], // 垂直刻度
      topSpacing: 0, // 标尺与窗口上间距
      leftSpacing: 0, //  标尺与窗口左间距
      isDrag: false,
      dragFlag: '', // 拖动开始标记，可能值x(从水平标尺开始拖动),y(从垂直标尺开始拖动)
      horizontalDottedLeft: -999, // 水平虚线位置
      verticalDottedTop: -999, // 垂直虚线位置
      rulerWidth: 0, // 垂直标尺的宽度
      rulerHeight: 0, // 水平标尺的高度
      dragLineId: '', // 被移动线的ID
      keyCode: {
        r: 82
      }, // 快捷键参数
      rulerToggle: true // 标尺辅助线显示开关
    }
  },
  computed: {
    wrapperStyle() {
      return {
        width : this.windowWidth + 'px',
        height : this.windowHeight + 'px',
        position: this.position
      }
    },
    contentStyle() {
      return {
        left: this.contentLayout.left + 'px',
        top: this.contentLayout.top + 'px',
        padding: this.left_top + 'px 0px 0px ' + this.left_top + 'px'
      }
    },
    lineList() {
      let hCount = 0;
      let vCount = 0;
      return this.value.map((item) => {
        const isH = item.type === 'h'
        return {
          id: `${item.type}_${isH ? hCount++ : vCount++}`,
          type: item.type,
          title: item.site + 'px',
          [isH ? 'top' : 'left']: item.site + this.size
        }
      })
    }
  },
  watch: {
    visible: {
      handler(visible) {
        this.rulerToggle = visible;
      },
      immediate: true
    }
  },
  mounted () {

    on(document, 'mousemove', this.dottedLineMove)
    on(document, 'mouseup', this.dottedLineUp)
    on(document, 'keyup', this.keyboard)
    this.init()
    const self = this // 绑定窗口调整大小onresize事件
    window.onresize = function () { // 如果直接使用this,this指向的不是vue实例
      self.xScale = []
      self.yScale = []
      self.init()
    }
  },
  beforeDestroy () {
    off(document, 'mousemove', this.dottedLineMove)
    off(document, 'mouseup', this.dottedLineUp)
    off(document, 'keyup', this.keyboard)
  },
  methods: {
    init () {    
      this.box()
      this.scaleCalc()
    },
    getLineStyle({type, top, left}) {
      return type === 'h' ? {top: top+ 'px'} : {left: left + 'px'}
    },
    handleDragLine({type, id}) {
      return type === 'h' ? this.dragHorizontalLine(id) : this.dragVerticalLine(id)
    },
    box () {
      if (this.isScaleRevise) { // 根据内容部分进行刻度修正
        const content = this.$refs.content
        const contentLeft = content.offsetLeft
        const contentTop = content.offsetTop
        for (let i = 0; i < contentLeft; i += 1) {
          if (i % 50 === 0 && i + 50 <= contentLeft) {
            this.xScale.push({ id: i })
          }
        }
        for (let i = 0; i < contentTop; i += 1) {
          if (i % 50 === 0 && i + 50 <= contentTop) {
            this.yScale.push({ id: i })
          }
        }
      }
      if (this.parent) {
        const style = window.getComputedStyle(this.$el.parentNode, null)
        this.windowWidth = parseInt(style.getPropertyValue('width'), 10)
        this.windowHeight = parseInt(style.getPropertyValue('height'), 10)
      } else {
        this.windowWidth = document.documentElement.clientWidth - this.leftSpacing
        this.windowHeight = document.documentElement.clientHeight - this.topSpacing
      }
      this.rulerWidth = this.$refs.verticalRuler.clientWidth
      this.rulerHeight = this.$refs.horizontalRuler.clientHeight
      this.setSpacing()
    }, // 获取窗口宽与高
    setSpacing () {
      this.topSpacing = this.$refs.horizontalRuler.getBoundingClientRect().y //.offsetParent.offsetTop
      this.leftSpacing = this.$refs.verticalRuler.getBoundingClientRect().x// .offsetParent.offsetLeft
    },
    scaleCalc () {
      for (let i = 0; i < this.windowWidth; i += 1) {
        if (i % 50 === 0) {
          this.xScale.push({ id: i })
        }
      }
      for (let i = 0; i < this.windowHeight; i += 1) {
        if (i % 50 === 0) {
          this.yScale.push({ id: i })
        }
      }
    }, // 计算刻度
    newHorizontalLine () {
      this.isDrag = true
      this.dragFlag = 'x'
    }, // 生成一个水平参考线
    newVerticalLine () {
      this.isDrag = true
      this.dragFlag = 'y'
    }, // 生成一个垂直参考线
    dottedLineMove ($event) {
      this.setSpacing()
      switch (this.dragFlag) {
        case 'x':
          if (this.isDrag) {
            this.verticalDottedTop = $event.pageY - this.topSpacing
          }
          break
        case 'y':
          if (this.isDrag) {
            this.horizontalDottedLeft = $event.pageX - this.leftSpacing
          }
          break
        case 'h':
          if (this.isDrag) {
            this.verticalDottedTop = $event.pageY - this.topSpacing
          }
          break
        case 'v':
          if (this.isDrag) {
            this.horizontalDottedLeft = $event.pageX - this.leftSpacing
          }
          break
        default:
          break
      }
    }, // 虚线移动
    dottedLineUp ($event) {
      if (this.isDrag) {
        this.setSpacing()
        this.isDrag = false
        const cloneList = JSON.parse(JSON.stringify(this.value))
        switch (this.dragFlag) {
          case 'x':
            cloneList.push({
              type: 'h',
              site: $event.pageY - this.topSpacing - this.size
            })
            this.$emit('input', cloneList)
            break
          case 'y':
            cloneList.push({
              type: 'v',
              site: $event.pageX - this.leftSpacing - this.size
            })
            this.$emit('input', cloneList)
            break
          case 'h':
            if ($event.pageY - this.topSpacing < this.rulerHeight) {
              let Index, id
              this.lineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              cloneList.splice(Index, 1, {
                type: 'h',
                site: -600
              })
            } else {
              let Index, id
              this.lineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              cloneList.splice(Index, 1, {
                type: 'h',
                site: $event.pageY - this.topSpacing - this.size
              })
            }
            this.$emit('input', cloneList)
            break
          case 'v':
            if ($event.pageX - this.leftSpacing < this.rulerWidth) {
              let Index, id
              this.lineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              cloneList.splice(Index, 1, {
                type: 'v',
                site: -600
              })
            } else {
              let Index, id
              this.lineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              cloneList.splice(Index, 1, {
                type: 'v',
                site: $event.pageX - this.leftSpacing - this.size
              })
            }
            this.$emit('input', cloneList)
            break
          default:
            break
        }
        this.verticalDottedTop = this.horizontalDottedLeft = -10
      }
    }, // 虚线松开
    horizontalDragRuler () {
      this.newHorizontalLine()
    }, // 水平标尺处按下鼠标
    verticalDragRuler () {
      this.newVerticalLine()
    }, // 垂直标尺处按下鼠标
    dragHorizontalLine (id) {
      this.isDrag = true
      this.dragFlag = 'h'
      this.dragLineId = id
    }, // 水平线处按下鼠标
    dragVerticalLine (id) {
      this.isDrag = true
      this.dragFlag = 'v'
      this.dragLineId = id
    }, // 垂直线处按下鼠标
    keyboard ($event) {
      if (this.isHotKey) {
        switch ($event.keyCode) {
          case this.keyCode.r:
            this.rulerToggle = !this.rulerToggle
            this.$emit('update:visible', this.rulerToggle)
            if (this.rulerToggle) {
              this.left_top = 18;
            } else {
              this.left_top = 0;
            }
            break
        }
      }
    }, // 键盘事件
  }
}
</script>

<style lang="scss">
.vue-ruler{
  &-wrapper {
    left: 0;
    top: 0;
    z-index: 999;
    overflow: hidden;
    user-select: none;
  }
  &-h,
  &-v,
  &-ref-line-v,
  &-ref-line-h,
  &-ref-dot-h,
  &-ref-dot-v {
    position: absolute;
    left: 0;
    top: 0;
    overflow: hidden;
    z-index: 999;
  }
  &-h,
  &-v,
  &-ref-line-v,
  &-ref-line-h,
  &-ref-dot-h,
  &-ref-dot-v {
    position: absolute;
    left: 0;
    top: 0;
    overflow: hidden;
    z-index: 999;
  }

  &-h {
    width: 100%;
    height: 18px;
    left: 18px;   
     background-color: #000000;
   // background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAASCAMAAAAuTX21AAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAlQTFRFMzMzAAAA////BqjYlAAAACNJREFUeNpiYCAdMDKRCka1jGoBA2JZZGshiaCXFpIBQIABAAplBkCmQpujAAAAAElFTkSuQmCC)
   background:url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAASCAYAAAAZk42HAAAACXBIWXMAAAsTAAALEwEAmpwYAAAF8WlUWHRYTUw6Y29tLmFkb2JlLnhtcAAAAAAAPD94cGFja2V0IGJlZ2luPSLvu78iIGlkPSJXNU0wTXBDZWhpSHpyZVN6TlRjemtjOWQiPz4gPHg6eG1wbWV0YSB4bWxuczp4PSJhZG9iZTpuczptZXRhLyIgeDp4bXB0az0iQWRvYmUgWE1QIENvcmUgNi4wLWMwMDIgNzkuMTY0NDg4LCAyMDIwLzA3LzEwLTIyOjA2OjUzICAgICAgICAiPiA8cmRmOlJERiB4bWxuczpyZGY9Imh0dHA6Ly93d3cudzMub3JnLzE5OTkvMDIvMjItcmRmLXN5bnRheC1ucyMiPiA8cmRmOkRlc2NyaXB0aW9uIHJkZjphYm91dD0iIiB4bWxuczp4bXA9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC8iIHhtbG5zOnhtcE1NPSJodHRwOi8vbnMuYWRvYmUuY29tL3hhcC8xLjAvbW0vIiB4bWxuczpzdEV2dD0iaHR0cDovL25zLmFkb2JlLmNvbS94YXAvMS4wL3NUeXBlL1Jlc291cmNlRXZlbnQjIiB4bWxuczpkYz0iaHR0cDovL3B1cmwub3JnL2RjL2VsZW1lbnRzLzEuMS8iIHhtbG5zOnBob3Rvc2hvcD0iaHR0cDovL25zLmFkb2JlLmNvbS9waG90b3Nob3AvMS4wLyIgeG1wOkNyZWF0b3JUb29sPSJBZG9iZSBQaG90b3Nob3AgMjIuMCAoV2luZG93cykiIHhtcDpDcmVhdGVEYXRlPSIyMDIxLTAzLTE1VDEyOjQzOjM4KzAxOjAwIiB4bXA6TWV0YWRhdGFEYXRlPSIyMDIxLTAzLTE1VDEyOjQzOjM4KzAxOjAwIiB4bXA6TW9kaWZ5RGF0ZT0iMjAyMS0wMy0xNVQxMjo0MzozOCswMTowMCIgeG1wTU06SW5zdGFuY2VJRD0ieG1wLmlpZDpmM2IzMjBiNy1mMjcyLWE0NGMtODk4Zi1jNjMyYWNhZmNlOWMiIHhtcE1NOkRvY3VtZW50SUQ9ImFkb2JlOmRvY2lkOnBob3Rvc2hvcDpiODU1Njg1ZS0xZTc4LWJiNGItOTBiOS0zYWQ3ZDU2MjQ3MzYiIHhtcE1NOk9yaWdpbmFsRG9jdW1lbnRJRD0ieG1wLmRpZDo5YTAzYmI5Yi04YWMzLTM4NDktYmRmYi03ZWIxZGQyOTFlNjMiIGRjOmZvcm1hdD0iaW1hZ2UvcG5nIiBwaG90b3Nob3A6Q29sb3JNb2RlPSIzIiBwaG90b3Nob3A6SUNDUHJvZmlsZT0ic1JHQiBJRUM2MTk2Ni0yLjEiPiA8eG1wTU06SGlzdG9yeT4gPHJkZjpTZXE+IDxyZGY6bGkgc3RFdnQ6YWN0aW9uPSJjcmVhdGVkIiBzdEV2dDppbnN0YW5jZUlEPSJ4bXAuaWlkOjlhMDNiYjliLThhYzMtMzg0OS1iZGZiLTdlYjFkZDI5MWU2MyIgc3RFdnQ6d2hlbj0iMjAyMS0wMy0xNVQxMjo0MzozOCswMTowMCIgc3RFdnQ6c29mdHdhcmVBZ2VudD0iQWRvYmUgUGhvdG9zaG9wIDIyLjAgKFdpbmRvd3MpIi8+IDxyZGY6bGkgc3RFdnQ6YWN0aW9uPSJzYXZlZCIgc3RFdnQ6aW5zdGFuY2VJRD0ieG1wLmlpZDpmM2IzMjBiNy1mMjcyLWE0NGMtODk4Zi1jNjMyYWNhZmNlOWMiIHN0RXZ0OndoZW49IjIwMjEtMDMtMTVUMTI6NDM6MzgrMDE6MDAiIHN0RXZ0OnNvZnR3YXJlQWdlbnQ9IkFkb2JlIFBob3Rvc2hvcCAyMi4wIChXaW5kb3dzKSIgc3RFdnQ6Y2hhbmdlZD0iLyIvPiA8L3JkZjpTZXE+IDwveG1wTU06SGlzdG9yeT4gPC9yZGY6RGVzY3JpcHRpb24+IDwvcmRmOlJERj4gPC94OnhtcG1ldGE+IDw/eHBhY2tldCBlbmQ9InIiPz7I+pFgAAAAeElEQVRIx+3WvQ6AIAwEYB5FiqP0/V/NQerAwlCTI2L8u+ES0siVbzPMSdckGt6ekCSXj0DUCCGEEEIIIYSQv0HitGzIdxLhXx4b2Te88GkQQwrRGbrDn2E7egoNe/T1EG92CnKwxO6EAMlgtDTnztR7EnPxUgF+dsDk+Xpr1z6sAAAAAElFTkSuQmCC')
   
      repeat-x; /*./image/ruler_h.png*/
  }

  &-v {
    width: 18px;
    height: 100%;
    top: 18px;
    background-color: #000000;
   // background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABIAAAAyCAMAAABmvHtTAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAlQTFRFMzMzAAAA////BqjYlAAAACBJREFUeNpiYGBEBwwMTGiAakI0NX7U9aOuHyGuBwgwAH6bBkAR6jkzAAAAAElFTkSuQmCC)
      background:url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABIAAAAyCAYAAABRYothAAAACXBIWXMAAAsTAAALEwEAmpwYAAAF8WlUWHRYTUw6Y29tLmFkb2JlLnhtcAAAAAAAPD94cGFja2V0IGJlZ2luPSLvu78iIGlkPSJXNU0wTXBDZWhpSHpyZVN6TlRjemtjOWQiPz4gPHg6eG1wbWV0YSB4bWxuczp4PSJhZG9iZTpuczptZXRhLyIgeDp4bXB0az0iQWRvYmUgWE1QIENvcmUgNi4wLWMwMDIgNzkuMTY0NDg4LCAyMDIwLzA3LzEwLTIyOjA2OjUzICAgICAgICAiPiA8cmRmOlJERiB4bWxuczpyZGY9Imh0dHA6Ly93d3cudzMub3JnLzE5OTkvMDIvMjItcmRmLXN5bnRheC1ucyMiPiA8cmRmOkRlc2NyaXB0aW9uIHJkZjphYm91dD0iIiB4bWxuczp4bXA9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC8iIHhtbG5zOnhtcE1NPSJodHRwOi8vbnMuYWRvYmUuY29tL3hhcC8xLjAvbW0vIiB4bWxuczpzdEV2dD0iaHR0cDovL25zLmFkb2JlLmNvbS94YXAvMS4wL3NUeXBlL1Jlc291cmNlRXZlbnQjIiB4bWxuczpkYz0iaHR0cDovL3B1cmwub3JnL2RjL2VsZW1lbnRzLzEuMS8iIHhtbG5zOnBob3Rvc2hvcD0iaHR0cDovL25zLmFkb2JlLmNvbS9waG90b3Nob3AvMS4wLyIgeG1wOkNyZWF0b3JUb29sPSJBZG9iZSBQaG90b3Nob3AgMjIuMCAoV2luZG93cykiIHhtcDpDcmVhdGVEYXRlPSIyMDIxLTAzLTE1VDEyOjM5OjUzKzAxOjAwIiB4bXA6TWV0YWRhdGFEYXRlPSIyMDIxLTAzLTE1VDEyOjM5OjUzKzAxOjAwIiB4bXA6TW9kaWZ5RGF0ZT0iMjAyMS0wMy0xNVQxMjozOTo1MyswMTowMCIgeG1wTU06SW5zdGFuY2VJRD0ieG1wLmlpZDplNDkyYTc5Yi0zNmQ3LTE4NDUtOTAzMy1iNGFhY2U5MzEwYzEiIHhtcE1NOkRvY3VtZW50SUQ9ImFkb2JlOmRvY2lkOnBob3Rvc2hvcDo5ODMyYjdlMS0xMGI5LTllNGUtYWEyOC1mZDcyODFjOTMyOTkiIHhtcE1NOk9yaWdpbmFsRG9jdW1lbnRJRD0ieG1wLmRpZDozYjc1MWVlNy01ZmUzLTcwNDMtOWZlYi1hMzBkZTUzZWI4MGUiIGRjOmZvcm1hdD0iaW1hZ2UvcG5nIiBwaG90b3Nob3A6Q29sb3JNb2RlPSIzIiBwaG90b3Nob3A6SUNDUHJvZmlsZT0ic1JHQiBJRUM2MTk2Ni0yLjEiPiA8eG1wTU06SGlzdG9yeT4gPHJkZjpTZXE+IDxyZGY6bGkgc3RFdnQ6YWN0aW9uPSJjcmVhdGVkIiBzdEV2dDppbnN0YW5jZUlEPSJ4bXAuaWlkOjNiNzUxZWU3LTVmZTMtNzA0My05ZmViLWEzMGRlNTNlYjgwZSIgc3RFdnQ6d2hlbj0iMjAyMS0wMy0xNVQxMjozOTo1MyswMTowMCIgc3RFdnQ6c29mdHdhcmVBZ2VudD0iQWRvYmUgUGhvdG9zaG9wIDIyLjAgKFdpbmRvd3MpIi8+IDxyZGY6bGkgc3RFdnQ6YWN0aW9uPSJzYXZlZCIgc3RFdnQ6aW5zdGFuY2VJRD0ieG1wLmlpZDplNDkyYTc5Yi0zNmQ3LTE4NDUtOTAzMy1iNGFhY2U5MzEwYzEiIHN0RXZ0OndoZW49IjIwMjEtMDMtMTVUMTI6Mzk6NTMrMDE6MDAiIHN0RXZ0OnNvZnR3YXJlQWdlbnQ9IkFkb2JlIFBob3Rvc2hvcCAyMi4wIChXaW5kb3dzKSIgc3RFdnQ6Y2hhbmdlZD0iLyIvPiA8L3JkZjpTZXE+IDwveG1wTU06SGlzdG9yeT4gPC9yZGY6RGVzY3JpcHRpb24+IDwvcmRmOlJERj4gPC94OnhtcG1ldGE+IDw/eHBhY2tldCBlbmQ9InIiPz4iNB9ZAAAAQElEQVRIDWNQVtL9A8T/KcUMykp6VMCjBuHSPBrYNDdoNGWPpuzRlD0a2KMpezRljwb2aMoeTdmjKXs0jAgYBACodfmN8XF2oAAAAABJRU5ErkJggg==')

      repeat-y; /*./image/ruler_v.png*/
  }

  &-v .n,
  &-h .n {
    position: absolute;
    font-size: 10px;
    line-height: 1;
    color: #333;
    cursor: default;
  }

  &-v .n {
    width: 8px;
    left: 3px;
    word-wrap: break-word;
  }

  &-h .n {
    top: 1px;
  }

  &-ref-line-v,
  &-ref-line-h,
  &-ref-dot-h,
  &-ref-dot-v {
    z-index: 998;
  }

  &-ref-line-h {
    width: 100%;
    height: 3px;
    background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAgAAAABCAMAAADU3h9xAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAZQTFRFSv//AAAAH8VRuAAAAA5JREFUeNpiYIACgAADAAAJAAE0lmO3AAAAAElFTkSuQmCC)
      repeat-x left center; /*./image/line_h.png*/
    cursor: n-resize; /*url(./image/cur_move_h.cur), move*/
  }

  &-ref-line-v {
    width: 3px;
    height: 100%;
    _height: 9999px;
    background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAAICAMAAAAPxGVzAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAZQTFRFSv//AAAAH8VRuAAAAA5JREFUeNpiYEAFAAEGAAAQAAGePof9AAAAAElFTkSuQmCC)
      repeat-y center top; /*./image/line_v.png*/
    cursor: w-resize; /*url(./image/cur_move_v.cur), move*/
  }

  &-ref-dot-h {
    width: 100%;
    height: 3px;
    background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAIAAAACCAMAAABFaP0WAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAZQTFRFf39/////F3PnHQAAAAJ0Uk5T/wDltzBKAAAAEElEQVR42mJgYGRgZAQIMAAADQAExkizYQAAAABJRU5ErkJggg==)
      repeat-x left 1px; /*./image/line_dot.png*/
    cursor: n-resize; /*url(./image/cur_move_h.cur), move*/
    top: -10px;
  }

  &-ref-dot-v {
    width: 3px;
    height: 100%;
    _height: 9999px;
    background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAIAAAACCAMAAABFaP0WAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAZQTFRFf39/////F3PnHQAAAAJ0Uk5T/wDltzBKAAAAEElEQVR42mJgYGRgZAQIMAAADQAExkizYQAAAABJRU5ErkJggg==)
      repeat-y 1px top; /*./image/line_dot.png*/
    cursor: w-resize; /*url(./image/cur_move_v.cur), move*/
    left: -10px;
  }
  &-content {
    position: absolute;
    z-index: 997;
  }
  &-content-mask{
    position: absolute;
    width: 100%;
    height: 100%;
    background: transparent;
    z-index: 998;
  }
}




</style>
