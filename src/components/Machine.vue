<template>
  <div id="mainMachine">
    <div id="title" color="#ace0f9"> 
      WW-T
      <span class="smaller">Trống điện tử</span>
    </div>
    <div id="controls">
      <machine-button color="#ace0f9" @click="pausePlay" :pressed="playing">✔</machine-button>
      <machine-button @click="randomizeSteps">Tạo các bước ngẫu nhiên</machine-button> 
      <machine-button @click="randomizeDrums">Tạo trống ngẫu nhiên</machine-button> 
      <machine-button @click="clearSteps">Làm mới</machine-button>
      <span class="knobLabel">Nhịp độ</span>
      <knob min="60" max="220" v-model="tempo" style="position: relative; top: 31px;"/>
    </div>
    <led v-for="(step,i) in stepCount" :key="i" :active="i==currentStep" />
    <div id="knobTitles">
        <li class="knobLabel">Gain</li>
        <li class="knobLabel">Pitch</li>
        <li class="knobLabel">Length</li>
        <li class="knobLabel">Env</li>
        <li class="knobLabel">Noise</li>
    </div>
    <div v-for="(inst,i) in instrumentCount" :key="i">
      <step-button 
        v-for="(step,j) in stepCount" 
        :key="j" 
        :value="pattern[i][j].active" 
        @input="val => pattern[i][j].active = val" 
        :highlight="currentStep==j" />
      <knob min="0" max="0.9"
      v-model="instruments[i].gain" />
      <knob min="60" max="8800"
      v-model="instruments[i].freq" />
      <knob min="0.05" max="0.9"
      v-model="instruments[i].decay" />
      <knob min="0.01" max="1"
      v-model="instruments[i].endPitch" />
      <knob min="0.01" max="1"
      v-model="instruments[i].sineNoiseMix" />
      <machine-button
        color="#8ddad5"
        :pressed="mutes[i]"
        style="float: right;"
        @click="mutes[i] = !mutes[i]">🤐</machine-button>
    </div>
    <span class="knobLabel">Trống đã cài đặt:</span>
    <machine-button 
      v-for="(preset,i) in presets" 
      :key="i" 
      color="#fad0c4"
      @click="loadPreset(preset)">{{ preset.name }}</machine-button>
  </div>
</template>

<script>
import StepButton from './StepButton'
import Knob from './Knob'
import Led from './Led'
import MachineButton from './MachineButton'
import _ from 'lodash'
import presets from '../presets'

let audioContext
if(window.AudioContext) {
  audioContext = new AudioContext()
} else {
  audioContext = new webkitAudioContext()
}

let bufferSize = 2 * audioContext.sampleRate,
    noiseBuffer = audioContext.createBuffer(1, bufferSize, audioContext.sampleRate),
    output = noiseBuffer.getChannelData(0);
for (var i = 0; i < bufferSize; i++) {
    output[i] = Math.random() * 2 - 1;
}

