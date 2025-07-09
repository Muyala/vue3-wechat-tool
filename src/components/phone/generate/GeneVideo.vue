<template>
  <a-tooltip title="生成视频前可以修改配置" placement="right">
    <div class="wtc-button" @click="handleGenerateVideo">生成视频</div>
  </a-tooltip>

  <a-drawer 
    :width="500" 
    title="生成视频" 
    placement="right" 
    :closable="false" 
    :destroyOnClose="true" 
    :open="drawerVisible" 
    @close="onClose"
  >
    <template #extra>
      <a-space>
        <a-button type="primary" :disabled="!videoUrl" @click="handleDownloadVideo">
          下载视频
        </a-button>
        <a-button danger type="link" shape="circle" :icon="h(CloseOutlined)" @click="onClose" />
      </a-space>
    </template>
    
    <!-- 配置选项 -->
    <div class="video-config" v-if="!generating">
      <a-form layout="vertical">
        <a-form-item label="视频质量">
          <a-select v-model:value="videoConfig.quality" style="width: 100%">
            <a-select-option value="high">高质量 (1080p)</a-select-option>
            <a-select-option value="medium">中等质量 (720p)</a-select-option>
            <a-select-option value="low">预览质量 (480p)</a-select-option>
          </a-select>
        </a-form-item>
        
        <a-form-item label="帧率(FPS)">
          <a-slider v-model:value="videoConfig.fps" :min="10" :max="60" />
          <span>当前: {{ videoConfig.fps }} FPS</span>
        </a-form-item>
        
        <a-form-item label="背景风格">
          <a-radio-group v-model:value="videoConfig.backgroundStyle">
            <a-radio-button value="light">浅色</a-radio-button>
            <a-radio-button value="dark">深色</a-radio-button>
            <a-radio-button value="blur">模糊背景</a-radio-button>
          </a-radio-group>
        </a-form-item>
      </a-form>
    </div>
    
    <!-- 生成状态 -->
    <div class="generation-status">
      <a-progress
        v-if="generating"
        type="circle"
        :percent="progressPercent"
        :status="progressStatus"
      />
      
      <div class="status-text">
        <template v-if="generating">
          <a-spin size="large" />
          <p>{{ statusText }}</p>
          <p v-if="estimatedTime">预计剩余时间: {{ estimatedTime }}秒</p>
        </template>
        <template v-else-if="videoUrl">
          <video :src="videoUrl" controls autoplay muted class="preview-video"></video>
          <p class="success-text">视频生成成功！</p>
        </template>
      </div>
    </div>
    
    <!-- 操作区域 -->
    <div class="action-area" v-if="!generating && !videoUrl">
      <a-button 
        type="primary" 
        size="large"
        @click="startVideoGeneration"
        :loading="starting"
      >
        开始生成视频
      </a-button>
    </div>
    
    <!-- 错误提示 -->
    <a-alert 
      v-if="errorMessage" 
      :message="errorMessage" 
      type="error" 
      show-icon
      style="margin-top: 20px;"
    />
  </a-drawer>
</template>

<script setup>
import { ref, h, reactive, onMounted } from "vue";
import { CloseOutlined } from '@ant-design/icons-vue';
import _ from "lodash";
import dayjs from "dayjs";
import { createFFmpeg, fetchFile } from '@ffmpeg/ffmpeg';
import eventBus from '@/utils/eventBus';
import { sleep } from "@/utils/utils";
import useStore from "@/store";
const { useChatStore } = useStore();

// 视频配置
const videoConfig = reactive({
  quality: 'high',
  fps: 30,
  backgroundStyle: 'light',
});

// 第一条延迟多久展示
const initInterval = 1000;
const drawerVisible = ref(false);
const videoUrl = ref("");
const generating = ref(false);
const starting = ref(false);
const progressPercent = ref(0);
const progressStatus = ref('active');
const statusText = ref("准备生成环境...");
const errorMessage = ref("");
const estimatedTime = ref(0);
const ffmpegRef = ref(null);
const workerRef = ref(null);

const handleGenerateVideo = async () => {
  drawerVisible.value = true;
};

const onClose = () => {
  drawerVisible.value = false;
  videoUrl.value = "";
  generating.value = false;
  starting.value = false;
  progressPercent.value = 0;
  errorMessage.value = "";
  
  // 清理资源
  if (ffmpegRef.value) {
    try {
      ffmpegRef.value.exit();
    } catch (e) {
      console.log('FFmpeg清理完成');
    }
  }
  
  if (workerRef.value) {
    workerRef.value.terminate();
    workerRef.value = null;
  }
};

