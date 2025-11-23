<template>
  <div class="app">
    <h1>Video to GIF Converter</h1>
    
    <div v-if="!videoFile" class="upload-section">
      <label for="video-upload" class="upload-label">
        <div class="upload-box">
          <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4M17 8l-5-5-5 5M12 3v12"/>
          </svg>
          <p>Click to upload video</p>
          <span>MP4, MOV, AVI, WebM</span>
        </div>
      </label>
      <input 
        id="video-upload" 
        type="file" 
        accept="video/*" 
        @change="handleVideoUpload"
      />
    </div>

    <div v-else class="converter">
      <div class="video-section">
        <div class="video-container">
          <video 
            ref="videoElement"
            :src="videoUrl"
            controls
            @loadedmetadata="onVideoLoaded"
          ></video>
          <img 
            v-if="watermarkFile && watermarkUrl"
            :src="watermarkUrl"
            ref="watermarkPreview"
            class="watermark-preview"
            :class="'position-' + watermarkPosition"
            :style="{
              transform: watermarkPosition === 'center' 
                ? `translate(-50%, -50%) scale(${watermarkScale / 100})` 
                : `scale(${watermarkScale / 100})`,
              opacity: watermarkOpacity,
              width: 'auto',
              height: 'auto'
            }"
            alt="Watermark preview"
          />
        </div>
        <button @click="resetVideo" class="reset-btn">Upload Different Video</button>
      </div>

      <div class="controls-section">
        <h2>GIF Settings</h2>
        
        <div class="control-group">
          <label>Start Time (seconds)</label>
          <input 
            type="number" 
            v-model.number="startTime" 
            :max="endTime - 0.1"
            step="0.1"
            min="0"
          />
          <input 
            type="range" 
            v-model.number="startTime" 
            :max="duration"
            step="0.1"
            min="0"
          />
        </div>

        <div class="control-group">
          <label>End Time (seconds)</label>
          <input 
            type="number" 
            v-model.number="endTime" 
            :min="startTime + 0.1"
            :max="duration"
            step="0.1"
          />
          <input 
            type="range" 
            v-model.number="endTime" 
            :max="duration"
            step="0.1"
            min="0"
          />
        </div>

        <div class="control-group">
          <label>Width (px)</label>
          <input type="number" v-model.number="width" min="100" max="1920" step="10" />
        </div>

        <div class="control-group">
          <label>Frame Rate (fps)</label>
          <input type="number" v-model.number="fps" min="5" max="30" />
        </div>

        <div class="watermark-section">
          <h3>Watermark (Optional)</h3>
          
          <div class="control-group">
            <label for="watermark-upload" class="watermark-upload-label">
              <div class="watermark-upload-box">
                <span v-if="!watermarkFile">📷 Upload Image (PNG recommended)</span>
                <span v-else>✓ {{ watermarkFile.name }}</span>
              </div>
            </label>
            <input 
              id="watermark-upload" 
              type="file" 
              accept="image/*" 
              @change="handleWatermarkUpload"
              style="display: none;"
            />
            <button v-if="watermarkFile" @click="removeWatermark" class="remove-watermark-btn">
              Remove Watermark
            </button>
          </div>

          <div v-if="watermarkFile" class="watermark-controls">
            <div class="control-group">
              <label>Position</label>
              <select v-model="watermarkPosition">
                <option value="top-left">Top Left</option>
                <option value="top-right">Top Right</option>
                <option value="bottom-left">Bottom Left</option>
                <option value="bottom-right">Bottom Right</option>
                <option value="center">Center</option>
              </select>
            </div>

            <div class="control-group">
              <label>Scale (%)</label>
              <input type="number" v-model.number="watermarkScale" min="10" max="200" />
              <input type="range" v-model.number="watermarkScale" min="10" max="200" />
            </div>

            <div class="control-group">
              <label>Opacity</label>
              <input type="range" v-model.number="watermarkOpacity" min="0.1" max="1" step="0.1" />
              <span class="opacity-value">{{ (watermarkOpacity * 100).toFixed(0) }}%</span>
            </div>
          </div>
        </div>

        <div class="info">
          <p>Duration: {{ (endTime - startTime).toFixed(1) }}s</p>
          <p>Output size: {{ width }}x{{ Math.round(width / aspectRatio) }}px</p>
        </div>

        <button 
          @click="convertToGif" 
          :disabled="isConverting || !ffmpegLoaded"
          class="convert-btn"
        >
          {{ isConverting ? 'Converting...' : ffmpegLoaded ? 'Convert to GIF' : 'Loading FFmpeg...' }}
        </button>

        <div v-if="progress > 0 && progress < 100" class="progress">
          <div class="progress-bar" :style="{ width: progress + '%' }"></div>
          <span>{{ progress }}%</span>
        </div>
      </div>

      <div v-if="gifUrl" class="preview-section">
        <h2>Preview</h2>
        <img :src="gifUrl" alt="Generated GIF" />
        <a :href="gifUrl" download="output.gif">
          <button>Download GIF</button>
        </a>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { FFmpeg } from '@ffmpeg/ffmpeg'
