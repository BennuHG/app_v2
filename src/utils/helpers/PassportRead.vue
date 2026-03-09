<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import { parse } from 'mrz';
import { createWorker } from 'tesseract.js';

const emit = defineEmits(['scanned', 'close']);

const video = ref(null);
const worker = ref(null);
const scannedMessage = ref('');
const results = ref([]);

let stream = null;
let intervalId = null;

function cropMRZ(canvas) {
  const width = canvas.width;
  const height = canvas.height;
  const mrzHeight = height * 0.3;

  const cropCanvas = document.createElement('canvas');
  cropCanvas.width = width;
  cropCanvas.height = mrzHeight;
  const ctx = cropCanvas.getContext('2d');
  ctx.filter = 'grayscale(100%) contrast(200%)';
  ctx.drawImage(canvas, 0, height - mrzHeight, width, mrzHeight, 0, 0, width, mrzHeight);

  return cropCanvas;
}

function normalizeMrzLine(line, targetLen = 44) {
  const charMap = { O: '0', I: '1', Z: '2', S: '5', G: '6', B: '8' };
  let s = line.replace(/\s/g, '<').replace(/[^A-Z0-9<]/g, (c) => charMap[c] ?? '<');
  if (s.length > targetLen) s = s.slice(0, targetLen);
  else if (s.length < targetLen) s = s.padEnd(targetLen, '<');
  return s;
}

async function captureFrame() {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');

  canvas.width = video.value.videoWidth;
  canvas.height = video.value.videoHeight;

  ctx.drawImage(video.value, 0, 0);

  return canvas;
}

async function analyzeMrz() {
  if (!video.value || !worker.value) return;
  if (video.value.videoWidth === 0 || video.value.videoHeight === 0) return;

  try {
    const frame = await captureFrame();
    const mrzImage = cropMRZ(frame);

    const {
      data: { text }
    } = await worker.value.recognize(mrzImage);

    const lines = text
      .split('\n')
      .map((l) => l.trim())
      .filter((l) => l.length >= 30)
      .map((l) => normalizeMrzLine(l, l.length > 36 ? 44 : 30));

    if (lines.length >= 2) {
      const parsed = parse(lines.slice(0, 2));
      results.value.push(parsed.documentNumber || JSON.stringify(parsed));
      emit('scanned', { ...parsed, code: parsed.documentNumber || JSON.stringify(parsed) });
    }
  } catch (_) {
    // parse() lanza si el formato no es válido; es esperado hasta que el OCR lea bien
  }
}

onMounted(async () => {
  try {
    stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: 'environment' }
    });

    video.value.srcObject = stream;
    await video.value.play();

    await new Promise((resolve) => {
      if (video.value.videoWidth > 0) return resolve();
      video.value.onloadedmetadata = resolve;
    });

    worker.value = await createWorker('eng');

    await worker.value.setParameters({
      tessedit_char_whitelist: 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789<'
    });

    intervalId = setInterval(analyzeMrz, 1500);
  } catch (error) {
    console.error('Scanner error:', error);
    scannedMessage.value = 'Error al iniciar el escáner';
  }
});

async function stopScanner() {
  if (intervalId) {
    clearInterval(intervalId);
    intervalId = null;
  }
  if (stream) {
    stream.getTracks().forEach((track) => track.stop());
    stream = null;
  }
  if (worker.value) {
    await worker.value.terminate();
    worker.value = null;
  }
  emit('close');
}

onBeforeUnmount(() => {
  if (intervalId) clearInterval(intervalId);
  stopScanner();
});
</script>

<template>
  <div class="scanner-container">
    <div class="scanner-video-wrapper">
      <video ref="video" class="scanner-video"></video>
      <div class="scan-line"></div>
    </div>

    <div v-if="scannedMessage" class="text-warning mt-2">
      {{ scannedMessage }}
    </div>

    <p v-if="results.length">Scanned: {{ results.join(', ') }}</p>

    <v-btn variant="outlined" color="primary" class="mt-2" @click="stopScanner"> Detener </v-btn>
  </div>
</template>

<style scoped>
.scanner-container {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.scanner-video-wrapper {
  position: relative;
  width: 100%;
  max-width: 400px;
  overflow: hidden;
  border-radius: 12px;
}

.scanner-video {
  width: 100%;
  display: block;
  border-radius: 12px;
}

.scan-line {
  position: absolute;
  left: 10%;
  width: 80%;
  height: 3px;
  background: red;
  animation: scan 2s linear infinite;
  pointer-events: none;
}

@keyframes scan {
  0% {
    top: 25%;
  }
  50% {
    top: 75%;
  }
  100% {
    top: 25%;
  }
}
</style>
