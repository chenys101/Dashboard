<template>
  <div
    ref="iframeWrapper"
    class="wrapper material-iframe"
    :style="{ borderRadius: element.borderRadius + 'px' }"
  >
    <input v-model.number="refreshDuration" class="small-input" />
    <template v-if="!needUseCache">
      <!-- 修改 iframe 的 src 为解密后的 blobUrl -->
      <iframe
        v-if="componentSetting.url && refreshFlag && blobUrl"
        ref="iframe"
        :src="blobUrl"
        :style="{
          width: '100%',
          height: '100%',
          pointerEvents: isLock ? 'all' : 'none'
        }"
        frameborder="0"
      />
      <Unset v-else-if="!blobUrl" :tips="`💫${$t('内容加载中...')}`" />
      <Unset v-else :tips="`💫${$t('IFrame路径丢失，请进行配置')}`" />
    </template>
  </div>
</template>

<script lang="ts" setup>
import { computed, nextTick, ref, watch, onBeforeUnmount, onMounted } from 'vue'
import Unset from '@/components/Tools/Unset.vue'
import { useStore } from '@/store'
import CryptoJS from 'crypto-js'

const props = defineProps({
  element: { type: Object, required: true },
  componentSetting: { type: Object, required: true }
})

const store = useStore()
const isLock = computed(() => store.isLock)
const isLowPreformance = computed(() => store.global.disabledDialogAnimation)

// 新增：存储解密后的 blob URL
const blobUrl = ref<string | null>(null)

// 解密函数（兼容文本和 ArrayBuffer）
const decryptContent = async (encryptedData: string, secret: string): Promise<string> => {
  try {
    const encryptedDataB64 = CryptoJS.enc.Base64.parse(encryptedData);
    const secretB64 = CryptoJS.enc.Utf8.parse(secret);
    const decryptedData = CryptoJS.AES.decrypt({ ciphertext: encryptedDataB64 }, secretB64, {
      mode: CryptoJS.mode.ECB,
      padding: CryptoJS.pad.Pkcs7
    });
    const decryptedDataStr = decryptedData.toString(CryptoJS.enc.Utf8);
    return decryptedDataStr
  } catch (error) {
    console.error('解密失败:', error)
    throw error
  }
}

// 获取并解密 URL 内容
const fetchAndDecrypt = async (url: string, secret?: string) => {
  try {
    const response = await fetch(url)
    if (!response.ok) throw new Error(`HTTP错误: ${response.status}`)
    
    const buffer = await response.arrayBuffer()
    let decryptedContent = new TextDecoder().decode(buffer)
    if (secret) {
      decryptedContent = await decryptContent(decryptedContent, secret)
    }

    // 创建 Blob 并生成 URL
    const blob = new Blob([decryptedContent], { type: response.headers.get('content-type') || 'text/plain' })
    const newBlobUrl = URL.createObjectURL(blob)
    
    // 释放旧的 blob URL 避免内存泄漏
    if (blobUrl.value) URL.revokeObjectURL(blobUrl.value)
      blobUrl.value = newBlobUrl
    } catch (error) {
      console.error('加载或解密失败:', error)
      blobUrl.value = null
    }
}

// 监听 URL 和密钥变化
watch(
  () => [props.componentSetting.url, props.componentSetting.aesSecret],
  async ([newUrl, newSecret]) => {
    if (!newUrl) return
    await fetchAndDecrypt(newUrl, newSecret)
  },
  { immediate: true }
)

// 以下部分与原代码保持一致（仅修改 refreshTimer 中刷新逻辑）
const refreshFlag = ref(false)
const refreshIframe = async () => {
  refreshFlag.value = false
  await nextTick()
  refreshFlag.value = true
  await fetchAndDecrypt(props.componentSetting.url, props.componentSetting.aesSecret)
}

// 清理 blob URL
onBeforeUnmount(() => {
  if (blobUrl.value) URL.revokeObjectURL(blobUrl.value)
})

let timer: number | null
// 新增 refreshDuration 的 ref 变量
const refreshDuration = ref<number>(props.componentSetting.duration || 0)

const refreshTimer = () => {
  refreshFlag.value = true
  // 优先使用 refreshDuration 的值
  const duration = refreshDuration.value ?? (props.componentSetting.duration || 0);
  console.warn('duration now value is ' + duration)
  const refreshInterval = duration * 60 * 1000
  if (timer) {
    window.clearInterval(timer)
    timer = null
  }
  if (!refreshInterval) return
  timer = window.setInterval(refreshIframe, refreshInterval)
}

watch(
  // 监听 refreshDuration 的变化
  () => refreshDuration.value,
  () => refreshTimer(),
  { immediate: true }
)

// 同时监听 props.componentSetting.duration 的变化
watch(
  () => props.componentSetting.duration,
  (newDuration) => {
    if (!refreshDuration.value) {
      refreshDuration.value = newDuration
    }
    refreshTimer()
  },
  { immediate: true }
)

const needUseCache = ref(false)
const iframe = ref()
const iframeWrapper = ref<HTMLElement>()

onMounted(() => {
  if (window.iframeCache && window.iframeCache.has(props.componentSetting.url)) {
    if(iframeWrapper.value) iframeWrapper.value.style.background = '#fff'
    needUseCache.value = true
    const cacheIframe = window.iframeCache.get(props.componentSetting.url)
    const dialogDelay = isLowPreformance.value ? 10 : 410
    setTimeout(() => {
      if (iframeWrapper.value) {
        const { top, left, width, height } = iframeWrapper.value.getBoundingClientRect()
        cacheIframe.style.cssText = `position: fixed;top:${top}px;left:${left}px;width:${width}px;height:${height}px;pointer-events: all;z-index: 10001;display: block;`
      }
    }, dialogDelay)
  }
})

onBeforeUnmount(() => {
  if (!window.iframeCache) {
    window.iframeCache = new Map()
  }
  if (!window.iframeCache.has(props.componentSetting.url) && props.componentSetting.useCache) {
    window.iframeCache.set(props.componentSetting.url, iframe.value)
    iframe.value.style.display = 'none'
    document.body.appendChild(iframe.value)
  }
  if (needUseCache.value) {
    if(iframeWrapper.value) iframeWrapper.value.style.background = '#fff'
    const cacheIframe = window.iframeCache.get(props.componentSetting.url)
    cacheIframe.style.display = 'none'
  }
})
</script>

<style lang="scss" scoped>
.wrapper {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  overflow: hidden;
}
.small-input {
  width: 20px; /* 调整宽度 */
  height: 15px; /* 调整高度 */
  font-size: 12px; /* 调整字体大小 */
  border: none; /* 设置无边框 */
  outline: none; /* 去除聚焦时的边框高亮效果 */
}
</style>
