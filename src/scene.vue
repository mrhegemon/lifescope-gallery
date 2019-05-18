<template>
  <a-scene :networked-scene="'serverURL: https://nxr.lifescope.io; app: lifescope-xr; room: ls-room; audio: true; adapter: easyrtc; connectOnLoad: true;'">

    <!-- Load assets -->
    <a-assets class="aframe-assets">
      <img id="sky" src="https://s3.amazonaws.com/lifescope-static/static/xr/gallery/skybox/nightsky.jpg"
        crossorigin="anonymous">
      <img id="earth" src="https://s3.amazonaws.com/lifescope-static/static/xr/components/globe/Albedo.jpg"
        crossorigin="anonymous">
      <!-- video controls -->
      <img id="video-play" src="https://s3.amazonaws.com/lifescope-static/static/xr/gallery/video_play.png"
        crossorigin="anonymous">
      <img id="video-pause" src="https://s3.amazonaws.com/lifescope-static/static/xr/gallery/video_pause.png"
        crossorigin="anonymous">
      <!-- gltf -->
      <!-- logo -->
      <a-gltf-model id="logo" src="https://s3.amazonaws.com/lifescope-static/static/xr/logo/logo.gltf"
                    crossorigin="anonymous">
      </a-gltf-model>
    </a-assets>

    <!-- gallery -->
    <gallery/>

    <avatarcomp ref="avatar"/>

    <!-- Sky id="Sky" -->
    <a-sky v-if="skybox==SkyboxEnum.STARS"
      id="starsky" src="#sky" rotation="90 0 90">
    </a-sky>
    <a-sun-sky v-else-if="skybox==SkyboxEnum.SUN"
      id="sunsky" material="side: back" :sun-sky-position="'time: ' + skytime">
    </a-sun-sky>

  </a-scene>
</template>

<script>
import { mapState } from 'vuex';

import axios from 'axios';

import socketIO from 'socket.io-client';
import easyrtc from '../static/easyrtc/easyrtc.js';

import gallery from "./components/gallery.vue";

import Avatar from "./avatar.js";
import avatarcomp from "./avatar.vue";

import { SkyboxEnum } from './store/modules/xr/modules/graphics';

export default {
    components: {
        gallery,
        avatarcomp
    },
    data() {
      return {
        avatar: {},
        SkyboxEnum: SkyboxEnum
      }
    },

    computed: {
      ...mapState('xr',
        ['roomName']
      ),

      ...mapState('xr/graphics',
        [
          'skybox',
          'skytime'
        ]
      )
    },

    mounted () {
      if (CONFIG.DEBUG) {console.log("App.vue mounted");};
      var self = this;

      var scene = document.querySelector('a-scene');  
      if (scene.hasLoaded) {
        self.onSceneLoaded();
      } else {
        scene.addEventListener('loaded', self.onSceneLoaded);
      }

      document.body.addEventListener('enter-vr', function (evt) {
        self.onEnterVR();
        document.body.addEventListener('exit-vr', function (event) {
          self.onExitVR();
        })
      });


      document.body.addEventListener('connected', function (evt) {
        if (CONFIG.DEBUG) {console.log('connected event. clientId =', evt.detail.clientId);};
        if (CONFIG.DEBUG) {console.log('roomName: ' + self.roomName);};

      });

      // make eyes invisible to user when the avatar is created
      document.body.addEventListener('entityCreated', function (evt) {
        if (evt.detail.el.id === 'playerRig') {
          document.getElementsByClassName('player')[0].getElementsByClassName('face')[0].setAttribute('visible', 'false');
          document.getElementsByClassName('player')[0].getElementsByClassName('head')[0].setAttribute('visible', 'false');
        }
      });

      if (!self.$route.query.room){
          self.$route.query.room = 'ls-room';
      }
          
      var queryRoom = this.$route.query.room || 'ls-room';

      this.$store.dispatch('xr/setRoomName', queryRoom).then(() => {
        this.$store.dispatch('xr/getRoomConfig').then(() => {
          this.$store.dispatch('xr/getObjs').then(() => {

            if (AFRAME.utils.device.isMobile()) {
              self.setupMobile();
            } else {
              self.setupDesktop();
            }
          })
        });
      });
      
    },


    methods: {
      onSceneLoaded () {
        if (CONFIG.DEBUG) {console.log("onSceneLoaded");}
        var self = this;
        self.$store.commit('xr/SET_SCENELOADED');
        self.$store.commit('xr/SET_ISMOBILE');
      },

      onEnterVR () {
        var self = this;
        if (CONFIG.DEBUG) {console.log('entered vr');};
        this.$refs.avatar.createRightHandNetworked();
        var playerRig = document.querySelector('#playerRig');
        playerRig.setAttribute('position', 'y', 0.2);

        if (AFRAME.utils.device.isMobile()) {
              this.teardownMobile();
        }
      },
      onExitVR () {
        if (CONFIG.DEBUG) {console.log('exited vr');};
        var rightHand = document.getElementById('rightHandController');
        rightHand.parentElement.removeChild(rightHand);

        if (AFRAME.utils.device.isMobile()) {
          this.setupMobile();
        }
      },

      setupMobile () {
        if (CONFIG.DEBUG) {console.log("isMobile");};
        this.$refs.avatar.setupMobile();
      },

      teardownMobile () {
        if (CONFIG.DEBUG) {console.log("teardownMobile");};
        var playerRig = document.getElementById('playerRig');
        if (playerRig) {
          playerRig.removeAttribute('virtual-gamepad-controls');
        }
        var sceneEl = document.getElementsByTagName('a-scene')[0];
        sceneEl.removeAttribute('look-on-mobile');
      },

      setupDesktop () {
        if (CONFIG.DEBUG) {console.log("!isMobile");};
        this.$refs.avatar.setupDesktop();
      },
    }
  }
</script>

<style src="./scene.css"></style>
