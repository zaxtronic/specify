<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>T-800 Vision Lite</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true" class="page">
      <div class="stage">
        <video
          v-show="mode === 'camera'"
          ref="videoRef"
          class="video"
          autoplay
          playsinline
          muted
        ></video>
        <img v-show="mode === 'image'" ref="imageRef" class="video" alt="Uploaded" />
        <video
          v-show="mode === 'video'"
          ref="uploadVideoRef"
          class="video"
          autoplay
          playsinline
          muted
          loop
        ></video>
        <canvas ref="canvasRef" class="overlay"></canvas>

        <div class="grid" v-if="showGrid"></div>
        <div class="scanline" v-if="showScanline"></div>
        <div class="reticle"></div>

        <div class="hud hud-right">
          <div class="hud-title">TARGET</div>
          <div class="hud-main">{{ targetLabel }}</div>
          <div class="hud-sub">{{ confidenceText }}</div>
          <div class="hud-status">{{ status }}</div>
          <div class="hud-fps">FPS: {{ fps.toFixed(1) }}</div>
          <div class="hud-mode">Mode: {{ highAccuracy ? 'HIGH' : 'FAST' }}</div>
          <div class="hud-meta">Source: {{ mode }}</div>
          <div class="hud-meta">Source size: {{ sourceSizeText }}</div>
          <div class="hud-meta">Video state: {{ videoStateText }}</div>
          <div class="hud-meta">Detect: {{ detectMs }} ms (avg {{ avgDetectMs }} ms)</div>
          <div class="hud-meta">Model load: {{ modelLoadMs }} ms</div>
          <div class="hud-meta">Last update: {{ lastUpdatedText }}</div>
        </div>

        <div class="hud hud-left">
          <div class="hud-title">DETECTIONS</div>
          <div v-if="detectionsList.length === 0" class="hud-empty">None</div>
          <div v-for="(d, i) in detectionsList" :key="i" class="hud-row">
            <span class="hud-label">{{ d.label }}</span>
            <span class="hud-score">{{ Math.round(d.score * 100) }}%</span>
          </div>
          <div class="hud-sep"></div>
          <div class="hud-title">COUNTS</div>
          <div v-if="detectionCounts.length === 0" class="hud-empty">None</div>
          <div v-for="(d, i) in detectionCounts" :key="i" class="hud-row">
            <span class="hud-label">{{ d.label }}</span>
            <span class="hud-score">{{ d.count }}</span>
          </div>
          <div class="hud-sep"></div>
          <div class="hud-title">TOTAL</div>
          <div class="hud-row">
            <span class="hud-label">Objects</span>
            <span class="hud-score">{{ totalObjects }}</span>
          </div>
        </div>

        <div class="controls">
          <button class="btn" @click="toggleRun">{{ isRunning ? 'Stop' : 'Start' }}</button>
          <button class="btn ghost" @click="settingsOpen = !settingsOpen">Settings</button>
        </div>

        <div class="media-controls">
          <button class="btn ghost" @click="switchToCamera">Camera</button>
          <label class="btn ghost file-btn">
            Upload Photo
            <input type="file" accept="image/*" @change="onPhotoSelected" />
          </label>
          <label class="btn ghost file-btn">
            Upload Video
            <input type="file" accept="video/*" @change="onVideoSelected" />
          </label>
        </div>

        <button v-if="mode === 'video' && needsPlay" class="play-btn" @click="playVideo">
          Tap to Play Video
        </button>

        <div class="settings" v-if="settingsOpen">
          <div class="settings-title">T-800 SETTINGS</div>

          <label class="settings-row">
            <span>Score Threshold: {{ scoreThreshold.toFixed(2) }}</span>
            <input type="range" min="0.05" max="0.6" step="0.05" v-model.number="scoreThreshold" />
          </label>

          <label class="settings-row">
            <span>Detection Interval: {{ detectionInterval }} ms</span>
            <input type="range" min="50" max="200" step="10" v-model.number="detectionInterval" />
          </label>

          <label class="settings-row">
            <span>Max Detections: {{ maxDetections }}</span>
            <input type="range" min="5" max="30" step="1" v-model.number="maxDetections" />
          </label>

          <label class="settings-row">
            <span>Input Size: {{ inputSize }} px</span>
            <input type="range" min="320" max="960" step="80" v-model.number="inputSize" />
          </label>

          <label class="settings-row toggle">
            <span>High Accuracy</span>
            <input type="checkbox" v-model="highAccuracy" />
          </label>

          <label class="settings-row toggle">
            <span>Fast Mode (downscale)</span>
            <input type="checkbox" v-model="fastMode" />
          </label>

          <label class="settings-row toggle">
            <span>Show Grid</span>
            <input type="checkbox" v-model="showGrid" />
          </label>

          <label class="settings-row toggle">
            <span>Show Scanline</span>
            <input type="checkbox" v-model="showScanline" />
          </label>

          <label class="settings-row toggle">
            <span>Focus Daily Objects Only</span>
            <input type="checkbox" v-model="focusOnly" />
          </label>

          <div class="settings-hint">
            Focus list: people, animals, tech, furniture, kitchen, street objects (expanded).
          </div>

          <label class="settings-row toggle">
            <span>Use Custom Focus List</span>
            <input type="checkbox" v-model="useCustomFocus" />
          </label>

          <label class="settings-row" v-if="useCustomFocus">
            <span>Custom list (comma separated)</span>
            <input type="text" v-model="customFocus" placeholder="person, laptop, bottle, chair" />
          </label>

          <div v-if="!useCustomFocus" class="settings-grid">
            <label><input type="checkbox" v-model="focusPeople" /> People</label>
            <label><input type="checkbox" v-model="focusAnimals" /> Animals</label>
            <label><input type="checkbox" v-model="focusTech" /> Tech</label>
            <label><input type="checkbox" v-model="focusHome" /> Home</label>
            <label><input type="checkbox" v-model="focusKitchen" /> Kitchen</label>
            <label><input type="checkbox" v-model="focusStreet" /> Street</label>
          </div>
        </div>
      </div>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar } from '@ionic/vue';
