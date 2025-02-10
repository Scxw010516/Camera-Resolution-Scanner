<template>
  <a-space direction="vertical" size="large" style="width: 100%">
    <!-- 摄像头选择 -->
    <a-card class="camera-selector">
      <a-select
        v-model:value="selectedDeviceId"
        :options="deviceOptions"
        placeholder="请选择摄像头"
        style="width: 100%"
        @change="onDeviceChange"
      />
    </a-card>

    <!-- 分辨率和帧率配置 -->
    <a-card>
      <template #title>测试配置</template>
      <template #extra>
        <a-button type="link" @click="resetToDefault">重置为默认配置</a-button>
      </template>

      <a-space direction="vertical" size="middle" style="width: 100%">
        <!-- 分辨率配置 -->
        <div>
          <div class="section-header">
            <h3 class="section-title">分辨率</h3>
            <a-button type="primary" @click="showAddResolutionModal = true">
              <template #icon><plus-outlined /></template>
              添加分辨率
            </a-button>
          </div>
          <div class="tag-container">
            <a-tag
              v-for="tag in resolutionTags"
              :key="tag"
              :closable="true"
              class="resolution-tag"
              @close="handleCloseResolution($event, tag)"
            >
              <div class="tag-content">
                <span class="resolution">{{ formatResolutionTagText(tag).resolution }}</span>
                <span class="megapixels">{{ formatResolutionTagText(tag).megapixels }}</span>
                <span v-if="formatResolutionTagText(tag).label" class="standard">
                  {{ formatResolutionTagText(tag).label }}
                </span>
              </div>
            </a-tag>
          </div>
        </div>

        <a-divider style="margin: 8px 0" />

        <!-- 帧率配置 -->
        <div>
          <div class="section-header">
            <h3 class="section-title">帧率</h3>
            <a-button type="primary" @click="showAddFrameRateModal = true">
              <template #icon><plus-outlined /></template>
              添加帧率
            </a-button>
          </div>
          <div class="tag-container">
            <a-tag
              v-for="tag in frameRateTags"
              :key="tag"
              :closable="true"
              class="framerate-tag"
              @close="handleCloseFrameRate($event, tag)"
            >
              <div class="tag-content">
                {{ formatFrameRateTagText(tag) }}
              </div>
            </a-tag>
          </div>
        </div>

        <!-- 扫描按钮 -->
        <a-button type="primary" :disabled="!canScan" :loading="isScanning" block @click="startScan">
          {{ isScanning ? "扫描中..." : "开始扫描" }}
        </a-button>

        <!-- 扫描进度 -->
        <template v-if="isScanning">
          <a-progress :percent="progress" :show-info="true" />
          <div class="text-center text-gray-500">{{ currentTest }}</div>
        </template>
      </a-space>
    </a-card>

    <!-- 扫描结果 -->
    <a-card v-if="resolutions.length > 0">
      <template #title>扫描结果</template>

      <a-list :data-source="sortedResolutions" :split="true">
        <template #renderItem="{ item }">
          <a-list-item>
            <template v-if="isCurrentPreview(item)">
              <a-card :class="['result-card', 'preview-active']" :bordered="false">
                <template #title>
                  <div class="flex justify-between items-center">
                    <span>{{ formatResolution(item) }}</span>
                  </div>
                </template>
                <template #extra>
                  <a-button type="primary" @click="togglePreview(item)"> 停止预览 </a-button>
                </template>

                <!-- 预览区域 -->
                <div class="inline-preview">
                  <video :ref="(el) => setVideoRef(el, item)" autoplay playsinline muted></video>
                  <div v-if="currentResolution" class="preview-info">
                    <a-descriptions :column="1" size="small">
                      <a-descriptions-item label="实际分辨率">
                        {{ currentResolution.actualWidth }}x{{ currentResolution.actualHeight }}
                      </a-descriptions-item>
                      <a-descriptions-item label="实际帧率">
                        {{ currentResolution.actualFrameRate.toFixed(1) }}fps
                      </a-descriptions-item>
                    </a-descriptions>
                  </div>
                </div>
              </a-card>
            </template>
            <template v-else>
              <a-card :class="['result-card']" :bordered="false" size="small">
                <template #title>
                  <div class="flex justify-between items-center">
                    <span>{{ formatResolution(item) }}</span>
                  </div>
                </template>
                <template #extra>
                  <a-button type="default" @click="togglePreview(item)"> 预览 </a-button>
                </template>
              </a-card>
            </template>
          </a-list-item>
        </template>
      </a-list>
    </a-card>

    <!-- 添加分辨率模态框 -->
    <a-modal v-model:open="showAddResolutionModal" title="添加自定义分辨率" @ok="handleAddResolution">
      <a-form :model="newResolution" layout="vertical">
        <a-form-item label="宽度" name="width" :rules="addResolutionRules.width" required>
          <a-input-number
            v-model:value="newResolution.width"
            :min="160"
            :max="8000"
            placeholder="请输入宽度 (160-8000)"
          />
        </a-form-item>
        <a-form-item label="高度" name="height" :rules="addResolutionRules.height" required>
          <a-input-number
            v-model:value="newResolution.height"
            :min="120"
            :max="6000"
            placeholder="请输入高度 (120-6000)"
          />
        </a-form-item>
        <a-form-item label="标签" name="label">
          <a-input v-model:value="newResolution.label" placeholder="可选，如：4K" />
        </a-form-item>
      </a-form>
    </a-modal>

    <!-- 添加帧率模态框 -->
    <a-modal v-model:open="showAddFrameRateModal" title="添加自定义帧率" @ok="handleAddFrameRate">
      <a-form :model="newFrameRate" layout="vertical">
        <a-form-item label="帧率" name="value" :rules="addFrameRateRules.value" required>
          <a-input-number v-model:value="newFrameRate.value" :min="1" :max="240" placeholder="请输入帧率 (1-240)" />
        </a-form-item>
      </a-form>
    </a-modal>
  </a-space>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick } from "vue";
