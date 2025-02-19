<template>
  <div id="sequencer">
    <div v-for="(row, index) in sequenceData" :key="index">
      <select v-model="row.sample">
        <option v-for="sample in availableSamples" :key="sample" :value="sample">
          {{ sample }}
        </option>
      </select>
      <div
        v-for="step in 16"
        :key="step"
        class="step"
        :class="{ active: row.steps[step] }"
        @click="toggleStep(row, step)"
      ></div>
      <button @click="deleteRow(index)">Delete Row</button>
    </div>
  </div>
  <button @click="addRow">Add Row</button>
  <div>
    <label for="bpm">BPM:</label>
    <input id="bpm" type="number" min="20" max="300" v-model.number="bpm" />
  </div>
  <button @click="togglePlay">{{ isPlaying ? 'Pause' : 'Play' }}</button>
</template>
<script setup>
import { ref, reactive, watch } from 'vue'
import * as Tone from 'tone'
const availableSamples = ['kick', 'snare', 'hihat', 'clap']
const sampler = new Tone.Sampler({
  kick: 'samples/kick.mp3',
  snare: 'samples/snare.mp3',
  hihat: 'samples/hihat.mp3',
  clap: 'samples/clap.mp3'
}).toDestination()
const sequenceData = reactive(
  availableSamples.map((sample) => ({ sample, steps: Array(16).fill(false) }))
)
let isPlaying = ref(false)
let bpm = ref(120)
const sequence = new Tone.Sequence(
  (time, col) => {
    for (const row of sequenceData) {
      if (row.steps[col]) {
        sampler.triggerAttack(row.sample, time)
      }
    }
  },
  Array.from({ length: 16 }, (_, i) => i),
  '16n'
)
Tone.Transport.bpm.value = bpm.value
watch(bpm, (newBpm) => {
  Tone.Transport.bpm.value = newBpm
})
function addRow() {
  sequenceData.push({ sample: availableSamples[0], steps: Array(16).fill(false) })
}
function deleteRow(index) {
  sequenceData.splice(index, 1)
}
function toggleStep(row, step) {
  row.steps[step] = !row.steps[step]
}
function togglePlay() {
  isPlaying.value = !isPlaying.value
  if (isPlaying.value) {
    Tone.Transport.start()
    sequence.start()
  } else {
    Tone.Transport.stop()
    sequence.stop()
  }
}
</script>
<style scoped>
.step {
  width: 20px;
  height: 20px;
  border: 1px solid #000;
}
.active {
  background-color: #000;
}
</style>