import { onMounted, onBeforeUnmount, ref, computed } from 'vue';
import * as tf from '@tensorflow/tfjs';
import * as cocoSsd from '@tensorflow-models/coco-ssd';

const videoRef = ref<HTMLVideoElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const imageRef = ref<HTMLImageElement | null>(null);
const uploadVideoRef = ref<HTMLVideoElement | null>(null);

const status = ref('Initializing camera...');
const fps = ref(0);
const target = ref<{ label: string; score: number } | null>(null);
const detectionsList = ref<Array<{ label: string; score: number }>>([]);
const totalObjects = ref(0);
const lastUpdatedAt = ref<number | null>(null);
const detectMs = ref(0);
const avgDetectMs = ref(0);
const modelLoadMs = ref(0);
const sourceSizeText = ref('—');
const videoStateText = ref('—');

const settingsOpen = ref(false);
const isRunning = ref(true);
const mode = ref<'camera' | 'image' | 'video'>('camera');
const videoReady = ref(false);
const showGrid = ref(false);
const showScanline = ref(false);
const highAccuracy = ref(true);
const detectionInterval = ref(60);
const scoreThreshold = ref(0.12);
const maxDetections = ref(20);
const inputSize = ref(640);
const focusOnly = ref(true);
const fastMode = ref(false);
const useCustomFocus = ref(false);
const customFocus = ref('');
const focusPeople = ref(true);
const focusAnimals = ref(true);
const focusTech = ref(true);
const focusHome = ref(true);
const focusKitchen = ref(true);
const focusStreet = ref(true);

