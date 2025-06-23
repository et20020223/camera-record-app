<template>
  <div class="signature-container">
    <div class="header">
      <h1>簽名錄影功能</h1>
      <p>點擊按鈕開啟簽名板並開始錄影</p>
    </div>

    <div class="main-content">
      <!-- 攝影機預覽區域 -->
      <div class="camera-section">
        <div class="video-container">
          <video
            ref="video"
            autoplay
            playsinline
            width="400"
            height="300"
            style="background: #111; border-radius: 8px; margin-bottom: 16px"
            :style="{ opacity: cameraStarted ? 1 : 0.5 }"
          ></video>
        </div>

        <div class="camera-controls">
          <div class="camera-status" v-if="cameraStarted">
            <span class="status-indicator"></span>
            鏡頭已啟動
          </div>

          <button
            @click="openSignatureDialog"
            :disabled="showDialog"
            class="btn success signature-btn"
          >
            開始簽名錄影
          </button>

          <!-- 切換鏡頭按鈕 -->
          <button
            @click="toggleCamera"
            :disabled="!cameraStarted || recording"
            class="btn secondary"
          >
            切換鏡頭
          </button>
        </div>
      </div>
    </div>

    <!-- 簽名對話框 -->
    <div v-if="showDialog" class="dialog-overlay" @click="handleOverlayClick">
      <div class="dialog-content" @click.stop>
        <div class="dialog-header">
          <h3>請在下方簽名</h3>
          <div class="recording-status" v-if="recording">
            <span class="recording-indicator"></span>
            錄影中...
          </div>
          <button @click="closeDialog" class="close-btn">&times;</button>
        </div>

        <div class="dialog-body">
          <!-- 全屏錄影預覽作為背景 -->
          <div class="fullscreen-recording-preview" v-if="recording">
            <video
              ref="recordingPreview"
              autoplay
              playsinline
              muted
              class="background-video"
            ></video>
          </div>

          <!-- 透明簽名板覆蓋在錄影預覽上 -->
          <div class="signature-overlay">
            <!-- 人臉檢測狀態顯示 -->
            <div class="face-detection-status" v-if="recording && faceDetectionEnabled">
              <div class="face-counter">
                <span class="face-icon">👤</span>
                <span>檢測到 {{ facesDetected }} 張人臉</span>
              </div>
            </div>

            <!-- 人臉檢測畫布 -->
            <canvas
              ref="faceDetectionCanvas"
              class="face-detection-canvas"
              v-if="recording && faceDetectionEnabled"
            ></canvas>

            <!-- 簽名板 -->
            <canvas ref="signatureCanvas" class="transparent-signature-canvas"></canvas>
          </div>

          <div class="dialog-controls">
            <button @click="clearSignature" :disabled="!hasSignature" class="btn secondary">
              清除簽名
            </button>

            <button @click="completeSignature" :disabled="!hasSignature" class="btn success">
              完成簽名
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- 結果展示區域 -->
    <div v-if="completedData" class="results-section">
      <h3>簽名結果</h3>

      <div class="results-grid">
        <!-- 簽名圖片 -->
        <div class="result-item">
          <h4>簽名</h4>
          <img :src="completedData.signatureDataUrl" alt="簽名" class="signature-image" />
          <a :href="completedData.signatureDataUrl" download="signature.png" class="download-btn">
            下載簽名
          </a>
        </div>

        <!-- 錄影結果 -->
        <div class="result-item">
          <h4>錄影</h4>
          <video :src="completedData.videoUrl" controls class="result-video"></video>
          <a
            :href="completedData.videoUrl"
            download="signature-recording.webm"
            class="download-btn"
          >
            下載錄影
          </a>
        </div>
      </div>

      <button @click="startNewSignature" class="btn primary new-signature-btn">開始新的簽名</button>
    </div>

    <!-- 錯誤訊息 -->
    <div v-if="errorMessage" class="error">
      <p>{{ errorMessage }}</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onUnmounted, nextTick } from 'vue'
import SignaturePad from 'signature_pad'
import { FaceDetection } from '@mediapipe/face_detection'
import { Camera } from '@mediapipe/camera_utils'

// DOM 引用
const video = ref<HTMLVideoElement | null>(null)
const signatureCanvas = ref<HTMLCanvasElement | null>(null)
const recordingPreview = ref<HTMLVideoElement | null>(null)
const faceDetectionCanvas = ref<HTMLCanvasElement | null>(null)

// 狀態管理
const cameraStarted = ref<boolean>(false)
const recording = ref<boolean>(false)
const hasSignature = ref<boolean>(false)
const errorMessage = ref<string>('')
const showDialog = ref<boolean>(false)
const faceDetectionEnabled = ref<boolean>(true)
const facesDetected = ref<number>(0)
const isUsingFrontCamera = ref<boolean>(true) // 追蹤目前使用的鏡頭