import { PlusOutlined } from "@ant-design/icons-vue";
import { message } from "ant-design-vue";

// 预设配置
const defaultResolutions = [
  { width: 160, height: 120, label: "QQVGA" },
  { width: 320, height: 240, label: "QVGA" },
  { width: 640, height: 480, label: "VGA" },
  { width: 800, height: 600, label: "SVGA" },
  { width: 1024, height: 768, label: "XGA" },
  { width: 1280, height: 960, label: "SXGA-" },
  { width: 1400, height: 1050, label: "SXGA+" },
  { width: 1600, height: 1200, label: "UXGA" },
  { width: 2048, height: 1536, label: "QXGA" },
  { width: 1280, height: 720, label: "HD" },
  { width: 1920, height: 1080, label: "Full HD" },
  { width: 2560, height: 1440, label: "QHD" },
  { width: 3840, height: 2160, label: "4K UHD" },
  { width: 4000, height: 3000, label: "12MP" },
  { width: 8000, height: 6000, label: "48MP" },
];

const defaultFrameRates = [
  { value: 5, label: "5 FPS" },
  { value: 10, label: "10 FPS" },
  { value: 15, label: "15 FPS" },
  { value: 20, label: "20 FPS" },
  { value: 24, label: "24 FPS" },
  { value: 25, label: "25 FPS" },
  { value: 30, label: "30 FPS" },
  { value: 50, label: "50 FPS" },
  { value: 60, label: "60 FPS" },
];

// 基础状态
const devices = ref([]);
const selectedDeviceId = ref(null);
const currentDevice = computed(() => devices.value.find((d) => d.deviceId === selectedDeviceId.value));

// 用户可编辑的列表
const resolutionsToTest = ref([...defaultResolutions]);
const frameRatesToTest = ref([...defaultFrameRates]);

// 标签列表
const resolutionTags = ref(defaultResolutions.map((res) => `${res.width}x${res.height}`));
const frameRateTags = ref(defaultFrameRates.map((fps) => fps.value.toString()));

// 扫描和预览状态
const isScanning = ref(false);
const progress = ref(0);
const resolutions = ref([]);
const currentTest = ref("");
const currentResolution = ref(null);
let stream = null;
const videoElements = new Map();