const focusCategories = {
  people: ['person'],
  animals: ['dog', 'cat', 'bird', 'horse', 'sheep', 'cow', 'bear', 'zebra', 'giraffe'],
  tech: ['tv', 'laptop', 'keyboard', 'mouse', 'cell phone', 'monitor', 'remote'],
  home: ['chair', 'sofa', 'bed', 'dining table', 'potted plant', 'vase', 'clock', 'toilet', 'sink'],
  kitchen: [
    'bottle', 'cup', 'wine glass', 'fork', 'knife', 'spoon', 'bowl', 'refrigerator',
    'microwave', 'oven', 'toaster', 'banana', 'apple', 'orange', 'sandwich', 'pizza', 'donut', 'cake',
  ],
  street: [
    'backpack', 'handbag', 'umbrella', 'suitcase', 'bench', 'bicycle', 'motorcycle', 'car', 'bus', 'truck',
    'traffic light', 'stop sign', 'fire hydrant', 'parking meter',
  ],
  misc: ['book', 'scissors'],
} as const;

const focusSet = computed(() => {
  if (useCustomFocus.value) {
    const parts = customFocus.value
      .split(',')
      .map((s) => s.trim().toLowerCase())
      .filter(Boolean);
    return new Set(parts);
  }

  const items: string[] = [];
  if (focusPeople.value) items.push(...focusCategories.people);
  if (focusAnimals.value) items.push(...focusCategories.animals);
  if (focusTech.value) items.push(...focusCategories.tech);
  if (focusHome.value) items.push(...focusCategories.home);
  if (focusKitchen.value) items.push(...focusCategories.kitchen);
  if (focusStreet.value) items.push(...focusCategories.street);
  items.push(...focusCategories.misc);
  return new Set(items);
});

const shouldFilter = computed(() => {
  if (useCustomFocus.value) return true;
  if (!focusPeople.value || !focusAnimals.value || !focusTech.value || !focusHome.value || !focusKitchen.value || !focusStreet.value) {
    return true;
  }
  return focusOnly.value;
});

let model: cocoSsd.ObjectDetection | null = null;
let animationId = 0;
let lastFrameTime = performance.now();
let lastDetectTime = 0;
let isDetecting = false;
let stream: MediaStream | null = null;
let lastDetections: cocoSsd.DetectedObject[] = [];
let detectionScale = { x: 1, y: 1 };
let frameCount = 0;
let detectSamples = 0;

const inputCanvas = document.createElement('canvas');
const inputCtx = inputCanvas.getContext('2d');
const needsPlay = ref(false);
let videoSizePoll: number | null = null;

const targetLabel = computed(() => target.value?.label ?? '—');
const confidenceText = computed(() =>
  target.value ? `${Math.round(target.value.score * 100)}% confidence` : 'No target',
);

const detectionCounts = computed(() => {
  const map = new Map<string, number>();
  for (const det of detectionsList.value) {
    map.set(det.label, (map.get(det.label) ?? 0) + 1);
  }
  return Array.from(map.entries())
    .map(([label, count]) => ({ label, count }))
    .sort((a, b) => b.count - a.count);
});

const lastUpdatedText = computed(() => {
  if (!lastUpdatedAt.value) return '—';
  const seconds = Math.max(0, Math.round((performance.now() - lastUpdatedAt.value) / 1000));
  return `${seconds}s ago`;
});

async function setupCamera() {
  const video = videoRef.value;
  if (!video) return;

  stream = await navigator.mediaDevices.getUserMedia({
    video: {
      facingMode: 'environment',
      width: { ideal: 1280 },
      height: { ideal: 720 },
    },
    audio: false,
  });
  video.srcObject = stream;

  await new Promise<void>((resolve) => {
    if (video.readyState >= 2) resolve();
    video.onloadedmetadata = () => resolve();
  });

  await video.play();
}

async function loadModel() {
  status.value = 'Loading model...';
  const start = performance.now();
  try {
    await tf.setBackend('webgl');
  } catch {
    await tf.setBackend('cpu');
  }
  await tf.ready();
  model = await cocoSsd.load();
  modelLoadMs.value = Math.round(performance.now() - start);
  status.value = 'Model ready';
}

