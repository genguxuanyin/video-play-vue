<template>
  <div ref="container" style="width: 100%; height: 100%; margin: 0; border: 0; padding: 0;">
    <canvas v-if="suportWebGL" ref="canvas" style="width: 100%; height: 100%;"></canvas>
    <div v-else>
      <slot>
        Your browser does not seem to support
        <a
          href="http://khronos.org/webgl/wiki/Getting_a_WebGL_Implementation"
          style="color:#000"
        >WebGL</a>.
        <br>'
      </slot>
    </div>
  </div>
</template>

<script>
import {
  Object3D,
  Vector2,
  Vector3,
  Color,
  Scene,
  Raycaster,
  WebGLRenderer,
  PerspectiveCamera,
  OrthographicCamera,
  AmbientLight,
  PointLight,
  HemisphereLight,
  DirectionalLight
} from "three";
import { getSize, getCenter } from "./util";
import { Stats } from "./stats/stats";
import { MapControls } from "./controls/MapControls";

const suportWebGL = (() => {
  try {
    var canvas = document.createElement("canvas");
    return !!(
      window.WebGLRenderingContext &&
      (canvas.getContext("webgl") || canvas.getContext("experimental-webgl"))
    );
  } catch (e) {
    return false;
  }
})();

export default {
  props: {
    width: {
      type: Number
    },
    height: {
      type: Number
    },
    position: {
      type: Object,
      default() {
        return { x: 0, y: 0, z: 0 };
      }
    },
    rotation: {
      type: Object,
      default() {
        return { x: 0, y: 0, z: 0 };
      }
    },
    scale: {
      type: Object,
      default() {
        return { x: 1, y: 1, z: 1 };
      }
    },
    lights: {
      type: Array,
      default() {
        return [
          {
            type: "ambientlight",
            color: 0xffffff,
            intensity: 0.8
          }
        ];
      }
    },
    cameraPosition: {
      type: Object,
      default() {
        return {
          x: 0,
          y: 0,
          z: 100
        };
      }
    },
    cameraRotation: {
      type: Object
    },
    cameraUp: {
      type: Object
    },
    cameraLookAt: {
      type: Object
    },
    backgroundColor: {
      default: 0xffffff
    },
    backgroundAlpha: {
      type: Number,
      default: 1
    },
    controllable: {
      type: Boolean,
      default: true
    },
    crossOrigin: {
      default: "anonymous"
    }
  },
  data() {
    return {
      statsEnabled: true,
      suportWebGL,
      size: {
        width: this.width,
        height: this.height
      },
      raycaster: new Raycaster(),
      mouse: new Vector2(),
      camera: new OrthographicCamera(
        window.innerWidth / -2,
        window.innerWidth / 2,
        window.innerHeight / 2,
        window.innerHeight / -2,
        1,
        20000
      ),
      scene: new Scene(),
      wrapper: new Object3D(),
      renderer: null,
      controls: null,
      stats: null,
      allLights: [],
      clock: typeof performance === "undefined" ? Date : performance,
      reqId: null // requestAnimationFrame id
    };
  },
  computed: {
    hasListener() {
      // 判断是否有鼠标事件监听，用于减少不必要的拾取判断
      const events = this._events;
      const result = {};

      ["on-mousemove", "on-mouseup", "on-mousedown", "on-click"].forEach(
        name => {
          result[name] = !!events[name] && events[name].length > 0;
        }
      );

      return result;
    }
  },
  mounted() {
    if (this.width === undefined || this.height === undefined) {
      this.size = {
        width: window.innerWidth, //this.$el.offsetWidth
        height: window.innerHeight //this.$el.offsetHeight
      };
    }

    this.renderer = new WebGLRenderer({
      antialias: true,
      alpha: true,
      canvas: this.$refs.canvas
    });
    this.renderer.shadowMap.enabled = true;

    // STATS
    if (this.statsEnabled) {
      this.stats = new Stats();
      this.stats.domElement.style.position = "absolute";
      this.stats.domElement.style.bottom = "0px";
      this.stats.domElement.style.zIndex = 100;
      this.$refs.container.appendChild(this.stats.domElement);
    }
    this.scene.add(this.wrapper);
    this.update();

    this.$el.addEventListener("mousedown", this.onMouseDown, false);
    this.$el.addEventListener("mousemove", this.onMouseMove, false);
    this.$el.addEventListener("mouseup", this.onMouseUp, false);
    this.$el.addEventListener("click", this.onClick, false);

    window.addEventListener("resize", this.onResize, false);

    this.animate();
  },
  beforeDestroy() {
    cancelAnimationFrame(this.reqId);

    this.$el.removeEventListener("mousedown", this.onMouseDown, false);
    this.$el.removeEventListener("mousemove", this.onMouseMove, false);
    this.$el.removeEventListener("mouseup", this.onMouseUp, false);
    this.$el.removeEventListener("click", this.onClick, false);

    window.removeEventListener("resize", this.onResize, false);
  },
  watch: {
    rotation: {
      deep: true,
      handler(val) {
        if (!this.wrapper) return;
        this.wrapper.rotation.set(val.x, val.y, val.z);
      }
    },
    position: {
      deep: true,
      handler(val) {
        if (!this.wrapper) return;
        this.wrapper.position.set(val.x, val.y, val.z);
      }
    },
    scale: {
      deep: true,
      handler(val) {
        if (!this.wrapper) return;
        this.wrapper.scale.set(val.x, val.y, val.z);
      }
    },
    lights: {
      deep: true,
      handler(val, oldVal) {
        this.updateLights();
      }
    },
    size: {
      deep: true,
      handler(val) {
        this.updateCamera();
        this.updateRenderer();
      }
    },
    backgroundAlpha() {
      this.updateRenderer();
    },
    backgroundColor() {
      this.updateRenderer();
    }
  },
  methods: {
    init() {},
    onResize() {
      if (this.width === undefined || this.height === undefined) {
        this.$nextTick(() => {
          this.size = {
            width: window.innerWidth, //this.$el.offsetWidth,
            height: window.innerHeight //this.$el.offsetHeight
          };
        });
      }
    },
    onMouseDown(event) {
      if (!this.hasListener["on-mousedown"]) return;

      const intersected = this.pick(event.clientX, event.clientY);
      this.$emit("on-mousedown", intersected);
    },
    onMouseMove(event) {
      if (!this.hasListener["on-mousemove"]) return;

      const intersected = this.pick(event.clientX, event.clientY);
      this.$emit("on-mousemove", intersected);
    },
    onMouseUp(event) {
      if (!this.hasListener["on-mouseup"]) return;

      const intersected = this.pick(event.clientX, event.clientY);
      this.$emit("on-mouseup", intersected);
    },
    onClick(event) {
      if (!this.hasListener["on-click"]) return;

      const intersected = this.pick(event.clientX, event.clientY);
      this.$emit("on-click", intersected);
    },
    pick(x, y) {
      if (!this.wrapper) return;

      const rect = this.$el.getBoundingClientRect();

      x -= rect.left;
      y -= rect.top;

      this.mouse.x = (x / this.size.width) * 2 - 1;
      this.mouse.y = -(y / this.size.height) * 2 + 1;

      this.raycaster.setFromCamera(this.mouse, this.camera);

      const intersects = this.raycaster.intersectObject(this.wrapper, true);

      return (intersects && intersects.length) > 0 ? intersects[0] : null;
    },
    update() {
      this.updateRenderer();
      this.updateCamera();
      this.updateLights();
      this.updateControls();
    },
    updateWrapper() {
      const wrapper = this.wrapper;

      if (!wrapper) return;

      const position = this.position;
      const rotation = this.rotation;
      const scale = this.scale;

      wrapper.position.set(position.x, position.y, position.z);
      wrapper.rotation.set(rotation.x, rotation.y, rotation.z);
      wrapper.scale.set(scale.x, scale.y, scale.z);
    },
    updateRenderer() {
      const renderer = this.renderer;

      renderer.setSize(this.size.width, this.size.height);
      renderer.setPixelRatio(window.devicePixelRatio || 1);
      renderer.setClearColor(new Color(this.backgroundColor).getHex());
      renderer.setClearAlpha(this.backgroundAlpha);
    },
    updateCamera() {
      const camera = this.camera;
      this.cameraPosition &&
        camera.position.set(
          this.cameraPosition.x,
          this.cameraPosition.y,
          this.cameraPosition.z
        );
    },
    updateLights() {
      this.scene.remove.apply(this.scene, this.allLights);

      this.allLights = [];

      this.lights.forEach(item => {
        if (!item.type) return;

        const type = item.type.toLowerCase();

        let light = null;

        if (type === "ambient" || type === "ambientlight") {
          const color = item.color || 0x404040;
          const intensity = item.intensity || 1;

          light = new AmbientLight(color, intensity);
        } else if (type === "point" || type === "pointlight") {
          const color = item.color || 0xffffff;
          const intensity = item.intensity || 1;
          const distance = item.distance || 0;
          const decay = item.decay || 1;

          light = new PointLight(color, intensity, distance, decay);

          if (item.position) {
            light.position.copy(item.position);
          }
        } else if (type === "directional" || type === "directionallight") {
          const color = item.color || 0xffffff;
          const intensity = item.intensity || 1;

          light = new DirectionalLight(color, intensity);

          if (item.position) {
            light.position.copy(item.position);
          }

          if (item.target) {
            light.target.copy(item.target);
          }
        } else if (type === "hemisphere" || type === "hemispherelight") {
          const skyColor = item.skyColor || 0xffffff;
          const groundColor = item.groundColor || 0xffffff;
          const intensity = item.intensity || 1;

          light = new HemisphereLight(skyColor, groundColor, intensity);

          if (item.position) {
            light.position.copy(item.position);
          }
        }

        this.allLights.push(light);
        this.scene.add(light);
      });
    },
    updateControls() {
      if (this.controllable && this.controls) return;

      if (this.controllable) {
        if (this.controls) return;
        this.controls = new MapControls(this.camera, this.renderer.domElement);
        this.controls.enableDamping = true; // an animation loop is required when either damping or auto-rotation are enabled
        this.controls.screenSpacePanning = true;
        this.controls.enableRotate = false;
        this.controls.panSpeed = 0.4;
        this.controls.minDistance = 100;
        this.controls.maxDistance = 500;

        // this.controls.addEventListener("change", this.render.bind(this));
      } else {
        if (this.controls) {
          this.controls.dispose();
          this.controls = null;
        }
      }
    },
    addObject(object) {
      this.wrapper.add(object);

      this.updateCamera();
      this.updateWrapper();
    },
    animate() {
      this.reqId = requestAnimationFrame(this.animate);
      this.render();
    },
    render() {
      this.renderer.render(this.scene, this.camera);
      this.stats && this.statsEnabled && this.stats.update();
      this.controls && this.controllable && this.controls.update();
    }
  }
};
</script>
