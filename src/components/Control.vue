<template>
  <div class="control" @click="playOrPause">{{menu == 1?'暂停':'播放'}}</div>
</template>

<script>
export default {
  name: "Control",
  data() {
    return {
      menu: 1
    };
  },
  methods: {
    playOrPause() {
      var video = document.getElementById("video");
      this.menu == 1 ? video.pause() : video.play();
      this.menu = +!this.menu;
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang='scss' scoped>
.control {
  position: fixed;
  bottom: 1rem /* 20/20 */;
  left: 50%;
  margin-left: -0.5rem /* 10/20 */;
  background-color: #00f;
  color: #fff;
  border-radius: 50%;
  width: 1rem /* 20/20 */;
  height: 1rem /* 20/20 */;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>