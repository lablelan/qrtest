<template>
  <div class="scanner-container">
    <!-- 扫描区域：视频流 + 扫描线动画 -->
    <div class="scan-area">
      <video 
        ref="video" 
        autoplay 
        playsinline 
        class="scan-video"
      ></video>
      <!-- 扫描线动画（视觉提示） -->
      <div class="scan-line"></div>
    </div>

    <!-- 控制按钮与结果展示 -->
    <div class="control-panel">
      <button 
        @click="startScan" 
        v-if="!isScanning"
        class="scan-btn start-btn"
      >
        开始扫描（支持一维/二维码）
      </button>
      <button 
        @click="stopScan" 
        v-if="isScanning"
        class="scan-btn stop-btn"
      >
        停止扫描
      </button>

      <!-- 扫描结果（成功） -->
      <p v-if="scanResult" class="result success">
        扫描成功：{{ scanResult }}
      </p>
      <!-- 扫描错误（权限/设备问题） -->
      <p v-if="errorMsg" class="result error">
        错误：{{ errorMsg }}
      </p>
    </div>
  </div>
</template>

<script setup>
import { ref, onUnmounted } from 'vue';
// 引入 ZXing 核心类（支持多格式条码解析）
import { BrowserMultiFormatReader } from '@zxing/library';

// 1. DOM 引用
const video = ref(null); // 视频容器（摄像头画面）

// 2. 状态管理
const isScanning = ref(false); // 是否正在扫描
const scanResult = ref(''); // 扫描结果
const errorMsg = ref(''); // 错误信息
let codeReader = null; // ZXing 解析器实例（全局保存，用于停止扫描）

// 3. 开始扫描逻辑
const startScan = async () => {
  try {
    // 重置状态
    scanResult.value = '';
    errorMsg.value = '';
    isScanning.value = true;

    // 初始化 ZXing 解析器（支持所有一维/二维码格式）
    codeReader = new BrowserMultiFormatReader();
    // 配置：优先使用后置摄像头（ideal: 'environment'）
    const constraints = {
      video: {
        facingMode: { ideal: 'environment' }, // 后置摄像头（手机/平板）
        width: { ideal: 1280 }, // 理想分辨率（平衡性能与清晰度）
        height: { ideal: 720 }
      }
    };

    // 4. 启动摄像头并开始解析
    // ZXing 会自动绑定视频流 + 实时解析，无需手动处理 canvas 帧
    const result = await codeReader.decodeFromConstraints(
      constraints,
      video.value, // 视频容器（展示摄像头画面）
      (result) => {
        // 实时解析回调（扫描到条码时触发）
        if (result) {
          scanResult.value = result.text; // 保存解析结果（条码内容）
          stopScan(); // 扫描成功后自动停止
        }
      }
    );

    // 兜底：若回调未触发，直接处理解析结果（兼容部分浏览器）
    if (result) {
      scanResult.value = result.text;
      stopScan();
    }

  } catch (err) {
    // 捕获错误（权限拒绝、无摄像头、设备不支持等）
    isScanning.value = false;
    errorMsg.value = getErrorMsg(err);
    console.error('扫描初始化失败：', err);
  }
};

// 5. 停止扫描逻辑
const stopScan = () => {
  if (!isScanning.value) return;

  // 停止 ZXing 解析器
  if (codeReader) {
    codeReader.stopContinuousDecode(); // 停止持续解析
    codeReader.reset(); // 重置解析器
    codeReader = null;
  }

  // 关闭摄像头流（释放设备资源）
  if (video.value?.srcObject) {
    video.value.srcObject.getTracks().forEach(track => track.stop());
    video.value.srcObject = null;
  }

  // 更新状态
  isScanning.value = false;
};

// 6. 错误信息格式化（提升用户体验）
const getErrorMsg = (err) => {
  const errMsg = err.message || '';
  if (errMsg.includes('permission denied')) {
    return '请授予摄像头权限（设置-隐私-摄像头）';
  } else if (errMsg.includes('no video device found')) {
    return '未检测到摄像头设备，请检查硬件';
  } else if (errMsg.includes('could not start video stream')) {
    return '摄像头启动失败，请重试或更换设备';
  } else {
    return '扫描出错，请稍后重试';
  }
};

// 7. 组件卸载时：强制停止扫描（避免内存泄漏）
onUnmounted(() => {
  stopScan();
});
</script>

<style scoped>
/* 容器样式：居中布局 */
.scanner-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  max-width: 500px;
  margin: 0 auto;
}

/* 扫描区域：固定比例 + 边框 */
.scan-area {
  position: relative;
  width: 100%;
  padding-top: 100%; /* 1:1 正方形比例（适配手机屏幕） */
  border: 2px solid #333;
  border-radius: 8px;
  overflow: hidden;
  background-color: #000;
}

/* 视频样式：填充扫描区域 */
.scan-video {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover; /* 保持画面比例，避免拉伸 */
}

/* 扫描线动画：从上到下滚动 */
.scan-line {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 4px;
  background-color: #409eff; /* 蓝色扫描线 */
  box-shadow: 0 0 10px 2px rgba(64, 158, 255, 0.8); /* 发光效果 */
  animation: scanMove 2s linear infinite; /* 动画：2秒循环 */
}

/* 扫描线动画关键帧 */
@keyframes scanMove {
  0% { top: 0; } /* 起始位置：顶部 */
  100% { top: 100%; } /* 结束位置：底部 */
}

/* 控制面板：按钮 + 结果区域 */
.control-panel {
  width: 100%;
  margin-top: 24px;
  text-align: center;
}

/* 按钮样式：区分开始/停止 */
.scan-btn {
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s;
}
.start-btn {
  background-color: #67c23a; /* 绿色：开始 */
  color: #fff;
}
.start-btn:hover {
  background-color: #52c41a;
}
.stop-btn {
  background-color: #f56c6c; /* 红色：停止 */
  color: #fff;
}
.stop-btn:hover {
  background-color: #ff4d4f;
}

/* 结果文本样式：区分成功/错误 */
.result {
  margin-top: 16px;
  font-size: 16px;
  line-height: 1.5;
}
.success {
  color: #67c23a; /* 成功：绿色 */
}
.error {
  color: #f56c6c; /* 错误：红色 */
}
</style>