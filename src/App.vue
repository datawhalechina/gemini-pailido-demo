<script setup>
import { ref, onMounted } from 'vue'

const API_KEY = "在此填入你自己的Gemini KEY"
const CAMERA_STATUS_ENUM = {
  DEFAULT: 1,
  LOADING: 2,
  OUTPUT: 3,
}

const styleSelectIndex = ref(0)
const styleList = ref([
  { id: 1, name: "小红书风格", prompt: "你是一位小红书博主，专注于为用户推荐种草有价值的物品。根据提供的图片信息，使用小红书风格创作一篇种草文案，只需要输出纯文字即可，不需要Markdown格式：", },
  { id: 2, name: "好评大师", prompt: "你是一名大众点评专家，专注于从消费者反馈中提取有价值的信息并提供详细的优缺点分析。根据提供的图片信息，写一篇好评，只需要输出纯文字即可，不需要Markdown格式：", },
  { id: 3, name: "一句话文案", prompt: "根据图中的信息，用非常精炼的一句话进行描述，需要有诗意，有洞察力，下面是图片信息，只需要输出纯文字即可，不需要Markdown格式：", },
])
const videoElement = ref()
const canvasElement = ref()
const videoStream = ref()
const cameraStatus = ref(CAMERA_STATUS_ENUM.DEFAULT)
const geminiText = ref("")

const onStyleClick = (index) => {
  styleSelectIndex.value = index
}
const drawVideoToCanvas = () => {
  const canvas = canvasElement.value;
  canvas.width = videoElement.value.clientWidth;
  canvas.height = videoElement.value.clientHeight;

  // 绘制内容
  const ctx = canvas.getContext("2d");
  const videoRatio = videoElement.value.videoWidth / videoElement.value.videoHeight;
  const canvasRatio = canvas.width / canvas.height;
  let drawWidth, drawHeight, startX, startY;
  if (videoRatio > canvasRatio) {
    // 视频宽高比大于画布宽高比，绘制高度等等于视频高度，绘制宽度等于视频高度除以画布宽高比
    drawHeight = videoElement.value.videoHeight;
    drawWidth = videoElement.value.videoHeight * canvasRatio;
    // 计算绘制起始点坐标
    startX = (videoElement.value.videoWidth - drawWidth) / 2;
    startY = 0;
  } else {
    // 视频宽高比小于等于画布宽高比，绘制宽度等于视频宽度，绘制高度等于视频宽度除以画布宽高比
    drawWidth = videoElement.value.videoWidth;
    drawHeight = videoElement.value.videoWidth / canvasRatio;
    // 计算绘制起始点坐标
    startX = 0;
    startY = (videoElement.value.videoHeight - drawHeight) / 2;
  }

  // 使用canvas的drawImage方法将视频绘制到画布上
  ctx.drawImage(
    videoElement.value,
    startX,
    startY,
    drawWidth,
    drawHeight,
    0,
    0,
    canvas.width,
    canvas.height
  );

  // 从dataURL中移除数据URL的前缀部分，以获取纯Base64编码的图像数据
  const dataURL = canvas.toDataURL("image/png");
  let base64Image = dataURL.replace(
    /^data:image\/(png|jpg);base64,/,
    ""
  );
  return base64Image
}
const requestGemini = (base64Image) => {
  // 请求Gemini API
  fetch(`https://generativelanguage.googleapis.com/v1/models/gemini-1.5-flash:generateContent?key=${API_KEY}`, {
    method: "POST",
    body: JSON.stringify({
      "contents": [
        {
          "role": "user",
          "parts": [
            { "text": styleList.value[styleSelectIndex.value].prompt },
            {
              "inlineData": {
                "mimeType": "image/png",
                "data": base64Image
              }
            }
          ]
        }
      ]
    }),
  }).then(res => res.json()).then(res => {
    cameraStatus.value = CAMERA_STATUS_ENUM.OUTPUT
    geminiText.value = res.candidates[0].content.parts[0].text
  })
}
const onShutterClick = () => {
  cameraStatus.value = CAMERA_STATUS_ENUM.LOADING
  const base64Image = drawVideoToCanvas()
  requestGemini(base64Image)
}
const onOutputTextClick = () => {
  navigator.clipboard.writeText(geminiText.value)
  alert("文字已复制到剪贴板")
}
const onCloseClick = () => {
  geminiText.value = ""
  cameraStatus.value = CAMERA_STATUS_ENUM.DEFAULT
}