function updateCanvasSize() {
  const source = getActiveSource();
  const canvas = canvasRef.value;
  if (!source || !canvas) return;
  const { width, height } = getSourceSize(source);
  if (width && height) {
    canvas.width = width;
    canvas.height = height;
    return;
  }
  const display = getDisplaySize(source);
  if (display.width && display.height) {
    canvas.width = display.width;
    canvas.height = display.height;
  }
}

function drawDetections() {
  const canvas = canvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.lineWidth = 2;
  ctx.font = '16px "Courier New", monospace';

  for (const det of lastDetections) {
    const [x, y, w, h] = det.bbox;
    const sx = x * detectionScale.x;
    const sy = y * detectionScale.y;
    const sw = w * detectionScale.x;
    const sh = h * detectionScale.y;

    ctx.strokeStyle = '#00ff9a';
    ctx.fillStyle = 'rgba(0, 255, 154, 0.12)';
    ctx.strokeRect(sx, sy, sw, sh);
    ctx.fillRect(sx, sy, sw, sh);

    const label = `${det.class} ${Math.round(det.score * 100)}%`;
    ctx.fillStyle = '#00ff9a';
    ctx.fillText(label, sx + 4, sy + 16);
  }
}

function updateInputCanvas() {
  const source = getActiveSource();
  if (!source || !inputCtx) return;
  let { width, height } = getSourceSize(source);
  if (!width || !height) {
    const display = getDisplaySize(source);
    width = display.width;
    height = display.height;
  }
  if (!width || !height) return;

  const ratio = height / width;
  inputCanvas.width = inputSize.value;
  inputCanvas.height = Math.round(inputSize.value * ratio);
  try {
    inputCtx.drawImage(source, 0, 0, inputCanvas.width, inputCanvas.height);
  } catch {
    inputCanvas.width = 0;
    inputCanvas.height = 0;
  }
}

async function warmup() {
  if (!model || !videoRef.value) return;
  status.value = 'Warming up...';
  updateInputCanvas();
  try {
    await model.detect(inputCanvas);
  } catch {
    // Ignore warmup errors.
  }
  status.value = 'Ready';
}

function ensureVideoPlaying() {
  if (mode.value !== 'video') return;
  const video = uploadVideoRef.value;
  if (!video) return;
  if (video.readyState >= 2 && video.videoWidth > 0 && video.videoHeight > 0) {
    videoReady.value = true;
  }
  if (mode.value === 'video') {
    const rect = video.getBoundingClientRect();
    videoStateText.value = `rs:${video.readyState} t:${video.currentTime.toFixed(1)} vw:${video.videoWidth} vh:${video.videoHeight} rw:${Math.round(rect.width)} rh:${Math.round(rect.height)}`;
  }
  if (video.readyState < 2) return;
  if (video.paused) {
    video
      .play()
      .then(() => {
        needsPlay.value = false;
      })
      .catch(() => {
        needsPlay.value = true;
      });
  }
}

function waitForVideoDimensions(video: HTMLVideoElement) {
  if (videoSizePoll) {
    window.clearInterval(videoSizePoll);
    videoSizePoll = null;
  }
  videoSizePoll = window.setInterval(() => {
    if (video.videoWidth > 0 && video.videoHeight > 0) {
      videoReady.value = true;
      if (videoSizePoll) {
        window.clearInterval(videoSizePoll);
        videoSizePoll = null;
      }
    }
  }, 100);
}

