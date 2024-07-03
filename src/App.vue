<template>
  <div id="app">
    <!-- <h1>Music Practice</h1> -->
    <div class="notes">
      <div v-for="(note, index) in notes" :key="index"
        :style="{ backgroundColor: getColorForNote(note[0]), opacity: 1 }" :class="{ 'playing': isNotePlaying(index) }"
        class="note-square">
        <div class="note">{{ note[0] }}</div>
        <div class="frequency">{{ note[1] < 10000 ? note[1].toFixed(2) : note[1].toFixed(0) }} Hz</div>
        </div>
      </div>
      <div style="margin: 20px;">
        <label for="bpm">BPM: </label>
        <input type="number" id="bpm" v-model="bpm" min="60" max="200" />
      </div>
      <button style="margin: 10px;" @click="playNotes" :disabled="notes.length === 0">Play</button>
      <button style="margin: 10px;" @click="generateNewNotes">Next</button>
      <button style="margin: 10px;" @click="generateOpenNewNotes">Next (Open)</button>
      <div class="notes">
        <div v-for="(note, index) in detectedNotes" :key="index"
          :style="{ backgroundColor: getColorForNote(note[0]), opacity: 0.8 }"
          :class="{ 'playing': isNoteLastPlaying(index) }" class="note-square">
          <div class="note">{{ note[0] }}</div>
          <div class="frequency">{{ note[1] < 10000 ? note[1].toFixed(2) : note[1].toFixed(0) }} Hz</div>
          </div>
        </div>
        <div>
          <p>{{ result }}</p>
        </div>
      </div>
</template>

<script>
import * as Tone from 'tone'
import { PitchDetector } from "pitchy"

