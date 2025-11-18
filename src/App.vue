<script setup lang="ts">
import { ref, computed } from 'vue'
import FrameView from './components/FrameView.vue'

interface SplitImage {
  leftHalf: string // Display quality (92%)
  rightHalf: string // Display quality (92%)
  leftHalfOriginal: string // Original quality (98%)
  rightHalfOriginal: string // Original quality (98%)
  originalName: string
  leftRotation: number
  rightRotation: number
  leftChecked: boolean
  rightChecked: boolean
}

const splitImages = ref<SplitImage[]>([])
const zoomLevel = ref(25) // Default 49% (2 per row)

// Import progress tracking
const isImporting = ref(false)
const importProgress = ref({ current: 0, total: 0 })
const shouldStopImport = ref(false)

// Download progress tracking
const isDownloading = ref(false)
const downloadProgress = ref({ current: 0, total: 0 })
const shouldStopDownload = ref(false)

// Checkbox state
const allChecked = computed({
  get: () => {
    if (splitImages.value.length === 0) return false
    return splitImages.value.every((img) => img.leftChecked && img.rightChecked)
  },
  set: (value: boolean) => {
    splitImages.value.forEach((img) => {
      img.leftChecked = value
      img.rightChecked = value
    })
  },
})

const hasCheckedItems = computed(() => {
  return splitImages.value.some((img) => img.leftChecked || img.rightChecked)
})

const checkedCount = computed(() => {
  return splitImages.value.reduce((count, img) => {
    return count + (img.leftChecked ? 1 : 0) + (img.rightChecked ? 1 : 0)
  }, 0)
})

const downloadButtonText = computed(() => {
  if (isDownloading.value) {
    return `${downloadProgress.value.current} / ${downloadProgress.value.total}`
  }
  const count = checkedCount.value
  if (count === 0) return 'Download Selected'
  return `Download ${count} Selected`
})

const importButtonText = computed(() => {
  if (isImporting.value) {
    return `${importProgress.value.current} / ${importProgress.value.total}`
  }
  return 'Add Images'
})

// Calculate max height based on zoom level
const maxImageHeight = computed(() => {
  // Scale height from 200px (at 25%) to 800px (at 100%)
  return Math.round(200 + (zoomLevel.value - 25) * (600 / 75))
})

const splitImageInHalf = (imageData: string, fileName: string): Promise<SplitImage> => {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.onload = () => {
      const width = img.width
      const height = img.height

      // Determine if we should split horizontally or vertically
      const isHorizontal = width > height

      if (isHorizontal) {
        // Split vertically (left and right halves)
        const halfWidth = width / 2

        // Create canvas for left half
        const leftCanvas = document.createElement('canvas')
        leftCanvas.width = halfWidth
        leftCanvas.height = height
        const leftCtx = leftCanvas.getContext('2d', { alpha: false })
        if (leftCtx) {
          leftCtx.imageSmoothingEnabled = false
          leftCtx.drawImage(img, 0, 0, halfWidth, height, 0, 0, halfWidth, height)
        }

        // Create canvas for right half
        const rightCanvas = document.createElement('canvas')
        rightCanvas.width = halfWidth
        rightCanvas.height = height
        const rightCtx = rightCanvas.getContext('2d', { alpha: false })
        if (rightCtx) {
          rightCtx.imageSmoothingEnabled = false
          rightCtx.drawImage(img, halfWidth, 0, halfWidth, height, 0, 0, halfWidth, height)
        }

        resolve({
          leftHalf: leftCanvas.toDataURL('image/jpeg', 0.92),
          rightHalf: rightCanvas.toDataURL('image/jpeg', 0.92),
          leftHalfOriginal: leftCanvas.toDataURL('image/png'),
          rightHalfOriginal: rightCanvas.toDataURL('image/png'),
          originalName: fileName,
          leftRotation: 0,
          rightRotation: 0,
          leftChecked: false,
          rightChecked: false,
        })
      } else {
        // Split horizontally (top and bottom halves)
        const halfHeight = height / 2

        // Create canvas for top half
        const topCanvas = document.createElement('canvas')
        topCanvas.width = width
        topCanvas.height = halfHeight
        const topCtx = topCanvas.getContext('2d', { alpha: false })
        if (topCtx) {
          topCtx.imageSmoothingEnabled = false
          topCtx.drawImage(img, 0, 0, width, halfHeight, 0, 0, width, halfHeight)
        }

        // Create canvas for bottom half
        const bottomCanvas = document.createElement('canvas')
        bottomCanvas.width = width
        bottomCanvas.height = halfHeight
        const bottomCtx = bottomCanvas.getContext('2d', { alpha: false })
        if (bottomCtx) {
          bottomCtx.imageSmoothingEnabled = false
          bottomCtx.drawImage(img, 0, halfHeight, width, halfHeight, 0, 0, width, halfHeight)
        }

        resolve({
          leftHalf: topCanvas.toDataURL('image/jpeg', 0.92),
          rightHalf: bottomCanvas.toDataURL('image/jpeg', 0.92),
          leftHalfOriginal: topCanvas.toDataURL('image/png'),
          rightHalfOriginal: bottomCanvas.toDataURL('image/png'),
          originalName: fileName,
          leftRotation: 0,
          rightRotation: 0,
          leftChecked: false,
          rightChecked: false,
        })
      }
    }
    img.onerror = reject
    img.src = imageData
  })
}

