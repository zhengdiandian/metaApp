<template>
  <div class="discover-container">
    <div class="pull-down-tip" :class="{ show: pullDownStatus > 0 }">
      <div v-if="pullDownStatus === 1">下拉刷新...</div>
      <div v-else-if="pullDownStatus === 2">释放刷新</div>
      <div v-else-if="pullDownStatus === 3">加载中...</div>
    </div>
    <div class="waterfall" ref="waterfall">
      <div class="column" v-for="(col, index) in columns" :key="index">
        <div class="item" v-for="item in col" :key="item.id">
          <div v-if="item.video" class="video-container">
            <video 
              :src="item.video" 
              controls
              playsinline
              webkit-playsinline
              x5-playsinline
              x5-video-player-type="h5"
              x5-video-player-fullscreen="true"
              preload="auto"
              @error="handleVideoError(item)"
            ></video>
          </div>
          <img v-else :src="item.cover" alt="" @load="handleImageLoad($event, item)">
          <h3>{{ item.title }}</h3>
          <p>{{ item.desc }}</p>
          <div class="item-footer">
            <div class="author-info">
              <img :src="item.avatar" class="avatar" alt="头像">
              <span class="author">{{ item.author }}</span>
            </div>
            <button class="like-btn" @click="toggleLike(item)" :class="{ liked: item.isLiked }">
              ♥ {{ item.likes }}
            </button>
          </div>
        </div>
      </div>
    </div>
    <div class="loading-tip" v-if="isLoading">正在加载更多...</div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, onBeforeUnmount } from 'vue'

export default defineComponent({
  name: 'CommunityDiscover',
  setup() {

interface DiscoverItem {
  id: number
  title: string
  desc: string
  cover: string
  video?: string
  height: number
  isPlaying?: boolean
  author: string
  avatar: string
  likes: number
  isLiked: boolean
}

    const columns = ref<DiscoverItem[][]>([[], []])
    const isLoading = ref(false)
    const isRefreshing = ref(false)
    const page = ref(1)
    const waterfall = ref<HTMLElement | null>(null)
    const startY = ref(0)
    const pullDownStatus = ref(0) // 0: 默认, 1: 下拉中, 2: 释放刷新, 3: 刷新中
    const pullDownProgress = ref(0) // 下拉进度 0-1
    const dataType = ref('default') // 数据类型: default | video | image

    // 处理触摸事件
    function handleTouchStart(e: TouchEvent) {
      if (window.scrollY <= 0) {
        startY.value = e.touches[0].pageY
        pullDownStatus.value = 1
      }
    }

    function handleTouchMove(e: TouchEvent) {
      if (pullDownStatus.value === 0) return
      
      const moveY = e.touches[0].pageY
      const diff = moveY - startY.value
      
      if (diff > 0 && window.scrollY <= 0) {
        e.preventDefault()
        const progress = Math.min(diff / 80, 1)
        pullDownStatus.value = progress >= 1 ? 2 : 1
        pullDownProgress.value = progress
      }
    }

    function handleTouchEnd() {
      if (pullDownStatus.value === 2) {
        pullDownStatus.value = 3
        refreshData().then(() => {
          pullDownStatus.value = 0
        })
      } else {
        pullDownStatus.value = 0
      }
    }

    // 解析URL参数
    function parseUrlParams() {
      const params = new URLSearchParams(window.location.search)
      const type = params.get('type')
      
      // 支持两种参数格式:
      // 1. ?type=video
      // 2. ?data=image
      if (type) {
        dataType.value = type
      } else {
        const data = params.get('data')
        if (data) {
          dataType.value = data
        }
      }
    }

    // Mock数据生成 - 混合视频和图片内容
    function generateMockData(count: number): DiscoverItem[] {
      const types = [
        {
          title: '233游戏视频',
          desc: '大家都在看的精彩视频',
          type: 'video'
        },
        {
          title: '233游戏精选图片',
          desc: '社区精选美图欣赏',
          type: 'image'
        }
      ]
      
      return Array.from({ length: count }, (_, i) => {
        const isVideo = Math.random() > 0.5
        const type = types[isVideo ? 0 : 1]
        
        const baseItem = {
          id: Date.now() + i,
          title: `${type.title} ${i + 1}`,
          desc: type.desc,
          cover: `https://picsum.photos/300/${Math.floor(Math.random() * 200) + 300}?random=${i}`,
          height: Math.floor(Math.random() * 200) + 300, // 随机高度300-500px
          author: `用户${Math.floor(Math.random() * 1000)}`,
          avatar: `https://picsum.photos/50/50?random=${i + 1000}`,
          likes: Math.floor(Math.random() * 1000),
          isLiked: false
        }
        
        if (type.type === 'video') {
          return {
            ...baseItem,
            video: `https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4?r=${i}`,
            poster: `https://picsum.photos/300/200?random=${i}`
          }
        }
        return baseItem
      })
    }

    // 加载数据
    async function loadData() {
  isLoading.value = true
  // 模拟API请求延迟
  await new Promise(resolve => setTimeout(resolve, 800))
  
  const newData = generateMockData(10)
  distributeItems(newData)
  page.value++
  isLoading.value = false
}

    // 刷新数据
    async function refreshData() {
  isRefreshing.value = true
  columns.value = [[], []]
  page.value = 1
  await loadData()
  isRefreshing.value = false
}

    // 分配项目到两列 - 交替分配实现错开效果
    function distributeItems(items: DiscoverItem[]) {
      let toggle = false
      items.forEach(item => {
        const targetCol = toggle ? 1 : 0
        columns.value[targetCol].push(item)
        toggle = !toggle
      })
    }

    // 滚动事件处理
    function handleScroll() {
  if (isLoading.value || !waterfall.value) return
  
  const { scrollTop, scrollHeight, clientHeight } = document.documentElement
  const threshold = 100
  
  if (scrollTop + clientHeight >= scrollHeight - threshold) {
    loadData()
  }
}

    onMounted(() => {
      parseUrlParams()
      loadData()
      window.addEventListener('scroll', handleScroll)
      window.addEventListener('touchstart', handleTouchStart)
      window.addEventListener('touchmove', handleTouchMove)
      window.addEventListener('touchend', handleTouchEnd)
    })

    onBeforeUnmount(() => {
      window.removeEventListener('scroll', handleScroll)
      window.removeEventListener('touchstart', handleTouchStart)
      window.removeEventListener('touchmove', handleTouchMove)
      window.removeEventListener('touchend', handleTouchEnd)
    })

    function handleVideoError(item: DiscoverItem) {
      console.error('视频加载失败:', item.video)
      // 替换为备用视频源
      item.video = 'https://sample-videos.com/video123/mp4/720/big_buck_bunny_720p_1mb.mp4'
      item.isPlaying = false
    }

    // 图片加载完成后计算item高度
    function toggleLike(item: DiscoverItem) {
      item.isLiked = !item.isLiked
      item.likes += item.isLiked ? 1 : -1
    }

    function handleImageLoad(e: Event, item: DiscoverItem) {
      const img = e.target as HTMLImageElement
      // 高度 = 图片高度 + 标题高度(约40px) + 描述高度(约40px) + 内边距(约24px) + 底部操作栏(约40px)
      item.height = img.naturalHeight + 40 + 40 + 24 + 40
    }

    return {
      columns,
      isLoading,
      isRefreshing,
      waterfall,
      pullDownStatus,
      dataType,
      handleVideoError,
      handleImageLoad,
      toggleLike
    }
  }
})
</script>