async function detectFrame() {
  const source = getActiveSource();
  if (!model || !source || !isRunning.value) return;
  if (isDetecting) return;

  const now = performance.now();
  if (now - lastDetectTime < detectionInterval.value) return;

  isDetecting = true;
  lastDetectTime = now;
  frameCount += 1;

  try {
    let detections: cocoSsd.DetectedObject[] = [];

    const sourceSize = getSourceSize(source);
    const displaySize = getDisplaySize(source);
    if (!sourceSize.width || !sourceSize.height) {
      if (displaySize.width && displaySize.height) {
        sourceSizeText.value = `${displaySize.width}x${displaySize.height} (display)`;
      } else {
        status.value = 'Loading media...';
        return;
      }
    } else {
      sourceSizeText.value = `${sourceSize.width}x${sourceSize.height}`;
    }

    const t0 = performance.now();
    if (mode.value === 'video') {
      updateInputCanvas();
      if (!inputCanvas.width || !inputCanvas.height) {
        status.value = 'Waiting for video frame...';
        return;
      }
      detections = await model.detect(inputCanvas);
      const w = sourceSize.width || displaySize.width;
      const h = sourceSize.height || displaySize.height;
      detectionScale = {
        x: w / inputCanvas.width,
        y: h / inputCanvas.height,
      };
    } else if (!fastMode.value || (highAccuracy.value && frameCount % 5 === 0)) {
      if (sourceSize.width === 0 || sourceSize.height === 0) {
        status.value = 'Waiting for camera frame...';
        return;
      }
      detections = await model.detect(source);
      detectionScale = { x: 1, y: 1 };
    } else {
      updateInputCanvas();
      if (!inputCanvas.width || !inputCanvas.height) {
        status.value = 'Waiting for frame...';
        return;
      }
      detections = await model.detect(inputCanvas);
      detectionScale = {
        x: sourceSize.width / inputCanvas.width,
        y: sourceSize.height / inputCanvas.height,
      };
    }
    const t1 = performance.now();
    detectMs.value = Math.round(t1 - t0);
    detectSamples += 1;
    avgDetectMs.value = Math.round(((avgDetectMs.value * (detectSamples - 1)) + detectMs.value) / detectSamples);

    const filtered = detections
      .filter((d) => d.score >= scoreThreshold.value)
      .filter((d) => (shouldFilter.value ? focusSet.value.has(d.class) : true))
      .sort((a, b) => b.score - a.score)
      .slice(0, maxDetections.value);

    lastDetections = filtered;
    const top = filtered[0];
    target.value = top ? { label: top.class, score: top.score } : null;
    detectionsList.value = filtered.map((d) => ({
      label: d.class,
      score: d.score,
    }));
    totalObjects.value = filtered.length;
    lastUpdatedAt.value = performance.now();
    status.value = filtered.length ? 'Detecting...' : 'No objects';
  } catch (err) {
    status.value = 'Detection error';
    console.error(err);
  } finally {
    isDetecting = false;
  }
}

function loop() {
  const now = performance.now();
  const delta = now - lastFrameTime;
  fps.value = 1000 / delta;
  lastFrameTime = now;

  ensureVideoPlaying();
  updateCanvasSize();
  drawDetections();
  void detectFrame();

  animationId = requestAnimationFrame(loop);
}

function toggleRun() {
  isRunning.value = !isRunning.value;
  status.value = isRunning.value ? 'Ready' : 'Paused';
}

async function init() {
  try {
    await loadModel();
  } catch (err) {
    status.value = 'Model load error';
    console.error(err);
    return;
  }

  if (mode.value === 'camera') {
    try {
      await setupCamera();
      await warmup();
    } catch (err) {
      status.value = 'Camera unavailable';
      console.error(err);
    }
  } else {
    status.value = 'Ready';
  }

  loop();
}

function getActiveSource(): HTMLVideoElement | HTMLImageElement | null {
  if (mode.value === 'camera') return videoRef.value;
  if (mode.value === 'video') return uploadVideoRef.value;
  return imageRef.value;
}

function getSourceSize(source: HTMLVideoElement | HTMLImageElement) {
  if (source instanceof HTMLVideoElement) {
    return { width: source.videoWidth, height: source.videoHeight };
  }
  return { width: source.naturalWidth, height: source.naturalHeight };
}

function getDisplaySize(source: HTMLVideoElement | HTMLImageElement) {
  if (source instanceof HTMLVideoElement) {
    const rect = source.getBoundingClientRect();
    return { width: Math.round(rect.width), height: Math.round(rect.height) };
  }
  return { width: source.width, height: source.height };
}

function stopCamera() {
  if (stream) {
    for (const track of stream.getTracks()) {
      track.stop();
    }
    stream = null;
  }
}

