<template>
  <div>
    <p>Upload a reference image:</p>
    <input type="file" @change="uploadReferenceImage" />

    <p>Upload an image to authenticate:</p>
    <input type="file" @change="uploadUserImage" />
    <button @click="authenticateFace" :disabled="!userImage || !referenceImageDescriptor">Authenticate with Face</button>
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
      referenceImageDescriptor: null
    };
  },
  async mounted() {
    await this.loadModels();
    this.loadedModels = true;
  },
  methods: {
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

  }
};
</script>