// 模态框状态
const showAddResolutionModal = ref(false);
const showAddFrameRateModal = ref(false);
const newResolution = ref({ width: null, height: null, label: "" });
const newFrameRate = ref({ value: null });

// 计算属性
const deviceOptions = computed(() => {
  return devices.value.map((device) => ({
    label: device.label || `Camera ${device.deviceId}`,
    value: device.deviceId,
  }));
});

const sortedResolutions = computed(() => {
  return [...resolutions.value].sort((a, b) => {
    const aPixels = a.width * a.height;
    const bPixels = b.width * b.height;
    if (aPixels === bPixels) {
      return b.frameRate - a.frameRate;
    }
    return bPixels - aPixels;
  });
});

const canScan = computed(() => {
  return selectedDeviceId.value && resolutionTags.value.length > 0 && frameRateTags.value.length > 0;
});

// 格式化函数
const formatResolutionTagText = (value) => {
  if (!value) return "";
  try {
    const [width, height] = value.split("x").map(Number);
    if (isNaN(width) || isNaN(height)) return value;

    const resInfo = resolutionsToTest.value.find((res) => res.width === width && res.height === height);
    if (!resInfo) return value;

    const mp = ((width * height) / 1000000).toFixed(1);
    return {
      resolution: `${width}x${height}`,
      megapixels: `${mp}MP`,
      label: resInfo.label || "",
    };
  } catch (error) {
    console.error("格式化分辨率标签时出错:", error);
    return { resolution: value, megapixels: "", label: "" };
  }
};

const formatFrameRateTagText = (value) => {
  if (!value) return "";
  try {
    const fps = parseInt(value);
    if (isNaN(fps)) return value;
    return `${fps} FPS`;
  } catch (error) {
    console.error("格式化帧率标签时出错:", error);
    return value;
  }
};

// 处理标签关闭事件
const handleCloseResolution = (e, tag) => {
  e?.preventDefault();
  const [width, height] = tag.split("x").map(Number);
  const index = resolutionsToTest.value.findIndex((res) => res.width === width && res.height === height);
  if (index !== -1) {
    resolutionsToTest.value.splice(index, 1);
    const tagIndex = resolutionTags.value.indexOf(tag);
    if (tagIndex !== -1) {
      resolutionTags.value.splice(tagIndex, 1);
    }
  }
};

const handleCloseFrameRate = (e, tag) => {
  e?.preventDefault();
  const fps = parseInt(tag);
  const index = frameRatesToTest.value.findIndex((item) => item.value === fps);
  if (index !== -1) {
    frameRatesToTest.value.splice(index, 1);
    const tagIndex = frameRateTags.value.indexOf(tag);
    if (tagIndex !== -1) {
      frameRateTags.value.splice(tagIndex, 1);
    }
  }
};

// 验证规则
const addResolutionRules = {
  width: [
    { required: true, message: "请输入宽度" },
    { type: "number", min: 160, max: 8000, message: "宽度必须在160-8000之间" },
  ],
  height: [
    { required: true, message: "请输入高度" },
    { type: "number", min: 120, max: 6000, message: "高度必须在120-6000之间" },
  ],
};

const addFrameRateRules = {
  value: [
    { required: true, message: "请输入帧率" },
    { type: "number", min: 1, max: 240, message: "帧率必须在1-240之间" },
  ],
};

// 错误处理函数
const handleCameraError = (error) => {
  console.error("摄像头错误:", error);

  switch (error.name) {
    case "NotReadableError":
      message.error(
        "无法访问摄像头，请检查：\n1. 确保没有其他程序正在使用摄像头\n2. 尝试重新插拔摄像头\n3. 重启浏览器\n4. 检查摄像头驱动是否正常"
      );
      break;
    case "NotAllowedError":
      message.error("未获得摄像头访问权限，请检查浏览器设置中的摄像头权限，并刷新页面重试");
      break;
    case "NotFoundError":
      message.error("未找到可用的摄像头设备，请确保摄像头已正确连接到电脑");
      break;
    default:
      message.error(error.message || "未知错误");
  }
};