const startVideoGeneration = async () => {
  if (starting.value) return;
  
  try {
    starting.value = true;
    errorMessage.value = "";
    
    // 初始化FFmpeg
    const ffmpeg = createFFmpeg({
      log: true,
      corePath: 'https://unpkg.com/@ffmpeg/core@0.10.0/dist/ffmpeg-core.js'
    });
    ffmpegRef.value = ffmpeg;
    
    statusText.value = "加载视频引擎...";
    progressStatus.value = 'active';
    
    await ffmpeg.load();
    starting.value = false;
    generating.value = true;
    
    // 克隆聊天记录
    const chatList = _.cloneDeep(useChatStore.chatList);
    
    // 准备原始数据
    statusText.value = "准备原始数据...";
    await sleep(300);
    
    // 1. 获取高质量的聊天序列截图
    const frames = await captureAllFrames(chatList);
    
    // 2. 在Web Worker中生成视频
    await renderVideoFrames(ffmpeg, frames, chatList);
    
    // 3. 生成最终视频
    progressPercent.value = 90;
    statusText.value = "合成最终视频...";
    await sleep(1000);
    
    const videoData = ffmpeg.FS('readFile', 'output.mp4');
    videoUrl.value = URL.createObjectURL(
      new Blob([videoData.buffer], { type: 'video/mp4' })
    );
    
    progressPercent.value = 100;
    progressStatus.value = 'success';
    statusText.value = "视频生成完成！";
  } catch (error) {
    console.error('视频生成失败:', error);
    errorMessage.value = `视频生成失败: ${error.message}`;
    progressStatus.value = 'exception';
  } finally {
    generating.value = false;
  }
};

// 高清截图函数
const captureHighQualityFrame = async (node) => {
  const scaleFactors = {
    high: 3,
    medium: 2,
    low: 1
  };
  
  const canvas = document.createElement('canvas');
  const context = canvas.getContext('2d');
  
  // 根据质量设置缩放
  const scale = scaleFactors[videoConfig.quality] || 2;
  
  // 设置Canvas尺寸（高质量）
  const width = node.clientWidth * scale;
  const height = node.clientHeight * scale;
  
  canvas.width = width;
  canvas.height = height;
  
  // 设置Canvas样式
  context.scale(scale, scale);
  context.imageSmoothingEnabled = true;
  context.imageSmoothingQuality = 'high';
  
  // 绘制背景
  if (videoConfig.backgroundStyle === 'blur') {
    const background = await htmlToBlob(document.body);
    context.drawImage(background, 0, 0, canvas.width, canvas.height);
    
    // 应用模糊效果
    context.filter = 'blur(10px) brightness(0.7)';
    context.drawImage(background, 0, 0, canvas.width, canvas.height);
    context.filter = 'none';
  } else {
    const bgColor = videoConfig.backgroundStyle === 'dark' ? '#121212' : '#f0f0f0';
    context.fillStyle = bgColor;
    context.fillRect(0, 0, canvas.width, canvas.height);
  }
  
  // 绘制聊天区域
  context.drawImage(node, 0, 0, node.clientWidth, node.clientHeight);
  
  // 添加水印
  context.fillStyle = 'rgba(255, 255, 255, 0.6)';
  context.font = '16px PingFang SC, sans-serif';
  context.fillText('Generated by WeChat Video', 20, canvas.height - 20);
  
  return canvas;
};

// 将HTML元素转换为图像Blob
const htmlToBlob = async (element) => {
  const canvas = document.createElement('canvas');
  const context = canvas.getContext('2d');
  canvas.width = element.clientWidth;
  canvas.height = element.clientHeight;
  
  const html2canvas = (await import('html2canvas')).default;
  const tempCanvas = await html2canvas(element, {
    scale: 1,
    useCORS: true,
    backgroundColor: null
  });
  
  context.drawImage(tempCanvas, 0, 0);
  return tempCanvas;
};

// 捕获所有帧
const captureAllFrames = async (chatList) => {
  const frames = [];
  const originalChatList = _.cloneDeep(useChatStore.chatList);
  
  try {
    // 重置聊天记录
    useChatStore.chatList = [];
    
    // 捕获初始帧（0条消息）
    await sleep(300);
    const node = document.querySelector('.phone-wrap');
    const initFrame = await captureHighQualityFrame(node);
    frames.push({ 
      frame: initFrame, 
      duration: initInterval 
    });
    
    // 捕获每条消息后的帧
    for (let i = 0; i < chatList.length; i++) {
      // 恢复当前消息
      useChatStore.chatList = chatList.slice(0, i + 1);
      eventBus.emit("sentChat");
      
      // 等待UI更新
      await sleep(200);
      
      // 捕获当前帧
      const currentFrame = await captureHighQualityFrame(node);
      
      // 获取消息间隔时间
      const maxInterval = useChatStore.generateConfig.maxInterval;
      const minInterval = useChatStore.generateConfig.minInterval;
      const intervalTime = chatList[i].intervalTime || 
        Math.floor(Math.random() * (maxInterval - minInterval + 1)) + minInterval;
      
      frames.push({
        frame: currentFrame,
        duration: intervalTime
      });
      
      // 更新进度
      progressPercent.value = Math.floor((i + 1) / chatList.length * 70);
      statusText.value = `生成帧 ${i + 1}/${chatList.length}...`;
      
      // 估算剩余时间
      const frameTime = Math.max(intervalTime / 1000, 0.2);
      const remaining = Math.floor(frameTime * (chatList.length - i));
      estimatedTime.value = remaining > 0 ? remaining : 1;
    }
    
    return frames;
  } catch (error) {
    throw new Error(`帧捕获失败: ${error.message}`);
  } finally {
    // 恢复原始聊天记录
    useChatStore.chatList = originalChatList;
  }
};