// 簽名和錄影實例
let signaturePad: SignaturePad | null = null
let mediaStream: MediaStream | null = null
let mediaRecorder: MediaRecorder | null = null
let recordedChunks: Blob[] = []

// 人臉檢測實例
let faceDetection: FaceDetection | null = null
let camera: Camera | null = null

// 完成後的資料
const completedData = ref<{
  signatureDataUrl: string
  videoUrl: string
} | null>(null)

// 啟動攝影機
const startCamera = async () => {
  if (!cameraStarted.value) {
    try {
      errorMessage.value = ''
      // 優先嘗試前置鏡頭
      try {
        mediaStream = await navigator.mediaDevices.getUserMedia({
          video: { facingMode: 'user' }, // 前置鏡頭
          audio: false,
        })
      } catch (frontCameraError) {
        console.warn('前置鏡頭不可用，嘗試使用預設鏡頭:', frontCameraError)
        // 如果前置鏡頭失敗，則使用任何可用的鏡頭
        mediaStream = await navigator.mediaDevices.getUserMedia({
          video: true,
          audio: false,
        })
      }

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

// 切換鏡頭
const toggleCamera = async () => {
  if (mediaStream && cameraStarted.value && !recording.value) {
    // 停止目前的串流
    mediaStream.getTracks().forEach((track) => track.stop())

    try {
      errorMessage.value = ''
      // 切換鏡頭方向
      isUsingFrontCamera.value = !isUsingFrontCamera.value

      mediaStream = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: isUsingFrontCamera.value ? 'user' : 'environment' },
        audio: true,
      })

      if (video.value) {
        video.value.srcObject = mediaStream
      }
    } catch (error) {
      console.error('切換鏡頭失敗:', error)
      errorMessage.value = '切換鏡頭失敗，請重試'
      // 切換失敗時恢復原狀態
      isUsingFrontCamera.value = !isUsingFrontCamera.value
    }
  }
}

// 初始化人臉檢測
const initFaceDetection = async () => {
  try {
    faceDetection = new FaceDetection({
      locateFile: (file) => {
        return `https://cdn.jsdelivr.net/npm/@mediapipe/face_detection/${file}`
      },
    })

    faceDetection.setOptions({
      model: 'short',
      minDetectionConfidence: 0.5,
    })

    faceDetection.onResults(onFaceDetectionResults)
  } catch (error) {
    console.error('人臉檢測初始化失敗:', error)
  }
}

// 定義人臉檢測相關類型
interface BoundingBox {
  xCenter: number
  yCenter: number
  width: number
  height: number
}

interface Detection {
  boundingBox: BoundingBox
}

interface FaceDetectionResults {
  detections: Detection[]
}

// 人臉檢測結果處理
const onFaceDetectionResults = (results: FaceDetectionResults) => {
  if (faceDetectionCanvas.value && results.detections) {
    const canvas = faceDetectionCanvas.value
    const ctx = canvas.getContext('2d')

    if (ctx) {
      // 清除之前的檢測框
      ctx.clearRect(0, 0, canvas.width, canvas.height)

      // 更新檢測到的人臉數量
      facesDetected.value = results.detections.length

      // 繪製人臉檢測框
      results.detections.forEach((detection: Detection) => {
        const bbox = detection.boundingBox
        if (bbox) {
          ctx.strokeStyle = '#00ff00'
          ctx.lineWidth = 2
          ctx.strokeRect(
            bbox.xCenter * canvas.width - (bbox.width * canvas.width) / 2,
            bbox.yCenter * canvas.height - (bbox.height * canvas.height) / 2,
            bbox.width * canvas.width,
            bbox.height * canvas.height,
          )
        }
      })
    }
  }
}

// 開啟簽名對話框並開始錄影
const openSignatureDialog = async () => {
  // 如果鏡頭還沒啟動，先啟動鏡頭
  if (!cameraStarted.value) {
    await startCamera()
  }

  // 如果鏡頭啟動失敗，不繼續執行
  if (!cameraStarted.value) {
    return
  }

  showDialog.value = true
  startRecording()

  // 等待 dialog 渲染完成後設置簽名板和錄影預覽
  await nextTick()
  setupSignaturePad()
  setupRecordingPreview()

  // 初始化人臉檢測
  if (faceDetectionEnabled.value) {
    await initFaceDetection()
    setupFaceDetection()
  }
}

