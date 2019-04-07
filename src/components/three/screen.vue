<script>
import mixin from "./model-mixin.vue";
import * as THREE from "three";

export default {
  name: "screen",
  mixins: [mixin],
  props: {
    screenWidth: {
      type: Number
    },
    screenHeight: {
      type: Number
    }
  },
  methods: {
    init() {
      this.uninit = false;
      var video = document.getElementById("video");
      // enableInlineVideo(video);
      video.play();
      var texture = new THREE.VideoTexture(video);
      //
      var movieMaterial = new THREE.MeshBasicMaterial({
        map: texture,
        side: THREE.DoubleSide
      });
      // the geometry on which the movie will be displayed;
      // movie image will be scaled to fit these dimensions.
      var movieGeometry = new THREE.PlaneGeometry(2560, 442, 4, 4);
      var movieScreen = new THREE.Mesh(movieGeometry, movieMaterial);
      //   movieScreen.position.set(0, 50, 0);
      this.wrapper.add(movieScreen);
    },
    initClick() {
      this.init();
    }
  }
};
</script>
