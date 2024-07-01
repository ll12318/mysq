<script setup>
import * as faceapi from "@vladmandic/face-api";
import { onMounted, onUnmounted, reactive, ref, watch } from "vue";

/**属性状态 */
const state = reactive({
  /**初始化模型加载 */
  netsLoadModel: true,
  /**算法模型 */
  netsType: "ssdMobilenetv1",
  /**模型参数 */
  netsOptions: {
    ssdMobilenetv1: undefined,
    tinyFaceDetector: undefined,
  },
  /**检测人脸 多或单 */
  detectFace: "detectAllFaces",
  /**面框分值, 面部轮廓, 面部表情, 年龄性别 */
  draws: ["box", "landmark", "expression", "ageGender"],
  /**视频元素 */
  videoEl: null,
  /**画布图层元素 */
  canvasEl: null,
  /**绘制定时器 */
  timer: 0,
  /**视频媒体参数配置 */
  constraints: {
    audio: false,
    video: {
      /**ideal（应用最理想的） */
      width: {
        min: 320,
        ideal: 1980,
        max: 1980,
      },
      height: {
        min: 200,
        ideal: 1080,
        max: 1080,
      },
      /**frameRate 受限带宽传输时，低帧率可能更适宜 */
      frameRate: {
        min: 7,
        ideal: 15,
        max: 30,
      },
      /**facingMode 摄像头前后切换 */
      facingMode: "environment",
    },
  },
  /**视频流 */
  stream: null,
});

/**初始化模型加载 */
async function fnLoadModel() {
  try {
    const BASE_DIR =  await ipcRenderer.invoke('get-path')
    console.log(BASE_DIR, 'BASE_DIR')
    const modelsPath = `${BASE_DIR}/models`;
    // 模型文件访问路径
  // const modelsPath = `http://192.168.100.206/models`;
  // const modelsPath = `D:\\web\\web\\vue3_electron\\electron-vite-vue\\release\\28.1.0\\win-unpacked\\resources\\public\\models`;
  //   const  modelsPath = `/models`;
  // 面部轮廓模型
  await faceapi.nets.faceLandmark68Net.load(modelsPath);
  // 面部表情模型
  await faceapi.nets.faceExpressionNet.load(modelsPath);
  // 年龄性别模型
  await faceapi.nets.ageGenderNet.load(modelsPath);

  // 模型参数-ssdMobilenetv1
  await faceapi.nets.ssdMobilenetv1.load(modelsPath);
  state.netsOptions.ssdMobilenetv1 = new faceapi.SsdMobilenetv1Options({
    minConfidence: 0.5, // 0 ~ 1
    maxResults: 50, // 0 ~ 100
  });
  // 模型参数-tinyFaceDetector
  await faceapi.nets.tinyFaceDetector.load(modelsPath);
  state.netsOptions.tinyFaceDetector = new faceapi.TinyFaceDetectorOptions({
    inputSize: 416, // 160 224 320 416 512 608
    scoreThreshold: 0.5, // 0 ~ 1
  });

  // 输出库版本
  console.log(
    `FaceAPI Version: ${faceapi?.version || "(not loaded)"
    } \nTensorFlow/JS Version: ${faceapi.tf?.version_core || "(not loaded)"
    } \nBackend: ${faceapi.tf?.getBackend() || "(not loaded)"
    } \nModels loaded: ${faceapi.tf.engine().state.numTensors} tensors`
  );

  // 节点元素
  state.videoEl = document.getElementById("page_draw-video");
  state.canvasEl = document.getElementById("page_draw-video-canvas");

  // 关闭模型加载
  state.netsLoadModel = false;
  } catch (error) {
    alert(JSON.stringify(error))
  }
}
const faceCount = ref(0);
/**根据模型参数识别绘制 */
async function fnRedraw() {
  try {
  if (!state.videoEl || !state.canvasEl) return;
  console.log("Run Redraw");

  // 暂停视频时清除定时
  if (state.videoEl.paused) {
    clearTimeout(state.timer);
    state.timer = 0;
    return;
  }
  // 识别绘制人脸信息
  const detect = await faceapi[state.detectFace](
    state.videoEl,
    state.netsOptions[state.netsType]
  )
    // 需引入面部轮廓模型
    .withFaceLandmarks()
    // 需引入面部表情模型
    .withFaceExpressions()
    // 需引入年龄性别模型
    .withAgeAndGender();
  faceCount.value = detect.length;
  document.title = `Face Count: ${detect.length}`;
  // 无识别数据时，清除定时重新再次识别
  if (!detect) {
    clearTimeout(state.timer);
    state.timer = 0;
    fnRedraw();
    return;
  }

  // 匹配元素大小
  const dims = faceapi.matchDimensions(state.canvasEl, state.videoEl, true);
  const result = faceapi.resizeResults(detect, dims);

  // 面框分值
  if (state.draws.includes("box")) {
    faceapi.draw.drawDetections(state.canvasEl, result);
  }
  // 面部轮廓
  if (state.draws.includes("landmark")) {
    // 需引入面部轮廓模型
    faceapi.draw.drawFaceLandmarks(state.canvasEl, result);
  }
  // 面部表情
  if (state.draws.includes("expression")) {
    // 需引入面部表情模型
    faceapi.draw.drawFaceExpressions(state.canvasEl, result, 0.05);
  }
  // 年龄性别
  if (state.draws.includes("ageGender")) {
    // 需引入年龄性别模型模型
    const drawItem = (item) => {
      const { age, gender, genderProbability } = item;
      new faceapi.draw.DrawTextField(
        [
          `${Math.round(age)} Age`,
          `${gender} (${Math.round(genderProbability)})`,
        ],
        item.detection.box.topRight
      ).draw(state.canvasEl);
    };
    // 多结果
    if (Array.isArray(result)) {
      result.forEach((item) => drawItem(item));
    } else {
      drawItem(result);
    }
  }

  // 定时器句柄
  state.timer = setTimeout(() => fnRedraw(), 0);
  } catch (error) {
    console.log(error);
  }
}
const show = ref(false);
/**启动摄像头视频媒体 */
async function fnOpen() {
  if (state.stream !== null) return;
  try {
    state.stream = {}; // 置为空对象，避免重复点击
    const stream = await navigator.mediaDevices.getUserMedia(state.constraints);
    state.stream = stream;
    state.videoEl.srcObject = stream;
    state.videoEl.play();
    setTimeout(() => fnRedraw(), 1000);
    show.value = true;
  } catch (error) {
    state.stream = null; // 置为空，可点击
    console.error(error);
    alert("视频媒体流获取错误: " + error);
  }
}

