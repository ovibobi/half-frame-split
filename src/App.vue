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
  const count = checkedCount.value
  if (count === 0) return 'Download Selected'
  return `Download ${count} Selected`
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
        const leftCtx = leftCanvas.getContext('2d')
        if (leftCtx) {
          leftCtx.drawImage(img, 0, 0, halfWidth, height, 0, 0, halfWidth, height)
        }

        // Create canvas for right half
        const rightCanvas = document.createElement('canvas')
        rightCanvas.width = halfWidth
        rightCanvas.height = height
        const rightCtx = rightCanvas.getContext('2d')
        if (rightCtx) {
          rightCtx.drawImage(img, halfWidth, 0, halfWidth, height, 0, 0, halfWidth, height)
        }

        resolve({
          leftHalf: leftCanvas.toDataURL('image/jpeg', 0.92),
          rightHalf: rightCanvas.toDataURL('image/jpeg', 0.92),
          leftHalfOriginal: leftCanvas.toDataURL('image/jpeg', 0.95),
          rightHalfOriginal: rightCanvas.toDataURL('image/jpeg', 0.95),
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
        const topCtx = topCanvas.getContext('2d')
        if (topCtx) {
          topCtx.drawImage(img, 0, 0, width, halfHeight, 0, 0, width, halfHeight)
        }

        // Create canvas for bottom half
        const bottomCanvas = document.createElement('canvas')
        bottomCanvas.width = width
        bottomCanvas.height = halfHeight
        const bottomCtx = bottomCanvas.getContext('2d')
        if (bottomCtx) {
          bottomCtx.drawImage(img, 0, halfHeight, width, halfHeight, 0, 0, width, halfHeight)
        }

        resolve({
          leftHalf: topCanvas.toDataURL('image/jpeg', 0.92),
          rightHalf: bottomCanvas.toDataURL('image/jpeg', 0.92),
          leftHalfOriginal: topCanvas.toDataURL('image/jpeg', 0.95),
          rightHalfOriginal: bottomCanvas.toDataURL('image/jpeg', 0.95),
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

const handleFileSelect = async (event: Event) => {
  const target = event.target as HTMLInputElement
  const files = target.files

  if (files) {
    const fileArray = Array.from(files).filter((file) => file.type.startsWith('image/'))

    // Process files one at a time with small delays to keep UI responsive
    for (let i = 0; i < fileArray.length; i++) {
      const file = fileArray[i]
      if (!file) continue

      const reader = new FileReader()

      reader.onload = async (e) => {
        const imageData = e.target?.result as string
        const split = await splitImageInHalf(imageData, file.name)
        splitImages.value.push(split)
      }

      reader.readAsDataURL(file) // Allow UI to breathe between processing files
      if (i % 2 === 1) {
        await new Promise((resolve) => setTimeout(resolve, 0))
      }
    }
  }
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
    const ctx = canvas.getContext('2d')

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

    // Apply rotation
    ctx.translate(canvas.width / 2, canvas.height / 2)
    ctx.rotate((rotation * Math.PI) / 180)
    ctx.drawImage(img, -img.width / 2, -img.height / 2)

    // Convert to blob and download
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
      0.95,
    )
  }

  img.src = imageData
}

// Download all checked halves
const downloadChecked = async () => {
  for (let i = 0; i < splitImages.value.length; i++) {
    const img = splitImages.value[i]
    if (!img) continue

    if (img.leftChecked) {
      downloadHalf(img.leftHalfOriginal, img.leftRotation, img.originalName, 'left')
      // Small delay to prevent overwhelming the browser
      await new Promise((resolve) => setTimeout(resolve, 100))
    }

    if (img.rightChecked) {
      downloadHalf(img.rightHalfOriginal, img.rightRotation, img.originalName, 'right')
      await new Promise((resolve) => setTimeout(resolve, 100))
    }
  }
}
</script>

<template>
  <div class="container">
    <div class="header-section">
      <h1>Half-Frame Film Splitter</h1>

      <div class="upload-section">
        <label for="file-input" class="upload-button">
          <span>Add Images</span>
          <input
            id="file-input"
            type="file"
            multiple
            accept="image/*"
            @change="handleFileSelect"
            class="file-input"
          />
        </label>

        <button v-if="splitImages.length > 0" @click="clearAll" class="clear-button">
          Clear All
        </button>
        <button
          v-if="splitImages.length > 0"
          @click="downloadChecked"
          :disabled="!hasCheckedItems"
          class="download-button"
        >
          {{ downloadButtonText }}
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
}

.upload-button:hover {
  background-color: #e6b886;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(212, 165, 116, 0.3);
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
</style>
