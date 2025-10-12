<template>
  <div class="min-h-screen flex flex-col items-center justify-center p-6">
    <!-- 标题 -->
    <h1 class="text-2xl font-bold text-gray-800 mb-6">
      视频/图片上传与灰度处理
    </h1>

    <!-- 上传区 -->
    <div class="flex flex-col items-center gap-4 w-full max-w-lg p-6">
      <input
        type="file"
        accept="video/*,image/*"
        class="block w-full text-sm text-gray-700
               file:mr-4 file:py-2 file:px-4
               file:rounded-full file:border-0
               file:text-sm file:font-semibold
               file:bg-blue-50 file:text-blue-700
               hover:file:bg-blue-100"
        @change="handleUpload"
      >

      <!-- 视频预览 -->
      <video
        v-show="fileType === 'video'"
        id="video"
        ref="videoRef"
        class="w-full rounded-lg shadow"
        controls
      />

      <!-- 图片预览 -->
      <img
        v-show="fileType === 'image'"
        id="image"
        ref="imageRef"
        class="w-full rounded-lg shadow"
      >

      <!-- 串口状态 -->
      <div class="flex items-center gap-2 mt-2">
        <span>串口状态:</span>
        <span :class="isSerialConnected ? 'text-green-600 font-bold' : 'text-red-600 font-bold'">
          {{ isSerialConnected ? "已连接 ✅" : "未连接 ❌" }}
        </span>
      </div>

      <!-- 播放状态 -->
      <div class="flex items-center gap-2 mt-2">
        <span>播放状态:</span>
        <span :class="isPlaying ? 'text-green-600 font-bold' : 'text-red-600 font-bold'">
          {{ isPlaying ? "播放中 ▶" : "已停止 ⏹" }}
        </span>
      </div>

      <!-- 控制按钮 -->
      <div class="flex flex-col gap-6 mt-6 w-full max-w-lg">
        <!-- 串口控制 -->
        <div class="flex gap-4 justify-center">
          <button
            class="px-4 py-2 bg-indigo-500 text-white rounded-xl shadow hover:bg-indigo-600 transition disabled:opacity-50"
            :disabled="isSerialConnected"
            @click="connectSerial"
          >
            🔌 串口连接
          </button>
          <button
            class="px-4 py-2 bg-gray-400 text-white rounded-xl shadow hover:bg-gray-500 transition disabled:opacity-50"
            :disabled="!isSerialConnected"
            @click="disconnectSerial"
          >
            ❎ 断开串口
          </button>
        </div>

        <!-- 播放控制 -->
        <div class="flex gap-4 justify-center">
          <button
            class="px-4 py-2 bg-sky-500 text-white rounded-xl shadow hover:bg-sky-600 transition disabled:opacity-50"
            :disabled="!isSerialConnected || !fileType || isPlaying"
            @click="startProcessing"
          >
            ▶ 开始播放
          </button>
          <button
            class="px-4 py-2 bg-gray-500 text-white rounded-xl shadow hover:bg-gray-600 transition disabled:opacity-50"
            :disabled="!isPlaying"
            @click="stopProcessing"
          >
            ⏹ 停止播放
          </button>
        </div>
      </div>
    </div>

    <!-- Canvas 显示区 -->
    <div
      v-show="fileType"
      class="mt-6"
    >
      <canvas
        id="canvas"
        ref="canvasRef"
        class="border border-gray-300 rounded-lg shadow"
      />
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue"

const videoRef = ref(null)
const imageRef = ref(null)
const canvasRef = ref(null)

let port = null
let writer = null
let imageTimer = null

const isSerialConnected = ref(false)
const isPlaying = ref(false)
const fileType = ref(null) // "video" | "image"

// 上传文件
function handleUpload(event) {
    const file = event.target.files[0]
    if (!file) return

    const url = URL.createObjectURL(file)

    if (file.type.startsWith("video")) {
        fileType.value = "video"
        videoRef.value.src = url
        videoRef.value.load()
    } else if (file.type.startsWith("image")) {
        fileType.value = "image"
        imageRef.value.src = url
    }
}

// 串口连接
async function connectSerial() {
    try {
        port = await navigator.serial.requestPort()
        await port.open({
            baudRate: 115200,
            dataBits: 8,
            stopBits: 1,
            parity: "none",
        })
        writer = port.writable.getWriter()
        isSerialConnected.value = true
    } catch (err) {
        console.error("串口连接失败:", err)
        alert("串口连接失败:" + err)
        isSerialConnected.value = false
    }
}

// 断开串口
async function disconnectSerial() {
    try {
        if (writer) {
            writer.releaseLock()
            writer = null
        }
        if (port) {
            await port.close()
            port = null
        }
        isSerialConnected.value = false
    } catch (err) {
        console.error("串口断开失败:", err)
    }
}

// 灰度处理 + 发送
function processFrame(source, ctx, targetWidth = 200, targetHeight = 150) {
    ctx.drawImage(source, 0, 0, targetWidth, targetHeight)

    const imageData = ctx.getImageData(0, 0, targetWidth, targetHeight)
    const data = imageData.data

    // 计算压缩后数组长度，每个字节存 8 个像素
    const byteLength = Math.ceil(targetWidth * targetHeight / 8)
    const grayArray = new Uint8Array(1 + byteLength)
    grayArray[0] = 0xAA

    for (let i = 0, bitIndex = 0; i < data.length; i += 4, bitIndex++) {
        // 灰度化 + 二值化
        let gray = 0.299 * data[i] + 0.587 * data[i + 1] + 0.114 * data[i + 2]
        gray = gray > 128 ? 1 : 0 // 1 位表示

        // 更新显示用的 imageData
        const displayGray = gray ? 255 : 0
        data[i] = data[i + 1] = data[i + 2] = displayGray

        // 将 8 个像素打包到一个字节里
        const bytePos = Math.floor(bitIndex / 8)
        const bitPos = 7 - (bitIndex % 8) // 高位先存
        grayArray[1 + bytePos] |= gray << bitPos
    }

    ctx.putImageData(imageData, 0, 0)
    console.log(grayArray)

    if (writer) {
        writer.write(grayArray).catch(err => console.error("串口写入失败:", err))
    }
}

// 开始处理
function startProcessing() {
    const canvas = canvasRef.value
    const ctx = canvas.getContext("2d")
    const targetWidth = 200
    const targetHeight = 150
    canvas.width = targetWidth
    canvas.height = targetHeight

    isPlaying.value = true

    if (fileType.value === "video") {
        const video = videoRef.value
        video.play()

        function frameCallback() {
            if (!isPlaying.value) return
            processFrame(video, ctx, targetWidth, targetHeight)
            video.requestVideoFrameCallback(frameCallback)
        }
        video.requestVideoFrameCallback(frameCallback)
    }

    if (fileType.value === "image") {
        const img = imageRef.value
        if (imageTimer) clearInterval(imageTimer)
        imageTimer = setInterval(() => {
            if (!isPlaying.value) return
            processFrame(img, ctx, targetWidth, targetHeight)
        }, 1000 / 10) // 10 fps
    }
}

// 停止处理
function stopProcessing() {
    isPlaying.value = false
    if (fileType.value === "video") {
        videoRef.value.pause()
    }
    if (fileType.value === "image" && imageTimer) {
        clearInterval(imageTimer)
        imageTimer = null
    }
}
</script>
