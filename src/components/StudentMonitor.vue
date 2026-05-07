<template>
  <div class="monitor-container">
    <div class="main-layout">
      <!-- Left Panel: Video Feed -->
      <div class="video-panel">
        <div class="video-card">
          <div class="video-wrapper">
            <video ref="videoEl" autoplay muted playsinline class="video-feed"></video>
            <canvas ref="canvasEl" class="video-overlay"></canvas>
            
            <!-- Status Overlay -->
            <div v-if="isMonitoring && currentStatus" class="status-overlay">
              <div class="status-item">
                <span class="label">状态</span>
                <a-tag :color="getStatusColor(currentStatus.status)" class="status-tag">{{ currentStatus.status }}</a-tag>
              </div>
              <div class="status-item">
                <span class="label">表情</span>
                <span class="value">{{ currentStatus.expression }}</span>
              </div>
              <div class="status-item">
                <span class="label">姿态</span>
                <span class="value">{{ currentStatus.posture }}</span>
              </div>
            </div>

            <div v-if="!cameraActive" class="video-placeholder">
              <div class="placeholder-content">
                <span class="icon">📷</span>
                <p>点击“开始监控”启动摄像头</p>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Right Panel: Controls & Timeline -->
      <div class="sidebar-panel">
        <div class="panel-section control-card glass-effect">
          <h3 class="panel-title">控制面板</h3>
          
          <a-alert v-if="loadingMessage" :message="loadingMessage" type="info" show-icon class="status-alert" />
          
          <div class="action-buttons">
            <a-button 
              type="primary" 
              size="large" 
              block 
              class="action-btn start-btn"
              @click="startMonitoring" 
              :disabled="isMonitoring || !!loadingMessage"
            >
              <template #icon><span>▶</span></template>
              开始监控
            </a-button>
            
            <a-button 
              danger 
              size="large" 
              block 
              class="action-btn stop-btn"
              @click="stopMonitoring" 
              :disabled="!isMonitoring"
            >
              <template #icon><span>⏹</span></template>
              停止监控
            </a-button>
          </div>

          <div class="settings-group">
            <span class="setting-label">记录间隔 (秒)</span>
            <a-input-number 
              v-model:value="intervalSeconds" 
              :min="1" 
              :max="60" 
              class="interval-input"
              :disabled="isMonitoring" 
            />
          </div>
        </div>

        <div class="panel-section timeline-card glass-effect">
          <div class="panel-header">
            <h3 class="panel-title">历史记录</h3>
            <a-tag v-if="history.length" color="blue">{{ history.length }} 条</a-tag>
          </div>
          
          <div class="timeline-scroll-area">
            <a-empty v-if="history.length === 0" description="暂无记录" :image="simpleImage" />
            <div v-else class="history-list">
              <div v-for="(item, index) in history" :key="index" class="history-item">
                <div class="time-badge">{{ formatTime(item.timestamp) }}</div>
                <div class="history-content">
                  <div class="main-status">
                    <a-badge :status="getStatusBadge(item.status)" />
                    <span class="status-text" :class="getStatusClass(item.status)">{{ item.status }}</span>
                  </div>
                  <div class="sub-status">
                    {{ item.expression }} · {{ item.posture }}
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue';
import * as faceapi from 'face-api.js';
import dayjs from 'dayjs';
import { message, Empty } from 'ant-design-vue';

const simpleImage = Empty.PRESENTED_IMAGE_SIMPLE;

const videoEl = ref(null);
const canvasEl = ref(null);
const loadingMessage = ref('正在加载 AI 模型...');
const cameraActive = ref(false);
const isMonitoring = ref(false);
const intervalSeconds = ref(3);
const history = ref([]);
const currentStatus = ref(null);

let detectionInterval = null;
let recordInterval = null;

const MODEL_URL = '/models';

onMounted(async () => {
  try {
    await Promise.all([
      faceapi.nets.tinyFaceDetector.loadFromUri(MODEL_URL),
      faceapi.nets.faceExpressionNet.loadFromUri(MODEL_URL),
    ]);
    loadingMessage.value = '';
  } catch (error) {
    console.error('Error loading models:', error);
    loadingMessage.value = 'AI 模型加载失败，请刷新页面重试';
  }
});

onUnmounted(() => {
  stopMonitoring();
});

const startMonitoring = async () => {
  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    message.error('当前浏览器不支持摄像头访问');
    return;
  }

  try {
    loadingMessage.value = '正在启动摄像头...';
    const stream = await navigator.mediaDevices.getUserMedia({ video: {} });
    videoEl.value.srcObject = stream;
    
    videoEl.value.onloadedmetadata = () => {
      videoEl.value.play();
      cameraActive.value = true;
      loadingMessage.value = '';
      isMonitoring.value = true;
      startDetectionLoop();
    };
  } catch (error) {
    console.error('Camera error:', error);
    message.error('无法访问摄像头: ' + error.message);
    loadingMessage.value = '';
  }
};