// 获取设备列表
const getDevices = async () => {
  try {
    const initialDevices = await navigator.mediaDevices.enumerateDevices();
    const videoDevices = initialDevices.filter((device) => device.kind === "videoinput");

    if (videoDevices.length === 0) {
      message.warning("未检测到摄像头设备");
      return;
    }

    // 如果有设备但没有标签信息，则请求权限
    if (videoDevices.some((device) => !device.label)) {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: true });
        stream.getTracks().forEach((track) => track.stop());
        const mediaDevices = await navigator.mediaDevices.enumerateDevices();
        devices.value = mediaDevices.filter((device) => device.kind === "videoinput");
        message.success("成功获取摄像头权限");
      } catch (error) {
        handleCameraError(error);
        devices.value = videoDevices;
      }
    } else {
      devices.value = videoDevices;
    }

    // 更新设备监听器
    const deviceChangeHandler = async () => {
      try {
        const updatedDevices = await navigator.mediaDevices.enumerateDevices();
        const newVideoDevices = updatedDevices.filter((device) => device.kind === "videoinput");

        // 检查设备数量变化
        const oldDeviceIds = new Set(devices.value.map((d) => d.deviceId));
        const newDeviceIds = new Set(newVideoDevices.map((d) => d.deviceId));

        if (oldDeviceIds.size < newDeviceIds.size) {
          message.success("检测到新的摄像头设备已连接");
        } else if (oldDeviceIds.size > newDeviceIds.size) {
          message.warning("检测到摄像头设备已断开");
          if (selectedDeviceId.value && !newDeviceIds.has(selectedDeviceId.value)) {
            stopPreview();
            selectedDeviceId.value = null;
          }
        }

        devices.value = newVideoDevices;
      } catch (error) {
        console.error("设备变化检测失败:", error);
      }
    };

    // 移除旧的监听器（如果存在）
    navigator.mediaDevices.removeEventListener("devicechange", deviceChangeHandler);
    navigator.mediaDevices.addEventListener("devicechange", deviceChangeHandler);
  } catch (error) {
    console.error("获取设备列表失败:", error);
    handleCameraError(error);
  }
};

// 检查摄像头权限状态
const checkPermissionStatus = async () => {
  try {
    const permissionStatus = await navigator.permissions.query({ name: "camera" });
    switch (permissionStatus.state) {
      case "granted":
        return true;
      case "prompt":
        message.info("需要获取摄像头权限，请在弹出的对话框中允许访问");
        return null;
      case "denied":
        message.error("摄像头权限已被拒绝，请在浏览器设置中重新授予摄像头访问权限，然后刷新页面");
        return false;
    }
  } catch (error) {
    console.error("检查权限状态失败:", error);
    return null; // 不支持权限查询API，返回null
  }
};

// 检查设备状态
const checkDeviceStatus = async (deviceId) => {
  if (!deviceId) return false;

  try {
    const stream = await navigator.mediaDevices.getUserMedia({
      video: {
        deviceId: { exact: deviceId },
        width: { ideal: 640 },
        height: { ideal: 480 },
      },
    });

    stream.getTracks().forEach((track) => track.stop());
    return true;
  } catch (error) {
    handleCameraError(error);
    return false;
  }
};

// 处理设备变更
const onDeviceChange = async (newDeviceId) => {
  if (selectedDeviceId.value === newDeviceId) return;

  stopPreview();
  selectedDeviceId.value = newDeviceId;
  resolutions.value = [];

  if (newDeviceId) {
    const isAvailable = await checkDeviceStatus(newDeviceId);
    if (!isAvailable) {
      message.warning("选中的摄像头可能无法正常使用，请检查设备状态");
    }
  }
};