// 設置人臉檢測
const setupFaceDetection = async () => {
  if (faceDetectionCanvas.value && recordingPreview.value && faceDetection) {
    const canvas = faceDetectionCanvas.value
    const video = recordingPreview.value

    // 設置檢測畫布尺寸
    canvas.width = video.videoWidth || 640
    canvas.height = video.videoHeight || 480

    // 創建攝像頭實例
    camera = new Camera(video, {
      onFrame: async () => {
        if (faceDetection) {
          await faceDetection.send({ image: video })
        }
      },
      width: canvas.width,
      height: canvas.height,
    })

    camera.start()
  }
}

// 關閉對話框
const closeDialog = () => {
  showDialog.value = false
  stopRecording()
  stopCamera()
  stopFaceDetection()
  hasSignature.value = false
  if (signaturePad) {
    signaturePad.clear()
  }
}

// 停止人臉檢測
const stopFaceDetection = () => {
  if (camera) {
    camera.stop()
    camera = null
  }
  if (faceDetection) {
    faceDetection.close()
    faceDetection = null
  }
  facesDetected.value = 0
}

// 點擊遮罩關閉對話框（可選）
const handleOverlayClick = () => {
  // 可以選擇是否允許點擊遮罩關閉
  // closeDialog()
}

// 設置簽名板
const setupSignaturePad = () => {
  if (signatureCanvas.value) {
    // 設置 canvas 大小 - 使用百分比來適應dialog大小
    const canvas = signatureCanvas.value
    const container = canvas.parentElement
    const dpr = window.devicePixelRatio || 1

    // 動態設置大小以適應容器
    const containerWidth = container?.clientWidth || 600
    const containerHeight = container?.clientHeight || 400
    const canvasWidth = containerWidth * 0.9
    const canvasHeight = containerHeight * 0.7

    canvas.width = canvasWidth * dpr
    canvas.height = canvasHeight * dpr
    canvas.style.width = canvasWidth + 'px'
    canvas.style.height = canvasHeight + 'px'

    const ctx = canvas.getContext('2d')
    if (ctx) {
      ctx.scale(dpr, dpr)
    }

    // 初始化透明簽名板
    signaturePad = new SignaturePad(canvas, {
      backgroundColor: 'rgba(0, 0, 0, 0)', // 完全透明背景
      penColor: 'rgb(0, 0, 0)', // 黑色筆跡
      minWidth: 3,
      maxWidth: 6,
    })

    // 監聽簽名更新事件
    signaturePad.addEventListener('endStroke', onSignatureUpdate)
  }
}

// 簽名更新時檢查是否有簽名
const onSignatureUpdate = () => {
  if (signaturePad) {
    hasSignature.value = !signaturePad.isEmpty()
  }
}

// 開始錄影
const startRecording = () => {
  if (mediaStream && !recording.value) {
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
        const videoUrl = URL.createObjectURL(blob)

        if (signaturePad && !signaturePad.isEmpty()) {
          completedData.value = {
            signatureDataUrl: signaturePad.toDataURL(),
            videoUrl: videoUrl,
          }
        }
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

// 停止錄影
const stopRecording = () => {
  if (mediaRecorder && recording.value) {
    mediaRecorder.stop()
    recording.value = false
  }
}

// 關閉鏡頭
const stopCamera = () => {
  if (mediaStream) {
    mediaStream.getTracks().forEach((track) => track.stop())
    mediaStream = null
  }
  if (video.value) {
    video.value.srcObject = null
  }
  cameraStarted.value = false
}

// 清除簽名
const clearSignature = () => {
  if (signaturePad) {
    signaturePad.clear()
    hasSignature.value = false
  }
}

// 完成簽名
const completeSignature = () => {
  if (hasSignature.value) {
    stopRecording()
    stopCamera()
    stopFaceDetection()
    showDialog.value = false
  }
}

// 開始新的簽名
const startNewSignature = () => {
  completedData.value = null
  hasSignature.value = false
  // 重置鏡頭狀態，讓用戶可以重新開始
  cameraStarted.value = false
}

// 設置錄影預覽
const setupRecordingPreview = () => {
  if (recordingPreview.value && mediaStream) {
    recordingPreview.value.srcObject = mediaStream
  }
}

// 組件卸載時清理資源
onUnmounted(() => {
  if (mediaStream) {
    mediaStream.getTracks().forEach((track) => track.stop())
  }
  if (mediaRecorder && recording.value) {
    mediaRecorder.stop()
  }
  stopFaceDetection()
})
</script>

<style scoped>
.signature-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.header {
  text-align: center;
  margin-bottom: 30px;
}

.header h1 {
  color: #333;
  margin-bottom: 10px;
}

.header p {
  color: #666;
  font-size: 16px;
}

.main-content {
  display: flex;
  justify-content: center;
  margin-bottom: 40px;
}

.camera-section {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.video-container {
  margin-bottom: 20px;
}

.camera-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
}

.signature-btn {
  margin-top: 10px;
}

.camera-status {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #28a745;
  font-weight: 500;
  margin-bottom: 10px;
}

.status-indicator {
  width: 12px;
  height: 12px;
  background-color: #28a745;
  border-radius: 50%;
}

.recording-status {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #dc3545;
  font-weight: 500;
}

.recording-indicator {
  width: 12px;
  height: 12px;
  background-color: #dc3545;
  border-radius: 50%;
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% {
    opacity: 1;
  }
  50% {
    opacity: 0.3;
  }
  100% {
    opacity: 1;
  }
}

/* Dialog 樣式 */
.dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.dialog-content {
  background: white;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  max-width: 700px;
  width: 90%;
  max-height: 80vh;
  overflow: auto;
}

.dialog-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 30px;
  border-bottom: 1px solid #eee;
}

.dialog-header h3 {
  margin: 0;
  color: #333;
}

.close-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #999;
  padding: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  transition: all 0.3s;
}

