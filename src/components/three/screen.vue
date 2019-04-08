<script>
import mixin from "./model-mixin.vue";
import * as THREE from "three";

export default {
  name: "screen",
  data(){
    return {
      movieScreen:null
    }
  },
  mixins: [mixin],
  props: {
    screenWidth: {
      type: Number,
      default:2560
    },
    screenHeight: {
      type: Number,
      default:375
    }
  },
  methods: {
    init() {
      this.movieScreen && this.wrapper.remove(this.movieScreen);
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
      var movieGeometry = new THREE.PlaneGeometry(this.screenWidth, this.screenHeight, 1, 1);
      var movieScreen = new THREE.Mesh(movieGeometry, movieMaterial);
      //   movieScreen.position.set(0, 50, 0);
      this.wrapper.add(movieScreen);
    }
  },
  watch:{
    screenHeight(){
      this.init();
      if(this.controls && this.controllable){
        this.controls.screenWidth = this.screenWidth;
        this.controls.screenHeight = this.screenHeight;
      }
    },
    screenWidth(){
      this.init();
      if(this.controls && this.controllable){
        this.controls.screenWidth = this.screenWidth;
        this.controls.screenHeight = this.screenHeight;
      }
    }
  }
};
</script>