<style scoped>
.discover-container {
  max-width: 100vw;
  padding: 12px;
  padding-top: 0;
  transition: padding-top 0.3s;
}

.discover-container.pulling {
  padding-top: 60px;
}

.waterfall {
  display: flex;
  gap: 12px;
}

.column {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.item {
  background: #fff;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.item img {
  width: 100%;
  display: block;
}

.video-container {
  position: relative;
}

.video-container video {
  width: 100%;
  display: block;
}

.video-thumbnail {
  display: none;
}

.video-container video {
  position: relative;
  z-index: 1;
}

.item h3 {
  padding: 8px 12px;
  font-size: 16px;
  margin: 0;
}

.item-header {
  display: flex;
  align-items: center;
  padding: 8px 12px;
}

.avatar {
  width: 18px !important;
  height: 18px;
  border-radius: 50%;
  margin-right: 4px;
}

.author {
  font-size: 12px;
  color: #666;
}

.item p {
  padding: 0 12px 12px;
  font-size: 12px;
  color: #666;
  margin: 0;
}

.item-footer {
  padding: 8px 12px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.author-info {
  display: flex;
  align-items: center;
}

.like-btn {
  border: none;
  background: #f5f5f5;
  padding: 4px 12px;
  border-radius: 20px;
  font-size: 12px;
  color: #666;
  cursor: pointer;
  display: flex;
  align-items: center;
}

.like-btn:hover {
  background: #eee;
}

.like-btn.liked {
  color: #ff2442;
  background: #ffebee;
}

.like-btn::before {
  content: "♥";
  margin-right: 4px;
  font-size: 14px;
}

.refresh-tip,
.loading-tip {
  text-align: center;
  padding: 12px;
  color: #666;
}

.pull-down-tip {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  text-align: center;
  padding: 12px;
  background: #f5f5f5;
  transform: translateY(-100%);
  transition: transform 0.3s;
}

.pull-down-tip.show {
  transform: translateY(0);
}
</style>
