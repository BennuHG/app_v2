<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import { parse } from 'mrz';
import { createWorker } from 'tesseract.js';

const emit = defineEmits(['scanned', 'close']);

const video = ref(null);
const worker = ref(null);

let stream = null;

async function captureFrame() {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');

  canvas.width = video.value.videoWidth;
  canvas.height = video.value.videoHeight;

  ctx.drawImage(video.value, 0, 0);

  return canvas;
}

function cropMRZ(canvas) {
  const ctx = canvas.getContext('2d');
  ctx.filter = 'grayscale(100%) contrast(200%)';

  const width = canvas.width;
  const height = canvas.height;

  const mrzHeight = height * 0.3;

  const imageData = ctx.getImageData(0, height - mrzHeight, width, mrzHeight);

  const cropCanvas = document.createElement('canvas');
  cropCanvas.width = width;
  cropCanvas.height = mrzHeight;

  cropCanvas.getContext('2d').putImageData(imageData, 0, 0);

  return cropCanvas;
}

async function analyzeMrz() {
  const frame = await captureFrame();
  const mrzImage = cropMRZ(frame);

  const {
    data: { text }
  } = await worker.value.recognize(mrzImage);

  console.log('OCR:', text);

  const lines = text
    .split('\n')
    .map((l) => l.trim())
    .filter((l) => l.length > 30);

  if (lines.length >= 2) {
    const parsed = parse(lines.slice(0, 2));
    console.log('Parsed MRZ:', parsed);

    emit('scanned', parsed);
  }
}

onMounted(async () => {
  try {
    stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: 'environment' }
    });

    video.value.srcObject = stream;
    await video.value.play();

    worker.value = await createWorker('eng');

    await worker.value.setParameters({
      tessedit_char_whitelist: 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789<'
    });

    setInterval(analyzeMrz, 1500);
  } catch (error) {
    console.error('Scanner error:', error);
  }
});

async function stopScanner() {
  if (stream) {
    stream.getTracks().forEach((track) => track.stop());
  }

  if (worker.value) {
    await worker.value.terminate();
  }

  emit('close');
}

onBeforeUnmount(() => {
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