// 使用FFmpeg渲染视频
const renderVideoFrames = async (ffmpeg, frames, chatList) => {
  statusText.value = "编码视频帧...";
  
  try {
    // 计算每帧应出现的持续时间 (毫秒)
    let frameDurations = frames.map(f => f.duration);
    
    // 写入帧到FFmpeg
    for (let i = 0; i < frames.length; i++) {
      const frame = frames[i].frame;
      const blob = await new Promise(resolve => frame.toBlob(resolve, 'image/png', 0.95));
      const data = await blob.arrayBuffer();
      ffmpeg.FS('writeFile', `frame-${i}.png`, new Uint8Array(data));
      
      // 更新进度
      progressPercent.value = 75 + Math.floor(i / frames.length * 15);
    }
    
    // 创建时间间隔文件
    let filterComplex = '';
    let inputs = '';
    let lastStream = '';
    
    // 构建复杂的过滤器图
    for (let i = 0; i < frames.length; i++) {
      inputs += `-i frame-${i}.png `;
      
      if (i === 0) {
        filterComplex += `[0]loop=1:${Math.ceil(frameDurations[0] / 1000 * videoConfig.fps)}[v0];`;
        lastStream = 'v0';
      } else {
        filterComplex += `[${i}]loop=1:${Math.ceil(frameDurations[i] / 1000 * videoConfig.fps)}[v${i}];`;
        
        if (i === frames.length - 1) {
          filterComplex += `${lastStream}[v${i}]concat=n=2[video];`;
        } else {
          filterComplex += `${lastStream}[v${i}]concat=n=2[v${i}temp];`;
          lastStream = `v${i}temp`;
        }
      }
    }
    
    // 设置视频尺寸（基于质量）
    const resolutions = {
      high: '1920x1080',
      medium: '1280x720',
      low: '854x480'
    };
    
    const resolution = resolutions[videoConfig.quality] || resolutions.medium;
    const [width, height] = resolution.split('x').map(Number);
    
    // 添加最终缩放滤镜
    filterComplex += `[video]scale=${width}:${height}:force_original_aspect_ratio=increase,crop=${width}:${height}[final]`;
    
    // 运行FFmpeg命令
    const args = [
      '-y',
      ...inputs.trim().split(' '),
      '-filter_complex', filterComplex,
      '-map', '[final]',
      '-r', String(videoConfig.fps),
      '-c:v', 'libx264',
      '-preset', 'medium',
      '-crf', '18',
      '-pix_fmt', 'yuv420p',
      'output.mp4'
    ];
    
    await ffmpeg.run(...args);
    
    return true;
  } catch (error) {
    console.error('视频编码错误:', error);
    throw new Error(`视频编码失败: ${error.message}`);
  }
};

const handleDownloadVideo = () => {
  if (!videoUrl.value) return;
  
  const link = document.createElement('a');
  link.href = videoUrl.value;
  link.download = `微信聊天记录-${dayjs().format('YYYYMMDDHHmmss')}.mp4`;
  link.target = '_blank';
  link.rel = 'noopener noreferrer';
  
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  
  // 添加反馈
  message.success('视频下载成功！');
};

// 组件卸载时清理资源
onMounted(() => {
  window.addEventListener('beforeunload', onClose);
});
</script>

<style scoped>
.wtc-button {
  padding: 8px 15px;
  background-color: #07c160;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  transition: all 0.3s;
}

.wtc-button:hover {
  background-color: #06ad56;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.generation-status {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin: 20px 0;
  min-height: 300px;
}

.status-text {
  text-align: center;
  margin-top: 20px;
}

.success-text {
  color: #52c41a;
  font-weight: bold;
  margin-top: 10px;
}

.preview-video {
  width: 100%;
  max-height: 300px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  background: #000;
}

.video-config {
  margin-bottom: 20px;
  padding: 15px;
  background-color: #f9f9f9;
  border-radius: 8px;
}

.action-area {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

.default-loading {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 300px;
}

:deep(.ant-progress-text) {
  font-size: 16px;
  font-weight: bold;
}
</style>
