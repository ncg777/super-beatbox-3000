<template>
  <v-app>
    <v-main>
      <v-responsive class="align-center mx-auto pa-4" max-width="900">
        <h1>Super Beatbox 3k</h1>
        <v-row>
          <v-col cols="12" :style="'position:relative'">
            <div :style="'position:absolute; right:1em;top:1.5em;'"><v-btn @click="showPitchHelp = true" :style="'z-index: 999;'">Key mapping</v-btn></div>
            <v-text-field 
              :label="`Drum Pitches (${drumPitchesInput.length})`" 
              v-model="drumPitchesInput" 
              placeholder="e.g. 36 38 39 42 43 46 47 49 50 53 75" 
              @update:modelValue="saveSettingsToLocalStorage" />
            </v-col>
        </v-row>
        <v-row>
          <v-col cols="12">
            <v-text-field 
              :label="`Sequence (${sequence.length})`"
              v-model="sequenceInput" 
              placeholder="e.g. 0 1 2..." 
              @update:modelValue="saveSettingsToLocalStorage" />
          </v-col>
        </v-row>
        <v-row>
          <v-col cols="12">
            <v-slider 
              :label="'Tempo (' + bpm + ' BPM)'" 
              min="1" step="1" max="499" 
              v-model.number="bpm" 
              @update:modelValue="saveSettingsToLocalStorage" />
          </v-col>
          <v-col cols="12">
            <v-slider 
              :label="'Velo bits/pitch (' + velobits + ')'" 
              min="1" max="7" step="1" 
              v-model.number="velobits" 
              @update:modelValue="saveSettingsToLocalStorage" />
          </v-col>
        </v-row>
        <v-row>
          <v-col cols="12">
            <v-slider 
              :label="'Numerator (' + numerator + ')'" 
              min="1" step="1" max="16" 
              v-model.number="numerator" 
              @update:modelValue="saveSettingsToLocalStorage" />
          </v-col>
          <v-col cols="12">
            <v-slider 
              :label="'Denominator (' + denominator + ')'" 
              min="1" step="1" max="16" 
              v-model.number="denominator" 
              @update:modelValue="saveSettingsToLocalStorage" />
          </v-col>
        </v-row>
        <button @click="toggleSequencer" class="stopplay">{{ isRunning ? '⏹️' : '▶️' }}</button>
        <button @click="copyURL" class="userbutton">📋 Copy URL</button>
        <button @click="downloadMIDI" class="downloadmidi">Download MIDI</button>
        <button @click="showHelp = true" class="userbutton">❓ Help</button>
      </v-responsive>
      <v-dialog v-model="showPitchHelp" max-width="800px">
        <v-card class="pa-4 bg-black">
          <v-card-title>
            <span class="text-h5">DR-220</span>
            <v-spacer></v-spacer>
            <v-btn icon @click="showPitchHelp = false" class="close-btn">
              <v-icon>mdi-close</v-icon>
            </v-btn>
          </v-card-title>
          <v-divider></v-divider>
          <v-card-text class="pa-4">
            <v-table :items="soundMappings" hide-default-footer>
              <thead>
              <tr>
                <th>MIDI Pitch</th>
                <th>SOUND</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in soundMappings" :key="item.pitch">
                <td>{{ item.pitch }}</td>
                <td>{{ item.sound }}</td>
              </tr>
            </tbody>
            </v-table>
          </v-card-text>
          <v-divider></v-divider>
          <v-card-actions class="pa-4">
            <v-spacer></v-spacer>
            <v-btn color="primary" @click="showPitchHelp = false">Close</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
      <v-dialog v-model="showHelp" max-width="800px">
        <v-card class="pa-4 bg-black">
          <v-card-title class="pa-4">
            <span class="text-h5 font-weight-bold">Help</span>
            <v-spacer></v-spacer>
            <v-btn icon @click="showHelp = false" class="close-btn">
              <v-icon>mdi-close</v-icon>
            </v-btn>
          </v-card-title>
          <v-divider></v-divider>
          <v-card-text class="pa-4">
            <h4 class="mb-2">How the Super Beatbox 3000 Works</h4>
            <p>
              This drum machine uses two sequences:
              <strong>Drum Pitch Sequence</strong> and <strong>Main Sequence</strong>.
            </p>
            <ul>
              <li><strong>Drum Pitch Sequence</strong>: Enter a space‑separated list of MIDI note numbers (e.g. "36 38 42 46").</li>
              <li><strong>Main Sequence</strong>: Each number is converted to binary and padded to (number of drum pitches × velobits) bits. It is then split into velobits‑bit chunks – each corresponding (in order) to a drum pitch. A chunk of 0 means “don’t play,” while a nonzero value triggers the sound with velocity proportional to its value.</li>
              <li><strong>Tempo, Numerator, Denominator</strong>: Control the timing and time signature of the loop.</li>
            </ul>
          </v-card-text>
          <v-divider></v-divider>
          <v-card-actions class="pa-4">
            <v-spacer></v-spacer>
            <v-btn color="primary" @click="showHelp = false">Close</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-main>
  </v-app>
