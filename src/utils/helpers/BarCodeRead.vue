<script setup>
import { BrowserMultiFormatReader } from '@zxing/browser';
import { BarcodeFormat, DecodeHintType } from '@zxing/library';
import { ref, onMounted, onBeforeUnmount } from 'vue';

const emit = defineEmits(['scanned', 'close']);

const video = ref(null);
const scannedMessage = ref('');
const results = ref([]);

let codeReader = null;
let scannerControls = null;
let msgTimer = null;

let lastScanned = null;
let scanLock = false;

function showAlreadyScanned() {
  scannedMessage.value = 'Código ya escaneado';
  if (msgTimer) clearTimeout(msgTimer);
  msgTimer = setTimeout(() => {
    scannedMessage.value = '';
  }, 1500);
}

onMounted(async () => {
  try {
    const hints = new Map();
    hints.set(DecodeHintType.POSSIBLE_FORMATS, [BarcodeFormat.CODE_128, BarcodeFormat.EAN_13]);

    codeReader = new BrowserMultiFormatReader(hints);

    scannerControls = await codeReader.decodeFromConstraints(
      {
        video: {
          facingMode: 'environment',
          width: { ideal: 1280 },
          height: { ideal: 720 }
        }
      },
      video.value,
      (result) => {
        if (!result || scanLock) return;

        const code = result.getText();

        if (code === lastScanned) {
          showAlreadyScanned();
          return;
        }

        scanLock = true;
        lastScanned = code;

        results.value.push(code);
        emit('scanned', { code });

        setTimeout(() => {
          scanLock = false;
        }, 1000);
      }
    );
  } catch (error) {
    console.error('Scanner error:', error);
  }
});

function stopScanner() {
  try {
    scannerControls?.stop();
  } catch (e) {
    console.warn(e);
  } finally {
    scannerControls = null;
    emit('close');
  }
}

onBeforeUnmount(() => {
  if (scannerControls) scannerControls.stop();
  if (msgTimer) clearTimeout(msgTimer);
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
  top: 50%;
  transform: translateY(-50%);
  background: red;
  pointer-events: none;
}
</style>