onMounted(() => {
  // 停止之前的流
  if (videoStream.value) {
    videoStream.value.getTracks().forEach((track) => track.stop());
  }
  // 获取用户媒体设备
  navigator.mediaDevices
    .getUserMedia({ video: { facingMode: "environment" } })
    .then((mediaStream) => {
      // 将video元素的流设置成摄像机的视频流
      videoStream.value = mediaStream;
      videoElement.value.srcObject = mediaStream;
      // 视频流加载完成后播放
      videoElement.value.onloadedmetadata = () => {
        videoElement.value.play();
      };
    })
    .catch(function (err) {
      console.log(err.name + ": " + err.message);
    });
})
</script>

<template>
  <div class="style-select">
    <div v-for="(style, index) in styleList" class="style-item"
      :class="{ 'style-item-select': styleSelectIndex === index }" @click="onStyleClick(index)">
      {{ style.name }}
    </div>
  </div>
  <div class="video-wrapper">
    <video ref="videoElement" class="video" autoplay playsinline></video>
    <canvas v-show="cameraStatus !== CAMERA_STATUS_ENUM.DEFAULT" ref="canvasElement" class="video-canvas"></canvas>
    <Transition>
      <div class="gemini-loading" v-if="cameraStatus === CAMERA_STATUS_ENUM.LOADING">让我想想……</div>
    </Transition>
    <Transition>
      <div class="gemini-output" v-if="cameraStatus === CAMERA_STATUS_ENUM.OUTPUT">
        <p class="gemini-text" @click="onOutputTextClick">{{ geminiText }}</p>
        <div class="close-button" @click="onCloseClick">关闭</div>
      </div>
    </Transition>
  </div>
  <div class="shutter-wrapper">
    <img class="shutter" src="./assets/shutter.svg" @click="onShutterClick" />
  </div>
</template>

<style scoped>
.style-select {
  width: 100vw;
  height: 8vh;
  display: flex;
  flex-direction: row;
  justify-content: space-evenly;
  align-items: center;
}

.style-item {
  color: #fff;
  font-size: 16px;
  width: 120px;
  height: 48px;
  line-height: 48px;
  text-align: center;
}

.style-item-select {
  color: #F59E0B;
  font-weight: bold;
}

.video-wrapper {
  margin: 4vw;
  border-radius: 2vw;
  width: 92vw;
  height: 60vh;
  overflow: hidden;
  position: relative;
}

.video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.video-canvas {
  width: 100%;
  height: 100%;
  position: absolute;
  left: 0;
  top: 0;
  z-index: 1;
}

.gemini-loading {
  position: absolute;
  width: 120px;
  height: 32px;
  line-height: 32px;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  margin: auto;
  background: rgba(0, 0, 0, 0.8);
  color: #fff;
  font-size: 14px;
  text-align: center;
  z-index: 2;
}

.gemini-output {
  position: absolute;
  width: 80vw;
  height: 50vh;
  left: 6vw;
  top: 5vh;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;
  gap: 16px;
  z-index: 3;
}

.gemini-text {
  padding: 6px;
  width: 80vw;
  height: 45vh;
  border-radius: 4px;
  background: rgba(0, 0, 0, 0.8);
  color: #fff;
  font-size: 14px;
  overflow: auto;
  white-space: break-spaces;
}

.close-button {
  width: 56px;
  height: 32px;
  line-height: 32px;
  border-radius: 4px;
  background: rgba(0, 0, 0, 0.8);
  color: #fff;
  font-size: 16px;
  text-align: center;
}

.shutter-wrapper {
  width: 100vw;
  height: 10vh;
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
}

.shutter {
  width: 20vw;
  height: 20vw;
}

.v-enter-active,
.v-leave-active {
  transition: opacity 0.5s ease;
}

.v-enter-from,
.v-leave-to {
  opacity: 0;
}
</style>
