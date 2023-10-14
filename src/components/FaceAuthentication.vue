<template>
  <div>
    <p>Capture an image from the camera:</p>
    <video v-if="!userImage" ref="video" width="720" height="560" autoplay></video>
    <img v-if="userImage" :src="capturedImageSrc" alt="Captured Image" width="720" height="560" />
    <button @click="authenticateFace" :disabled="!userImage">Authenticate with Face</button>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, onBeforeUnmount } from 'vue';
import * as faceapi from 'face-api.js';
import image1 from '/src/assets/images/1.png';
import image2 from '/src/assets/images/2.png';

export default defineComponent({
  name: 'FaceAuthentication',
  setup() {
    const loadedModels = ref<boolean>(false);
    const userImage = ref<HTMLImageElement | null>(null);
    const referenceImageDescriptor = ref<Float32Array | null>(null);
    const capturedImageSrc = ref<string | null>(null);
    const datasetDescriptors = ref<Float32Array[]>([]);

    const video = ref<HTMLVideoElement | null>(null);

    const loadModels = async () => {
      await faceapi.nets.ssdMobilenetv1.loadFromUri('/models');
      await faceapi.nets.faceLandmark68Net.loadFromUri('/models');
      await faceapi.nets.faceRecognitionNet.loadFromUri('/models');
    };

    const prepareDataset = async (datasetUrls: string[]) => {
      for (let imageUrl of datasetUrls) {
        const response = await fetch(imageUrl);
        const imageBlob = await response.blob();
        const image = await faceapi.bufferToImage(imageBlob);
        const detections = await faceapi.detectAllFaces(image).withFaceLandmarks().withFaceDescriptors();
        if (detections.length > 0) {
          datasetDescriptors.value.push(detections[0].descriptor);
        }
      }
    };

    const startCamera = async () => {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: {} });
        if (video.value) {
          video.value.srcObject = stream;
        }
        setTimeout(captureImageFromCamera, 2000);
      } catch (error) {
        console.error("Error accessing the camera:", error);
      }
    };

    const captureImageFromCamera = async () => {
      if (video.value) {
        const canvas = document.createElement('canvas');
        canvas.width = video.value.videoWidth;
        canvas.height = video.value.videoHeight;
        canvas.getContext('2d')?.drawImage(video.value, 0, 0);

        canvas.toBlob(async (blob) => {
          if (blob) {
            userImage.value = await faceapi.bufferToImage(blob);
            capturedImageSrc.value = URL.createObjectURL(blob);
          }
        }, 'image/png');
      }
    };

    const authenticateFace = async () => {
      if (!userImage.value) {
        console.error("Please capture the face first.");
        return;
      }

      const detections = await faceapi.detectAllFaces(userImage.value).withFaceLandmarks().withFaceDescriptors();
      if (detections.length === 0) {
        console.error("No faces detected in the user image.");
        return;
      }

      const labeledFaceDescriptors = new faceapi.LabeledFaceDescriptors('knownFaces', datasetDescriptors.value);
      const faceMatcher = new faceapi.FaceMatcher(labeledFaceDescriptors, 0.6);
      const match = faceMatcher.findBestMatch(detections[0].descriptor);

      if (match.label === "knownFaces") {
        alert("Authenticated successfully!");
      } else {
        alert("Face not recognized.");
      }
    };

    onMounted(async () => {
      await loadModels();
      await prepareDataset([image1, image2]);
      console.log("Dataset prepared.");
      await startCamera();
    });

    onBeforeUnmount(() => {
      const videoEl = video.value;
      if (videoEl && videoEl.srcObject) {
        videoEl.srcObject.getTracks().forEach(track => track.stop());
      }
    });

    return {
      userImage,
      capturedImageSrc,
      authenticateFace
    };
  }
});
</script>

<style>
@media (max-width: 767px) {
  video, img {
    display: block;
    margin: 0 auto;
    width: 90%;
  }
}
</style>
