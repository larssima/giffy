# Video to GIF Converter

A local web app for converting videos to GIFs with full control over timing, size, and quality.

## Features

- 🎬 Upload any video format (MP4, MOV, AVI, WebM)
- ⏱️ Set custom start and end times
- 📐 Adjust output dimensions
- 🎞️ Control frame rate (5-30 fps)
- 👁️ Live preview before download
- 🔒 100% client-side processing (videos never leave your machine)

## Setup

```bash
npm install
npm run dev
```

Open http://localhost:5173 in your browser.

## How to Use

1. Click to upload a video file
2. Adjust the start/end times using sliders or number inputs
3. Set your desired width and frame rate
4. Click "Convert to GIF"
5. Preview and download your GIF

## Tech Stack

- Vue 3 + TypeScript
- Vite
- FFmpeg.wasm (browser-based video processing)
