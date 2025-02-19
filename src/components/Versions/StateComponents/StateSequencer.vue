<script async setup>
import {
  ref,
  reactive,
  watch,
  onMounted,
  onUnmounted,
  TransitionGroup,
  Transition,
  computed
} from 'vue'
import { storeToRefs } from 'pinia'
import * as Tone from 'tone'
// Pack with sample names
import { useSequenceStore } from '@/stores/sequence.js'
import { getSampleData, getSampleUrl } from '@/composables/getSampleData.js'

import {
  createSampleObject,
  createSequenceArraySteps,
  createSequenceArrayIndex
} from '@/helpers/toneHelpers.js'

import BaseIcon from '@/components/BaseIcon.vue'
import BaseButton from '@/components/BaseButton.vue'
import SampleSelect from '@/components/SampleSelect.vue'
import SequenceItem from '@/components/Versions/HitLoopSequencerV/SequenceItemRenderless.vue'
import StateSequenceItem from '@/components/Versions/StateComponents/StateSequenceItem.vue'
import SequenceItemArc from '@/components/SequenceItemArc.vue'

// Base url for the api

const store = await useSequenceStore()
// store values to vuejs ref
const {
  availableNotes,
  activeNotes,
  currentStepIndex,
  sequenceData,
  sampleData,
  isPlaying,
  columns,
  getSequenceData,
  sampleObject
} = storeToRefs(store)

const { toggleStep, updateSequenceURL, addSequence, togglePlayPause, setCurrentStepIndex } = store

const bpm = ref(120)

const sampleTypeList = ref(['Crash', 'Kick', 'Sfx', 'Snare', 'Hi-Hat'])

const highlighted = ref(-1)

const newSequenceData = getSequenceData

const tick = (time, col) => {
  const sampler = new Tone.Sampler({
    urls: store.sampleObject,
    onload: () => {
      console.log('loaded')
    }
  }).toDestination()

  Tone.Draw.schedule(() => {
    if (isPlaying.value === true) {
      highlighted.value = col
      setCurrentStepIndex(col)
    }
  }, time)

  for (const row of newSequenceData.value) {
    if (row.steps[col]) {
      Tone.loaded().then(() => {
        sampler.triggerAttackRelease(row.sample, '16n', Tone.now())
      })
    }
  }
}

const sequence = new Tone.Sequence(tick, createSequenceArrayIndex(columns.value), '16n')

Tone.Transport.bpm.value = bpm.value

watch(bpm, (newBpm) => {
  Tone.Transport.bpm.value = newBpm
})

const togglePlay = () => {
  togglePlayPause()
  if (isPlaying.value) {
    Tone.Transport.start(Tone.now())
    sequence.start()
  } else {
    Tone.Transport.stop(Tone.now())
    sequence.stop()
  }
}

const onKeyDown = (event) => {
  if (event.code === 'Space') {
    togglePlay()
    event.preventDefault()
  }
}

onMounted(() => {
  window.addEventListener('keydown', onKeyDown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', onKeyDown)
})
</script>

<template>
  <div id="sequencer" v-if="sequenceData">
    <TransitionGroup name="fade">
      <template v-for="(row, index) in sequenceData" :key="index">
        <StateSequenceItem
          :selectedValue="row.url"
          :item="row"
          :id="index"
          :columns="columns"
          :highlighted="currentStepIndex"
          @toggle-step="toggleStep"
          :sampleTypeList="sampleTypeList"
          :sampleData="sampleData"
        />
      </template>
    </TransitionGroup>
    <StateSequenceItem empty>
      <button v-show="availableNotes > activeNotes" @click="addSequence()">
        <BaseIcon name="add" />
      </button>
    </StateSequenceItem>
  </div>
  <div class="controlls">
    <div>
      <label for="bpm">BPM:</label>
      <input id="bpm" type="number" min="20" max="300" v-model.number="bpm" />
    </div>
    <BaseButton v-if="!isPlaying" @click="togglePlay" icon="play_arrow" />
    <BaseButton v-else @click="togglePlay" icon="pause" />
  </div>
</template>

<style scoped lang="scss">
.controlls {
  padding: 1em;
  display: flex;
  align-items: center;
  gap: 1em;
  position: fixed;
  bottom: 0;
  justify-content: center;
  justify-items: center;
  // margin: 0 auto;
  left: 0;
  right: 0;
}
.sequencer {
  font-size: 1.5em;
}

.step-container {
  display: flex;
  flex-direction: column;
  margin-top: 0.1em;
  margin-bottom: 0.1em;
}
// .step:first-child {
//   height: 10rem;
// }
.step {
  width: 3em;
  height: 3em;
  border: 1px solid var(--color-black);
  // transition: all 1ms ease-in-out;
  position: absolute;

  &.highlighted {
    border: 4px solid var(--color-orange);
  }
}

svg {
  width: 10rem;
  line {
    cursor: pointer;
  }
}
.arc-item {
  stroke: #cbcbcb;
}
.active {
  stroke-opacity: 50%;
  stroke: #2ecd71;
}

.highlighted {
  stroke-opacity: 50%;

  // stroke: green;
}

.highlighted.active {
  stroke-width: 22;
}

#sequencer {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); /* see notes below */
  grid-gap: 1em;
  padding: 2rem;

  // div {
  //   display: flex;
  //   flex-wrap: wrap;
  //   flex-direction: row;

  //   div {
  //     display: flex;
  //   }
  // }
}
.row {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 2rem;
  overflow: hidden;
}

select {
  width: 100%;
}
.active {
  background-color: var(--color-black-mute);
}

/* 1. declare transition */
.fade-move,
.fade-enter-active,
.fade-leave-active {
  transition: all 0.5s cubic-bezier(0.55, 0, 0.1, 1);
}

/* 2. declare enter from and leave to state */
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
  transform: scaleY(0.01) translate(30px, 0);
}

/* 3. ensure leaving items are taken out of layout flow so that moving
      animations can be calculated correctly. */
.fade-leave-active {
  position: absolute;
}
</style>
