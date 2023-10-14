<template>
  <div>
    <p>Upload a reference image:</p>
    <input type="file" @change="uploadReferenceImage" />

    <div v-if="referenceImageDescriptor">
      <p>Capture an image from the camera:</p>
      <video v-if="!userImage" ref="video" width="720" height="560" autoplay></video>
      <img v-if="userImage" :src="capturedImageSrc" alt="Captured Image" width="720" height="560" />
      <button @click="authenticateFace" :disabled="!userImage">Authenticate with Face</button>
    </div>
  </div>
</template>

<script>
import * as faceapi from 'face-api.js';
import { LabeledFaceDescriptors } from 'face-api.js';

export default {
  data() {
    return {
      loadedModels: false,
      userImage: null,
      referenceImageDescriptor: null,
      capturedImageSrc: null
    };
  },
  async mounted() {
    await this.loadModels();
    this.loadedModels = true;
  },
  methods: {
    async startCamera() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: {} });
        this.$refs.video.srcObject = stream;

        // Запуск таймера для автоматической съемки через 2 секунды
        setTimeout(this.captureImageFromCamera, 2000);
      } catch (error) {
        console.error("Error accessing the camera:", error);
      }
    },

    beforeDestroy() {
      const video = this.$refs.video;
      if (video && video.srcObject) {
        video.srcObject.getTracks().forEach(track => track.stop());
      }
    },

    async captureImageFromCamera() {
      const canvas = document.createElement('canvas');
      canvas.width = this.$refs.video.videoWidth;
      canvas.height = this.$refs.video.videoHeight;
      canvas.getContext('2d').drawImage(this.$refs.video, 0, 0);

      canvas.toBlob(async (blob) => {
        this.userImage = await faceapi.bufferToImage(blob);
        this.capturedImageSrc = URL.createObjectURL(blob);
      }, 'image/png');
    },

    async loadModels() {
      await faceapi.nets.ssdMobilenetv1.loadFromUri('/models');
      await faceapi.nets.faceLandmark68Net.loadFromUri('/models');
      await faceapi.nets.faceRecognitionNet.loadFromUri('/models');
    },

    async uploadReferenceImage(event) {
      if (this.loadedModels && event.target.files.length > 0) {
        const image = await faceapi.bufferToImage(event.target.files[0]);
        const detections = await faceapi.detectAllFaces(image).withFaceLandmarks().withFaceDescriptors();
        if (detections.length > 0) {
          this.referenceImageDescriptor = detections[0].descriptor;
        } else {
          console.error("No faces detected in the reference image.");
        }
      }
    },

    async uploadUserImage(event) {
      if (this.loadedModels && event.target.files.length > 0) {
        this.userImage = await faceapi.bufferToImage(event.target.files[0]);
      }
    },

    async authenticateFace() {
      if (!this.userImage || !this.referenceImageDescriptor) {
        console.error("Please upload both images.");
        return;
      }

      const detections = await faceapi.detectAllFaces(this.userImage).withFaceLandmarks().withFaceDescriptors();

      if (detections.length === 0) {
        console.error("No faces detected in the user image.");
        return;
      }

      const labeledFaceDescriptors = new faceapi.LabeledFaceDescriptors('reference', [this.referenceImageDescriptor]);
      const faceMatcher = new faceapi.FaceMatcher(labeledFaceDescriptors, 0.6);
      const match = faceMatcher.findBestMatch(detections[0].descriptor);

      if (match.label === "reference") {
        alert("Authenticated successfully!");
      } else {
        alert("Face not recognized.");
      }
    }
  },
  watch: {
    referenceImageDescriptor() {
      if (this.referenceImageDescriptor) {
        this.startCamera();
      }
    }
  }
};
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