const stopMonitoring = () => {
  isMonitoring.value = false;
  cameraActive.value = false;
  
  if (videoEl.value && videoEl.value.srcObject) {
    videoEl.value.srcObject.getTracks().forEach(track => track.stop());
    videoEl.value.srcObject = null;
  }

  if (detectionInterval) clearInterval(detectionInterval);
  if (recordInterval) clearInterval(recordInterval);
  
  if (canvasEl.value) {
    const ctx = canvasEl.value.getContext('2d');
    ctx.clearRect(0, 0, canvasEl.value.width, canvasEl.value.height);
  }
};

const startDetectionLoop = () => {
  const canvas = canvasEl.value;
  const video = videoEl.value;
  
  const displaySize = { width: video.videoWidth, height: video.videoHeight };
  faceapi.matchDimensions(canvas, displaySize);

  detectionInterval = setInterval(async () => {
    if (!video || video.paused || video.ended) return;

    const detections = await faceapi.detectAllFaces(video, new faceapi.TinyFaceDetectorOptions())
      .withFaceExpressions();

    const resizedDetections = faceapi.resizeResults(detections, displaySize);
    
    const ctx = canvas.getContext('2d');
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    faceapi.draw.drawDetections(canvas, resizedDetections);
    
    // Custom drawing for cleaner look? Keeping default for now but could be customized
    // faceapi.draw.drawFaceExpressions(canvas, resizedDetections); 

    if (detections.length > 0) {
      const detection = detections[0];
      const status = analyzeStatus(detection);
      currentStatus.value = status;
    } else {
      currentStatus.value = { status: '离开', expression: '无', posture: '未知' };
    }
  }, 100);

  recordInterval = setInterval(async () => {
    if (currentStatus.value) {
      history.value.unshift({
        timestamp: new Date(),
        ...currentStatus.value
      });
      if (history.value.length > 100) history.value.pop();
    }
  }, intervalSeconds.value * 1000);
};

const analyzeStatus = (detection) => {
  const { expressions } = detection;
  const box = detection.detection.box;
  
  const sorted = Object.entries(expressions).sort((a, b) => b[1] - a[1]);
  const dominant = sorted[0][0];
  
  let status = '专注';
  
  if (dominant === 'happy') status = '开心';
  else if (dominant === 'sad' || dominant === 'fearful' || dominant === 'angry') status = '困惑/焦虑';
  else if (dominant === 'surprised') status = '惊讶';
  else if (dominant === 'neutral') status = '专注';
  else status = '分心';

  const expressionMap = {
    neutral: '平静',
    happy: '开心',
    sad: '悲伤',
    angry: '生气',
    fearful: '害怕',
    disgusted: '厌恶',
    surprised: '惊讶'
  };

  const videoWidth = videoEl.value.videoWidth;
  const centerX = box.x + box.width / 2;
  const relativeX = centerX / videoWidth;
  
  let posture = '端正';
  if (relativeX < 0.3 || relativeX > 0.7) {
    posture = '侧倾';
    if (status === '专注') status = '分心'; 
  }
  
  return {
    status,
    expression: expressionMap[dominant] || dominant,
    posture
  };
};

const getStatusColor = (status) => {
  switch (status) {
    case '开心': return 'success';
    case '专注': return 'processing';
    case '困惑/焦虑': return 'warning';
    case '惊讶': return 'cyan';
    case '分心': return 'error';
    case '离开': return 'default';
    default: return 'default';
  }
};

const getStatusBadge = (status) => {
  switch (status) {
    case '开心': return 'success';
    case '专注': return 'processing';
    case '困惑/焦虑': return 'warning';
    case '惊讶': return 'default';
    case '分心': return 'error';
    default: return 'default';
  }
};

const getStatusClass = (status) => {
  switch (status) {
    case '开心': return 'text-success';
    case '专注': return 'text-primary';
    case '困惑/焦虑': return 'text-warning';
    case '分心': return 'text-danger';
    default: return 'text-default';
  }
};

const formatTime = (date) => {
  return dayjs(date).format('HH:mm:ss');
};
</script>

<style scoped>
/* Main Container */
.monitor-container {
  height: 100%;
  padding: 0; /* Removed padding to fill screen */
  background-color: #f0f2f5;
  box-sizing: border-box;
}

.monitor-container * {
  box-sizing: border-box; /* Ensure all children use border-box */
}

.main-layout {
  display: flex;
  height: 100%;
  gap: 0; /* Removed gap for full integration */
  width: 100%; /* Full width */
  margin: 0;
}

/* Video Panel (Left) */
.video-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0; /* Prevent overflow */
  overflow: hidden; /* Ensure video doesn't break layout */
  padding: 0; /* No padding for full screen fit */
}

/* Right Sidebar Panel */
.sidebar-panel {
  width: 350px;
  display: flex;
  flex-direction: column;
  gap: 0;
  flex-shrink: 0;
  padding: 0; /* No padding for full screen fit */
  background: #f0f2f5;
  border-left: 1px solid rgba(0,0,0,0.05); /* Separator */
}

