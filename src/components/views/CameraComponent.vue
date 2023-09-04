<template>
  <div>
    <video ref="videoElement" width="640" height="480" autoplay></video>
    <button @click="snapPhoto">Snap Photo</button>
  </div>
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
/* Video styling */
video {
  width: 100%; /* Responsive width */
  max-width: 640px; /* Maximum width */
  border: 1px solid #ccc; /* Border around the video */
  border-radius: 8px; /* Rounded corners */
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1); /* Shadow for depth */
  margin-bottom: 20px; /* Space below the video element */
}

/* Button styling */
button {
  background-color: #007bff; /* Primary color */
  color: #fff; /* Text color */
  display: block;
  margin: auto;
  border: none; /* Remove default border */
  border-radius: 4px; /* Rounded corners */
  font-size: 16px; /* Text size */
  cursor: pointer; /* Hand cursor on hover */
  transition: background-color 0.3s; /* Smooth transition for hover effect */
}

button:hover {
  background-color: #0056b3; /* Darker shade for hover effect */
}

/* Canvas styling */
canvas {
  display: block; /* Block display to fit the container */
  width: 100%; /* Responsive width */
  max-width: 640px; /* Maximum width */
  border: 1px solid #ccc; /* Border around the canvas */
  border-radius: 8px; /* Rounded corners */
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1); /* Shadow for depth */
  margin-top: 20px; /* Space above the canvas element */
}
</style>