// Helper to process a single file
const processFile = (file: File): Promise<SplitImage> => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = async (e) => {
      try {
        const imageData = e.target?.result as string
        const split = await splitImageInHalf(imageData, file.name)
        resolve(split)
      } catch (error) {
        reject(error)
      }
    }
    reader.onerror = reject
    reader.readAsDataURL(file)
  })
}

// Stop the import process
const stopImport = () => {
  shouldStopImport.value = true
}

const handleFileSelect = async (event: Event) => {
  const target = event.target as HTMLInputElement
  const files = target.files

  if (!files || files.length === 0) return

  const fileArray = Array.from(files).filter((file) => file.type.startsWith('image/'))

  if (fileArray.length === 0) return

  isImporting.value = true
  shouldStopImport.value = false
  importProgress.value = { current: 0, total: fileArray.length }

  // Process files in small batches to keep UI responsive
  const batchSize = 2

  for (let i = 0; i < fileArray.length; i += batchSize) {
    // Check if user requested to stop
    if (shouldStopImport.value) {
      break
    }

    const batch = fileArray.slice(i, i + batchSize)

    // Process batch in parallel
    const results = await Promise.all(batch.map((file) => processFile(file)))

    // Add results to the array
    splitImages.value.push(...results)

    // Update progress
    importProgress.value.current = Math.min(i + batchSize, fileArray.length)

    // Give the UI time to update between batches
    if (i + batchSize < fileArray.length) {
      await new Promise((resolve) => setTimeout(resolve, 10))
    }
  }

  // Keep the progress indicator visible for a moment before hiding
  await new Promise((resolve) => setTimeout(resolve, 500))
  isImporting.value = false
  shouldStopImport.value = false

  // Reset the file input so the same files can be selected again if needed
  target.value = ''
}

const removeImagePair = (index: number) => {
  splitImages.value.splice(index, 1)
}

const clearAll = () => {
  splitImages.value = []
}

const rotateToDirection = (
  pairIndex: number,
  side: 'left' | 'right',
  deltaRotation: 0 | 90 | -90 | 180,
) => {
  const image = splitImages.value[pairIndex]
  if (!image) return

  const currentRotation = side === 'left' ? image.leftRotation : image.rightRotation
  const newRotation = currentRotation + deltaRotation

  if (side === 'left') {
    image.leftRotation = newRotation
  } else {
    image.rightRotation = newRotation
  }
}

const toggleCheck = (pairIndex: number, side: 'left' | 'right') => {
  const image = splitImages.value[pairIndex]
  if (!image) return

  if (side === 'left') {
    image.leftChecked = !image.leftChecked
  } else {
    image.rightChecked = !image.rightChecked
  }
}

// Function to download a single half (reusable)
const downloadHalf = (imageData: string, rotation: number, fileName: string, sideLabel: string) => {
  const img = new Image()
  img.onload = () => {
    const canvas = document.createElement('canvas')
    const ctx = canvas.getContext('2d', { alpha: false })

    if (!ctx) return

    const normalizedRotation = ((rotation % 360) + 360) % 360

    // Set canvas dimensions based on rotation
    if (normalizedRotation === 90 || normalizedRotation === 270) {
      canvas.width = img.height
      canvas.height = img.width
    } else {
      canvas.width = img.width
      canvas.height = img.height
    }

    // Disable image smoothing for better quality
    ctx.imageSmoothingEnabled = false

    // Apply rotation
    ctx.translate(canvas.width / 2, canvas.height / 2)
    ctx.rotate((rotation * Math.PI) / 180)
    ctx.drawImage(img, -img.width / 2, -img.height / 2)

    // Convert to blob and download as PNG for maximum quality
    canvas.toBlob(
      (blob) => {
        if (!blob) return

        const url = URL.createObjectURL(blob)
        const a = document.createElement('a')
        a.href = url
        a.download = `${fileName.replace(/\.[^/.]+$/, '')}_${sideLabel}.jpg`
        a.click()
        URL.revokeObjectURL(url)
      },
      'image/jpeg',
      0.98,
    )
  }

  img.src = imageData
}