import { fetchFile, toBlobURL } from '@ffmpeg/util'

const videoFile = ref<File | null>(null)
const videoUrl = ref('')
const videoElement = ref<HTMLVideoElement | null>(null)
const duration = ref(0)
const startTime = ref(0)
const endTime = ref(5)
const width = ref(480)
const fps = ref(24)
const aspectRatio = ref(16/9)
const gifUrl = ref('')
const isConverting = ref(false)
const progress = ref(0)
const ffmpegLoaded = ref(false)

// Watermark settings
const watermarkFile = ref<File | null>(null)
const watermarkUrl = ref('')
const watermarkPosition = ref('bottom-right')
const watermarkOpacity = ref(1)
const watermarkScale = ref(100)

const ffmpeg = new FFmpeg()

onMounted(async () => {
  await loadFFmpeg()
})

async function loadFFmpeg() {
  try {
    const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.10/dist/esm'
    
    ffmpeg.on('log', ({ message }) => {
      console.log('FFmpeg:', message)
    })
    
    await ffmpeg.load({
      coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
      wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm'),
    })
    
    ffmpegLoaded.value = true
    console.log('FFmpeg loaded successfully')
  } catch (error) {
    console.error('Failed to load FFmpeg:', error)
    alert('Failed to load FFmpeg. Please refresh the page and try again.')
  }
}

function handleVideoUpload(event: Event) {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]
  if (file) {
    videoFile.value = file
    videoUrl.value = URL.createObjectURL(file)
  }
}

function handleWatermarkUpload(event: Event) {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]
  if (file) {
    watermarkFile.value = file
    watermarkUrl.value = URL.createObjectURL(file)
  }
}

function removeWatermark() {
  if (watermarkUrl.value) URL.revokeObjectURL(watermarkUrl.value)
  watermarkFile.value = null
  watermarkUrl.value = ''
}

function onVideoLoaded() {
  if (videoElement.value) {
    duration.value = videoElement.value.duration
    endTime.value = duration.value
    aspectRatio.value = videoElement.value.videoWidth / videoElement.value.videoHeight
  }
}

function resetVideo() {
  if (videoUrl.value) URL.revokeObjectURL(videoUrl.value)
  if (gifUrl.value) URL.revokeObjectURL(gifUrl.value)
  videoFile.value = null
  videoUrl.value = ''
  gifUrl.value = ''
  progress.value = 0
}

function getWatermarkOverlayPosition() {
  const padding = 10
  const positions: Record<string, string> = {
    'top-left': `${padding}:${padding}`,
    'top-right': `W-w-${padding}:${padding}`,
    'bottom-left': `${padding}:H-h-${padding}`,
    'bottom-right': `W-w-${padding}:H-h-${padding}`,
    'center': '(W-w)/2:(H-h)/2'
  }
  return positions[watermarkPosition.value] || positions['bottom-right']
}

async function convertToGif() {
  if (!videoFile.value || !ffmpegLoaded.value) return
  
  isConverting.value = true
  progress.value = 0
  
  try {
    const inputName = 'input.mp4'
    const watermarkName = 'watermark.png'
    const outputName = 'output.gif'
    
    await ffmpeg.writeFile(inputName, await fetchFile(videoFile.value))
    
    // Load watermark if provided
    if (watermarkFile.value) {
      await ffmpeg.writeFile(watermarkName, await fetchFile(watermarkFile.value))
    }
    
    const height = Math.round(width.value / aspectRatio.value)
    
    ffmpeg.on('progress', ({ progress: p }) => {
      progress.value = Math.round(p * 100)
    })
    
    // Build video filter with optional watermark overlay
    let videoFilter = `fps=${fps.value},scale=${width.value}:${height}:flags=lanczos`
    
    if (watermarkFile.value) {
      const scale = watermarkScale.value / 100
      const position = getWatermarkOverlayPosition()
      const alpha = watermarkOpacity.value
      
      // Scale watermark with high quality and overlay with proper alpha blending
      if (alpha < 1) {
        videoFilter = `[0:v]fps=${fps.value},scale=${width.value}:${height}:flags=lanczos[main];[1:v]scale=iw*${scale}:ih*${scale}:flags=lanczos,format=rgba,colorchannelmixer=aa=${alpha}[wm];[main][wm]overlay=${position}`
      } else {
        videoFilter = `[0:v]fps=${fps.value},scale=${width.value}:${height}:flags=lanczos[main];[1:v]scale=iw*${scale}:ih*${scale}:flags=lanczos[wm];[main][wm]overlay=${position}`
      }
    }
    
    videoFilter += ',split[s0][s1];[s0]palettegen[p];[s1][p]paletteuse'
    
    console.log('Video filter:', videoFilter)
    
    const ffmpegArgs = [
      '-i', inputName,
    ]
    
    if (watermarkFile.value) {
      ffmpegArgs.push('-i', watermarkName)
    }
    
    ffmpegArgs.push(
      '-ss', startTime.value.toString(),
      '-t', (endTime.value - startTime.value).toString(),
      '-filter_complex', videoFilter,
      '-loop', '0',
      outputName
    )
    
    await ffmpeg.exec(ffmpegArgs)
    
    const data = await ffmpeg.readFile(outputName)
    const blob = new Blob([data], { type: 'image/gif' })
    
    if (gifUrl.value) URL.revokeObjectURL(gifUrl.value)
    gifUrl.value = URL.createObjectURL(blob)
    
  } catch (error) {
    console.error('Conversion failed:', error)
    alert('Failed to convert video. Please try again.')
  } finally {
    isConverting.value = false
  }
}
</script>

