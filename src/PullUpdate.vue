<template>
  <section class="scroll-load"
           :style="{
             height: (height || windowHeight) + 'px',
             'padding-top': paddingTop + 'px',
             'padding-bottom': paddingBottom + 'px'
           }"
           @touchstart="touchStart"
           @touchmove="touchMove"
           @touchend="touchEnd"
           @touchcancel="touchEnd">

    <header v-if="refreshing"
            class="refresh-area"
            :style="{height: refreshHeight + 'px'}">
      <slot name="refreshing">
        <icon icon="loading" size="20"/>
        <span class="action-text">刷新中···</span>
      </slot>
    </header>
    <header v-else-if="downPulling"
            class="refresh-area"
            :style="{height: Math.abs(contentTranslateY) + 'px'}">
      <slot name="downPullingRefresh" v-if="downRefresh">
        <div :style="{transform: 'rotate(' + refreshRotate + 'deg)'}"><icon icon="refresh" size="20"/></div>
        <span class="action-text">松开刷新</span>
      </slot>
      <slot name="downPullingNotRefresh" v-else>
        <div :style="{transform: 'rotate(' + refreshRotate + 'deg)'}"><icon icon="refresh" size="20"/></div>
        <span class="action-text">下拉刷新</span>
      </slot>
    </header>

    <div class="content-area"
         ref="contentBody"
         :style="{ transform: 'translateY(' + contentTranslateY + 'px)' }"
         @scroll="scrollHandler">
      <slot></slot>
      <slot name="over" v-if="!refreshing && over">
        <div class="load-over-area">
          <div :class="['load-over-text', {'fix-at-bottom': !this.contentBodyScroll}]">- 没有更多了 -</div>
        </div>
      </slot>
    </div>

    <footer v-if="loading"
            class="load-area"
            :style="{height: loadHeight + 'px'}">
      <slot name="loading">
        <icon icon="loading" size="20"/>
        <span class="action-text">加载中···</span>
      </slot>
    </footer>
    <footer v-else-if="upPulling"
            class="load-area"
            :style="{height: Math.abs(contentTranslateY) + 'px'}">
      <slot name="upPulling">松开加载</slot>
    </footer>

  </section>