// 测试分辨率
const testResolution = async (deviceId, width, height, frameRate) => {
  if (!deviceId) return false;

  const constraints = {
    video: {
      deviceId: { exact: deviceId },
      width: { ideal: width },
      height: { ideal: height },
      frameRate: { ideal: frameRate },
    },
  };

  let retryCount = 0;
  while (retryCount < 2) {
    try {
      const testStream = await navigator.mediaDevices.getUserMedia(constraints);
      const videoTrack = testStream.getVideoTracks()[0];
      const settings = videoTrack.getSettings();

      // 检查实际获得的值是否接近请求的值（允许5%的误差）
      const widthMatch = Math.abs(settings.width - width) / width <= 0.05;
      const heightMatch = Math.abs(settings.height - height) / height <= 0.05;
      const frameRateMatch = Math.abs(settings.frameRate - frameRate) / frameRate <= 0.05;

      testStream.getTracks().forEach((track) => track.stop());
      return widthMatch && heightMatch && frameRateMatch;
    } catch (error) {
      if (error.name === "OverconstrainedError") {
        return false;
      }

      if (error.name === "NotReadableError" && retryCount < 1) {
        await new Promise((resolve) => setTimeout(resolve, 1000));
        retryCount++;
        continue;
      }

      console.error(`测试分辨率 ${width}x${height}@${frameRate}fps 时发生错误:`, error);
      return false;
    }
  }
  return false;
};

// 开始扫描
const startScan = async () => {
  if (!selectedDeviceId.value || !canScan.value) return;

  // 检查权限状态
  const hasPermission = await checkPermissionStatus();
  if (hasPermission === false) return;

  // 检查设备状态
  const isAvailable = await checkDeviceStatus(selectedDeviceId.value);
  if (!isAvailable) {
    message.error("摄像头当前无法使用，请检查设备状态后重试");
    return;
  }

  try {
    isScanning.value = true;
    resolutions.value = [];
    progress.value = 0;

    message.info("开始扫描支持的分辨率和帧率...");

    const selectedResolutionsList = resolutionsToTest.value.filter((res) =>
      resolutionTags.value.includes(`${res.width}x${res.height}`)
    );

    const selectedFrameRatesList = frameRatesToTest.value.filter((fps) =>
      frameRateTags.value.includes(fps.value.toString())
    );

    const totalTests = selectedResolutionsList.length * selectedFrameRatesList.length;
    let completedTests = 0;
    let supportedCount = 0;

    for (const res of selectedResolutionsList) {
      for (const fps of selectedFrameRatesList) {
        currentTest.value = `${res.width}x${res.height} @ ${fps.value}fps`;
        const supported = await testResolution(selectedDeviceId.value, res.width, res.height, fps.value);

        if (supported) {
          resolutions.value.push({
            width: res.width,
            height: res.height,
            frameRate: fps.value,
          });
          supportedCount++;
        }

        completedTests++;
        progress.value = Math.round((completedTests / totalTests) * 100);
        await new Promise((resolve) => setTimeout(resolve, 0));
      }
    }

    currentTest.value = "";
    isScanning.value = false;

    if (supportedCount > 0) {
      message.success(`扫描完成，找到 ${supportedCount} 个支持的配置`);
    } else {
      message.warning("扫描完成，未找到支持的配置");
    }
  } catch (error) {
    console.error("扫描失败:", error);
    message.error("扫描过程中发生错误: " + error.message);
    isScanning.value = false;
  }
};

// 切换预览状态
const togglePreview = async (resolution) => {
  if (isCurrentPreview(resolution)) {
    stopPreview();
  } else {
    // 先设置当前分辨率，触发UI更新
    currentResolution.value = {
      ...resolution,
      actualWidth: 0,
      actualHeight: 0,
      actualFrameRate: 0,
    };

    // 等待DOM更新完成
    await nextTick();

    // 开始预览
    await previewResolution(resolution);
  }
};