export default {
  data() {
    return {
      notes: [],
      noteColors: {
        'C': '#2c3e50', 'D': '#8e44ad', 'E': '#2980b9', 'F': '#27ae60',
        'G': '#783900', 'A': '#d35400', 'B': '#c0392b'
      },
      bpm: 120,
      result: '',
      isPlaying: false,
      synth: null,
      correctNotes: [],
      detectedNotes: [],
      currentNoteIndex: -1,
      audioContext: null,
      microphone: null,
      analyserNode: null,
      bufferSize: 1024 * 2
    }
  },
  methods: {
    generateNewNotes() {
      ; (new AudioContext()).resume()
      this.currentNoteIndex = -1
      this.notes = this.generateRandomNotes()
      this.result = ''
      this.correctNotes = []
      this.playNotes()
    },
    generateRandomNotes() {
      const possibleNotes = ['C', 'D', 'E', 'F', 'G', 'A', 'B']
      const octaves = [3, 4, 5]
      let notes = []

      for (let i = 0; i < 8; i++) {
        const note = possibleNotes[Math.floor(Math.random() * possibleNotes.length)]
        const octave = octaves[Math.floor(Math.random() * octaves.length)]
        const frequency = Tone.Frequency(`${note}${octave}`).toFrequency()
        notes.push([`${note}${octave}`, frequency])
      }
      return notes
    },
    generateOpenNewNotes() {
      this.currentNoteIndex = -1
      this.notes = []
      const possibleNotes = ["E4", "B3", "G3", "D3", "A2", "E2"]
      for (let i = 0; i < 8; i++) {
        const note = possibleNotes[Math.floor(Math.random() * possibleNotes.length)]
        const frequency = Tone.Frequency(note).toFrequency()
        this.notes.push([note, frequency])
      }
      this.result = ''
      this.correctNotes = []
      this.playNotes()
    },
    getColorForNote(note) {
      return this.noteColors[note[0]]
    },
    async playNotes() {
      if (this.notes.length === 0) return

      this.synth = new Tone.Synth().toDestination()
      const now = Tone.now()
      this.isPlaying = true
      this.currentNoteIndex = -1
      Tone.getTransport().bpm.value = this.bpm
      Tone.getTransport().start()

      const synth = new Tone.Synth().toDestination()
      this.notes.forEach((note, index) => {
        synth.triggerAttackRelease(note[0], '8n', now + (index * (60 / this.bpm)))
        // this.currentNoteIndex = index

        // Clear currentNoteIndex after short delay to remove highlight effect
        setTimeout(() => {
          // this.currentNoteIndex = -1
          this.currentNoteIndex = index
        }, (index * (60 / this.bpm)) * 1000) // Highlight for 0.5 seconds
      })

      // Record user input and check against notes
      // const recordedNotes = await this.recordUserInput()
      // this.checkNotes(recordedNotes)
    },
    async recordUserInput() {
      // Simulated user input
      return ['C4', 'D4', 'E4', 'F4', 'G4', 'A4', 'B4', 'C5']
    },
    checkNotes(recordedNotes) {
      this.correctNotes = this.notes.map((note, index) => recordedNotes[index] === note ? note : null)
      if (this.correctNotes.filter(note => note).length === this.notes.length) {
        this.result = 'Correct!'
      } else {
        this.result = 'Try Again!'
      }
    },
    isNotePlaying(index) {
      return index === this.currentNoteIndex
    },
    isNoteLastPlaying(index) {
      return index === this.detectedNotes.length - 1
    },
    autoCorrelate(buf, sampleRate) {
      let SIZE = buf.length
      let rms = 0

      for (let i = 0; i < SIZE; i++) {
        let val = buf[i]
        rms += val * val
      }
      rms = Math.sqrt(rms / SIZE)
      if (rms < 0.01)
        // not enough signal
        return -1

      let r1 = 0,
        r2 = SIZE - 1,
        thres = 0.2
      for (let i = 0; i < SIZE / 2; i++)
        if (Math.abs(buf[i]) < thres) {
          r1 = i
          break
        }
      for (let i = 1; i < SIZE / 2; i++)
        if (Math.abs(buf[SIZE - i]) < thres) {
          r2 = SIZE - i
          break
        }

      buf = buf.slice(r1, r2)
      SIZE = buf.length

      let c = new Array(SIZE).fill(0)
      for (let i = 0; i < SIZE; i++)
        for (let j = 0; j < SIZE - i; j++) c[i] = c[i] + buf[j] * buf[j + i]

      let d = 0
      while (c[d] > c[d + 1]) d++
      let maxval = -1,
        maxpos = -1
      for (let i = d; i < SIZE; i++) {
        if (c[i] > maxval) {
          maxval = c[i]
          maxpos = i
        }
      }
      let T0 = maxpos

      let x1 = c[T0 - 1],
        x2 = c[T0],
        x3 = c[T0 + 1]
      let a = (x1 + x3 - 2 * x2) / 2
      let b = (x3 - x1) / 2
      if (a) T0 = T0 - b / (2 * a)

      return sampleRate / T0
    },
    setupMicrophone() {
      navigator.mediaDevices.getUserMedia({ audio: true })
        .then(stream => {
          this.audioContext = new AudioContext()
          this.microphone = this.audioContext.createMediaStreamSource(stream)
          this.analyserNode = this.audioContext.createAnalyser()
          this.analyserNode.fftSize = this.bufferSize

          this.microphone.connect(this.analyserNode)

          this.processMicrophoneInput()
        })
        .catch(error => {
          console.error('Error accessing microphone:', error)
        })
    },
    processMicrophoneInput() {
      const dataArray = new Float32Array(this.bufferSize)

      const detectPitch = () => {
        this.analyserNode.getFloatTimeDomainData(dataArray)

        const frequency = this.autoCorrelate(dataArray, this.audioContext.sampleRate)
        // console.log(maxFrequencyIndex, this.audioContext.sampleRate, this.bufferSize, frequency)
        console.log(frequency)
        if (frequency !== -1) {
          const detectedNote = this.frequencyToNoteName(frequency)
          this.detectedNotes.push([detectedNote, frequency])
          if (this.detectedNotes.length > 8) {
            this.detectedNotes.shift()
          }
        }

        setTimeout(() => {
          requestAnimationFrame(detectPitch)
        }, 300)

      }

      detectPitch()
    },
    frequencyToNoteName(frequency) {
      // console.log(frequency)
      // Convert frequency to musical note name (e.g., 'C4', 'D#5')
      const noteNumber = 12 * (Math.log2(frequency / 440)) + 69
      const noteIndex = Math.round(noteNumber)
      const octave = Math.floor(noteIndex / 12) - 1
      const noteNames = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']
      const noteName = noteNames[noteIndex % 12]
      return `${noteName}${octave}`
    }
  },
  mounted() {
    // Real-time note detection setup
    this.setupMicrophone()
  }
}
</script>

<style>
#app {
  text-align: center;
  font-family: 'Courier New', Courier, monospace;
}

.notes {
  display: flex;
  justify-content: center;
  margin: 0.4rem;
}

.notes div {
  margin: 0.4rem;
  padding: 0.8rem;
  color: white;
  font-weight: bold;
  font-size: 2rem;
  /* Use monospace font */
}

.note-square {
  width: 6rem;
  /* Adjust width to maintain square shape */
  height: 6rem;
  /* Adjust height to maintain square shape */
  margin: 0.2rem;
  padding: 0.2rem;
  color: white;
  font-weight: bold;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
}


.playing {
  background-color: yellow;
  /* Highlight color */
  box-shadow: 0 0 4rem rgba(255, 255, 0, 0.7);
  /* Shadow effect */
}

.notes .note {
  margin: 0;
  padding: 0;
}

.notes .frequency {
  font-size: 0.5em;
  /* Smaller font size for frequency */
  margin: 0;
  padding: 0;
}
</style>
