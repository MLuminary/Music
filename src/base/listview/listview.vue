<template>
  <scroll ref="listview" class="listview" :data="data" :listenScroll="listenScroll" @scroll="scroll" :probeType="probeType">
    <ul>
      <li v-for="(group,index) in data" :key="index" class="list-group" ref="listGroup">
        <h2 class="list-group-title">{{group.title}}</h2>
        <ul>
          <li @click="selectItem(item)" v-for="(item,index) in group.items" :key="index" class="list-group-item">
            <img class="avatar" v-lazy="item.avator">
            <span class="name">{{item.name}}</span>
          </li>
        </ul>
      </li>
    </ul>
    <div class="list-shortcut" @touchstart="onShortCutTouchStart" @touchmove.stop.prevent="onShortCutTouchMove">
      <ul>
        <li class="item" v-for="(item,index) in shortcutList" :key="index" :data-index="index" :class="{ current : currentIndex === index}">
          {{item}}
        </li>
      </ul>
    </div>
    <div class="list-fixed" v-show="fixedTitle" ref="fixed">
      <h1 class="fixed-title">{{fixedTitle}}</h1>
    </div>
    <div v-show="!data.length" class="loading-container">
      <loading></loading>
    </div>
  </scroll>
</template>

<script type="text/ecmascript-6">
import Scroll from 'base/scroll/scroll'
import { getData } from 'common/js/dom'
import Loading from 'base/loading/loading'

const ANCHOR_HEIGHT = 18 // list-shortcut 中 item 的高度
const TITLE_HEIGHT = 30 // title 的高度

export default {
  created() {
    // 因为此值并不需要设置 seter 和 geter
    this.touch = {}
    this.listenScroll = true // 是否开启监听 scroll 事件
    this.listHeight = [] // 存储 listGroup 高度区间
    this.probeType = 3
  },
  props: {
    data: {
      type: Array,
      default: () => []
    }
  },
  components: {
    Scroll,
    Loading
  },
  data() {
    return {
      scrollY: -1, // 此时滑动的 y 坐标
      currentIndex: 0, // 此时展现的模块 index
      diff: 0 // 滑动 y 坐标距离当前模块底部的距离
    }
  },
  computed: {
    shortcutList() {
      return this.data.map(group => {
        return group.title.substr(0, 1)
      })
    },
    fixedTitle() {
      if (this.scrollY > 0) {
        return ''
      }
      return this.data[this.currentIndex]
        ? this.data[this.currentIndex].title
        : ''
    }
  },
  methods: {
    selectItem(item) {
      // 点击后弹出点击的对象
      this.$emit('select', item)
    },
    onShortCutTouchStart(e) {
      let anchorIndex = parseInt(getData(e.target, 'index'))
      let firstTouch = e.touches[0] // 第一个手指的位置
      this.touch.y1 = firstTouch.pageY // 获取刚触摸时的 y 坐标值
      this.touch.anchorIndex = anchorIndex // 存储最开始的索引
      this._scrollTo(anchorIndex)
    },
    onShortCutTouchMove(e) {
      let firstTouch = e.touches[0]
      this.touch.y2 = firstTouch.pageY
      // 手指按住在 y 轴上的偏移除每个 item 高度来算出当前所在的字母
      let delta = ((this.touch.y2 - this.touch.y1) / ANCHOR_HEIGHT) | 0
      let anchorIndex = this.touch.anchorIndex + delta
      this._scrollTo(anchorIndex)
    },
    _scrollTo(index) {
      // 此方法并不会触发 scroll 事件
      if (!index && index !== 0) {
        return
      }
      // 边界判定
      if (index < 0) {
        index = 0
      } else if (index > this.listHeight.length - 2) {
        index = this.listHeight.length - 2
      }
      // 更改 scrollY 便会引起监听函数的执行
      this.scrollY = -this.listHeight[index]
      this.$refs.listview.scrollToElement(this.$refs.listGroup[index], 0) // 0 为动画的滚动时间
    },
    refresh() {
      this.$refs.listview.refresh()
    },
    scroll(pos) {
      this.scrollY = pos.y // 获取滚动的 y
    },
    _calculateHeight() {
      this.listHeight = [] // 存储每个 listGroup 的高度区间
      const list = this.$refs.listGroup
      let height = 0
      this.listHeight.push(height)
      for (let i = 0; i < list.length; i++) {
        let item = list[i]
        height += item.clientHeight
        this.listHeight.push(height)
      }
    }
  },
  // 监听
  watch: {
    data() {
      setTimeout(() => {
        this._calculateHeight()
      }, 20)
    },
    // 参数为新的 scrollY 值
    scrollY(newY) {
      const listHeight = this.listHeight
      // 当滚动到顶部 newY > 0
      if (newY > 0) {
        this.currentIndex = 0
        return
      }
      // 在中间部分滚动
      for (let i = 0; i < listHeight.length - 1; i++) {
        let heightTop = listHeight[i]
        let heightDown = listHeight[i + 1]
        // 右边注意要开
        if (-newY >= heightTop && -newY < heightDown) {
          this.currentIndex = i
          this.diff = heightDown + newY // 此时滑动量 y 距离此模块底部的距离
          return
        }
      }
      //  当滚动到底部时，且 -newY 大于最后一个元素的上限
      this.currentIndex = listHeight.length - 2
    },
    diff(newVal) {
      let fixedTop =
        newVal > 0 && newVal < TITLE_HEIGHT ? newVal - TITLE_HEIGHT : 0

      this.$refs.fixed.style.transform = `translate3d(0,${fixedTop}px,0)`
    }
  }
}
</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
@import '~common/stylus/variable'
.listview
  position relative
  width 100%
  height 100%
  overflow hidden
  background $color-background
  .list-group
    padding-bottom 30px
    .list-group-title
      height 30px
      line-height 30px
      padding-left 20px
      font-size $font-size-small
      color $color-text-l
      background $color-highlight-background
    .list-group-item
      display flex
      align-items center
      padding 20px 0 0 30px
      .avatar
        width 50px
        height 50px
        border-radius 50%
      .name
        margin-left 20px
        color $color-text-l
        font-size $font-size-medium
  .list-shortcut
    position absolute
    z-index 30
    right 0
    top 50%
    transform translateY(-50%)
    width 20px
    padding 20px 0
    border-radius 10px
    text-align center
    background $color-background-d
    font-family Helvetica
    .item
      padding 3px
      line-height 1
      color $color-text-l
      font-size $font-size-small
      &.current
        color $color-theme
  .list-fixed
    position absolute
    top 0
    left 0
    width 100%
    .fixed-title
      height 30px
      line-height 30px
      padding-left 20px
      font-size $font-size-small
      color $color-text-l
      background $color-highlight-background
  .loading-container
    position absolute
    width 100%
    top 50%
    transform translateY(-50%)
</style>