export default {
  name: 'machine',
  components: { StepButton, Knob, Led, MachineButton},
  data () {
    return {
      pattern: [],
      stepCount: 16,
      instruments: [],
      currentStep: 0,
      secondsPerStep: 0,
      lastScheduledTime: 0,
      nextStepTime: 0,
      mutes: [],
      playing: true,
      tempo: 120,
      audioTime: undefined,
      presets: presets.presets
    }
  },
  computed: {
    instrumentCount(){
      return this.instruments.length
    }
  },
  mounted () {
    this.loadPreset(this.presets[0])
    if(!window.AudioContext) this.playing = false // Safari fix
  },
  methods:{
    pausePlay(){
      this.playing = !this.playing
      audioContext.createOscillator().start(0,0,0.1)
    },
    loadPreset(preset){
      let loadedPreset = _.cloneDeep(preset)
      this.pattern = loadedPreset.pattern,
      this.tempo = loadedPreset.tempo,
      this.instruments = loadedPreset.instruments
      for(let mute in this.mutes){
        this.mutes[mute] = false
      }
    },
    randomizeSteps(){
      for (let inst in this.pattern){
        for (let step in this.pattern[inst]){
          this.pattern[inst][step].active = (Math.random()>0.5)
        }
      }
    },
    randomizeDrums(){
      for (let inst of this.instruments){
        inst.freq = Math.random()*3000+100
        inst.decay = Math.random()*0.8+0.01
        inst.endPitch = Math.random()*0.8+0.01
        inst.sineNoiseMix = Math.random()*1+0.01
      }
    },    
    clearSteps(){
      for (let inst in this.pattern){
        for (let step in this.pattern[inst]){
          this.pattern[inst][step].active = false
        }
      }
      for(let mute in this.mutes){
        this.mutes[mute] = false
      }
    },
    scheduleNote(instrument,startTime){
      let osc = audioContext.createOscillator()
      let mainGainNode = audioContext.createGain()
      let whiteNoise = audioContext.createBufferSource();

      let oscVol = audioContext.createGain()
      osc.connect(oscVol)
      oscVol.gain.setValueAtTime((1-instrument.sineNoiseMix)*2, startTime)
      oscVol.connect(mainGainNode)
      mainGainNode.connect(audioContext.destination)
      osc.start(startTime)
      osc.stop(startTime+instrument.decay)
      osc.frequency.setValueAtTime(instrument.freq, startTime)
      osc.frequency.exponentialRampToValueAtTime(instrument.freq*instrument.endPitch, startTime+instrument.decay)

      let noiseVol = audioContext.createGain()
      whiteNoise.buffer = noiseBuffer;
      whiteNoise.loop = true;
      whiteNoise.connect(noiseVol);
      noiseVol.gain.setValueAtTime(instrument.sineNoiseMix*2, startTime)
      noiseVol.connect(mainGainNode)
      whiteNoise.start(startTime);
      whiteNoise.stop(startTime+instrument.decay)
      mainGainNode.gain.setValueAtTime(instrument.gain, startTime)
      mainGainNode.gain.exponentialRampToValueAtTime(0.01, startTime+instrument.decay)


    },
    getSchedule(step,currentTime){
      let stepTime = step * this.secondsPerStep + ( currentTime - currentTime % (this.secondsPerStep * this.stepCount))
      if (stepTime<currentTime) { // skip to the next pattern if it's already too late
        stepTime += this.secondsPerStep * this.stepCount
      }
      return stepTime
    },
    updateAudioTime(){
      if(this.playing){
        const LOOK_AHEAD = 0.1
        this.secondsPerStep = 60/this.tempo/4
        this.audioTime = audioContext.currentTime
        this.currentStep = Math.floor(this.audioTime/this.secondsPerStep % this.stepCount)
        for (let inst in this.pattern){
          if(!this.mutes[inst]){
            for (let step in this.pattern[inst]){
              if (this.pattern[inst][step].active){
                let schedule = this.getSchedule(step, this.audioTime)
                if (schedule > 0 && schedule-this.audioTime<LOOK_AHEAD && schedule>this.lastScheduledTime){
                  this.scheduleNote(this.instruments[inst], schedule)
                }
              }
            }
          }
        }

        this.lastScheduledTime = this.audioTime+LOOK_AHEAD
      }
      requestAnimationFrame(this.updateAudioTime)
    }
  },
  created (){

    // Add some instruments

    for (let i=0; i<4; i++){
      this.instruments.push({
        freq: 100,
        gain: 0.2,
        decay: 0.01,
        endPitch: 0.01,
        sineNoiseMix: 0.01
      })
      this.mutes.push(false)
      this.randomizeDrums()
    }

    // Create empty patterns

    for(let i=0; i<this.instrumentCount; i++){
      this.pattern.push([])
      for(let j=0; j<this.stepCount; j++){
        this.pattern[i].push({active: false})
      }
    }

    this.updateAudioTime()
  }
}
</script>

<style scoped>

  #title{
    font-size: 30px;
    background-image: linear-gradient(to top, #3f51b1 0%, #5a55ae 13%, #7b5fac 25%, #8f6aae 38%, #a86aa4 50%, #cc6b8e 62%, #f18271 75%, #f3a469 87%, #f7c978 100%);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    font-weight: bold;
  }

  #title .smaller{
    font-size: 12px;
    background: linear-gradient(to bottom, rgba(255,255,255,0.15) 0%, rgba(0,0,0,0.15) 100%), radial-gradient(at top center, rgba(255,255,255,0.40) 0%, rgba(0,0,0,0.40) 120%) #989898;
    background-blend-mode: multiply,multiply;
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    font-weight: normal;
  }

  #mainMachine{
    margin: 0 auto;
    margin-top: 50px;
  }
  #knobTitles{
    display: inline-block;
    margin-left: 6px;
  }

  #controls{
    text-align: left;
  }

  .knobLabel{
    display: inline-block;
    font-size: 12px;
    margin: 0px 7px;
    color: #666;
  }
</style>
