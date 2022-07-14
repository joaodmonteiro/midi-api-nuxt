<template>
  <div class="container">
    <button @click="startAudioContext">Start</button>
    <div class="notes-container">
      <div class="note" :class="key" v-for="(note, key, index) in oscillators" :key="key"
        :style="{ '--velocity': note.velocity }">
        <div class="note-text">{{ note.noteName }}</div>
      </div>
    </div>
    <div></div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'

export default Vue.extend({
  name: 'IndexPage',

  data() {
    return {
      audioContext: null,
      oscillators: {},
      noteBars: [],
    }
  },

  async mounted() {
    const midiAccess = await navigator.requestMIDIAccess();
    for (let input of midiAccess.inputs.values()) {
      input.onmidimessage = this.getMIDIMessage;
    }

    window.AudioContext = window.AudioContext || window.webKitAudioContext;

  },

  methods: {
    getMIDIMessage(msg) {
      if (msg) {
        if (msg.data[0] === 144) {
          this.noteOn(msg.data[1], msg.data[2]);
        } else {
          this.noteOff(msg.data[1]);
        }
      }
    },

    startAudioContext() {
      this.audioContext = new AudioContext();
    },

    noteOn(note: number, velocity: number) {
      const osc = this.audioContext.createOscillator();

      const oscGain = this.audioContext.createGain();
      oscGain.gain.value = 0.2;

      const velocityGainAmout = (1 / 127) * velocity;
      const velocityGain = this.audioContext.createGain();
      velocityGain.gain.value = velocityGainAmout;

      osc.type = 'sine';
      osc.frequency.value = this.midiToFreq(note);
      osc.connect(oscGain);
      oscGain.connect(velocityGain);
      velocityGain.connect(this.audioContext.destination);

      osc.gain = oscGain;

      this.$set(this.oscillators, note.toString(), { noteName: this.getNoteName(note % 12, Math.floor(note / 12)), osc, velocity: velocityGainAmout });

      osc.start();
    },

    noteOff(note: number) {
      const osc = this.oscillators[note.toString()];
      const oscGain = osc.osc.gain;

      oscGain.gain.setValueAtTime(oscGain.gain.value, this.audioContext.currentTime);
      oscGain.gain.exponentialRampToValueAtTime(0.0001, this.audioContext.currentTime + 2);

      setTimeout(() => {
        osc.osc.stop();
        osc.osc.disconnect();
      }, 5000);

      this.$delete(this.oscillators, note.toString());
    },

    midiToFreq(note: number): number {
      const a = 440;
      return (a / 32 * (2 ** ((note - 9) / 12)));
    },

    getNoteName(note: number, octave: number) {
      const noteValue = {
        0: "C",
        1: "C#",
        2: "D",
        3: "D#",
        4: "E",
        5: "F",
        6: "F#",
        7: "G",
        8: "G#",
        9: "A",
        10: "A#",
        11: "B",
      };

      return (noteValue[note] + octave);
    }
  }
})
</script>

<style>
.container {
  height: 100vh;
}

.notes-container {
  display: flex;
  gap: 10px;
  justify-content: center;
  align-items: flex-end;
  height: 100%;
}

.note {
  flex-grow: var(--velocity);
  background-color: black;
  color: white;
  height: 200px;
  height: calc(100vh * var(--velocity));
  display: grid;
  place-items: center;
  /* animation-name: noteIn;
  animation-duration: 100ms;
  animation-timing-function: ease-in-out; */
}

.note-text {
  font-size: calc(200px * var(--velocity));
}

/* @keyframes noteIn {
  from {
    transform: scaleY(0.4) translateY(300%);
  }

  to {
    transform: scaleY(1) translateY(0);
  }
} */
</style>
