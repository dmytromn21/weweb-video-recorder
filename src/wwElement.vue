<template>
  <div class="container">
    <button @click="handleShowRecordModal">Record Video</button>
    <button @click="triggerFileUplaodInput" :disabled="isRecording || isUploading">Upload Video</button>
    <input type="file" @change="handleUploadFileInput" style="display: none" ref="fileUploadInput"/>
  </div>
  <div class="modal" v-if="showModal" @click.self="showModal = false">
    <transition name="fade">
      <div class="modal-content" @click.stop>
        <h3 style="text-align: left;">Record a Video</h3>
        <span class="close-modal" @click="showModal = false">&times;</span>
        <div class="camera-container">
          <div class="spinner" :hidden="!isUploading"></div>
          <div class="recording-spinner" :hidden="!isRecording"></div>
          <!-- <video :hidden="isUploaded" ref="camera" autoplay></video> -->
          <video ref="camera" autoplay></video>
          <!-- <div :hidden="!isUploaded">
            <mux-player
              class="video-player"
              ref="muxplayer"
              playback-id=""
              metadata-video-title="Test video title"
              metadata-viewer-user-id="user-id-007"
            ></mux-player>
          </div> -->
        </div>
        <div class="btn-container">
          <div style="display: flex; gap: 8px;">
            <button @click="handleRecording" :disabled="isUploading">{{ getButtonTitle() }}</button>
            <button @click="handleUploadRecordedVideo" :disabled="isRecording || isUploading || recordedBlobs.length == 0 || isUploaded">Keep this Video</button>
            <button @click="deleteRecordedVideo" :disabled="recordedBlobs.length == 0 || isUploaded || isUploading" class="delete-video-button">
              <svg width="24px" height="24px" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg" stroke="#ffffff"><g id="SVGRepo_bgCarrier" stroke-width="0"></g><g id="SVGRepo_tracerCarrier" stroke-linecap="round" stroke-linejoin="round"></g><g id="SVGRepo_iconCarrier"> <path d="M6 7V18C6 19.1046 6.89543 20 8 20H16C17.1046 20 18 19.1046 18 18V7M6 7H5M6 7H8M18 7H19M18 7H16M10 11V16M14 11V16M8 7V5C8 3.89543 8.89543 3 10 3H14C15.1046 3 16 3.89543 16 5V7M8 7H16" stroke="#ffffff" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"></path> </g></svg>
            </button>
          </div>
          <!-- <button @click="handleClickTestCam" :hidden="isTestingCam" :disabled="isRecording || isUploading">{{isTestingCam ? "Turn off Cam" : "Turn on Cam"}}</button> -->
          <!-- <input type="file" @change="playbackFromFile" accept="video/webm" style="display: none" ref="fileInput"/>
          <button @click="triggerFileInput" :disabled="isRecording">Playback</button> -->
        </div>
      </div>
    </transition>
  </div>