</template>

<script lang="ts">
import { defineComponent, markRaw } from 'vue';
import * as Tone from 'tone';
import { Midi } from '@tonejs/midi';

interface SoundMapping {
  pitch: string;
  sound: string;
}
export default defineComponent({
  name: 'SuperBeatbox3000',
  data() {
    const params = new URLSearchParams(window.location.search);
    return {
      bpm: parseInt(params.get("bpm") ?? localStorage.getItem("bb3k_bpm") ?? "90"),
      numerator: parseInt(params.get("numerator") ?? localStorage.getItem("bb3k_numerator") ?? "4"),
      denominator: parseInt(params.get("denominator") ?? localStorage.getItem("bb3k_denominator") ?? "4"),
      drumPitchesInput: params.get("drumPitches") ?? localStorage.getItem("bb3k_drumPitches") ?? "36 38 39 42 43 46 47 49 50 53 75",
      sequenceInput: params.get("sequence") ?? localStorage.getItem("bb3k_sequence") ?? "1 0 32 8 2 0 32 8",
      velobits: parseInt(params.get("velobits") ?? localStorage.getItem("bb3k_velobits") ?? "1"),
      isRunning: false,
      loop: null as Tone.Loop | null,
      counter: 0,
      showHelp: false,
      showPitchHelp: false,
      soundMappings: [
        { pitch: "36", sound: "Bassdrum" },
        { pitch: "38", sound: "Snaredrum" },
        { pitch: "39", sound: "Clap" },
        { pitch: "42", sound: "Hat Closed" },
        { pitch: "43", sound: "Tom L" },
        { pitch: "46", sound: "Hat Open" },
        { pitch: "47", sound: "Tom M" },
        { pitch: "49", sound: "Crash" },
        { pitch: "50", sound: "Tom H" },
        { pitch: "53", sound: "Ride" },
        { pitch: "75", sound: "Clave" }
      ] as SoundMapping[],
      sampler: markRaw(new Tone.Sampler(
        {
          urls: {
          C2: "BossDR-220/Bassdrum.mp3",
          D2: "BossDR-220/Snaredrum.mp3",
          "D#2": "BossDR-220/Clap.mp3",
          "F#2": "BossDR-220/HatClosed.mp3",
          A2: "BossDR-220/TomL.mp3",
          "A#2": "BossDR-220/HatOpen.mp3",
          B2:"BossDR-220/TomM.mp3",
          "C#3": "BossDR-220/Crash.mp3",
          D3: "BossDR-220/TomH.mp3",
          F3: "BossDR-220/Ride.mp3",
          "D#5": "BossDR-220/Clave.mp3"
          },
          onload: () => {console.log('Samples loaded')},
          attack:0.001,
          release: 0.01,
          curve: 'exponential'
        }
        ).toDestination()),
    };
  },
  computed: {
    // The quantization value (in seconds) for each step.
    quant() {
      return 240.0 / (this.bpm * this.numerator * this.denominator);
    },
    // Main sequence as an array of numbers.
    sequence(): number[] {
      return this.sequenceInput
        .split(' ')
        .map((n: string) => parseInt(n.trim()))
        .filter((n: number) => !isNaN(n));
    },
    // Drum pitch sequence parsed into an array of MIDI note numbers.
    drumPitches(): number[] {
      return this.drumPitchesInput
        .split(' ')
        .map((n: string) => parseInt(n.trim()))
        .filter((n: number) => !isNaN(n));
    },
    // Compute, for each step, the drum triggers.
    // Each step’s number is covelobitserted to binary, padded to (drumPitches.length * velobits) bits,
    // and split into velobits‑bit segments. A segment value of 0 means no trigger.
    actualDrumTriggers(): { pitch: number, velocity: number }[][] {
      return this.sequence.map((num: number) => {
        const numPitches = this.drumPitches.length;
        const totalBits = numPitches * this.velobits;
        let binStr = Math.abs(num).toString(2).padStart(totalBits, '0').split('').reverse().join('');
        let triggers: { pitch: number, velocity: number }[] = [];
        for (let i = 0; i < numPitches; i++) {
          const segment = binStr.substring(i * this.velobits, (i+1) * this.velobits);
          const value = parseInt(segment.split('').reverse().join(''), 2);
          if (value > 0) {
            const maxVal = Math.pow(2, this.velobits) - 1;
            const velocity = value *(127/maxVal);

            triggers.push({ pitch: this.drumPitches[i], velocity });
          }
        }
        return triggers;
      });
    },
    formattedDate() {
      return ((timestamp: number) =>
        `${new Date(timestamp).getUTCFullYear()}${String(new Date(timestamp).getUTCMonth() + 1).padStart(2, '0')}${String(new Date(timestamp).getUTCDate()).padStart(2, '0')}T${String(new Date(timestamp).getUTCHours()).padStart(2, '0')}${String(new Date(timestamp).getUTCMinutes()).padStart(2, '0')}${String(new Date(timestamp).getUTCSeconds()).padStart(2, '0')}Z`
      )(Date.now());
    },
  },
  methods: {
    async playStep(when: Tone.Unit.Seconds, counter:number) {
      const triggers = this.actualDrumTriggers[counter % this.actualDrumTriggers.length];
      if (triggers.length > 0 && this.sampler) {
        let dur = 1;
        while(this.actualDrumTriggers[(counter+dur)%this.actualDrumTriggers.length].length == 0) dur++;

        for(let trigger of triggers) {
          this.sampler.triggerAttackRelease(
            Tone.Frequency(trigger.pitch, 'midi').toNote(),
            (dur*this.quant).toString()+"s",
            when,
            trigger.velocity/127.0
          );
        }
      }
    },
    async toggleSequencer() {
      if (this.isRunning) {
        this.stopSequencer();
      } else {
        this.startSequencer();
      }
    },
    async copyURL() {
      const url = encodeURI(`https://ncg777.github.io/super-beatbox-3000?bpm=${this.bpm}&numerator=${this.numerator}&denominator=${this.denominator}&drumPitches=${this.drumPitchesInput}&velobits=${this.velobits}&sequence=${this.sequenceInput}`);
      await navigator.clipboard.writeText(url);
      window.alert("URL copied to clipboard.");
    },
    async startSequencer() {
      if (this.isRunning) return;
      this.isRunning = true;
      this.counter = 0;
      await Tone.start();
      await Tone.loaded();
      console.log('Audio context started');
      this.saveSettingsToLocalStorage();
      const that = this;
      if (this.loop === null) {
        this.loop = new Tone.Loop(async (time: Tone.Unit.Seconds) => {
          that.playStep(time, that.counter);
          that.counter = (that.counter + 1) % that.actualDrumTriggers.length;
        }, this.quant.toString() + "s");
      }
      this.loop.start(0);
      Tone.getTransport().start();
      console.log('Sequencer started');
    },
    stopSequencer() {
      if (!this.isRunning) return;
      this.isRunning = false;
      this.loop?.stop();
      Tone.getTransport().stop();
      console.log('Sequencer stopped');
    },
    saveSettingsToLocalStorage() {
      localStorage.setItem("bb3k_bpm", this.bpm.toString());
      localStorage.setItem("bb3k_numerator", this.numerator.toString());
      localStorage.setItem("bb3k_denominator", this.denominator.toString());
      localStorage.setItem("bb3k_drumPitches", this.drumPitchesInput);
      localStorage.setItem("bb3k_velobits", this.velobits.toString());
      localStorage.setItem("bb3k_sequence", this.sequenceInput);
      if (this.loop) {
        this.loop.interval = this.quant + "s";
      }
      Tone.getTransport().bpm.value = this.bpm;
      Tone.getTransport().timeSignature = [this.numerator, this.denominator];
    },
    async getMidi(): Promise<Midi> {
      const midi = new Midi();
      const track = midi.addTrack();
      
      midi.header.setTempo(this.bpm);
      for (let i = 0; i < this.actualDrumTriggers.length; i++) {
        const triggers = this.actualDrumTriggers[i];
        let dur = 1;
        while(this.actualDrumTriggers[(i+dur)%this.actualDrumTriggers.length].length == 0) dur++;
        const duration = this.quant * dur;
        for(let trigger of triggers) {
          track.addNote({
            midi: trigger.pitch,
            time: i * this.quant,
            duration: duration,
            velocity: trigger.velocity/127
          });
        }
        // MIDI channel 10 (channel index 9) is standard for drums.
        track.channel = 9
      }
      return midi;
    },
    async downloadMIDI() {
      const midiData = (await this.getMidi()).toArray();
      const blob = new Blob([midiData], { type: 'audio/midi' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `SBbox3k-${this.formattedDate}-${this.bpm}BPM-${this.numerator}on${this.denominator}.mid`;
      a.click();
      URL.revokeObjectURL(url);
    },
  },
  beforeUnmount() {
    this.stopSequencer();
  },
  mounted() {
    this.saveSettingsToLocalStorage();
  }
});
</script>

<style scoped>
body, * {
  color: #00aa00;
  background-color: #000000;
}
h1 {
  text-align: center;
  margin-bottom: 16pt;
}
.downloadmidi, .userbutton {
  padding: 10px;
  font-size: 18px;
  width: 100%;
  margin-bottom: 10px;
}
.stopplay {
  padding: 2px;
  font-size: 50px;
  width: 100%;
  margin-bottom: 5px;
}
.close-btn {
  position: absolute;
  top: 16px;
  right: 16px;
}
</style>
