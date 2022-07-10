<template>

  <main>
    
    <a-space style="width: 100%;" direction="vertical" align="center" size="large">

      <h1>Instagram Story Music</h1>

      <div v-if="step === 0">
        <a-button @click="connectSpotify" type="primary">Connect Spotify</a-button>
      </div>

      <div v-if="step === 1">
        <a-space direction="vertical" size="large">
          <a-upload
            list-type="picture-card"
            accept="image/*"
            :show-upload-list="false"
            :before-upload="beforeUpload"
          >

            <img v-if="imageUrl" :src="imageUrl" />
            <div v-else>
              <PlusOutlined></PlusOutlined>
              <div class="ant-upload-text">Upload</div>
            </div>

          </a-upload>
          <a-button type="primary" @click="advance">Continue</a-button>
        </a-space>
      </div>

      <div v-if="step === 2">
        <a-space direction="vertical" size="large">

          <a-input @pressEnter="requestTracks" style="width: 500px;" v-model:value="spotifySearch" placeholder="Search Spotify">
            <template #prefix>
              <SearchOutlined />
            </template>
          </a-input>
          <div>
            <p v-if="isSearch">Search results for "{{ spotifySearch }}":</p>
            <p v-else>Your recently played tracks:</p>
            <a-spin :spinning="loadingTracks">
              <div class="track-container">
                <div @click="selectTrack(track)" class="track" v-for="track, i in spotifyTracks" :key="i">
                  <img :src="track.album.images[0].url" />
                  <div>
                    <span>{{ track.name }}</span>
                    <span>{{ track.artists.map(e => e.name).join(', ') }}</span>
                  </div>
                </div>
              </div>
            </a-spin>
          </div>
        </a-space>
      </div>

      <div v-if="step === 3">
        <a-input-number v-model:value="settings.length" :min="5" :max="15" addon-after="seconds" />
        <a-slider 
          v-model:value="settings.start"
          :max="Math.floor(selectedTrack.duration_ms / 1000)"
          :tip-formatter="val => `${String(Math.floor(val / 60)).padStart(2, '0')}:${String(val%60).padStart(2, '0')}`" />
        <a-button type="primary" @click="advance">Continue</a-button>
      </div>

      <a-steps :current="step">
        <a-step title="Connect Spotify" />
        <a-step title="Select Image" />
        <a-step title="Select Song" />
        <a-step title="Song Settings" />
        <a-step title="Done" />
      </a-steps>

    </a-space>

  </main>

  <div class="overlay" v-if="loading">
    <h4 style="color: white;" v-if="recording">RECORDING... PLEASE STAY ON THIS TAB.</h4>
    <a-spin size="large" />
  </div>

  <a-modal v-model:visible="alert" title="Before you continue" @ok="confirmStart">
    <p>When pressing 'OK', the selected song will start playing in your browser, and permission to record your screen will be demanded. Please accept, and make sure to enable audio recording.</p>
    <p>After the selected time, the recording will stop and your story will be generated. But while the recording is going, please stay on this page.</p>
    <p>Ready to go?</p>
  </a-modal>

</template>

<style scoped>
main {
  max-height: 90%;
  width: 70%;
  margin: 0 auto;
  position: absolute;
  top: 50%;
  left: 50%;
  padding: 30px;
  transform: translate(-50%, -50%);
  box-shadow: rgba(149, 157, 165, 0.2) 0px 8px 24px;
}
main h1 {
  text-align: center;
}
.overlay {
  position: fixed;
  top: 0; left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  font-size: 60px;
  background: rgba(0,0,0,0.5);
  z-index: 1001;
  text-align: center;
}
.track-container {
  max-height: 450px;
  overflow-y: scroll;
  display: flex;
  width: 500px;
  flex-direction: column;
  gap: 10px;
  padding: 10px;
}
.track {
  display: flex;
  height: 100px;
  width: 100%;
  padding: 10px;
  border-radius: 5px;
  box-shadow: rgba(149, 157, 165, 0.2) 0px 8px 24px;
  gap: 12px;
  cursor: pointer;
  transition: background 0.1s ease;
}
.track:hover {
  background: rgba(149, 157, 165, 0.2);
}
.track > img {
  height: 100%;
}
.track > div {
  display: flex;
  flex-direction: column;
  text-align: left;
  justify-content: center;
}
.track > div > span:nth-child(2) {
  color: grey;
}
</style>

<style>
@import 'ant-design-vue/dist/antd.css';
.ant-space-item {
  width: 100%;
  text-align: center;
}
.ant-upload > img {
  max-width: 100%;
  max-height: 100%;
}
</style>

<script>
import { PlusOutlined, SearchOutlined, LoadingOutlined } from '@ant-design/icons-vue';
import { message } from 'ant-design-vue';
import convertVideoToAudio from './video-to-audio.js';

const redirect = 'http://localhost:3000/';