<style scoped>
.app {
  text-align: center;
}

h1 {
  font-size: 2.5rem;
  margin-bottom: 2rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

h2 {
  font-size: 1.5rem;
  margin-bottom: 1.5rem;
  color: #f3f4f6;
}

.upload-section {
  margin: 4rem auto;
  max-width: 500px;
}

.upload-label {
  cursor: pointer;
}

.upload-box {
  border: 2px dashed #374151;
  border-radius: 1rem;
  padding: 3rem;
  transition: all 0.3s;
}

.upload-box:hover {
  border-color: #2563eb;
  background: #1a1a1a;
}

.upload-box svg {
  margin-bottom: 1rem;
  color: #6b7280;
}

.upload-box p {
  font-size: 1.125rem;
  margin-bottom: 0.5rem;
  color: #e0e0e0;
}

.upload-box span {
  font-size: 0.875rem;
  color: #6b7280;
}

.converter {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2rem;
  margin-top: 2rem;
}

.video-section {
  text-align: center;
}

.video-container {
  position: relative;
  display: inline-block;
  max-width: 600px;
  width: 100%;
}

.video-section video {
  width: 100%;
  max-width: 600px;
  border-radius: 0.5rem;
  margin-bottom: 1rem;
  display: block;
}

.watermark-preview {
  position: absolute;
  max-width: 100%;
  max-height: 100%;
  pointer-events: none;
  transform-origin: center;
  transition: all 0.3s ease;
}

.watermark-preview.position-top-left {
  top: 10px;
  left: 10px;
  transform-origin: top left;
}

.watermark-preview.position-top-right {
  top: 10px;
  right: 10px;
  transform-origin: top right;
}

.watermark-preview.position-bottom-left {
  bottom: 50px;
  left: 10px;
  transform-origin: bottom left;
}

.watermark-preview.position-bottom-right {
  bottom: 50px;
  right: 10px;
  transform-origin: bottom right;
}

.watermark-preview.position-center {
  top: 50%;
  left: 50%;
  transform-origin: center;
}

.reset-btn {
  background: #374151;
}

.reset-btn:hover {
  background: #4b5563;
}

.controls-section {
  background: #1a1a1a;
  padding: 2rem;
  border-radius: 1rem;
  text-align: left;
}

.control-group {
  margin-bottom: 1.5rem;
}

.control-group input[type="number"],
.control-group input[type="text"],
.control-group select {
  width: 100%;
  margin-bottom: 0.5rem;
}

.watermark-section {
  background: #0f0f0f;
  padding: 1.5rem;
  border-radius: 0.5rem;
  margin: 1.5rem 0;
}

.watermark-section h3 {
  font-size: 1.125rem;
  margin-bottom: 1rem;
  color: #9ca3af;
}

.watermark-controls {
  margin-top: 1rem;
}

.watermark-upload-box {
  border: 2px dashed #374151;
  border-radius: 0.5rem;
  padding: 1rem;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s;
  background: #1a1a1a;
}

.watermark-upload-box:hover {
  border-color: #2563eb;
  background: #252525;
}

.watermark-upload-label {
  cursor: pointer;
  display: block;
  margin-bottom: 0.5rem;
}

.remove-watermark-btn {
  width: 100%;
  background: #dc2626;
  margin-top: 0.5rem;
}

.remove-watermark-btn:hover {
  background: #b91c1c;
}

.opacity-value {
  display: inline-block;
  margin-left: 0.5rem;
  font-size: 0.875rem;
  color: #9ca3af;
}

.info {
  background: #0f0f0f;
  padding: 1rem;
  border-radius: 0.5rem;
  margin: 1.5rem 0;
}

.info p {
  margin: 0.25rem 0;
  font-size: 0.875rem;
}

.convert-btn {
  width: 100%;
  padding: 1rem;
  font-size: 1.125rem;
}

.progress {
  margin-top: 1rem;
  background: #0f0f0f;
  border-radius: 0.5rem;
  height: 2rem;
  position: relative;
  overflow: hidden;
}

.progress-bar {
  background: linear-gradient(90deg, #667eea 0%, #764ba2 100%);
  height: 100%;
  transition: width 0.3s;
}

.progress span {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-weight: 600;
}

.preview-section {
  grid-column: 1 / -1;
  margin-top: 2rem;
}

.preview-section img {
  max-width: 100%;
  border-radius: 0.5rem;
  margin-bottom: 1rem;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.5);
}

@media (max-width: 768px) {
  .converter {
    grid-template-columns: 1fr;
  }
}
</style>
