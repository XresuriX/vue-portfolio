<template>
  <div class="hls-player">
    <video 
      ref="videoEl" 
      controls 
      playsinline
      class="video-js"
      :poster="posterUrl"
    ></video>
    <div v-if="error" class="hls-error">{{ error }}</div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'
import Hls from 'hls.js'

const props = defineProps({
  src: { type: String, required: true }, // e.g., 'http://localhost:8888/test/index.m3u8'
  autoplay: { type: Boolean, default: false },
  posterUrl: { type: String, default: '' }
})

const emit = defineEmits(['ready', 'error', 'ended'])

const videoEl = ref(null)
const hlsInstance = ref(null)
const error = ref(null)

const initPlayer = () => {
  if (!videoEl.value) return
  
  // Cleanup previous instance
  if (hlsInstance.value) {
    hlsInstance.value.destroy()
    hlsInstance.value = null
  }
  error.value = null

  const video = videoEl.value
  
  // 🌐 NATIVE HLS SUPPORT (Safari/iOS)
  if (video.canPlayType('application/vnd.apple.mpegurl')) {
    video.src = props.src
    attachListeners(video)
    return
  }

  // 🌐 HLS.JS FOR CHROME/FIREFOX/EDGE
  if (Hls.isSupported()) {
    hlsInstance.value = new Hls({
      debug: false,
      enableWorker: true,
      lowLatencyMode: true, // 🔑 CRITICAL FOR MEDIAMTX LOW-LATENCY HLS
      backBufferLength: 15,
      liveSyncDurationCount: 3,
      liveMaxLatencyDurationCount: 6
    })

    hlsInstance.value.loadSource(props.src)
    hlsInstance.value.attachMedia(video)
    
    hlsInstance.value.on(Hls.Events.MANIFEST_PARSED, () => {
      emit('ready')
      if (props.autoplay) video.play().catch(e => console.warn('Autoplay blocked:', e))
    })
    
    hlsInstance.value.on(Hls.Events.ERROR, (event, data) => {
      if (data.fatal) {
        switch (data.type) {
          case Hls.ErrorTypes.NETWORK_ERROR:
            error.value = 'Network error - reconnecting...'
            hlsInstance.value.startLoad()
            break
          case Hls.ErrorTypes.MEDIA_ERROR:
            error.value = 'Media error - recovering...'
            hlsInstance.value.recoverMediaError()
            break
          default:
            error.value = `Stream error: ${data.details}`
            emit('error', data)
            hlsInstance.value.destroy()
        }
      }
    })
  } else {
    error.value = 'HLS not supported in this browser'
    emit('error', new Error('Unsupported browser'))
  }
}

const attachListeners = (video) => {
  video.addEventListener('error', (e) => {
    error.value = `Playback error: ${e.message}`
    emit('error', e)
  })
  video.addEventListener('ended', () => emit('ended'))
}

// Lifecycle
onMounted(() => initPlayer())
watch(() => props.src, () => initPlayer())
onUnmounted(() => {
  if (hlsInstance.value) hlsInstance.value.destroy()
})
</script>

<style scoped>
.hls-player { width: 100%; max-width: 100%; margin: 1rem auto; }
.video-js { 
  width: 100%; 
  border-radius: 8px; 
  background: #000; 
  display: block; 
}
.hls-error { 
  color: #ff6b6b; 
  background: #2c1b1b; 
  padding: 0.75rem; 
  border-radius: 6px; 
  margin-top: 0.5rem; 
  font-family: system-ui; 
  font-size: 0.9rem; 
}
</style>