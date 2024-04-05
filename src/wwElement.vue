<template>
  <div class="container">
    <div class="camera-container">
      <div class="spinner" :hidden="!isUploading"></div>
      <div class="recording-spinner" :hidden="!isRecording"></div>
      <video :hidden="isUploaded" ref="camera" autoplay></video>
      <div :hidden="!isUploaded">
        <mux-player
          class="video-player"
          ref="muxplayer"
          playback-id=""
          metadata-video-title="Test video title"
          metadata-viewer-user-id="user-id-007"
        ></mux-player>
      </div>
    </div>
    <div class="btn-container">
      <button @click="startRecording" :disabled="isRecording || isUploading">Start Recording</button>
      <button @click="stopRecording" :disabled="!isRecording || isUploading">Stop Recording</button>
      <button @click="testCamera" :hidden="isTestingCam" :disabled="isRecording || isUploading">Test Camera</button>
      <button @click="pauseCamera" :hidden="!isTestingCam" :disabled="isRecording || isUploading">Pause</button>
      <!-- <input type="file" @change="playbackFromFile" accept="video/webm" style="display: none" ref="fileInput"/>
      <button @click="triggerFileInput" :disabled="isRecording">Playback</button> -->
      <input type="file" @change="handleUploadFileInput" style="display: none" ref="fileUploadInput"/>
      <button @click="triggerFileUplaodInput" :disabled="isRecording || isUploading">Upload Video</button>
    </div>
  </div>
</template>
<script>
export default {
  name: "VideoRecording",
  data() {
    return {
      api_url: 'https://proxy.cors.sh/' + 'http://clownfish-app-9zwdy.ondigitalocean.app:8080',
      isUploading: false,
      isUploaded: false,
      isRecording: false,
      isTestingCam: false,
      recordedBlobs: [],
      mediaRecorder: null,
    };
  },
  methods: {
    async init() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
        this.$refs.camera.srcObject = stream;
        this.mediaRecorder = new MediaRecorder(stream, { mimeType: 'video/webm' });
        this.mediaRecorder.ondataavailable = (event) => {
          if (event.data && event.data.size > 0) {
            this.recordedBlobs.push(event.data);
          }
        };
      } catch (e) {
        alert('Error accessing media devices. Please check your camera again.');
        return;
      }
    },
    startRecording() {
      // Ensure the stream is available; if not, re-initialize it
      if (!this.$refs.camera.srcObject) {
        this.init().then(() => {
          this._startRecording();
        });
      } else {
        this._startRecording();
      }
    },

    _startRecording() {
      if(this.$refs.camera.srcObject) {
        this.recordedBlobs = [];
        this.mediaRecorder.start();
        this.isRecording = true;
      }
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
          this.$refs.camera.srcObject = null;
          // Ensure there's recorded data
          if (this.recordedBlobs.length > 0) {
          const blob = new Blob(this.recordedBlobs, { type: 'video/webm' });
          this.uploadVideoToMux(blob);
          } else {
          console.error('No recorded data available');
          }
      }).catch((error) => {
          console.error('Error stopping media recorder:', error);
      });
    },
    async testCamera() {
      try {
        this.isTestingCam = true;
        const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
        this.$refs.camera.srcObject = stream;
      } catch (e) {
        this.isTestingCam = false;
        alert("Camera test faild. Please check your web camera.");
      }
    },
    pauseCamera () {
      this.$refs.camera.srcObject = null;
      this.isTestingCam = false;
    },
    triggerFileInput() {
      this.$refs.fileInput.click();
    },
    triggerFileUplaodInput() {
      this.$refs.fileUploadInput.click();
    },
    playbackFromFile(event) {
      const file = event.target.files[0];
      if (file && file.type === 'video/webm') {
        const url = URL.createObjectURL(file);
        this.$refs.camera.srcObject = null; // Important: Reset srcObject to null
        this.$refs.camera.src = url;
        this.$refs.camera.load(); // Reload the video element to apply the new source
        this.$refs.camera.play();
      } else {
        alert('Please select a valid .webm video file.');
      }
    },
    handleUploadFileInput(event) {
      const file = event.target.files[0];
      this.uploadVideoToMux(file);
    },
    async uploadVideoToMux(blob) {
      this.isUploading = true;
      const uploadConfigResponse = await fetch(`${this.api_url}/video-upload/get-upload-url`, {headers: {'x-cors-api-key': "temp_368b76b526936e794eb3e109cc7fb026"}});
      const uploadConfig = await uploadConfigResponse.json();
      const uploadURL = uploadConfig.url;
      const uploadID = uploadConfig.id;
      try {
        await fetch(uploadURL, {
          method: 'PUT',
          body: blob,
          headers: { "content-type": blob.type}
        });
        const uploadDataResponse = await fetch(`${this.api_url}/video-upload/get-playback-id?upload_id=${uploadID}`, {headers: {'x-cors-api-key': "temp_368b76b526936e794eb3e109cc7fb026"}})
        const uploadData = await uploadDataResponse.json();
        setTimeout(() => {
          this.isUploaded = true;
          this.$refs.muxplayer.setAttribute("playback-id", uploadData.playback_ids[0].id);
          this.isUploading = false;
        }, 5000);
      } catch(error) {
        console.error(error);
      }
    }
  },
  mounted() {
    if (document.getElementById('mux-player')) return; // was already loaded
    var scriptTag = document.createElement("script");
    scriptTag.src = "https://cdn.jsdelivr.net/npm/@mux/mux-player";
    scriptTag.id = "mux-player";
    document.getElementsByTagName('head')[0].appendChild(scriptTag);
  },
};
</script>

<style scoped>
.spinner {
  top: 48%;
  left: 48%;
  position: absolute;
  border: 4px solid rgba(0, 0, 0, 0.1);
  width: 40px;
  height: 40px;
  border-radius: 50%;
  border-left-color: #09f;
  animation: spin 1s ease infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
}
.camera-container {
  background-color: black;
  position: relative;
  width: 640px;
  height: 480px;
  border: 1px solid black;
}
video {
  width: 640px;
  height: 480px;
}
.video-player {
  width: 640px;
  height: 480px;
}
button {
  background-color: #007bff;
  color: white;
  font-weight: 550;
  border: 1px solid #007bff;
  padding: 10px 0px;
  cursor: pointer;
  border-radius: 5px;
  font-size: 16px;
  width: 150px;
  display: flex;
  justify-content: center;
}

button:disabled {
  background-color: grey;
  border: 1px solid grey;
}
.btn-container {
  width: 640px;
  margin-top: 10px;
  margin-bottom: 10px;
  display: flex;
  justify-content: space-between;
}
.recording-spinner {
  position: absolute;
  top: 10px; /* Adjust as needed */
  right: 10px; /* Adjust as needed */
  width: 15px; /* Size of the spinner */
  height: 15px; /* Size of the spinner */
  border-radius: 50%;
  background-color: red;
  box-shadow: 0 0 0 0 rgba(255, 0, 0, 1);
  transform: scale(1);
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0% {
    transform: scale(0.95);
    box-shadow: 0 0 0 0 rgba(255, 0, 0, 0.7);
  }
  
  70% {
    transform: scale(1);
    box-shadow: 0 0 0 10px rgba(255, 0, 0, 0);
  }
  
  100% {
    transform: scale(0.95);
    box-shadow: 0 0 0 0 rgba(255, 0, 0, 0);
  }
}
</style>