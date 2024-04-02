<template>
  <div class="container">
    <video ref="video" controls autoplay></video>
    <div class="btn-container">
      <button @click="startRecording" :disabled="isRecording">Start Recording</button>
      <button @click="stopRecording" :disabled="!isRecording">Stop Recording</button>
      <input type="file" @change="playbackFromFile" accept="video/webm" style="display: none" ref="fileInput"/>
      <button @click="triggerFileInput" :disabled="isRecording">Playback</button>
    </div>
    <div v-if="isRecording" class="recording-indicator">Recording...</div>
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
      // Ensure the stream is available; if not, re-initialize it
      if (!this.$refs.video.srcObject) {
        this.init().then(() => {
          this._startRecording();
        });
      } else {
        this._startRecording();
      }
    },

    _startRecording() {
      this.recordedBlobs = [];
      this.mediaRecorder.start();
      this.isRecording = true;
    },
    stopRecording() {
      return new Promise((resolve, reject) => {
          // Handler to be called when the last blob of data is received
          this.mediaRecorder.onstop = resolve;
          this.mediaRecorder.onerror = (event) => reject(event.name);

          // Stop the media recorder
          this.mediaRecorder.stop();
      }).then(() => {
          this.isRecording = false;

          // Ensure there's recorded data
          if (this.recordedBlobs.length > 0) {
          const blob = new Blob(this.recordedBlobs, { type: 'video/webm' });
          const url = URL.createObjectURL(blob);
          const a = document.createElement('a');
          a.style.display = 'none';
          a.href = url;
          a.download = 'recorded-video.webm';
          document.body.appendChild(a);
          a.click();
          setTimeout(() => {
              document.body.removeChild(a);
              window.URL.revokeObjectURL(url);
          }, 100);
          } else {
          console.error('No recorded data available');
          }
      }).catch((error) => {
          console.error('Error stopping media recorder:', error);
      });
    },
    triggerFileInput() {
      this.$refs.fileInput.click();
    },
    playbackFromFile(event) {
      const file = event.target.files[0];
      if (file && file.type === 'video/webm') {
        const url = URL.createObjectURL(file);
        this.$refs.video.srcObject = null; // Important: Reset srcObject to null
        this.$refs.video.src = url;
        this.$refs.video.load(); // Reload the video element to apply the new source
        this.$refs.video.play();
      } else {
        alert('Please select a valid .webm video file.');
      }
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
.recording-indicator {
  color: red;
  margin-top: 10px;
}
</style>