export default {
  data() {
    return {
      step: 0,
      token: '',
      imageUrl: '',
      storyDuration: 15,
      device_id: '',
      player: null,
      spotifySearch: '',
      isSearch: false,
      spotifyTracks: [],
      recording: false,
      settings: {
        start: 0,
        length: 15
      },
      loadingTracks: false,
      loading: false,
      selectedTrack: null,
      alert: false
    }
  },
  watch: {
    async spotifySearch(newVal) {
      if (!newVal) { // DISPLAY RECENTLY PLAYED
        this.loadRecentlyPlayed();
      }
    },
    recording(newVal) {
      this.loading = newVal;
    }
  },
  methods: {
    connectSpotify() {
      const scopes = 'user-read-recently-played user-modify-playback-state streaming'.split(' ').join('+');
      const state = Math.floor(Math.random() * 0xffffff).toString(16);
      sessionStorage.setItem('spotify_state', state);
      location.assign(`
        https://accounts.spotify.com/authorize
          ?response_type=code
          &client_id=b408c3ec79a741928415fff8bc557a4c
          &redirect_uri=${redirect}
          &scope=${scopes}`.replace(/\s/g, ''));
          // &state=${state}
    },
    beforeUpload(file) {
      this.imageUrl = URL.createObjectURL(file);
      return false;
    },
    async advance() {
      if (this.step === 1) { // IMAGE UPLOAD
        if (this.imageUrl === '') return message.error('No image selected');
        await this.loadRecentlyPlayed();
        this.step++;
        return;
      }

      if (this.step === 2) { // SONG SELECTION
        this.step++;
        return;
      }

      if (this.step === 3) { // SONG SETTINGS
        if (!this.settings.length || this.settings.length < 5 || this.settings.length > 15) return message.error('Invalid length');
        if (this.settings.start + this.settings.length > this.selectedTrack.duration_ms / 1000) return message.error('Invalid start');

        this.alert = true;

        return;
      }
    },
    selectTrack(track) {
      this.selectedTrack = track;
      this.advance();
    },
    async requestTracks() {
      this.loadingTracks = true;
      this.isSearch = true;
      const data = await fetch(`https://api.spotify.com/v1/search?q=${this.spotifySearch}&type=track`, {
        headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.token}`
        }
      });
      this.spotifyTracks = (await data.json()).tracks.items;
      this.loadingTracks = false;
    },
    async loadRecentlyPlayed() {
      this.loadingTracks = true;
      this.isSearch = false;
      const data = await fetch('https://api.spotify.com/v1/me/player/recently-played', {
        headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.token}`
        }
      });
      this.spotifyTracks = (await data.json()).items.map(e => e.track);
      this.loadingTracks = false;
    },
    confirmStart() {
      this.alert = false;
      this.step++;
      this.createStory();
    },
    async createStory() {
      this.createStoryClientSide();
    },
    async createStoryClientSide() {
      const track = this.selectedTrack;
      const { start, length } = this.settings;

      const stream = await navigator.mediaDevices.getDisplayMedia({
        video: true, audio: true
      });

      const recorder = new MediaRecorder(stream, {
        mimeType: MediaRecorder.isTypeSupported("video/webm; codecs=vp9") 
             ? "video/webm; codecs=vp9" 
             : "video/webm"
      });

      const chunks = [];
      recorder.addEventListener('dataavailable', ({ data }) => {
        chunks.push(data);
      });

      recorder.addEventListener('stop', async function() {
        console.log(chunks[0].type);
        const blob = new Blob(chunks, {
          type: chunks[0].type
        });
        const url = URL.createObjectURL(blob);
        console.log('[VIDEO]', url);

        const output = await convertVideoToAudio(new File([blob], 'video'), 'mp3');
        console.log('[AUDIO]', output.data);

        
      });



      await fetch(`https://api.spotify.com/v1/me/player/play?device_id=${this.device_id}`, {
        method: 'PUT',
        body: JSON.stringify({
          uris: [track.uri],
          position_ms: start * 1000
        }),
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.token}`
        },
      });

      this.recording = true;
      recorder.start();

      await this.delay(length * 1000);

      const tracks = stream.getTracks();
      for (const track of tracks) {
        track.stop();
      }
      recorder.stop();

      await fetch(`https://api.spotify.com/v1/me/player/pause?device_id=${this.device_id}`, {
        method: 'PUT',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.token}`
        },
      });

      this.recording = false;
    },
    delay(ms = 0) {
      return new Promise(resolve => setTimeout(resolve, ms));
    }
  },
  async mounted() {

    window.onSpotifyWebPlaybackSDKReady = async _ => {
      // const token = (location.hash.match(/(?<=#access_token=).+?(?=&)/) ?? [])[0];
      // const urlState = (location.hash.match(/(?<=state=).+?(?=&|$)/) ?? [])[0];
      // const savedState = sessionStorage.getItem('spotify_state');

      // if (token && urlState !== savedState) window.location.assign('/');
      
      const code = new URL(location).searchParams.get('code');
      console.log('[CODE]', code);

      if (!code) return;
      this.loading = true;

      if (!this.token) {
        const response = await fetch(`https://accounts.spotify.com/api/token?grant_type=authorization_code&code=${code}&redirect_uri=${redirect}`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
            'Authorization': `Basic YjQwOGMzZWM3OWE3NDE5Mjg0MTVmZmY4YmM1NTdhNGM6MTM1ZjcxMmExMDA4NGY5MGI3YmM4OGIwY2FiNTI0Y2Y=`
          }
        });
  
        const data = await response.json();
        const token = data.access_token;
        
        console.log('[TOKEN]', token);
        if (!token) location.assign('/');
  
        this.token = token;
      }

      this.player = new Spotify.Player({
        name: 'Instagram Story Music',
        getOAuthToken: async cb => {
          cb(this.token);
        },
        volume: 1
      });

      this.player.addListener('ready', ({ device_id }) => {
        console.log('[DEVICE ID]', device_id);

        this.device_id = device_id;
        this.step = 1;
        this.loading = false;
      });

      // Not Ready
      this.player.addListener('not_ready', ({ device_id }) => {
        message.info('Device ID has gone offline ' + device_id);
      });

      this.player.addListener('initialization_error', err => {
        message.error('initialization_error');
        console.error(err);
      });

      this.player.addListener('authentication_error', err => {
        message.error('authentication_error');
        console.error(err);
      });

      this.player.addListener('account_error', err => {
        message.error('account_error ');
        console.error(err);
      });

      this.player.connect();
    }

  },
  components: {
    PlusOutlined, SearchOutlined, LoadingOutlined
  }
}
</script>