// 预览分辨率
const previewResolution = async (resolution) => {
  if (!selectedDeviceId.value) return;

  try {
    const videoKey = `${resolution.width}x${resolution.height}@${resolution.frameRate}`;
    const videoElement = videoElements.get(videoKey);

    if (!videoElement) {
      throw new Error("视频元素未找到，请稍后重试");
    }

    const loadingKey = "previewLoading";
    message.loading({ content: "正在启动预览...", key: loadingKey, duration: 0 });

    // 首先尝试使用精确约束
    let constraints = {
      video: {
        deviceId: { exact: selectedDeviceId.value },
        width: { exact: resolution.width },
        height: { exact: resolution.height },
        frameRate: { exact: resolution.frameRate },
      },
    };

    try {
      stream = await navigator.mediaDevices.getUserMedia(constraints);
    } catch (error) {
      console.log("精确约束失败，尝试使用理想约束", error);

      // 如果精确约束失败，尝试使用理想约束
      constraints = {
        video: {
          deviceId: { exact: selectedDeviceId.value },
          width: { ideal: resolution.width },
          height: { ideal: resolution.height },
          frameRate: { ideal: resolution.frameRate },
        },
      };

      stream = await navigator.mediaDevices.getUserMedia(constraints);
    }

    const videoTrack = stream.getVideoTracks()[0];
    videoElement.srcObject = stream;

    // 设置加载超时
    const timeoutPromise = new Promise((_, reject) => {
      setTimeout(() => {
        reject(new Error("视频加载超时"));
      }, 5000);
    });

    // 等待视频加载完成或超时
    await Promise.race([
      new Promise((resolve) => {
        videoElement.onloadedmetadata = () => {
          const settings = videoTrack.getSettings();

          // 检查实际获得的分辨率和帧率
          const actualWidth = settings.width;
          const actualHeight = settings.height;
          const actualFrameRate = settings.frameRate;

          // 计算误差百分比
          const widthDiff = Math.abs(actualWidth - resolution.width) / resolution.width;
          const heightDiff = Math.abs(actualHeight - resolution.height) / resolution.height;
          const frameRateDiff = Math.abs(actualFrameRate - resolution.frameRate) / resolution.frameRate;

          // 如果误差太大，显示警告
          if (widthDiff > 0.1 || heightDiff > 0.1 || frameRateDiff > 0.1) {
            message.warning("摄像头实际输出与请求的参数有较大差异");
          }

          currentResolution.value = {
            ...resolution,
            actualWidth,
            actualHeight,
            actualFrameRate,
          };

          resolve();
        };
      }),
      timeoutPromise,
    ]);

    message.destroy(loadingKey);
    message.success("预览已启动");

    // 开始播放视频
    try {
      await videoElement.play();
    } catch (error) {
      console.error("视频播放失败:", error);
      message.error("视频播放失败，请重试");
      stopPreview();
      return;
    }

    // 监听视频错误
    videoElement.onerror = (error) => {
      console.error("视频播放过程中出错:", error);
      message.error("视频播放出错，请重试");
      stopPreview();
    };
  } catch (error) {
    console.error("预览错误:", error);
    message.destroy();

    if (error.name === "OverconstrainedError") {
      message.error("摄像头不支持该分辨率或帧率，请尝试其他配置");
    } else if (error.name === "NotReadableError") {
      message.error("无法访问摄像头，请确保没有其他程序正在使用该设备");
    } else if (error.name === "NotAllowedError") {
      message.error("未获得摄像头访问权限，请在浏览器设置中允许访问");
    } else if (error.message === "视频加载超时") {
      message.error("视频加载超时，请检查设备连接并重试");
    } else {
      message.error(`预览失败: ${error.message || "未知错误"}`);
    }

    currentResolution.value = null;
    stopPreview();
  }
};

// 停止预览
const stopPreview = () => {
  if (stream) {
    stream.getTracks().forEach((track) => track.stop());
    stream = null;
  }

  // 清除所有视频元素的源
  videoElements.forEach((videoEl) => {
    if (videoEl && videoEl.srcObject) {
      videoEl.srcObject = null;
    }
  });

  currentResolution.value = null;
};

// 检查是否是当前预览的分辨率
const isCurrentPreview = (resolution) => {
  return (
    currentResolution.value &&
    currentResolution.value.width === resolution.width &&
    currentResolution.value.height === resolution.height &&
    currentResolution.value.frameRate === resolution.frameRate
  );
};

// 设置视频引用
const setVideoRef = (el, resolution) => {
  const videoKey = `${resolution.width}x${resolution.height}@${resolution.frameRate}`;
  if (el) {
    videoElements.set(videoKey, el);
  } else {
    videoElements.delete(videoKey);
  }
};

// 初始化数据
const initializeData = () => {
  // 重置测试列表
  resolutionsToTest.value = [...defaultResolutions];
  frameRatesToTest.value = [...defaultFrameRates];

  // 重置标签列表
  resolutionTags.value = defaultResolutions.map((res) => `${res.width}x${res.height}`);
  frameRateTags.value = defaultFrameRates.map((fps) => fps.value.toString());
};