</template>
<script>
export default {
  name: "VideoRecording",
  props: {
    content: { type: Object, required: true },
  },
  data() {
    return {
      api_url: 'https://proxy.cors.sh/' + 'http://clownfish-app-9zwdy.ondigitalocean.app:8080',
      isUploading: false,
      isUploaded: false,
      isRecording: false,
      isTestingCam: false,
      recordedBlobs: [],
      mediaRecorder: null,
      showModal: false,
      isPlaying: false,
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
    async handleShowRecordModal() {
      this.showModal = true;
      this.recordedBlobs = [];
      this.isUploaded = false;
      this.isRecording = false;
      this.mediaRecorder = null;
      this.isPlaying = false;
      this.isUploading = false;
      this.testCamera();
    },
    handleRecording() {
      if(!this.isRecording && this.recordedBlobs.length == 0) {
        this.startRecording();
      }
      if(this.isRecording){
        this.stopRecording();
      }
      if(this.recordedBlobs.length > 0) {
        this.playRecording();
      }
    },
    startRecording() {
      if(this.isTestingCam) {
        this.pauseCamera();
      }
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
    async stopRecording() {
      return new Promise((resolve, reject) => {
          // Handler to be called when the last blob of data is received
          this.mediaRecorder.onstop = resolve;
          this.mediaRecorder.onerror = (event) => reject(event.name);
          // Stop the media recorder
          this.mediaRecorder.stop();
      }).then(() => {
          this.isRecording = false;
          this.$refs.camera.srcObject = null;
          const url = URL.createObjectURL(new Blob(this.recordedBlobs, { type: 'video/webm' }));
          this.$refs.camera.src = url;
          this.$refs.camera.load(); // Reload the video element to apply the new source
          this.$refs.camera.onended = () => {
            this.isPlaying = false;
          };
          this.$refs.camera.pause();
      }).catch((error) => {
          console.error('Error stopping media recorder:', error);
      });
    },
    playRecording() {
      if (this.recordedBlobs.length > 0) {
        // Check if the video is paused
        if (this.$refs.camera.paused) {
          this.$refs.camera.play().then(() => {
            this.isPlaying = true;
          }).catch(error => {
            console.error("Error attempting to play video:", error);
          });
        } else {
          this.$refs.camera.pause();
          this.isPlaying = false;
        }
      } else {
        alert('Please select a valid .webm video file.');
      }
    },
    deleteRecordedVideo() {
      // Clear the recorded blobs
      this.recordedBlobs = [];
      // Reset the video element
      this.$refs.camera.srcObject = null;
      this.$refs.camera.src = '';
      // Reset states as necessary
      this.isPlaying = false;
      this.isUploaded = false;

      this.testCamera();
      // Additional cleanup as needed
    },
    getButtonTitle() {
      if(!this.isRecording && this.recordedBlobs.length == 0) {
        return "Start Recording";
      }
      if(this.isRecording){
        return "Stop Recording";
      }
      if (this.recordedBlobs.length > 0 && !this.isPlaying) {
        return "Play";
      }
      if (this.recordedBlobs.length > 0 && this.isPlaying) {
        return "Pause";
      }
    },
    handleUploadRecordedVideo() {
      if (this.recordedBlobs.length > 0) {
        const blob = new Blob(this.recordedBlobs, { type: 'video/webm' });
        this.uploadVideoToMux(blob);
      } else {
        console.error('No recorded data available');
      }
    },
    // handleClickTestCam () {
    //   if(this.isTestingCam) {
    //     this.pauseCamera();
    //   } else {
    //     this.testCamera();
    //   }
    // },
    async testCamera() {
      try {
        this.isTestingCam = true;
        const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
        this.$refs.camera.srcObject = stream;
        this.mediaRecorder = new MediaRecorder(stream, { mimeType: 'video/webm' });
        this.mediaRecorder.ondataavailable = (event) => {
          if (event.data && event.data.size > 0) {
            this.recordedBlobs.push(event.data);
          }
        };
      } catch (e) {
        console.log("Can't turn on your camera. Please check your web camera.");
        this.isTestingCam = false;
      }
    },
    pauseCamera () {
      this.$refs.camera.srcObject = null;
      this.mediaRecorder = null;
      this.isTestingCam = false;
      this.$refs.camera.pause();
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
      // const uploadID = uploadConfig.id;
      try {
        await fetch(uploadURL, {
          method: 'PUT',
          body: blob,
          headers: {"content-type": blob.type}
        });
        // const uploadDataResponse = await fetch(`${this.api_url}/video-upload/get-playback-id?upload_id=${uploadID}`, {headers: {'x-cors-api-key': "temp_368b76b526936e794eb3e109cc7fb026"}})
        // const uploadData = await uploadDataResponse.json();
        // setTimeout(() => {
        //   this.isUploaded = true;
        //   this.$refs.muxplayer.setAttribute("playback-id", uploadData.playback_ids[0].id);
        //   this.isUploading = false;
        // }, 5000);
        this.isUploaded = true;
        this.isUploading = false;
        this.isPlaying = false;
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
  align-items: center;
  justify-content: center;
  gap: 8px;
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
  padding: 8px 0px;
  cursor: pointer;
  border-radius: 5px;
  font-size: 16px;
  width: 150px;
  display: flex;
  align-items: center;
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

/* Modal background */
.modal {
  position: fixed;
  z-index: 999;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

/* Modal content */
.modal-content {
  background: white;
  padding: 20px;
  border-radius: 5px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.26);
}

/* Fade transition for the modal */
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter, .fade-leave-to /* .fade-leave-active in <2.1.8 */ {
  opacity: 0;
}

.modal-content {
  position: relative;
  /* Other styles... */
  padding: 16px; /* Adjust padding to ensure content does not overlap with the close icon */
  /* padding-top: 48px; */
  padding-top: 0px;
}

.close-modal {
  position: absolute;
  top: 10px; /* Adjust as needed */
  right: 10px; /* Adjust as needed */
  cursor: pointer;
  font-size: 24px; /* Adjust size as needed */
  font-weight: bold;
}

/* Optional: Change color or add effects on hover */
.close-modal:hover {
  color: #f00; /* Example: Change color on hover */
}
.delete-video-button {
  background-color: #ff4d4f; /* Red color for delete action */
  color: white;
  border: none;
  cursor: pointer;
  border-radius: 4px;
  display: flex;
  width: 40px;
}
</style>