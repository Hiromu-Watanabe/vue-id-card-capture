<template>
  <div class="camera-wrapper">
    <video ref="videoElement" width="640" height="480" autoplay></video>
    <img
      src="../../assets/transparent_license.png"
      alt="免許証撮影時のガイドの役割を持った免許証イラスト画像"
      class="license-guide"
    />
  </div>
  <button
    @click="snapPhoto"
    :disabled="!validResult"
    :class="{ active_button: validResult, disable_button: !validResult }"
  >
    Snap Photo
  </button>
  <p>{{ validResult ? "" : "免許証を枠内に正しく収めて撮影してください" }}</p>
  <canvas ref="canvasElement" width="640" height="480"></canvas>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from "vue";

const videoElement = ref<HTMLVideoElement | null>(null);
const canvasElement = ref<HTMLCanvasElement | null>(null);
const validResult = ref<boolean>(false);
const licenseEdgeData = ref<ImageData | null>(null);
let validationInterval: number | null = null;

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

  // 撮影ガイド画像のエッジ取得
  await getLicenseEdgeData();

  // 500ミリ秒ごとにバリデーションを実行
  validationInterval = window.setInterval(() => {
    validResult.value = validateImage();
  }, 500);
});

onBeforeUnmount(() => {
  if (validationInterval !== null) {
    clearInterval(validationInterval);
  }
});

// 事前に免許証イラストからエッジデータを取得
const getLicenseEdgeData = async () => {
  const licenseImage = new Image();
  licenseImage.src = "src/assets/transparent_license.png";

  await new Promise<void>((resolve, reject) => {
    licenseImage.onload = function () {
      const canvas = document.createElement("canvas");
      canvas.width = licenseImage.width;
      canvas.height = licenseImage.height;
      const ctx = canvas.getContext("2d");
      if (ctx) {
        ctx.drawImage(licenseImage, 0, 0);
        const licenseImageData = ctx.getImageData(0, 0, licenseImage.width, licenseImage.height);
        if (licenseImageData) {
          licenseEdgeData.value = sobelFilter(licenseImageData);
        }
      }
      resolve();
    };

    licenseImage.onerror = function () {
      console.error("Failed to load the image.");
      reject(new Error("Failed to load the image."));
    };
  });
};

/** スクショ機能(ボタン押下時のvideoの状態を画像としてcanvasに描画) */
const snapPhoto = () => {
  console.log("ボタンイベント発火");
  if (videoElement.value && canvasElement.value) {
    const context = canvasElement.value.getContext("2d");
    if (context) {
      context.drawImage(videoElement.value, 0, 0, 640, 480);
    }
  }
};

/**
 * バリデーション機能
 * - ガイドの枠内に免許証が正しく収められているか判定
 */
const validateImage = () => {
  if (videoElement.value) {
    // 一時的なcanvasを作成してバリデーションを行う
    const tempCanvas = document.createElement("canvas");
    tempCanvas.width = 640;
    tempCanvas.height = 480;
    const context = tempCanvas.getContext("2d");
    if (context) {
      context.drawImage(videoElement.value, 0, 0, 640, 480);

      // ガイド画像の位置とサイズを取得
      const guideWidth = tempCanvas.width * 0.7; // 70% of canvas width
      const guideHeight = (guideWidth * 480) / 640; // Maintain aspect ratio
      const guideX = (tempCanvas.width - guideWidth) / 2;
      const guideY = (tempCanvas.height - guideHeight) / 2;

      // ガイドの位置とサイズをもとに、canvasから該当する部分の画像データを取得
      const imageData = context.getImageData(guideX, guideY, guideWidth, guideHeight);
      const edges = sobelFilter(imageData);

      // // エッジの数をカウント
      // const edgeCount = Array.from(edges.data).filter((value, index) => index % 4 === 0 && value > 128).length;

      if (edges && licenseEdgeData.value) {
        // 免許証のエッジデータと比較
        const similarity = calculateSimilarity(edges, licenseEdgeData.value);
        console.log(similarity);
        // エッジの形状が90%以上類似していればtrueを返す
        return similarity > 0.9;
      }
    }
  }
  return false;
};

/**
 * ソーベルフィルタ
 * 画像内の色の変化が急な部分 (エッジ) を検出
 */
const sobelFilter = (imageData: ImageData): ImageData => {
  const width = imageData.width;
  const height = imageData.height;
  const output = new ImageData(width, height);
  const sobelX = [
    [-1, 0, 1],
    [-2, 0, 2],
    [-1, 0, 1],
  ];
  const sobelY = [
    [-1, -2, -1],
    [0, 0, 0],
    [1, 2, 1],
  ];

  for (let y = 1; y < height - 1; y++) {
    for (let x = 1; x < width - 1; x++) {
      let gx = 0;
      let gy = 0;
      for (let ky = -1; ky <= 1; ky++) {
        for (let kx = -1; kx <= 1; kx++) {
          const index = ((y + ky) * width + (x + kx)) * 4;
          // Convert to grayscale using the average method
          const pixel = imageData.data[index] + imageData.data[index + 1] + imageData.data[index + 2];
          gx += pixel * sobelX[ky + 1][kx + 1];
          gy += pixel * sobelY[ky + 1][kx + 1];
        }
      }
      const magnitude = Math.sqrt(gx * gx + gy * gy);
      const index = (y * width + x) * 4;
      output.data[index] = magnitude;
      output.data[index + 1] = magnitude;
      output.data[index + 2] = magnitude;
      output.data[index + 3] = 255;
    }
  }

  return output;
};

/**
 * Calculate the cosine similarity between two vectors.
 * @param {Array} vecA - The first vector.
 * @param {Array} vecB - The second vector.
 * @returns {number} The cosine similarity between vecA and vecB.
 */
const cosineSimilarity = (vecA: Array<number>, vecB: Array<number>) => {
  const dotProduct = vecA.reduce((sum, a, idx) => sum + a * vecB[idx], 0);
  const magnitudeA = Math.sqrt(vecA.reduce((sum, a) => sum + a * a, 0));
  const magnitudeB = Math.sqrt(vecB.reduce((sum, b) => sum + b * b, 0));
  console.log(dotProduct / (magnitudeA * magnitudeB));
  return dotProduct / (magnitudeA * magnitudeB);
};

/**
 * Calculate the similarity between two edge data.
 * @param {ImageData} edgesA - The first edge data.
 * @param {ImageData} edgesB - The second edge data (license edge data).
 * @returns {number} The similarity between edgesA and edgesB.
 */
const calculateSimilarity = (edgesA: ImageData, edgesB: ImageData): number => {
  const vecA = Array.from(edgesA.data).filter((_, idx) => idx % 4 === 0);
  const vecB = Array.from(edgesB.data).filter((_, idx) => idx % 4 === 0);
  return cosineSimilarity(vecA, vecB);
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
.active_button {
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
.disable_button {
  background-color: #808080;
  color: #fff;
  display: block;
  margin: auto;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button {
  background-color: #808080;
  color: #fff;
  display: block;
  margin: auto;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s;
}

/* button:hover {
  background-color: #0056b3;
} */

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
