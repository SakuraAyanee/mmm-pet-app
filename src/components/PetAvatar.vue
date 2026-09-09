<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
import { isTauri } from '@tauri-apps/api/core'
import { cursorPosition, getCurrentWindow } from '@tauri-apps/api/window'
import type { PetState } from '../types/pet'
import mamimiImage from '../assets/characters/mamimi/mamimi-cutout.png'

const petState = ref<PetState>({
  x: 0,
  y: 0,
  mood: 'idle',
  })
  
const petName = ref('MMM')
const affection = ref(0)
const petColor = ref('hsl(32 100% 78%)')
const petImage = ref<HTMLImageElement | null>(null)
const dragHandle = ref<HTMLButtonElement | null>(null)
const isPointerOverPet = ref(false)

const appWindow = getCurrentWindow()
const alphaThreshold = 12
let alphaPixels: Uint8ClampedArray | null = null
let imageWidth = 0
let imageHeight = 0
let pointerTimer: ReturnType<typeof setInterval> | undefined
let isCheckingPointer = false
let ignoresCursorEvents = false
let isDraggingWindow = false

function movePet() {
  const distance = 120

  petState.value.x = Math.round((Math.random() * 2 - 1) * distance)
  petState.value.y = Math.round((Math.random() * 2 - 1) * distance)
  petState.value.mood = 'happy'
  petColor.value = `hsl(${Math.floor(Math.random() * 360)} 75% 78%)`
  affection.value += 1
}

async function startWindowDrag() {
  isDraggingWindow = true
  await appWindow.startDragging()
}

function preparePetAlphaMap() {
  const image = petImage.value

  if (!image || image.naturalWidth === 0 || image.naturalHeight === 0) {
    return
  }

  const canvas = document.createElement('canvas')
  canvas.width = image.naturalWidth
  canvas.height = image.naturalHeight

  const context = canvas.getContext('2d', { willReadFrequently: true })
  if (!context) {
    return
  }

  context.drawImage(image, 0, 0)
  alphaPixels = context.getImageData(0, 0, canvas.width, canvas.height).data
  imageWidth = canvas.width
  imageHeight = canvas.height
}

function isPointInsideElement(
  element: HTMLElement | null,
  x: number,
  y: number,
) {
  if (!element) {
    return false
  }

  const rect = element.getBoundingClientRect()
  return x >= rect.left && x <= rect.right && y >= rect.top && y <= rect.bottom
}

function isPointOnOpaquePetPixel(x: number, y: number) {
  const image = petImage.value
  if (!image || !alphaPixels || imageWidth === 0 || imageHeight === 0) {
    return false
  }

  const rect = image.getBoundingClientRect()
  const scale = Math.min(rect.width / imageWidth, rect.height / imageHeight)
  const renderedWidth = imageWidth * scale
  const renderedHeight = imageHeight * scale
  const renderedLeft = rect.left + (rect.width - renderedWidth) / 2
  const renderedTop = rect.top + (rect.height - renderedHeight) / 2

  if (
    x < renderedLeft ||
    x >= renderedLeft + renderedWidth ||
    y < renderedTop ||
    y >= renderedTop + renderedHeight
  ) {
    return false
  }

  const pixelX = Math.floor((x - renderedLeft) / scale)
  const pixelY = Math.floor((y - renderedTop) / scale)
  const alphaIndex = (pixelY * imageWidth + pixelX) * 4 + 3

  return alphaPixels[alphaIndex] > alphaThreshold
}

async function updateCursorEventMode() {
  if (isCheckingPointer || !alphaPixels) {
    return
  }

  isCheckingPointer = true

  try {
    const [pointer, windowPosition, scaleFactor] = await Promise.all([
      cursorPosition(),
      appWindow.innerPosition(),
      appWindow.scaleFactor(),
    ])
    const x = (pointer.x - windowPosition.x) / scaleFactor
    const y = (pointer.y - windowPosition.y) / scaleFactor
    const receivesCursorEvents =
      isDraggingWindow ||
      isPointOnOpaquePetPixel(x, y) ||
      isPointInsideElement(dragHandle.value, x, y)

    isPointerOverPet.value = receivesCursorEvents && !isDraggingWindow

    const shouldIgnoreCursorEvents = !receivesCursorEvents
    if (shouldIgnoreCursorEvents !== ignoresCursorEvents) {
      await appWindow.setIgnoreCursorEvents(shouldIgnoreCursorEvents)
      ignoresCursorEvents = shouldIgnoreCursorEvents
    }
  } finally {
    isCheckingPointer = false
  }
}