// Stop the download process
const stopDownload = () => {
  shouldStopDownload.value = true
}

// Download all checked halves
const downloadChecked = async () => {
  // Count total items to download
  const total = splitImages.value.reduce((count, img) => {
    return count + (img.leftChecked ? 1 : 0) + (img.rightChecked ? 1 : 0)
  }, 0)

  if (total === 0) return

  isDownloading.value = true
  shouldStopDownload.value = false
  downloadProgress.value = { current: 0, total }

  for (let i = 0; i < splitImages.value.length; i++) {
    // Check if user requested to stop
    if (shouldStopDownload.value) {
      break
    }

    const img = splitImages.value[i]
    if (!img) continue

    if (img.leftChecked) {
      if (shouldStopDownload.value) break
      downloadHalf(img.leftHalfOriginal, img.leftRotation, img.originalName, 'left')
      downloadProgress.value.current++
      // Small delay to prevent overwhelming the browser
      await new Promise((resolve) => setTimeout(resolve, 100))
    }

    if (img.rightChecked) {
      if (shouldStopDownload.value) break
      downloadHalf(img.rightHalfOriginal, img.rightRotation, img.originalName, 'right')
      downloadProgress.value.current++
      await new Promise((resolve) => setTimeout(resolve, 100))
    }
  }

  // Keep the progress indicator visible for a moment before hiding
  await new Promise((resolve) => setTimeout(resolve, 500))
  isDownloading.value = false
  shouldStopDownload.value = false
}
</script>

