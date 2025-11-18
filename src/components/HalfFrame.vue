<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  imageSrc: string
  rotation: number
  checked: boolean
  label: string
  side: 'left' | 'right'
  index: number
  maxImageHeight: number
}

const props = defineProps<Props>()

const emit = defineEmits<{
  rotate: [index: number, side: 'left' | 'right', deltaRotation: 0 | 90 | -90 | 180]
  toggleFullscreen: [side: 'left' | 'right']
  download: [side: 'left' | 'right']
  toggleCheck: [side: 'left' | 'right']
}>()

const imageStyle = computed(() => {
  return {
    transform: `rotate(${props.rotation}deg)`,
  }
})

const containerStyle = computed(() => {
  return { maxHeight: `${props.maxImageHeight}px` }
})
</script>

<template>
  <div class="split-half" :style="containerStyle" @click="emit('toggleFullscreen', side)">
    <img :src="imageSrc" :alt="`${label} half`" :style="imageStyle" />
    <span class="half-label">{{ label }}</span>

    <!-- Checkbox -->
    <label class="checkbox-container" @click.stop>
      <input type="checkbox" :checked="checked" @change="emit('toggleCheck', side)" />
    </label>

    <!-- Download button -->
    <button class="download-btn" @click.stop="emit('download', side)" title="Download this image">
      <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
        <path d="M19 9h-4V3H9v6H5l7 7 7-7zM5 18v2h14v-2H5z" />
      </svg>
    </button>

    <!-- Directional rotation buttons -->
    <div class="rotation-controls">
      <button
        class="direction-btn direction-top"
        @click.stop="emit('rotate', index, side, 0)"
        title="Rotate top up"
      >
        <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
          <path d="M8 2L4 6h8z" />
        </svg>
      </button>
      <button
        class="direction-btn direction-right"
        @click.stop="emit('rotate', index, side, -90)"
        title="Rotate right up"
      >
        <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
          <path d="M10 8L6 4v8z" />
        </svg>
      </button>
      <button
        class="direction-btn direction-bottom"
        @click.stop="emit('rotate', index, side, 180)"
        title="Rotate bottom up"
      >
        <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
          <path d="M8 14L12 10H4z" />
        </svg>
      </button>
      <button
        class="direction-btn direction-left"
        @click.stop="emit('rotate', index, side, 90)"
        title="Rotate left up"
      >
        <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
          <path d="M6 8L10 12V4z" />
        </svg>
      </button>
    </div>
  </div>
</template>

<style scoped>
.split-half {
  position: relative;
  background-color: #0a0a0a;
  cursor: pointer;
  overflow: visible;
  display: flex;
  align-items: center;
  justify-content: center;
  aspect-ratio: 1 / 1;
}

.split-half:hover .rotation-controls {
  opacity: 1;
}

.checkbox-container {
  position: absolute;
  top: 0.5rem;
  left: 0.5rem;
  cursor: pointer;
  z-index: 3;
  background-color: rgba(20, 20, 20, 0.9);
  padding: 0.375rem;
  border-radius: 2px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  transition: all 0.2s ease;
  border: 1px solid #2a2a2a;
}

.checkbox-container:hover {
  background-color: rgba(30, 30, 30, 1);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
  border-color: #3a3a3a;
}

.checkbox-container input[type='checkbox'] {
  cursor: pointer;
  width: 17px;
  height: 17px;
  margin: 0;
  accent-color: #d4a574;
}

.split-half img {
  max-width: 100%;
  max-height: 100%;
  width: auto;
  height: auto;
  display: block;
  transition: transform 0.3s ease;
  will-change: transform;
}

.download-btn {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;
  background-color: rgba(212, 165, 116, 0.9);
  color: #0a0a0a;
  border: none;
  width: 32px;
  height: 32px;
  border-radius: 2px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transition: all 0.2s ease;
  pointer-events: all;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  z-index: 10;
}

.split-half:hover .download-btn {
  opacity: 1;
}

.download-btn:hover {
  background-color: rgba(230, 184, 134, 1);
  transform: scale(1.05);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
}

.rotation-controls {
  position: absolute;
  inset: 0;
  pointer-events: none;
  opacity: 0;
  transition: opacity 0.3s ease;
}

.direction-btn {
  position: absolute;
  background-color: rgba(212, 165, 116, 0.9);
  color: #0a0a0a;
  border: none;
  width: 32px;
  height: 32px;
  border-radius: 2px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
  pointer-events: all;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

.direction-btn:hover {
  background-color: rgba(230, 184, 134, 1);
  transform: scale(1.05);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
}

.direction-top {
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
}

.direction-top:hover {
  transform: translateX(-50%) scale(1.1);
}

.direction-right {
  top: 50%;
  right: 10px;
  transform: translateY(-50%);
}

.direction-right:hover {
  transform: translateY(-50%) scale(1.1);
}

.direction-bottom {
  bottom: 10px;
  left: 50%;
  transform: translateX(-50%);
}

.direction-bottom:hover {
  transform: translateX(-50%) scale(1.1);
}

.direction-left {
  top: 50%;
  left: 10px;
  transform: translateY(-50%);
}

.direction-left:hover {
  transform: translateY(-50%) scale(1.1);
}

.half-label {
  position: absolute;
  bottom: 0.5rem;
  left: 0.5rem;
  background-color: rgba(20, 20, 20, 0.9);
  color: #d4a574;
  padding: 0.375rem 0.625rem;
  border-radius: 2px;
  font-size: 0.6875rem;
  font-weight: 600;
  z-index: 2;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  letter-spacing: 0.1em;
  text-transform: uppercase;
  font-family: 'Courier New', monospace;
  border: 1px solid #2a2a2a;
}
</style>