function switchToCamera() {
  mode.value = 'camera';
  videoReady.value = false;
  needsPlay.value = false;
  void setupCamera();
  isRunning.value = true;
}

function onPhotoSelected(event: Event) {
  const input = event.target as HTMLInputElement;
  const file = input.files?.[0];
  if (!file || !imageRef.value) return;
  stopCamera();
  mode.value = 'image';
  isRunning.value = true;
  videoReady.value = false;
  lastDetections = [];
  detectionsList.value = [];
  target.value = null;
  totalObjects.value = 0;
  lastUpdatedAt.value = null;
  status.value = 'Loading image...';
  const url = URL.createObjectURL(file);
  imageRef.value.onload = () => {
    URL.revokeObjectURL(url);
    status.value = 'Ready';
  };
  imageRef.value.src = url;
}

function onVideoSelected(event: Event) {
  const input = event.target as HTMLInputElement;
  const file = input.files?.[0];
  if (!file || !uploadVideoRef.value) return;
  stopCamera();
  mode.value = 'video';
  isRunning.value = true;
  videoReady.value = false;
  lastDetections = [];
  detectionsList.value = [];
  target.value = null;
  totalObjects.value = 0;
  lastUpdatedAt.value = null;
  status.value = 'Loading video...';
  const url = URL.createObjectURL(file);
  uploadVideoRef.value.preload = 'auto';
  uploadVideoRef.value.muted = true;
  uploadVideoRef.value.loop = true;
  uploadVideoRef.value.playsInline = true;
  uploadVideoRef.value.controls = false;
  uploadVideoRef.value.onplaying = () => {
    videoReady.value = true;
    waitForVideoDimensions(uploadVideoRef.value!);
  };
  if ('requestVideoFrameCallback' in HTMLVideoElement.prototype) {
    const cb = () => {
      videoReady.value = true;
      uploadVideoRef.value?.requestVideoFrameCallback(cb);
    };
    uploadVideoRef.value.requestVideoFrameCallback(cb);
  }
  uploadVideoRef.value.onloadedmetadata = () => {
    uploadVideoRef.value!.currentTime = 0;
    videoReady.value = true;
    waitForVideoDimensions(uploadVideoRef.value!);
    uploadVideoRef.value
      ?.play()
      .then(() => {
        status.value = 'Ready';
        needsPlay.value = false;
      })
      .catch(() => {
        status.value = 'Tap to play';
        needsPlay.value = true;
      });
  };
  uploadVideoRef.value.oncanplay = () => {
    if (status.value !== 'Ready') status.value = 'Ready';
    videoReady.value = true;
    waitForVideoDimensions(uploadVideoRef.value!);
  };
  uploadVideoRef.value.onloadeddata = () => {
    videoReady.value = true;
    waitForVideoDimensions(uploadVideoRef.value!);
  };
  uploadVideoRef.value.onerror = () => {
    status.value = 'Video load error';
  };
  uploadVideoRef.value.src = url;
  uploadVideoRef.value.load();
}

function playVideo() {
  if (!uploadVideoRef.value) return;
  uploadVideoRef.value
    .play()
    .then(() => {
      needsPlay.value = false;
      status.value = 'Ready';
    })
    .catch(() => {
      needsPlay.value = true;
    });
}

onMounted(() => {
  void init();
});

onBeforeUnmount(() => {
  if (animationId) cancelAnimationFrame(animationId);
  if (stream) {
    for (const track of stream.getTracks()) {
      track.stop();
    }
  }
});
</script>

<style scoped>
.page {
  --background: #0b0f0e;
}