<template>
  <div class="container">
    <div class="header-section">
      <div class="header-top">
        <h1>Half-Frame Film Splitter</h1>
        <a
          href="https://github.com/ovibobi/half-frame-split"
          target="_blank"
          rel="noopener noreferrer"
          class="github-link"
          title="View on GitHub"
        >
          <svg width="24" height="24" viewBox="0 0 24 24" fill="currentColor">
            <path
              d="M12 0c-6.626 0-12 5.373-12 12 0 5.302 3.438 9.8 8.207 11.387.599.111.793-.261.793-.577v-2.234c-3.338.726-4.033-1.416-4.033-1.416-.546-1.387-1.333-1.756-1.333-1.756-1.089-.745.083-.729.083-.729 1.205.084 1.839 1.237 1.839 1.237 1.07 1.834 2.807 1.304 3.492.997.107-.775.418-1.305.762-1.604-2.665-.305-5.467-1.334-5.467-5.931 0-1.311.469-2.381 1.236-3.221-.124-.303-.535-1.524.117-3.176 0 0 1.008-.322 3.301 1.23.957-.266 1.983-.399 3.003-.404 1.02.005 2.047.138 3.006.404 2.291-1.552 3.297-1.23 3.297-1.23.653 1.653.242 2.874.118 3.176.77.84 1.235 1.911 1.235 3.221 0 4.609-2.807 5.624-5.479 5.921.43.372.823 1.102.823 2.222v3.293c0 .319.192.694.801.576 4.765-1.589 8.199-6.086 8.199-11.386 0-6.627-5.373-12-12-12z"
            />
          </svg>
        </a>
      </div>

      <div class="upload-section">
        <!-- Add Images button (normal state) -->
        <label v-if="!isImporting" for="file-input" class="upload-button">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
            <path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z" />
          </svg>
          <span>{{ importButtonText }}</span>
          <input
            id="file-input"
            type="file"
            multiple
            accept="image/*"
            @change="handleFileSelect"
            class="file-input"
          />
        </label>

        <!-- Stop button during import -->
        <button v-if="isImporting" @click="stopImport" class="upload-button importing">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
            <path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z" />
          </svg>
          <span class="progress-text">{{ importButtonText }}</span>
          <div class="progress-bar-container">
            <div
              class="progress-bar-fill"
              :style="{ width: `${(importProgress.current / importProgress.total) * 100}%` }"
            ></div>
          </div>
          <span class="stop-icon">✕</span>
        </button>

        <button v-if="splitImages.length > 0" @click="clearAll" class="clear-button">
          Remove All
        </button>

        <!-- Download button with integrated progress -->
        <button
          v-if="splitImages.length > 0 && !isDownloading"
          @click="downloadChecked"
          :disabled="!hasCheckedItems"
          class="download-button"
        >
          <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
            <path d="M19 9h-4V3H9v6H5l7 7 7-7zM5 18v2h14v-2H5z" />
          </svg>
          <span>{{ downloadButtonText }}</span>
        </button>

        <!-- Stop button during download -->
        <button
          v-if="splitImages.length > 0 && isDownloading"
          @click="stopDownload"
          class="download-button downloading"
        >
          <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
            <path d="M19 9h-4V3H9v6H5l7 7 7-7zM5 18v2h14v-2H5z" />
          </svg>
          <span class="progress-text">{{ downloadButtonText }}</span>
          <div class="progress-bar-container">
            <div
              class="progress-bar-fill"
              :style="{ width: `${(downloadProgress.current / downloadProgress.total) * 100}%` }"
            ></div>
          </div>
          <span class="stop-icon">✕</span>
        </button>
      </div>

      <div v-if="splitImages.length > 0" class="images-info">
        <label class="checkbox-label">
          <input type="checkbox" v-model="allChecked" />
          <span>Select All</span>
        </label>
        <p>{{ splitImages.length }} frame(s) split into {{ splitImages.length * 2 }} images</p>

        <div class="zoom-control">
          <label for="zoom-slider">
            <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
              <path d="M3 6h10v4H3z" />
            </svg>
          </label>
          <input
            id="zoom-slider"
            type="range"
            min="25"
            max="100"
            step="5"
            v-model.number="zoomLevel"
            class="zoom-slider"
          />
          <label>
            <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
              <path d="M3 6h4v10H3zM9 6h4v10H9z" />
            </svg>
          </label>
        </div>
      </div>
    </div>

    <div class="preview-grid">
      <FrameView
        v-for="(imagePair, index) in splitImages"
        :key="`${imagePair.originalName}-${index}`"
        :image-pair="imagePair"
        :index="index"
        :max-image-height="maxImageHeight"
        @remove="removeImagePair"
        @rotate="rotateToDirection"
        @toggle-check="toggleCheck"
        @download="
          (side: 'left' | 'right') =>
            downloadHalf(
              side === 'left' ? imagePair.leftHalfOriginal : imagePair.rightHalfOriginal,
              side === 'left' ? imagePair.leftRotation : imagePair.rightRotation,
              imagePair.originalName,
              side,
            )
        "
      />
    </div>
  </div>
</template>

<style>
/* Global styles to prevent double scrollbar */
html,
body {
  margin: 0;
  padding: 0;
  height: 100vh;
  overflow: hidden;
}

#app {
  height: 100vh;
}
</style>

<style scoped>
.container {
  margin: 0;
  padding: 0;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #1a1a1a;
}

.header-section {
  padding: 2rem 2.5rem;
  background-color: #0a0a0a;
  border-bottom: 1px solid #2a2a2a;
  flex-shrink: 0;
  box-shadow: 0 2px 16px rgba(0, 0, 0, 0.3);
}

h1 {
  color: #f5f5f5;
  margin: 0 0 1.5rem 0;
  font-size: 1.75rem;
  font-weight: 600;
  letter-spacing: 0.05em;
  text-transform: uppercase;
  font-family: 'Courier New', monospace;
}

.header-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.github-link {
  color: #a3a3a3;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0.5rem;
  border-radius: 2px;
}

.github-link:hover {
  color: #d4a574;
  background-color: rgba(212, 165, 116, 0.1);
  transform: translateY(-1px);
}

.upload-section {
  display: flex;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
}

.upload-button {
  display: inline-block;
  padding: 0.625rem 1.25rem;
  background-color: #d4a574;
  color: #0a0a0a;
  border-radius: 2px;
  cursor: pointer;
  font-weight: 600;
  font-size: 0.875rem;
  transition: all 0.2s ease;
  border: none;
  letter-spacing: 0.025em;
  text-transform: uppercase;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  position: relative;
  overflow: hidden;
}

.upload-button.importing {
  background-color: #d4a574;
  color: #0a0a0a;
  cursor: pointer;
}

.upload-button.importing:hover {
  background-color: #c09563;
}

.upload-button.importing .stop-icon {
  margin-left: auto;
  font-weight: bold;
  font-size: 1rem;
}

