<template>
  <div class="container">
    <video ref="video" controls autoplay></video>
    <div class="btn-container">
      <button @click="startRecording" :disabled="isRecording">Start Recording</button>
      <button @click="stopRecording" :disabled="!isRecording">Stop Recording</button>
      <button @click="playback" :disabled="!recordedBlobs.length">Playback</button>
    </div>
  </div>
</template>

<script>
export default {
  name: "VideoRecording",
  data() {
    return {
      isRecording: false,
      recordedBlobs: [],
      mediaRecorder: null,
    };
  },
  methods: {
    async init() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
        this.$refs.video.srcObject = stream;
        this.mediaRecorder = new MediaRecorder(stream, { mimeType: 'video/webm' });

        this.mediaRecorder.ondataavailable = (event) => {
          if (event.data && event.data.size > 0) {
            this.recordedBlobs.push(event.data);
          }
        };
      } catch (e) {
        console.error('Error accessing media devices.', e);
      }
    },
    startRecording() {
      this.recordedBlobs = [];
      this.mediaRecorder.start();
      this.isRecording = true;
    },
    stopRecording() {
      this.mediaRecorder.stop();
      this.isRecording = false;
    },
    playback() {
      const blob = new Blob(this.recordedBlobs, { type: 'video/webm' });
      this.$refs.video.src = URL.createObjectURL(blob);
      this.$refs.video.controls = true;
      this.$refs.video.play();
    },
  },
  mounted() {
    this.init();
  },
};
</script>

<style scoped>
  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
  }
  video {
    border: 1px solid black;
    width: 640px;
    height: 480px;
  }
  button {
    background-color: #007bff;
    color: white; 
    border: 1px solid #007bff; 
    padding: 10px 20px; 
    cursor: pointer; 
    border-radius: 5px; 
    font-size: 16px;
  }
  .btn-container {
    margin-top: 10px;
    margin-bottom: 10px;
    display: flex;
    gap: 12px;
  }
</style>