function stopWindowDrag() {
  isDraggingWindow = false
}

onMounted(() => {
  if (!isTauri()) {
    return
  }

  preparePetAlphaMap()
  window.addEventListener('mouseup', stopWindowDrag)
  void updateCursorEventMode()
  pointerTimer = setInterval(() => void updateCursorEventMode(), 80)
})

onUnmounted(() => {
  if (!isTauri()) {
    return
  }

  if (pointerTimer) {
    clearInterval(pointerTimer)
  }

  window.removeEventListener('mouseup', stopWindowDrag)

  if (ignoresCursorEvents) {
    void appWindow.setIgnoreCursorEvents(false)
  }
})
</script>

<template>
  <section
    class="pet-avatar"
    :class="{ 'pet-avatar--interactive': isPointerOverPet }"
    aria-label="桌面宠物"
  >
    <button
      ref="dragHandle"
      type="button"
      class="pet-avatar__drag-handle"
      aria-label="拖动窗口"
      title="拖动窗口"
      @mousedown.left.prevent="startWindowDrag"
    >
      <span></span>
      <span></span>
      <span></span>
    </button>

    <button
      type="button"
      class="pet-avatar__body"
      :style="{
        transform: `translate(${petState.x}px, ${petState.y}px)`,
      }"
      aria-label="点击移动宠物"
      @click="movePet"
    >
      <img
        ref="petImage"
        :src="mamimiImage"
        :alt="`${petName}，当前情绪：${petState.mood}`"
        @load="preparePetAlphaMap"
      />
    </button>
    
    <p>{{ petName }}</p>
    <p class="pet-avatar__mood" :style="{ color: petColor }">
      当前情绪：{{ petState.mood }}
    </p>

    <p>被摸次数：{{ affection }}</p>
  </section>
</template>

<style scoped>
.pet-avatar {
  position: relative;
  display: grid;
  justify-items: center;
  gap: 0.75rem;
}

.pet-avatar__drag-handle {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;
  z-index: 1;
  display: grid;
  width: 2.25rem;
  height: 2.25rem;
  padding: 0.5rem;
  gap: 0.25rem;
  border: 1px solid rgb(255 255 255 / 55%);
  border-radius: 0.5rem;
  place-content: center;
  background: rgb(35 22 48 / 72%);
  box-shadow: 0 0.25rem 0.75rem rgb(0 0 0 / 18%);
  cursor: grab;
  opacity: 0;
  pointer-events: none;
  visibility: hidden;
  transform: translateY(-0.4rem) scale(0.92);
  transition:
    opacity 160ms ease,
    transform 180ms cubic-bezier(0.22, 1, 0.36, 1),
    visibility 0s linear 160ms;
}

.pet-avatar__drag-handle:active {
  cursor: grabbing;
}

.pet-avatar__drag-handle span {
  display: block;
  width: 1rem;
  height: 0.125rem;
  border-radius: 999px;
  background: #fff;
}

.pet-avatar--interactive .pet-avatar__drag-handle,
.pet-avatar:focus-within .pet-avatar__drag-handle {
  opacity: 1;
  pointer-events: auto;
  visibility: visible;
  transform: translateY(0) scale(1);
  transition-delay: 180ms, 180ms, 180ms;
}

.pet-avatar__body {
  border: 0;
  padding: 0;
  background: transparent;
  cursor: pointer;
  line-height: 0;
  transition: transform 300ms ease;
}

.pet-avatar__body img {
  display: block;
  width: min(72vw, 22rem);
  height: min(68vh, 31rem);
  object-fit: contain;
  filter: drop-shadow(0 0.75rem 0.75rem rgb(0 0 0 / 20%));
  transition: filter 200ms ease;
}

.pet-avatar__body:hover img {
  filter: drop-shadow(0 1rem 1rem rgb(0 0 0 / 28%));
}

p {
  margin: 0;
}
</style>