.stage {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
  background: radial-gradient(circle at top, #0e1b17 0%, #0b0f0e 55%, #040605 100%);
}

.video {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.overlay {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.grid {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background-image:
    linear-gradient(rgba(0, 255, 154, 0.08) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0, 255, 154, 0.08) 1px, transparent 1px);
  background-size: 40px 40px;
}

.scanline {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background: linear-gradient(
    to bottom,
    rgba(0, 255, 154, 0.0) 0%,
    rgba(0, 255, 154, 0.08) 50%,
    rgba(0, 255, 154, 0.0) 100%
  );
  animation: scan 4s linear infinite;
  mix-blend-mode: screen;
}

.reticle {
  display: none;
}

@keyframes scan {
  0% { transform: translateY(-100%); }
  100% { transform: translateY(100%); }
}

.hud {
  position: absolute;
  width: 200px;
  padding: 12px;
  border: 1px solid rgba(0, 255, 154, 0.6);
  background: rgba(6, 16, 14, 0.6);
  color: #00ff9a;
  font-family: "Courier New", monospace;
  text-transform: uppercase;
  box-shadow: 0 0 24px rgba(0, 255, 154, 0.15);
}

.hud-right {
  right: 16px;
  bottom: 16px;
}

.hud-left {
  left: 16px;
  bottom: 16px;
  color: #9affd8;
  border-color: rgba(0, 255, 154, 0.35);
}

.hud-title {
  font-size: 11px;
  letter-spacing: 1px;
  opacity: 0.75;
}

.hud-main {
  font-size: 16px;
  margin-top: 6px;
}

.hud-sub {
  font-size: 12px;
  opacity: 0.9;
  margin-top: 4px;
}

.hud-status,
.hud-fps,
.hud-mode,
.hud-meta {
  font-size: 11px;
  margin-top: 6px;
  opacity: 0.7;
}

.hud-row {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  padding: 2px 0;
}

.hud-label {
  max-width: 120px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.hud-score {
  opacity: 0.8;
}

.hud-empty {
  font-size: 11px;
  opacity: 0.6;
}

.hud-sep {
  height: 1px;
  margin: 6px 0;
  background: rgba(0, 255, 154, 0.2);
}

.controls {
  position: absolute;
  top: 14px;
  left: 14px;
  display: flex;
  gap: 8px;
}

.media-controls {
  position: absolute;
  top: 14px;
  right: 14px;
  display: flex;
  gap: 8px;
}

.file-btn {
  position: relative;
  overflow: hidden;
  cursor: pointer;
}

.file-btn input {
  position: absolute;
  inset: 0;
  opacity: 0;
  cursor: pointer;
}

.btn {
  background: #00ff9a;
  color: #02110b;
  border: none;
  padding: 8px 12px;
  font-weight: 700;
  font-family: "Courier New", monospace;
  text-transform: uppercase;
}

.btn.ghost {
  background: transparent;
  color: #00ff9a;
  border: 1px solid rgba(0, 255, 154, 0.5);
}

.settings {
  position: absolute;
  top: 56px;
  left: 14px;
  width: 260px;
  padding: 12px;
  background: rgba(6, 16, 14, 0.75);
  border: 1px solid rgba(0, 255, 154, 0.4);
  color: #b6ffe3;
  font-family: "Courier New", monospace;
  font-size: 12px;
  text-transform: uppercase;
  z-index: 2;
}

.settings-title {
  font-size: 12px;
  margin-bottom: 10px;
  letter-spacing: 1px;
}

.settings-row {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 10px;
}

.settings-row.toggle {
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
}

.settings-row input[type="range"] {
  width: 100%;
}

.settings-row input[type="text"] {
  width: 100%;
  background: rgba(0, 255, 154, 0.08);
  color: #b6ffe3;
  border: 1px solid rgba(0, 255, 154, 0.3);
  padding: 6px 8px;
  font-family: "Courier New", monospace;
}

.settings-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 6px 10px;
  margin: 6px 0 10px;
}

.settings-grid label {
  display: flex;
  gap: 6px;
  align-items: center;
}

.settings-hint {
  font-size: 10px;
  opacity: 0.7;
}

.play-btn {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0, 255, 154, 0.85);
  color: #02110b;
  border: none;
  padding: 10px 14px;
  font-weight: 700;
  font-family: "Courier New", monospace;
  text-transform: uppercase;
}
</style>