.close-btn:hover {
  background-color: #f5f5f5;
  color: #333;
}

.dialog-body {
  padding: 0;
  position: relative;
  height: 500px;
  overflow: hidden;
}

/* 全屏錄影預覽樣式 */
.fullscreen-recording-preview {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
}

.background-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 0 0 12px 12px;
}

/* 簽名覆蓋層 */
.signature-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 2;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

/* 人臉檢測狀態 */
.face-detection-status {
  position: absolute;
  top: 20px;
  left: 20px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 10px 15px;
  border-radius: 20px;
  font-size: 14px;
  z-index: 4;
}

.face-counter {
  display: flex;
  align-items: center;
  gap: 8px;
}

.face-icon {
  font-size: 16px;
}

/* 人臉檢測畫布 */
.face-detection-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 3;
}

/* 透明簽名板 */
.transparent-signature-canvas {
  width: 90%;
  height: 70%;
  cursor: crosshair;
  background: transparent;
  border: 2px dashed rgba(255, 255, 255, 0.8);
  border-radius: 8px;
}

.dialog-controls {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 15px;
  justify-content: center;
  z-index: 3;
}

/* 結果展示區域 */
.results-section {
  background: #f8f9fa;
  padding: 30px;
  border-radius: 12px;
  text-align: center;
}

.results-section h3 {
  color: #333;
  margin-bottom: 30px;
}

.results-grid {
  display: flex;
  flex-direction: column;
  gap: 40px;
  margin-bottom: 30px;
  max-width: 600px;
  margin-left: auto;
  margin-right: auto;
}

.result-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}

.result-item h4 {
  color: #333;
  margin-bottom: 15px;
}

.signature-image {
  width: 100%;
  max-width: 500px;
  height: auto;
  min-height: 250px;
  border: 1px solid #ddd;
  border-radius: 8px;
  margin-bottom: 15px;
  background: white;
  object-fit: contain;
}

.result-video {
  width: 100%;
  max-width: 500px;
  height: auto;
  border-radius: 8px;
  margin-bottom: 15px;
}

.new-signature-btn {
  margin-top: 20px;
}

/* 按鈕樣式 */
.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 500;
  transition: all 0.3s;
  min-width: 140px;
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

.secondary {
  background-color: #6c757d;
  color: white;
}

.secondary:hover:not(:disabled) {
  background-color: #545b62;
}

.success {
  background-color: #28a745;
  color: white;
}

.success:hover:not(:disabled) {
  background-color: #1e7e34;
}

.download-btn {
  display: inline-block;
  padding: 10px 20px;
  background-color: #17a2b8;
  color: white;
  text-decoration: none;
  border-radius: 6px;
  font-size: 14px;
  transition: background-color 0.3s;
}

.download-btn:hover {
  background-color: #138496;
}

.error {
  background-color: #f8d7da;
  border: 1px solid #f5c6cb;
  color: #721c24;
  padding: 15px;
  border-radius: 6px;
  margin-top: 20px;
  text-align: center;
}

/* 響應式設計 */
@media (max-width: 768px) {
  .dialog-content {
    width: 95%;
    margin: 20px;
  }

  .dialog-header {
    padding: 15px 20px;
  }

  .dialog-body {
    height: 400px;
    padding: 0;
  }

  .results-grid {
    gap: 30px;
    max-width: 100%;
  }

  video {
    width: 100% !important;
    height: auto !important;
  }

  .dialog-controls {
    bottom: 10px;
    align-items: center;
    gap: 10px;
  }

  .transparent-signature-canvas {
    width: 95%;
    height: 75%;
  }

  .signature-container {
    padding: 15px;
  }

  .signature-image {
    max-width: 100%;
    min-height: 200px;
  }

  .result-video {
    max-width: 100%;
  }
}
</style>
