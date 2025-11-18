<script setup lang="ts">
import { defineProps, ref, computed, onMounted, onUnmounted } from 'vue'
import HalfFrame from './HalfFrame.vue'

interface Props {
  imagePair: {
    leftHalf: string
    rightHalf: string
    leftHalfOriginal: string
    rightHalfOriginal: string
    originalName: string
    leftRotation: number
    rightRotation: number
    leftChecked: boolean
    rightChecked: boolean
  }
  index: number
  maxImageHeight: number
}

const props = defineProps<Props>()

const emit = defineEmits<{
  remove: [index: number]
  rotate: [index: number, side: 'left' | 'right', deltaRotation: 0 | 90 | -90 | 180]
  download: [side: 'left' | 'right']
  toggleCheck: [index: number, side: 'left' | 'right']
}>()

const fullscreenSide = ref<'left' | 'right' | null>(null)
const zoomLevel = ref(1)
const panX = ref(0)
const panY = ref(0)

const toggleFullscreen = (side: 'left' | 'right') => {
  if (fullscreenSide.value === side) {
    fullscreenSide.value = null
    zoomLevel.value = 1
    panX.value = 0
    panY.value = 0
  } else {
    fullscreenSide.value = side
    zoomLevel.value = 1
    panX.value = 0
    panY.value = 0
  }
}

const fullscreenImageSrc = computed(() => {
  return fullscreenSide.value === 'left' ? props.imagePair.leftHalf : props.imagePair.rightHalf
})

const fullscreenImageStyle = computed(() => {
  const rotation =
    fullscreenSide.value === 'left' ? props.imagePair.leftRotation : props.imagePair.rightRotation
  return {
    transform: `translate(${panX.value}px, ${panY.value}px) rotate(${rotation}deg) scale(${zoomLevel.value})`,
  }
})

const toggleCheckbox = (side: 'left' | 'right') => {
  emit('toggleCheck', props.index, side)
}

const handleEscape = (event: KeyboardEvent) => {
  if (event.key === 'Escape' && fullscreenSide.value !== null) {
    fullscreenSide.value = null
    zoomLevel.value = 1
    panX.value = 0
    panY.value = 0
  }
}

const handleWheel = (event: WheelEvent) => {
  if (fullscreenSide.value !== null) {
    event.preventDefault()

    const delta = event.deltaY > 0 ? -0.1 : 0.1
    const newZoom = Math.max(0.5, Math.min(5, zoomLevel.value + delta))

    if (newZoom !== zoomLevel.value) {
      // Get mouse position relative to viewport center
      const mouseX = event.clientX - window.innerWidth / 2
      const mouseY = event.clientY - window.innerHeight / 2

      // Calculate the zoom factor change
      const zoomFactor = newZoom / zoomLevel.value

      // Adjust pan to zoom towards mouse position
      // The point under the mouse should stay in the same place
      panX.value = mouseX + (panX.value - mouseX) * zoomFactor
      panY.value = mouseY + (panY.value - mouseY) * zoomFactor

      zoomLevel.value = newZoom
    }
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleEscape)
  window.addEventListener('wheel', handleWheel, { passive: false })
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleEscape)
  window.removeEventListener('wheel', handleWheel)
})
</script>

<template>
  <div class="frame-view">
    <div class="pair-header">
      <span class="pair-name">{{ imagePair.originalName }}</span>
      <button @click="emit('remove', index)" class="remove-button">✕</button>
    </div>
    <div class="split-container">
      <HalfFrame
        :image-src="props.imagePair.leftHalf"
        :rotation="props.imagePair.leftRotation"
        :checked="props.imagePair.leftChecked"
        label="Left"
        side="left"
        :index="props.index"
        :max-image-height="props.maxImageHeight"
        @rotate="(idx, side, delta) => emit('rotate', idx, side, delta)"
        @toggle-fullscreen="toggleFullscreen"
        @download="emit('download', 'left')"
        @toggle-check="toggleCheckbox"
      />
      <HalfFrame
        :image-src="props.imagePair.rightHalf"
        :rotation="props.imagePair.rightRotation"
        :checked="props.imagePair.rightChecked"
        label="Right"
        side="right"
        :index="props.index"
        :max-image-height="props.maxImageHeight"
        @rotate="(idx, side, delta) => emit('rotate', idx, side, delta)"
        @toggle-fullscreen="toggleFullscreen"
        @download="emit('download', 'right')"
        @toggle-check="toggleCheckbox"
      />
    </div>
  </div>

  <!-- Fullscreen overlay - using Teleport to render at body level -->
  <Teleport to="body">
    <div v-if="fullscreenSide" class="fullscreen-overlay" @click="toggleFullscreen(fullscreenSide)">
      <img
        :src="fullscreenImageSrc"
        :style="fullscreenImageStyle"
        alt="Fullscreen view"
        class="fullscreen-image"
      />

      <!-- Directional rotation buttons for fullscreen -->
      <div class="fullscreen-rotation-controls">
        <button
          class="fullscreen-direction-btn fullscreen-direction-top"
          @click.stop="emit('rotate', props.index, fullscreenSide, 0)"
          title="Rotate top up"
        >
          <svg width="24" height="24" viewBox="0 0 16 16" fill="currentColor">
            <path d="M8 2L4 6h8z" />
          </svg>
        </button>
        <button
          class="fullscreen-direction-btn fullscreen-direction-right"
          @click.stop="emit('rotate', props.index, fullscreenSide, -90)"
          title="Rotate right up"
        >
          <svg width="24" height="24" viewBox="0 0 16 16" fill="currentColor">
            <path d="M10 8L6 4v8z" />
          </svg>
        </button>
        <button
          class="fullscreen-direction-btn fullscreen-direction-bottom"
          @click.stop="emit('rotate', props.index, fullscreenSide, 180)"
          title="Rotate bottom up"
        >
          <svg width="24" height="24" viewBox="0 0 16 16" fill="currentColor">
            <path d="M8 14L12 10H4z" />
          </svg>
        </button>
        <button
          class="fullscreen-direction-btn fullscreen-direction-left"
          @click.stop="emit('rotate', props.index, fullscreenSide, 90)"
          title="Rotate left up"
        >
          <svg width="24" height="24" viewBox="0 0 16 16" fill="currentColor">
            <path d="M6 8L10 12V4z" />
          </svg>
        </button>
      </div>

      <!-- Download button for fullscreen -->
      <button
        class="fullscreen-download-btn"
        @click.stop="emit('download', fullscreenSide)"
        title="Download this image"
      >
        <svg width="24" height="24" viewBox="0 0 24 24" fill="currentColor">
          <path d="M19 9h-4V3H9v6H5l7 7 7-7zM5 18v2h14v-2H5z" />
        </svg>
      </button>
    </div>
  </Teleport>
