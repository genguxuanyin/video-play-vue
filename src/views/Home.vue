<template>
  <div class="home">
    <div class="screen" v-show="uninit">
      <div class="overlay" v-if="uninit">
        <button @click="init">Click to Play</button>
        <p>Automatic video playback with audio requires a user interaction.</p>
      </div>
      <video
        id="video"
        src="../assets/1.mp4"
        loop
        style="object-fit:fill;display:none"
        webkit-playsinline="true"
        x-webkit-airplay="true"
        playsinline="true"
        x5-video-player-type="h5"
        x5-video-orientation="h5"
        x5-video-player-fullscreen="true"
        preload="auto"
      ></video>
    </div>
    <screen ref="screen"></screen>
  </div>
</template>

<script>
import screen from "../components/three/screen";
export default {
  name: "Home",
  data() {
    return {
      uninit: true
    };
  },
  components: {
    screen
  },
  methods:{
    init(){
      this.uninit = false;
      this.$refs.screen.init();
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang='scss' scoped>
</style>
