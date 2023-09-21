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
  <p>{{ validResult ? "正常に撮影されました" : "免許証を枠内に正しく収めて撮影してください" }}</p>
  <canvas ref="canvasElement" width="640" height="480"></canvas>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";

const videoElement = ref<HTMLVideoElement | null>(null);
const canvasElement = ref<HTMLCanvasElement | null>(null);
const validResult = ref<boolean>(false);

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
      if (validateImage()) {
        validResult.value = true;
        console.log("Validation passed!");
      } else {
        validResult.value = false;
        console.log("Validation failed!");
      }
    }
  }
};

/**
 * バリデーション機能
 * - ガイドの枠内に免許証が正しく収められているか判定
 * - 現状は色味(白系統が占める割合が70%以上)で判定
 */
const validateImage = () => {
  if (!canvasElement.value) return false;

  const context = canvasElement.value.getContext("2d");
  if (!context) return false;

  const imageData = context.getImageData(0, 0, canvasElement.value.width, canvasElement.value.height);
  const data = imageData.data;

  let whitePixelCount = 0;
  const totalPixels = data.length / 4;

  for (let i = 0; i < data.length; i += 4) {
    const r = data[i];
    const g = data[i + 1];
    const b = data[i + 2];

    // TODO: 免許証が正しく枠内に収まったかどうかのバリデーションの仕方を探す => 「色味で判定はない方法」かつ「OpenCVなどの重い処理を行わず軽量に動くこと」を必要条件
    // 白色または白系統の色を検出 (一旦 RGB 値がそれぞれ 200 以上としているが部屋のライトの色味や光量によって左右されてしまう)
    if (r > 200 && g > 200 && b > 200) {
      whitePixelCount++;
    }
  }

  const whitePixelRatio = whitePixelCount / totalPixels;

  // 白色の割合が 70% 以上かどうかを確認
  return whitePixelRatio > 0.7;
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