</template>
<script>
  import Icon from './Icon'

  export default {
    name: 'PullUpdate',

    components: { Icon },

    props: {
      height: { // 组件的高度，可不传。如果要传，最小要40
        type: Number,
        default: 0,
        validator: (val) => {
          return !val || val >= 40
        }
      },

      down: { // 下拉刷新，可不传。不传则不触发
        type: Function,
        default: null
      },

      up: { // 上拉加载，可不传。不传则不触发
        type: Function,
        default: null
      },

      scroll: { // 滚动加载，可不传。不传则不触发
        type: Function,
        default: null
      },

      minLoadDistance: { // 滚动加载时，预加载的最小距离。到组件底部的距离，小于这个值就会触发加载
        type: Number,
        default: 20,
        validator: (val) => {
          return val >= 0
        }
      }
    },

    data () {
      return {
        windowHeight: 0, // 当前窗口所允许的最大高度，如果没有传入height则采用这个值
        refreshHeight: 32, // 刷新中头部提示的节点高度
        loadHeight: 40, // 加载中脚部提示的节点高度

        refreshing: false, // 刷新中
        downPulling: false, // 下拉中
        downRefresh: false, // 是否到达下拉刷新的阈值

        loading: false, // 加载中
        upPulling: false, // 上拉中

        contentTranslateY: 0, // 内容节点偏移量
        contentBodyScroll: false, // 内容是否有滚动
        target: null, // touch事件发生的第一个节点，不支持多点滑动
        startY: 0, // 上滑到顶部或下滑到底部时，记录的touch位置
        scrollY: 0, // 内容节点滚动的位置

        over: false // 是否加载完成，如果设为true，所有加载的方法不再调用
      }
    },

    computed: {
      paddingTop () { // 刷新中给头部提示留的位置
        return this.refreshing ? this.refreshHeight : 0
      },

      paddingBottom () { // 没有更多或加载中给底部提示留的位置
        return this.loading ? this.loadHeight : 0
      },

      refreshRotate () { // 默认下拉刷新效果中的旋转角度
        let height = Math.max(this.height || this.windowHeight, 1) // 防止除0
        return Math.abs(this.contentTranslateY) / height * 360 * 1.8
      }
    },

    watch: {
      contentTranslateY (newVal, oldVal) {
        const MIN_DOWN = 80 // 下拉超过这个值才触发刷新
        const MIN_UP = -40 // 上拉超过这个值才触发加载
        if (typeof this.down === 'function') {
          if (oldVal > MIN_DOWN && newVal === 0) {
            // 松开刷新
            this.downPulling = false
            this.refreshing = true
            this.down.call(null, this)
          } else {
            // 下拉
            this.downPulling = newVal >= 28
            this.downRefresh = newVal > MIN_DOWN
          }
        }

        if (!this.over && typeof this.up === 'function' && !this.loading) {
          if (this.upPulling && newVal === 0) {
            // 松开加载
            this.upPulling = false
            this.loading = true
            this.up.call(null, this)
          } else {
            // 上拉
            this.upPulling = newVal <= MIN_UP
          }
        }
      }
    },

    mounted () {
      this.reCalcHeight()
      this.observeContent()
    },

    methods: {
      scrollHandler (evt) {
        let content = evt.target
        if (!this.over && !this.loading && content.scrollTop > this.scrollY) {
          // 判断是否滚动到底部
          let distanceToBottom = content.scrollHeight - content.scrollTop - content.clientHeight
          if ((typeof this.scroll === 'function') && (distanceToBottom < this.minLoadDistance)) {
            this.loading = true
            this.scroll.call(null, this)
          }
        }

        this.scrollY = content.scrollTop
      },

      touchStart (evt) {
        let touch = evt.touches[0]

        if (!this.target) {
          this.target = touch.target
          this.startY = touch.pageY
        }
      },

      touchMove (evt) {
        let content = this.$refs.contentBody
        let touch = this.getStartTouchPoint(evt.touches)

        if (!touch || touch.pageY < 0) {
          // 分期乐ios app有bug，滑动到app头部时，不能及时触发 touchend/touchcancel，这里修正下
          this.touchEnd()
          return
        }

        let newTransY = touch.pageY - this.startY
        let up = newTransY < 0
        let down = newTransY > 0
        let atTop = content.scrollTop < 1
        let atBottom = content.scrollTop + content.clientHeight >= content.scrollHeight

        if ((down && !atTop) || (up && !atBottom)) {
          // 滑动未到两端
          this.contentTranslateY = 0
          this.startY = touch.pageY
        } else if ((down && atTop) || (up && atBottom)) {
          // 滑动到一端（顶部或底部）并继续滑动
          evt.preventDefault()
          this.contentTranslateY = newTransY / 2.4 // 拉动比例
        } else {
          // 修正主体的位置
          this.contentTranslateY = 0
        }
      },

      touchEnd () {
        this.target = null
        this.contentTranslateY = 0
      },

      getStartTouchPoint (touches) {
        for (let touch of touches) {
          if (touch.target === this.target) {
            return touch
          }
        }
        return null
      },

      loaded (over) {
        this.refreshing = false
        this.loading = false
        this.over = over === true
      },

      reCalcHeight () {
        this.windowHeight = window.innerHeight - getOffsetPageTop(this.$el)
      },

      observeContent () {
        let content = this.$refs.contentBody
        let observer = new MutationObserver(() => {
          this.contentBodyScroll = content.scrollHeight > content.clientHeight
        })
        observer.observe(content, {
          childList: true,
          subtree: true
        })
      }
    }
  }

  /**
   * 节点到body顶部的
   * @param dom
   * @returns {number}
   */
  function getOffsetPageTop (dom) {
    let top = 0
    while (dom && dom !== document.body) {
      top += dom.offsetTop
      dom = dom.offsetParent
      top += dom.clientTop
    }
    return top
  }
</script>
<style scoped lang="less">
  .scroll-load {
    position: relative;
    overflow: hidden;
    box-sizing: border-box;

    .refresh-area,
    .load-area {
      box-sizing: border-box;
      position: absolute;
      width: 100%;
      color: #888;
      font-size: 14px;
      text-align: center;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .refresh-area {
      top: 0;
    }
    .load-area {
      bottom: 0;
    }

    .action-text {
      padding-left: 4px;
    }

    .content-area {
      box-sizing: border-box;
      height: 100%;
      overflow-y: auto;
      position: relative;
    }

    .load-over-area {
      height: 42px;
    }

    .load-over-text {
      width: 100%;
      line-height: 42px;
      text-align: center;
      color: #888;
      font-size: 14px;

      &.fix-at-bottom {
        position: absolute;
        bottom: 0;
      }
    }
  }
</style>