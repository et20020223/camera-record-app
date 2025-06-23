<template>
  <div class="camera">
    <div style="padding: 32px; max-width: 500px; margin: auto">
      <h1>相機錄影功能</h1>
      <div class="video-container">
        <video
          ref="video"
          autoplay
          playsinline
          width="400"
          height="300"
          style="background: #111; border-radius: 8px; margin-bottom: 16px"
        ></video>
      </div>

      <div class="controls" style="margin: 16px 0; text-align: center">
        <button @click="startCamera" :disabled="cameraStarted" class="btn primary">開啟鏡頭</button>
        <button @click="startRecording" :disabled="!cameraStarted || recording" class="btn success">
          開始錄影
        </button>
        <button @click="stopRecording" :disabled="!recording" class="btn danger">停止錄影</button>
      </div>

      <div v-if="videoUrl" class="result" style="margin-top: 24px">
        <h3>錄影結果</h3>
        <video
          :src="videoUrl"
          controls
          width="400"
          height="300"
          style="border-radius: 8px; margin-bottom: 12px"
        ></video>
        <div>
          <a :href="videoUrl" download="recorded-video.webm" class="download-btn"> 下載影片 </a>
        </div>
      </div>

      <div v-if="errorMessage" class="error" style="margin-top: 16px">
        <p>{{ errorMessage }}</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const video = ref<HTMLVideoElement | null>(null)
const cameraStarted = ref<boolean>(false)
const recording = ref<boolean>(false)
const videoUrl = ref<string>('')
const errorMessage = ref<string>('')

let mediaStream: MediaStream | null = null
let mediaRecorder: MediaRecorder | null = null
let recordedChunks: Blob[] = []

const startCamera = async () => {
  if (!cameraStarted.value) {
    try {
      errorMessage.value = ''
      mediaStream = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: true,
      })
      if (video.value) {
        video.value.srcObject = mediaStream
      }
      cameraStarted.value = true
    } catch (error) {
      console.error('無法訪問攝像頭:', error)
      errorMessage.value = '無法訪問攝像頭，請確保已授權攝像頭權限'
    }
  }
}

const startRecording = () => {
  if (mediaStream) {
    try {
      recordedChunks = []
      mediaRecorder = new MediaRecorder(mediaStream)

      mediaRecorder.ondataavailable = (event) => {
        if (event.data.size > 0) {
          recordedChunks.push(event.data)
        }
      }

      mediaRecorder.onstop = () => {
        const blob = new Blob(recordedChunks, { type: 'video/webm' })
        videoUrl.value = URL.createObjectURL(blob)
      }

      mediaRecorder.start()
      recording.value = true
      errorMessage.value = ''
    } catch (error) {
      console.error('錄影啟動失敗:', error)
      errorMessage.value = '錄影啟動失敗，請重試'
    }
  }
}

const stopRecording = () => {
  if (mediaRecorder && recording.value) {
    mediaRecorder.stop()
    recording.value = false
  }
}
</script>

<style scoped>
.camera {
  text-align: center;
}

.video-container {
  display: flex;
  justify-content: center;
}

.controls {
  display: flex;
  gap: 12px;
  justify-content: center;
  flex-wrap: wrap;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 500;
  transition: all 0.3s;
  min-width: 120px;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.primary {
  background-color: #007bff;
  color: white;
}

.primary:hover:not(:disabled) {
  background-color: #0056b3;
}

.success {
  background-color: #28a745;
  color: white;
}

.success:hover:not(:disabled) {
  background-color: #1e7e34;
}

.danger {
  background-color: #dc3545;
  color: white;
}

.danger:hover:not(:disabled) {
  background-color: #c82333;
}

.download-btn {
  display: inline-block;
  padding: 10px 20px;
  background-color: #17a2b8;
  color: white;
  text-decoration: none;
  border-radius: 6px;
  font-size: 16px;
  transition: background-color 0.3s;
}

.download-btn:hover {
  background-color: #138496;
}

.error {
  background-color: #f8d7da;
  border: 1px solid #f5c6cb;
  color: #721c24;
  padding: 12px;
  border-radius: 6px;
}

.result {
  text-align: center;
}

@media (max-width: 768px) {
  .camera {
    padding: 16px;
  }

  video {
    width: 100% !important;
    max-width: 350px;
    height: auto !important;
  }

  .controls {
    flex-direction: column;
    align-items: center;
  }
}
</style>
