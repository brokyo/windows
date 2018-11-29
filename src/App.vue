<template>
  <div>
    <div v-if="!start" id="intro">
      <p>Random number drone. Drag visualization to change shape.</p>
      <p>Oscillators: {{oscillators}}</p>
      <input v-model="oscillators" type="range" min="2" max="20">
      <p>Base Frequency: {{baseFrequency}}</p>
      <input v-model="baseFrequency" type="range" min="40" max="600">
      <button @click="startDrone">Start</button>
    </div>
    <div v-show="start" id="viz">
      <button id="fullscreenButton" @click="fullScreen">Go Full Screen</button>
      <div id="canvasContainer">
        <canvas id="pixels"></canvas>
      </div>
    </div>
    <logo id="logo"></logo>
  </div>
</template>

<script>
import logo from './logo.vue'

const Tone = require('tone');
const _ = require('underscore');
const StartAudioContext = require('startaudiocontext');
// EUGH I did not name this and it's literally the only thing that does what I need :/
const nipplejs = require('nipplejs');

export default {
  name: 'Windows',
 components: {
      logo
  },
  data() {
    return {
      canvasConfig: {
        h: 255,
        s: 11,
        l: 30,
        rw: 100,
        rh: 10,
        percent: 0.0001,
        interval: 250,
      },
      startLoc: {
        x: 0,
        y: 0,
      },
      baseFrequency: 440,
      oscillators: 8,
      start: false,
    };
  },
  computed: {
    isMobile() {
      return (typeof window.orientation !== 'undefined') || (navigator.userAgent.indexOf('IEMobile') !== -1);
    },
  },
  methods: {
    fullScreen() {
      const canvas = document.getElementById('canvasContainer');
      if (canvas.RequestFullScreen) {
        canvas.RequestFullScreen();
      } else if (canvas.webkitRequestFullScreen) {
        canvas.webkitRequestFullScreen();
      } else if (canvas.mozRequestFullScreen) {
        canvas.mozRequestFullScreen();
      } else if (canvas.msRequestFullscreen) {
        canvas.msRequestFullscreen();
      } else {
        console.log("This browser doesn't supporter fullscreen");
      }
    },
    startDrone() {
      this.start = true;
      this.createDrone();
      this.createCanvas();
      this.createControls();
    },
    createCanvas() {
      const canvas = document.getElementById('pixels');
      const ctx = canvas.getContext('2d');

      // const w = window.screen.width * window.devicePixelRatio;
      // const h = window.screen.height * window.devicePixelRatio;
      // const pixels = Math.round((w * h) / 2);


      window.setInterval(() => {
        // for (var i = 0; i < pixels * this.canvasConfig.percent; i++) {
        for (let i = 0; i < _.random(10, 70); i++) {
          const h = this.canvasConfig.h + Math.random() * 3;
          const s = this.canvasConfig.s + Math.random() * 3;
          const cw = this.canvasConfig.rw * _.random(1, 1.1);
          const ch = this.canvasConfig.rh * _.random(1, 1.1);
          ctx.fillStyle = `hsl(${h},${s}%,${this.canvasConfig.l}%)`;
          ctx.fillRect(_.random(-40, 600), _.random(-40, 600), cw, ch);
        }
      }, this.canvasConfig.interval);
    },
    createDrone() {
      // Gain is the output node but also determines how loud
      // TODO: Maybe loop this or something
      const droneOut = new Tone.Gain(0.25);
      droneOut.toMaster();

      // Creates an output chain for each oscillator, generates random playback, & pans audio [final two update constantly]
      function createOsc(freq) {
        // Create panner
        const panner = new Tone.Panner3D();
        const max = 20;
        const min = -20;
        let x = _.random(min, max);
        let y = _.random(min, max);
        let z = _.random(min, max);
        panner.setPosition(x, y, z);
        panner.connect(droneOut);

        const filter = new Tone.Filter({ frequency: freq, type: 'lowpass', Q: 50 });
        filter.connect(panner);

        const noiseSource = new Tone.Noise('pink');
        noiseSource.start();
        noiseSource.connect(filter);

        // Update panner position ever 500ms
        setInterval(() => {
          x += _.random(-0.1, 0.1);
          y += _.random(-0.1, 0.1);
          z += _.random(-0.1, 0.1);
          panner.setPosition(x, y, z);
        }, 500);
      }

      const scale = [0.0, 2.0, 4.0, 6.0, 7.0, 9.0, 11.0, 12.0, 14.0];

      // Run createOsc() for each oscillator
      // // function takes bounded random frequency
      for (let i = 0; i < this.oscillators; i++) {
        // Get random note from scale and smudge
        let freq = this.baseFrequency * Math.pow(1.05945946, _.sample(scale));
        freq += Math.random() * 4 - 2;
        createOsc(freq);
      }
    },
    createControls() {
      const options = {
        zone: document.getElementById('canvasContainer'),
        threshold: 0.01,
      };
      const manager = nipplejs.create(options);
      manager.on('start', (e, data) => {
        this.startLoc.x = data.position.x;
        this.startLoc.y = data.position.y;
      });

      manager.on('move', (e, data) => {
        const heightCalc = Math.abs(data.position.y - this.startLoc.y);
        const widthCalc = Math.abs(data.position.x - this.startLoc.x);
        heightCalc === 0 ? this.canvasConfig.rh = this.canvasConfig.rh : this.canvasConfig.rh = heightCalc / 2;
        widthCalc === 0 ? this.canvasConfig.rw = this.canvasConfig.rw : this.canvasConfig.rw = widthCalc * 2;


        this.canvasConfig.h = data.angle.degree;
      });
    },
  },
  mounted() {
    if (this.isMobileDevice) {
      StartAudioContext(Tone.context);
    }
  },
};
</script>
<style lang="scss" scoped>
  #intro {
    position: absolute;
    top: 10px;
    left: 0;
    right: 0;
    margin-left: auto;
    margin-right: auto;
    width: 400px; /* Need a specific value to work */
    z-index: 1;
    padding: 10px;
    border: 1px solid black;

    button {
      height: 40px;
      width: 100%;
    }
  }

  #fullscreenButton {
    position: absolute;
    right: 0px;
    z-index: 1;
  }

  canvas {
    width: 100vw;
    height: 100vh;

  }

  #canvasContainer {
    position: absolute;
    top: 0;
    left: 0;
    z-index: 0;
  }

  #logo {
      position: fixed;
      bottom: 20px;
      right: 20px;
      opacity: 0.5;
  }

  #logo:hover {
      opacity: 1;
  }

</style>