.upload-button:hover {
  background-color: #e6b886;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(212, 165, 116, 0.3);
}

.upload-button .progress-bar-container {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 3px;
  background-color: rgba(0, 0, 0, 0.2);
  overflow: hidden;
}

.upload-button .progress-bar-fill {
  height: 100%;
  background-color: rgba(0, 0, 0, 0.3);
  transition: width 0.2s ease;
}

.upload-button .progress-text {
  font-family: 'Courier New', monospace;
  font-weight: 600;
}

.file-input {
  display: none;
}

.clear-button {
  padding: 0.625rem 1.25rem;
  background-color: transparent;
  color: #dc2626;
  border: 1.5px solid #7f1d1d;
  border-radius: 2px;
  cursor: pointer;
  font-weight: 600;
  font-size: 0.875rem;
  transition: all 0.2s ease;
  letter-spacing: 0.025em;
  text-transform: uppercase;
}

.clear-button:hover {
  background-color: #450a0a;
  border-color: #dc2626;
  transform: translateY(-1px);
}

.download-button {
  padding: 0.625rem 1.25rem;
  background-color: #4a5568;
  color: #f5f5f5;
  border: none;
  border-radius: 2px;
  cursor: pointer;
  font-weight: 600;
  font-size: 0.875rem;
  transition: all 0.2s ease;
  letter-spacing: 0.025em;
  text-transform: uppercase;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  position: relative;
  overflow: hidden;
}

.download-button.downloading {
  background-color: #d4a574;
  color: #0a0a0a;
  cursor: pointer;
}

.download-button.downloading:hover {
  background-color: #c09563;
}

.download-button.downloading .stop-icon {
  margin-left: auto;
  font-weight: bold;
  font-size: 1rem;
}

.download-button:hover:not(:disabled) {
  background-color: #5a657a;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(74, 85, 104, 0.3);
}

.download-button:disabled {
  background-color: #2a2a2a;
  color: #666;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.download-button .progress-bar-container {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 3px;
  background-color: rgba(0, 0, 0, 0.2);
  overflow: hidden;
}

.download-button .progress-bar-fill {
  height: 100%;
  background-color: rgba(0, 0, 0, 0.3);
  transition: width 0.2s ease;
}

.download-button .progress-text {
  font-family: 'Courier New', monospace;
  font-weight: 600;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  cursor: pointer;
  user-select: none;
  font-size: 0.875rem;
  color: #a3a3a3;
  font-weight: 500;
  letter-spacing: 0.025em;
}

.checkbox-label input[type='checkbox'] {
  cursor: pointer;
  width: 17px;
  height: 17px;
  accent-color: #d4a574;
}

.images-info {
  margin-bottom: 0;
  color: #737373;
  display: flex;
  align-items: center;
  gap: 2rem;
  font-size: 0.875rem;
}

.images-info p {
  margin: 0;
  color: #a3a3a3;
  font-family: 'Courier New', monospace;
  letter-spacing: 0.025em;
}

.zoom-control {
  display: flex;
  align-items: center;
  gap: 0.625rem;
  color: #737373;
}

.zoom-slider {
  width: 160px;
  cursor: pointer;
  accent-color: #d4a574;
}

.preview-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 1.25rem;
  padding: 2rem 2.5rem;
  overflow-y: auto;
  flex: 1;
  align-content: flex-start;
}

/* Mobile Responsive Styles */
@media (max-width: 768px) {
  .header-section {
    padding: 1.25rem 1rem;
  }

  h1 {
    font-size: 1.25rem;
    margin-bottom: 1rem;
  }

  .upload-section {
    flex-wrap: wrap;
    gap: 0.5rem;
  }

  .upload-button,
  .clear-button,
  .download-button {
    padding: 0.75rem 1rem;
    font-size: 0.8125rem;
    flex: 1;
    min-width: 0;
  }

  .images-info {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.75rem;
  }

  .zoom-control {
    width: 100%;
  }

  .zoom-slider {
    flex: 1;
  }

  .preview-grid {
    padding: 1rem;
    gap: 1rem;
  }
}

@media (max-width: 480px) {
  .header-section {
    padding: 1rem 0.75rem;
  }

  h1 {
    font-size: 1.125rem;
  }

  .upload-button,
  .clear-button,
  .download-button {
    width: 100%;
    flex: none;
  }

  .preview-grid {
    padding: 0.75rem;
    gap: 0.75rem;
  }

  .images-info p {
    font-size: 0.75rem;
  }
}
</style>