.video-card {
  flex: 1;
  background: #000;
  border-radius: 0; /* No corners */
  overflow: hidden;
  box-shadow: none;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
}

.video-wrapper {
  width: 100%;
  height: 100%;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
}

.video-feed {
  width: 100%;
  height: 100%;
  object-fit: contain; /* Ensure video isn't stretched */
  max-height: 100%;
}

.video-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.video-placeholder {
  position: absolute;
  color: rgba(255, 255, 255, 0.7);
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.placeholder-content .icon {
  font-size: 4rem;
  margin-bottom: 1rem;
  display: block;
}

/* Status Overlay on Video */
.status-overlay {
  position: absolute;
  top: 20px;
  left: 20px;
  padding: 16px;
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  z-index: 10;
  color: #fff;
  background: rgba(0, 0, 0, 0.65); /* Dark background for better contrast */
  backdrop-filter: blur(8px);
  border: 1px solid rgba(255, 255, 255, 0.15);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  min-width: 120px;
}

.status-item {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.status-item .label {
  font-size: 0.75rem;
  opacity: 0.7;
  margin-bottom: 4px;
  color: #ccc;
}

.status-item .value {
  font-weight: 600;
  font-size: 1.1rem;
}

/* Right Sidebar Panel */
.sidebar-panel {
  width: 350px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  flex-shrink: 0;
  padding: 8px 8px 8px 0;
}

.glass-effect {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.3);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}

.panel-section {
  border-radius: 0;
  padding: 20px;
  display: flex;
  flex-direction: column;
}

.panel-title {
  margin: 0 0 16px 0;
  font-size: 1.1rem;
  font-weight: 600;
  color: #1f2937;
}

.control-card {
  flex-shrink: 0;
  border-bottom: 1px solid rgba(0,0,0,0.05);
}

.status-alert {
  margin-bottom: 16px;
}

.action-buttons {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 20px;
}

.action-btn {
  height: 48px;
  font-size: 1rem;
  font-weight: 500;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.settings-group {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding-top: 16px;
  border-top: 1px solid #eee;
}

.interval-input {
  width: 100px;
}

/* Timeline */
.timeline-card {
  flex: 1;
  min-height: 0; /* Important for scrolling */
  overflow: hidden;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.timeline-scroll-area {
  flex: 1;
  overflow-y: auto;
  padding-right: 4px;
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.history-item {
  display: flex;
  gap: 12px;
  padding: 12px;
  background: #f9fafb;
  border-radius: 8px;
  border-left: 3px solid transparent;
  transition: all 0.2s;
}

.history-item:hover {
  background: #fff;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}

.time-badge {
  font-family: monospace;
  font-weight: 600;
  color: #6b7280;
  font-size: 0.9rem;
  padding-top: 2px;
}

.history-content {
  flex: 1;
}

.main-status {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 4px;
}

.status-text {
  font-weight: 600;
}

.sub-status {
  font-size: 0.85rem;
  color: #9ca3af;
}

/* Status Colors */
.text-success { color: #52c41a; border-left-color: #52c41a; }
.text-primary { color: #1890ff; border-left-color: #1890ff; }
.text-warning { color: #faad14; border-left-color: #faad14; }
.text-danger { color: #f5222d; border-left-color: #f5222d; }
.text-default { color: #d9d9d9; border-left-color: #d9d9d9; }

/* Mobile Adaptation */
@media (max-width: 768px) {
  .monitor-container {
    padding: 0; /* Full screen on mobile */
    height: 100%;
    display: flex;
    flex-direction: column;
  }
  
  .main-layout {
    flex-direction: column;
    height: 100%;
    gap: 0; /* Remove gap for tighter fit */
  }
  
  .video-panel {
    height: 40vh; /* Fixed height for video */
    flex: none;
    border-radius: 0; /* Remove corners on mobile */
  }
  
  .video-card {
    border-radius: 0;
    box-shadow: none;
  }
  
  .sidebar-panel {
    width: 100%;
    flex: 1; /* Take remaining space */
    min-height: 0; /* Allow scrolling inside */
    gap: 0;
    background: #f0f2f5;
    padding: 0; /* Reset padding for mobile */
  }

  .control-card {
    border-radius: 0;
    border-top: 1px solid rgba(0,0,0,0.05);
    border-bottom: 1px solid rgba(0,0,0,0.05);
    padding: 12px;
  }
  
  .timeline-card {
    border-radius: 0;
    height: auto;
    flex: 1; /* Fill remaining space */
    background: transparent;
    box-shadow: none;
    border: none;
  }
  
  .panel-section {
    margin-bottom: 0;
  }

  /* Adjust buttons for mobile */
  .action-buttons {
    flex-direction: row;
    margin-bottom: 12px;
  }
  
  .settings-group {
    padding-top: 10px;
  }
}
</style>
