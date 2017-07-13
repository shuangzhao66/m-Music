<template>
	<scroll 
		class="listview"
		ref="listview"
		:data="data"
		:listen-scroll="listenScroll"
		:probe-type="probeType"
		@scroll="scroll"
	>
		<ul>
			<li v-for="group in data" class="list-group" ref="listGroup">
				<h2 class="list-group-title">{{group.title}}</h2>
				<ul>
					<li @click="selectItem(item)" v-for="item in group.items" class="list-group-item">
						<img v-lazy="item.avatar" class="avatar" />
						<span class="name">{{item.name}}</span>
					</li>
				</ul>
			</li>
		</ul>
		<div class="list-shortcut" @touchstart="onShortcutTouchStart" 
			@touchmove.stop.prevent="onShortcutTouchMove">
			<ul>
				<li v-for="(item, index) in shortcutList" :data-index="index" class="item"
					:class="{'current': currentIndex===index}">
					{{item}}
				</li>
			</ul>
		</div>
		<div class="list-fixed" ref="fixed" v-show="fixedTitle">
			<div class="fixed-title">{{fixedTitle}}</div>
		</div>
		<div v-show="!data.length" class="loading-container">
			<loading></loading>
		</div>
	</scroll>
</template>

<script type="text/x-ecmascript-6">
	import Scroll from 'base/scroll/scroll'
	import Loading from 'base/loading/loading'
	import {getData} from 'common/js/dom'
	
	const TITLE_HEIGHT = 30
	const ANCHER_HEIGHT = 18
	export default {
		props: {
			data: {
				type: Array,
				default: []
			}
		},
		computed: {
			//截取右侧列表只要一个字 热
			shortcutList() {
				return this.data.map( (group) => {
					return group.title.substr(0, 1)
				})
			},
			
			fixedTitle() {
				if (this.scrollY > 0) {
					return ''
				}
				//console.log(this.data[this.currentIndex].title)
				return this.data[this.currentIndex] ? this.data[this.currentIndex].title : ''
			}
			
		},
		data() {
			return {
				scrollY: -1,
				currentIndex: 0,
				diff: -1
			}
		},
		methods: {
			selectItem (item) {
				this.$emit('select', item)
			},
			//点击右侧，滚动到左侧对应的索引
			onShortcutTouchStart (e) {
				//e.target拿到当前点击的li getDate 用来获取属性的值 当前的索引
				let anchorIndex = getData(e.target, 'index')
				let firstTouch = e.touches[0]
				this.touch.y1 = firstTouch.pageY
				this.touch.anchorIndex = anchorIndex
				
				this._scrollTo(anchorIndex)
				
			},
			//判断手指第二次的位置除以行高  向下取整 判断对应的索引应该加还是减， 滚动到对应的位置
			onShortcutTouchMove (e) {
				let firstTouch = e.touches[0]
				this.touch.y2 = firstTouch.pageY
				let delta = (this.touch.y2 - this.touch.y1) / ANCHER_HEIGHT | 0
				let anchorIndex = parseInt(this.touch.anchorIndex) + delta
				this._scrollTo(anchorIndex)
			},
			//每个字母对应的区间的高度，放进listHeight数组
			_calculateHeight() {
				this.listHeight = []
				const list = this.$refs.listGroup
				let height = 0
				this.listHeight.push(height)
				for(let i = 0; i < list.length; i++) {
					let item = list[i]
					height += item.clientHeight
					this.listHeight.push(height)
				}
			},
			//传入的index是右侧字母列表的索引
			_scrollTo(index) {
				if(!index && index !== 0) {
					return 
				}
				if(index < 0) {
					index = 0
				} else if (index > this.listHeight.length - 2) {
					index = this.listHeight.length - 2
				}
				this.scrollY = -this.listHeight[index]
				this.$refs.listview.scrollToElement(this.$refs.listGroup[index], 0)
			},
			scroll(pos) {
				//console.log(pos.y)
				this.scrollY = pos.y
			}
			,
			refresh() {
		        this.$refs.listview.refresh()
		    },
		},
		created() {
	      this.probeType = 3
	      this.listenScroll = true
	      this.touch = {}
	      this.listHeight = []
		},
		watch: {
			data() {
				setTimeout( () => {
					this._calculateHeight()
				}, 20)
			},
			scrollY (newY) {
				const listHeight = this.listHeight
				//滚动到顶部 newY > 0
				if (newY > 0) {
					this.currentIndex = 0
					return 
				}
				for(let i = 0; i < listHeight.length -1; i++) {
					let height1 = listHeight[i]
					let height2 = listHeight[i + 1]
					if(-newY >= height1 && -newY < height2) {
						this.currentIndex = i
						//diff会
						this.diff = height2 + newY
						return
					}
				}
				
				//滚动到底部 -newY大于最后一个元素的上限
				this.currentIndex = listHeight.length - 2
			},
			//diff表示 上一个区间的值
			diff (newVal){
				let fixedTop = (newVal > 0 && newVal < TITLE_HEIGHT) ? newVal - TITLE_HEIGHT : 0
				if(this.fixedTop === fixedTop) {
					return 
				}
				this.fixedTop = fixedTop
				this.$refs.fixed.style.transform = `translate3d(0,${fixedTop}px,0)`
			}
		},
		components: {
			Scroll,
			Loading
		}
	}
</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
  @import "~common/stylus/variable"

  .listview
    position: relative
    width: 100%
    height: 100%
    overflow: hidden
    background: $color-background
    .list-group
      padding-bottom: 30px
      .list-group-title
        height: 30px
        line-height: 30px
        padding-left: 20px
        font-size: $font-size-small
        color: $color-text-l
        background: $color-highlight-background
      .list-group-item
        display: flex
        align-items: center
        padding: 20px 0 0 30px
        .avatar
          width: 50px
          height: 50px
          border-radius: 50%
        .name
          margin-left: 20px
          color: $color-text-l
          font-size: $font-size-medium
    .list-shortcut
      position: absolute
      z-index: 30
      right: 0
      top: 50%
      transform: translateY(-50%)
      width: 20px
      padding: 20px 0
      border-radius: 10px
      text-align: center
      background: $color-background-d
      font-family: Helvetica
      .item
        padding: 3px
        line-height: 1
        color: $color-text-l
        font-size: $font-size-small
        &.current
          color: $color-theme
    .list-fixed
      position: absolute
      top: 0
      left: 0
      width: 100%
      .fixed-title
        height: 30px
        line-height: 30px
        padding-left: 20px
        font-size: $font-size-small
        color: $color-text-l
        background: $color-highlight-background
    .loading-container
      position: absolute
      width: 100%
      top: 50%
      transform: translateY(-50%)
</style>