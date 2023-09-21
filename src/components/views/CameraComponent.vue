<template>
  <div class="camera-wrapper">
    <video ref="videoElement" width="640" height="480" autoplay></video>
    <img
      src="../../assets/transparent_license.png"
      alt="免許証撮影時のガイドの役割を持った免許証イラスト画像"
      class="license-guide"
    />
  </div>
  <button @click="snapPhoto">Snap Photo</button>
  <canvas ref="canvasElement" width="640" height="480"></canvas>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";

const videoElement = ref<HTMLVideoElement | null>(null);
const canvasElement = ref<HTMLCanvasElement | null>(null);

/**
 * 1.カメラ映像を取得
 * 2.Videoタグに取得したカメラ映像(MediaStream)を埋め込む
 * 3.メディアの再生を開始
 * ※ play()メソッド: https://developer.mozilla.org/ja/docs/Web/API/HTMLMediaElement/play
 */
onMounted(async () => {
  if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ video: true });
      if (videoElement.value) {
        videoElement.value.srcObject = stream;
        await videoElement.value.play();
      }
    } catch (err) {
      console.error("Error accessing the camera", err);
    }
  }
});

/** スクショ機能(ボタン押下時のvideoの状態を画像としてcanvasに描画) */
const snapPhoto = () => {
  if (videoElement.value && canvasElement.value) {
    const context = canvasElement.value.getContext("2d");
    if (context) {
      context.drawImage(videoElement.value, 0, 0, 640, 480);
    }
  }
};
</script>

<style scoped>
/* Camera wrapper styling */
.camera-wrapper {
  position: relative;
  width: 100%;
  max-width: 640px;
  margin-bottom: 20px;
}

/* Video styling */
video {
  width: 100%;
  max-width: 640px;
  border: 1px solid #ccc;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

/* License guide styling */
.license-guide {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 70%;
  max-width: 640px;
  z-index: 1; /* Ensure the guide is above the video */
}

/* Button styling */
button {
  background-color: #007bff;
  color: #fff;
  display: block;
  margin: auto;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #0056b3;
}

/* Canvas styling */
canvas {
  display: block;
  width: 100%;
  max-width: 640px;
  border: 1px solid #ccc;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  margin-top: 20px;
}
</style>
