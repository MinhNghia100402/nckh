<template>
  <div class="home-container">
    <div class="main-element">
      <div class="main-container">
        <div class="main-content">
          <div class="header">
            <div class="process">
              <div class="icon-x">
                <!-- <IconX/> -->
              </div>
              <div class="progress-tool">
                <div class="progress-container">
                  <div class="progress-bar" :style="{ width: progress + '%', height: '100%', backgroundColor: 'green', position: 'absolute' }"></div>
                </div>
              </div>
            </div>
          </div>
          <p>READ THE SENTENCES ALOUD</p>
          <div class="content">
            <p>{{ currentSentence }}</p>
          </div>
          <div class="display-result">
            <div class="score-result">
              <div class="circle">
                <div class="circle-inner">
                  <span class="score">{{ score }}%</span>
                </div>
              </div>
            </div>
          </div>
          <audio ref="audioPlayer" controls style="display: none;"></audio>
          <div id="waveform">

          </div>
          <div class="icon-container">
            <div class="volume" @click="playRecordedAudio">
              <IconVolume />
            </div>
            <div class="icon-mic" :class="{ 'recording': isRecording }" @click="toggleRecording">
              <IconMic />
            </div>
            <div class="ear">
              <IconEar />
            </div>
          </div>
        </div>
      </div>
    </div>
    <div class="footer">
      <div class="divider"></div>
      <div class="button-container">
        <button @click="previousSentence">Quay lại</button>
        <button @click="nextSentence">Bỏ qua</button>
      </div>
    </div>
  </div>
</template>

<script>
import IconMic from '@/components/icons/IconMic.vue';
import IconEar from '@/components/icons/IconEar.vue';
import IconVolume from '@/components/icons/IconVolume.vue';
import axios from 'axios';
import WaveSurfer from 'wavesurfer.js';

export default {
  name: 'HomeView',
  components: {
    IconMic,
    IconEar,
    IconVolume,
  },
  data() {
    return {
      apiData: [],
      currentIndex: 0,
      progress: 0,
      isRecording: false,
      recordedAudio: null,
      mediaRecorder: null,
      recordedChunks: [],
      audioSaved: false,
      activeButton: null,
      score: 70,
      getalldata: false,
      length_data: 0,
      wavesurfer: null,
    };
  },
  computed: {
    currentSentence() {
      return this.apiData[this.currentIndex] || '';
    }
  },
  created() {
    this.fetchData();
  },
  mounted() {
    this.initWaveSurfer();
  },
  methods: {
    initWaveSurfer() {
      this.wavesurfer = WaveSurfer.create({
        container: '#waveform',
        waveColor: 'violet',
        progressColor: 'purple',
        cursorColor: 'red',
        cursorWidth: 2,
        height: 128,
        responsive: true,
        backend: 'WebAudio'
      });

      this.wavesurfer.on('ready', () => {
        console.log('WaveSurfer is ready');
      });

      this.wavesurfer.on('error', (e) => {
        console.error(e);
      });
    },
    setActive(button) {
      this.activeButton = button;
    },
    playRecordedAudio() {
      if (this.recordedChunks.length > 0) {
        let recordedBlob = new Blob(this.recordedChunks, { type: 'audio/wav' });
        const audioUrl = URL.createObjectURL(recordedBlob);
        this.$refs.audioPlayer.src = audioUrl;
        this.$refs.audioPlayer.play();
        this.wavesurfer.load(audioUrl);
      } else {
        console.error('Không có file âm thanh ghi được.');
      }
    },
    toggleRecording() {
      this.isRecording = !this.isRecording;
      if (this.isRecording) {
        this.startRecording();
      } else {
        this.stopRecording();
      }
    },
    startRecording() {
      navigator.mediaDevices.getUserMedia({ audio: true })
        .then(stream => {
          this.mediaRecorder = new MediaRecorder(stream);
          this.recordedChunks = [];
          this.mediaRecorder.ondataavailable = event => {
            this.recordedChunks.push(event.data);
          };
          this.mediaRecorder.start();
          document.querySelector('.icon-mic').style.border = "3px solid red";
        })
        .catch(error => {
          console.error('Lỗi truy cập microphone:', error);
        });
    },
    stopRecording() {
      if (this.mediaRecorder && this.mediaRecorder.state !== 'inactive') {
        this.mediaRecorder.stop();
        document.querySelector('.icon-mic').style.border = "1px solid black";
      }
    },
    async sendAudio() {
      if (this.recordedChunks.length > 0) {
        let recordedBlob = new Blob(this.recordedChunks, { type: 'audio/wav' });
        this.recordedAudio = recordedBlob;
        let formData = new FormData();
        formData.append('audio', recordedBlob);
        try {
          const response = await axios.post('/api/save_audio', formData, {
            headers: {
              'Content-Type': 'multipart/form-data'
            }
          });
          console.log('File âm thanh đã được gửi thành công.', response);
        } catch (error) {
          console.error('Lỗi khi gửi file âm thanh:', error);
        }
      } else {
        console.error('Không có file âm thanh ghi được.');
      }
    },
    async fetchData() {
      try {
        const response = await axios.get('/api/getall');
        this.apiData = response.data.data;
        console.log(this.apiData);
        this.length_data = this.apiData.length;
      } catch (error) {
        console.error('Lỗi khi lấy dữ liệu:', error);
      }
    },
    nextSentence() {
      if (this.currentIndex < this.apiData.length - 1) {
        this.currentIndex++;
        this.progress = (this.currentIndex / this.apiData.length) * 100;
      }
    },
    previousSentence() {
      if (this.currentIndex > 0) {
        this.currentIndex--;
        this.progress = (this.currentIndex / this.apiData.length) * 100;
      }
    },
  },
  onAudioTimeUpdate() {
  const currentTime = this.$refs.audioPlayer.currentTime;
  const duration = this.$refs.audioPlayer.duration;
  this.progress = (currentTime / duration) * 100;
},

};
</script>

<style scoped>
/* CSS cho giao diện */
@import url('./../assets/home.css');

#waveform {
  width: 100%;
  height: 128px; /* hoặc bất kỳ chiều cao nào bạn muốn */
  background-color: #f0f0f0; /* màu nền để dễ nhìn hơn */
}
</style>