// 添加自定义分辨率
const handleAddResolution = () => {
  if (newResolution.value.width && newResolution.value.height) {
    const width = newResolution.value.width;
    const height = newResolution.value.height;
    const label = newResolution.value.label || "";

    // 检查是否已存在
    const resKey = `${width}x${height}`;
    const exists = resolutionTags.value.includes(resKey);

    if (exists) {
      message.warning("该分辨率已存在");
      return;
    }

    const newRes = { width, height, label };
    resolutionsToTest.value.push(newRes);
    resolutionTags.value.push(resKey);
    showAddResolutionModal.value = false;
    newResolution.value = { width: null, height: null, label: "" };
    message.success("添加成功");
  }
};

// 添加自定义帧率
const handleAddFrameRate = () => {
  if (newFrameRate.value.value) {
    const fps = newFrameRate.value.value;
    const fpsStr = fps.toString();

    // 检查是否已存在
    const exists = frameRateTags.value.includes(fpsStr);

    if (exists) {
      message.warning("该帧率已存在");
      return;
    }

    const newFps = { value: fps, label: `${fps} FPS` };
    frameRatesToTest.value.push(newFps);
    frameRateTags.value.push(fpsStr);
    showAddFrameRateModal.value = false;
    newFrameRate.value.value = null;
    message.success("添加成功");
  }
};

// 重置为默认配置
const resetToDefault = () => {
  initializeData();
  message.success("已重置为默认配置");
};

// 格式化函数
const formatResolution = (res) => {
  const megapixels = ((res.width * res.height) / 1000000).toFixed(1);
  return `${res.width}x${res.height} (${megapixels}MP) @ ${res.frameRate}fps`;
};

// 生命周期钩子
onMounted(() => {
  initializeData();
  getDevices();
});

onUnmounted(() => {
  stopPreview();
});
</script>

<style scoped>
.camera-selector {
  margin-bottom: 16px;
}

.inline-preview {
  margin-top: 16px;
}

.inline-preview video {
  width: 100%;
  max-height: 400px;
  object-fit: contain;
  background: #000;
  border-radius: 8px;
}

.preview-info {
  margin-top: 12px;
  padding: 12px;
  background: #f8f8f8;
  border-radius: 4px;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
  flex-wrap: nowrap;
}

.section-title {
  margin: 0;
  font-size: 16px;
  font-weight: 500;
}

.section-divider {
  height: 1px;
  background: #f0f0f0;
  margin: 24px 0;
  width: 100%;
}

.tag-container {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
  width: 100%;
  padding: 4px 0;
}

.tag-container :deep(.ant-tag) {
  margin: 0;
  cursor: pointer;
  padding: 4px 8px;
  height: 32px;
  width: 100%;
  display: inline-flex;
  align-items: center;
  justify-content: space-between;
  white-space: nowrap;
  overflow: hidden;
  position: relative;
}

.tag-container :deep(.ant-tag .anticon-close) {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
}

.tag-container :deep(.ant-tag .tag-content) {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  flex: 1;
  justify-content: center;
  margin-right: 8px;
  font-size: 14px;
}

.resolution-tag {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: #f0f5ff;
  border-color: #adc6ff;
  color: #1f2937;
  padding-right: 24px !important;
}

.resolution-tag .resolution {
  font-weight: 500;
  white-space: nowrap;
}

.resolution-tag .megapixels {
  color: #666;
  font-size: 0.9em;
  white-space: nowrap;
}

.resolution-tag .standard {
  color: #1890ff;
  font-size: 0.9em;
  font-weight: 500;
  white-space: nowrap;
}

.framerate-tag {
  background: #e6f7ff;
  border-color: #91d5ff;
  color: #1890ff;
  font-weight: 500;
  padding-right: 24px !important;
}

.result-card {
  width: 100%;
  transition: all 0.3s ease;
}

.result-card.preview-active {
  background: #fafafa;
}

:deep(.ant-card) {
  transition: all 0.3s ease;
}

:deep(.ant-btn) {
  transition: all 0.3s ease;
}

:deep(.ant-input-number) {
  width: 100%;
}

:deep(.ant-descriptions-item-label) {
  color: #666;
}
</style>