</template>

<style scoped>
.frame-view {
  border: 1px solid #2a2a2a;
  border-radius: 2px;
  overflow: hidden;
  background-color: #0a0a0a;
  transition:
    transform 0.2s,
    box-shadow 0.2s;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.5);
}

.frame-view:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.6);
  border-color: #3a3a3a;
}

.pair-header {
  padding: 0.875rem 1rem;
  background-color: #141414;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #2a2a2a;
}

.pair-name {
  font-size: 0.8125rem;
  color: #d4a574;
  font-weight: 500;
  letter-spacing: 0.05em;
  font-family: 'Courier New', monospace;
}

.split-container {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1px;
  background-color: #2a2a2a;
}

.remove-button {
  background-color: transparent;
  color: #dc2626;
  border: 1px solid #7f1d1d;
  border-radius: 2px;
  width: 28px;
  height: 28px;
  cursor: pointer;
  font-size: 1rem;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.remove-button:hover {
  background-color: #450a0a;
  border-color: #dc2626;
  transform: scale(1.05);
}
</style>

<style>
/* Non-scoped styles for teleported fullscreen overlay */
.fullscreen-overlay {
  position: fixed;
  inset: 0;
  background-color: rgba(0, 0, 0, 0.98);
  backdrop-filter: blur(8px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10000;
  cursor: pointer;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.fullscreen-image {
  max-width: min(95vw, 95vh);
  max-height: min(95vw, 95vh);
  width: auto;
  height: auto;
  object-fit: contain;
  transition: transform 0.3s ease;
}

.fullscreen-rotation-controls {
  position: fixed;
  inset: 0;
  pointer-events: none;
}

.fullscreen-direction-btn {
  position: absolute;
  background-color: rgba(212, 165, 116, 0.9);
  color: #0a0a0a;
  border: none;
  width: 48px;
  height: 48px;
  border-radius: 2px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
  pointer-events: all;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.4);
}

.fullscreen-direction-btn:hover {
  background-color: rgba(230, 184, 134, 1);
  transform: scale(1.05);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.5);
}

.fullscreen-direction-top {
  top: 40px;
  left: 50%;
  transform: translateX(-50%);
}

.fullscreen-direction-top:hover {
  transform: translateX(-50%) scale(1.1);
}

.fullscreen-direction-right {
  top: 50%;
  right: 40px;
  transform: translateY(-50%);
}

.fullscreen-direction-right:hover {
  transform: translateY(-50%) scale(1.1);
}

.fullscreen-direction-bottom {
  bottom: 40px;
  left: 50%;
  transform: translateX(-50%);
}

.fullscreen-direction-bottom:hover {
  transform: translateX(-50%) scale(1.1);
}

.fullscreen-direction-left {
  top: 50%;
  left: 40px;
  transform: translateY(-50%);
}

.fullscreen-direction-left:hover {
  transform: translateY(-50%) scale(1.1);
}

.fullscreen-download-btn {
  position: fixed;
  top: 40px;
  right: 40px;
  background-color: rgba(212, 165, 116, 0.9);
  color: #0a0a0a;
  border: none;
  width: 48px;
  height: 48px;
  border-radius: 2px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
  pointer-events: all;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.4);
  z-index: 10001;
}

.fullscreen-download-btn:hover {
  background-color: rgba(230, 184, 134, 1);
  transform: scale(1.05);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.5);
}

/* Mobile Responsive Styles */
@media (max-width: 768px) {
  .pair-header {
    padding: 0.75rem 0.875rem;
  }

  .pair-name {
    font-size: 0.75rem;
  }

  .remove-button {
    width: 32px;
    height: 32px;
    font-size: 1.125rem;
  }

  .fullscreen-direction-btn {
    width: 56px;
    height: 56px;
  }

  .fullscreen-download-btn {
    width: 56px;
    height: 56px;
  }

  .fullscreen-direction-top,
  .fullscreen-download-btn {
    top: 20px;
  }

  .fullscreen-direction-right {
    right: 20px;
  }

  .fullscreen-direction-bottom {
    bottom: 20px;
  }

  .fullscreen-direction-left {
    left: 20px;
  }

  .fullscreen-download-btn {
    right: 20px;
  }
}

@media (max-width: 480px) {
  .pair-header {
    padding: 0.625rem 0.75rem;
  }

  .pair-name {
    font-size: 0.6875rem;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .remove-button {
    width: 28px;
    height: 28px;
  }
}
</style>