/**结束摄像头视频媒体 */
function fnClose() {
  if (state.stream === null) return;
  state.videoEl.pause();
  state.videoEl.srcObject = null;
  state.stream.getTracks().forEach((track) => track.stop());
  state.stream = null;
  clearTimeout(state.timer);
  state.timer = 0;

  setTimeout(() => {
    // 清空画布
    state.canvasEl
      .getContext("2d")
      .clearRect(0, 0, state.canvasEl.width, state.canvasEl.height);
  }, 500);
}

// 摄像头前后切换 启用时，关闭后重开
watch(
  () => state.constraints.video.facingMode,
  () => {
    if (state.stream !== null) {
      fnClose();
      fnOpen();
    } else {
      fnClose();
    }
  }
);

onMounted(() => {
  fnLoadModel();
});
setTimeout(() => {
  fnOpen();
}, 1000);
 // 每隔5分钟刷新一次页面
 setInterval(() => {
  location.reload();
}, 300000);

onUnmounted(() => {
  fnClose();
});
</script>

<template>
  <span @click="fnOpen()"  class="hui" v-show="!show">
   
  </span>
  <div  v-show="show">
    <span v-if="faceCount > 1" class="hong"></span>
    <span v-else class="lu"></span>
  </div>
  <div class="page" >
    <div class="page_option">
      <div>
        <label>检测到人脸：{{ faceCount }}</label>
        <label>摄像头视频媒体：</label>
       
        &nbsp;
        <button @click="fnClose()">结束</button>
      </div>
      <div>
        <label>前置or后置切换：</label>
        <select v-model="state.constraints.video.facingMode">
          <option value="user">前置</option>
          <option value="environment">后置</option>
        </select>
      </div>
      <div>
        <label>面框分值：</label>
        <input type="checkbox" value="box" v-model="state.draws" />
      </div>
      <div>
        <label>面部轮廓：</label>
        <input type="checkbox" value="landmark" v-model="state.draws" />
      </div>
      <div>
        <label>面部表情：</label>
        <input type="checkbox" value="expression" v-model="state.draws" />
      </div>
      <div>
        <label>年龄性别：</label>
        <input type="checkbox" value="ageGender" v-model="state.draws" />
      </div>
      <div>
        <label>算法模型：</label>
        <select v-model="state.netsType">
          <option value="ssdMobilenetv1">SSD Mobilenet V1</option>
          <option value="tinyFaceDetector">Tiny Face Detector</option>
        </select>
      </div>
      <div>
        <label>检测人脸：</label>
        <select v-model="state.detectFace">
          <option value="detectSingleFace">Detect Single Face</option>
          <option value="detectAllFaces">Detect All Faces</option>
        </select>
      </div>
    </div>

    <div class="page_load" v-show="state.netsLoadModel">Load Model...</div>
    <div class="page_draw" v-show="!state.netsLoadModel">
      <div class="page_draw" v-show="!state.netsLoadModel">
        <video id="page_draw-video" poster="/images/720x480.png" src="/videos/test.mp4" muted playsinline></video>
        <canvas id="page_draw-video-canvas"></canvas>
      </div>
    </div>
  </div>
</template>

<style >
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
body {
  margin: 0
}
.page{
  display: none
}
.page_draw{
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
}
.page_draw video{
  width: 100%;
  height: 100%;
  object-fit: cover;
}
.page_draw canvas{
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}
.hong {
  display: inline-block;
  width: 20px;
  height: 20px;
  background-color: red;
  opacity: 0.9;
}
.lu {
  display: inline-block;
  width: 20px;
  height: 20px;
  background-color: green;
  opacity: 0.9;
}

.hui {
  display: inline-block;
  width: 20px;
  height: 20px;
  background-color: gray;
  opacity: 0.9;
}

</style>
