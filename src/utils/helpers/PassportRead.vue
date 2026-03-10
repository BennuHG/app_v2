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

function cropMRZ(canvas, useBottomPercent = 0.5, binThreshold = 140, targetHeight = 120) {
  const width = canvas.width;
  const height = canvas.height;
  const mrzHeight = Math.floor(height * useBottomPercent);

  const cropCanvas = document.createElement('canvas');
  cropCanvas.width = width;
  cropCanvas.height = mrzHeight;
  const ctx = cropCanvas.getContext('2d');
  ctx.filter = 'grayscale(100%) contrast(200%)';
  ctx.drawImage(canvas, 0, height - mrzHeight, width, mrzHeight, 0, 0, width, mrzHeight);

  const imageData = ctx.getImageData(0, 0, width, mrzHeight);
  const data = imageData.data;
  for (let i = 0; i < data.length; i += 4) {
    const gray = 0.299 * data[i] + 0.587 * data[i + 1] + 0.114 * data[i + 2];
    const v = gray < binThreshold ? 0 : 255;
    data[i] = data[i + 1] = data[i + 2] = v;
  }
  ctx.putImageData(imageData, 0, 0);

  if (mrzHeight > targetHeight) {
    const scale = targetHeight / mrzHeight;
    const w = Math.round(width * scale);
    const out = document.createElement('canvas');
    out.width = w;
    out.height = targetHeight;
    const outCtx = out.getContext('2d');
    outCtx.imageSmoothingEnabled = false;
    outCtx.drawImage(cropCanvas, 0, 0, w, targetHeight);
    return out;
  }

  return cropCanvas;
}

function normalizeMrzLine(line, targetLen) {
  let s = line.replace(/\s/g, '<').replace(/[^A-Z0-9<]/g, '<');
  if (s.length > targetLen) s = s.slice(0, targetLen);
  else if (s.length < targetLen) s = s.padEnd(targetLen, '<');
  return s;
}

function extractMrzLines(text) {
  const rawLines = text
    .split(/[\r\n]+/)
    .map((l) =>
      l
        .trim()
        .replace(/\s/g, '<')
        .replace(/[^A-Z0-9<]/g, '<')
    )
    .filter((l) => l.length >= 20);

  const lines = [];
  for (const l of rawLines) {
    if (l.length === 30) lines.push(normalizeMrzLine(l, 30));
    else if (l.length === 36) lines.push(normalizeMrzLine(l, 36));
    else if (l.length === 44) lines.push(normalizeMrzLine(l, 44));
    else if (l.length > 44) lines.push(normalizeMrzLine(l.slice(0, 44), 44));
    else if (l.length > 30) lines.push(normalizeMrzLine(l.slice(0, 30), 30));
  }
  return lines;
}

async function captureFrame() {
  const canvas = document.createElement('canvas');
  canvas.width = video.value.videoWidth;
  canvas.height = video.value.videoHeight;
  canvas.getContext('2d').drawImage(video.value, 0, 0);
  return canvas;
}

async function analyzeMrz() {
  if (!video.value || !worker.value) return;
  if (video.value.videoWidth === 0 || video.value.videoHeight === 0) return;

  try {
    const frame = await captureFrame();
    const mrzImage = cropMRZ(frame, 0.5);

    const {
      data: { text }
    } = await worker.value.recognize(mrzImage);

    if (import.meta.env.DEV && text?.trim()) {
      console.log('[PassportRead] OCR:', text);
    }

    const lines = extractMrzLines(text || '');
    if (lines.length < 2) return;

    let parsed = null;

    let linesToParse = [...lines];
    linesToParse = normalizeSex(linesToParse);

    if (linesToParse.length >= 3 && linesToParse[0].length === 30 && linesToParse[1].length === 30 && linesToParse[2].length === 30) {
      parsed = parse(linesToParse.slice(0, 3));
    } else if (linesToParse.length >= 2 && linesToParse[0].length === 44 && linesToParse[1].length === 44) {
      parsed = parse(linesToParse.slice(0, 2));
    } else if (linesToParse.length >= 2 && linesToParse[0].length === 36 && linesToParse[1].length === 36) {
      parsed = parse(linesToParse.slice(0, 2));
    } else if (linesToParse.length >= 2 && linesToParse[0].length === 30 && linesToParse[1].length === 30) {
      parsed = parse(linesToParse.slice(0, 2));
    }

    if (parsed) {
      const code = parsed.documentNumber || parsed.documentCode || JSON.stringify(parsed);
      results.value.push(code);
      console.log(parsed);
      parsed.details.forEach((d) => {
        if (d.valid === false) {
          console.log(d.field, d.error, d.ranges);
        }
      });
      emit('scanned', {
        documentNumber: parsed.documentNumber,
        format: parsed.format,
        lastName: parsed.fields?.lastName,
        firstName: parsed.fields?.firstName,
        birthDate: parsed.fields?.birthDate,
        expirationDate: parsed.fields?.expirationDate,
        nationality: parsed.fields?.nationality
      });
    }
  } catch (e) {
    if (import.meta.env.DEV) console.warn('[PassportRead] parse error:', e.message);
  }
}

function normalizeSex(lines) {
  if (lines.length >= 2 && lines[0].length === 30 && lines[1].length === 30) {
    const l1 = lines[1];
    if (l1[7] === 'H') {
      lines[1] = l1.slice(0, 7) + 'M' + l1.slice(8);
    }
  }
  return lines;
}

function acceptDocument(parsed) {
  if (!parsed || parsed.format !== 'TD1') return !!parsed?.valid;

  const errors = parsed.details.filter((d) => d.valid === false);
}

onMounted(async () => {
  try {
    const isMobile = /Android|iPhone|iPad|iPod/i.test(navigator.userAgent);
    stream = await navigator.mediaDevices.getUserMedia({
      video: isMobile ? { facingMode: 'environment' } : { facingMode: 'environment', width: { ideal: 1280 }, height: { ideal: 720 } }
    });

    video.value.srcObject = stream;
    video.value.muted = true;
    video.value.setAttribute('playsinline', '');
    video.value.setAttribute('webkit-playsinline', '');
    await video.value.play();

    await new Promise((resolve) => {
      if (video.value.videoWidth > 0) return resolve();
      video.value.onloadedmetadata = resolve;
    });

    worker.value = await createWorker('eng');
    await worker.value.setParameters({
      tessedit_char_whitelist: 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789<'
    });

    intervalId = setInterval(analyzeMrz, 1200);
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
      <video ref="video" class="scanner-video" playsinline muted autoplay></video>
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
