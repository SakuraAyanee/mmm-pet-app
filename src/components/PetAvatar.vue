<script setup lang="ts">
import { computed, nextTick, onMounted, onUnmounted, ref } from 'vue'
import { isTauri } from '@tauri-apps/api/core'
import { cursorPosition, getCurrentWindow } from '@tauri-apps/api/window'
import mamimiImage from '../assets/characters/mamimi/mamimi-cutout.png'
import SpineAvatar from './SpineAvatar.vue'

const hitMaskImage = ref<HTMLImageElement | null>(null)
const petBody = ref<HTMLElement | null>(null)
const petAvatar = ref<HTMLElement | null>(null)
const dragHandle = ref<HTMLButtonElement | null>(null)
const contextMenu = ref<HTMLElement | null>(null)
const resizeHandle = ref<HTMLButtonElement | null>(null)
const resizeResetButton = ref<HTMLButtonElement | null>(null)
const resizeConfirmButton = ref<HTMLButtonElement | null>(null)
const isDragHandleVisible = ref(false)
const isContextMenuOpen = ref(false)
const contextMenuPosition = ref({ x: 0, y: 0 })
const isResizing = ref(false)
const petScale = ref(1)
const confirmedPetScale = ref(1)
const resizeFrameStyle = ref<Record<string, string>>({})

const appWindow = isTauri() ? getCurrentWindow() : null
const alphaThreshold = 12
const dragThreshold = 6
const dragHandleHideDelay = 420
const defaultPetScale = 1
const minimumPetScale = 0.6
const maximumPetScale = 1.4
const basePetWidth = 384
const basePetHeight = 544
const petScaleStorageKey = 'mmm-pet-scale'
let alphaPixels: Uint8ClampedArray | null = null
let imageWidth = 0
let imageHeight = 0
let pointerTimer: ReturnType<typeof setInterval> | undefined
let isCheckingPointer = false
let ignoresCursorEvents = false
let isDraggingWindow = false
let petPointerStart: { x: number; y: number } | null = null
let petDragStarted = false
let dragHandleHideTimer: ReturnType<typeof setTimeout> | undefined
let resizePointerStart: { x: number; y: number } | null = null
let resizeStartScale = defaultPetScale

const petDisplayWidth = computed(() => basePetWidth * petScale.value)
const petDisplayHeight = computed(() => basePetHeight * petScale.value)

const petBodyStyle = computed(() => ({
  width: `${petDisplayWidth.value}px`,
  height: `${petDisplayHeight.value}px`,
}))

const petScalePercent = computed(() => Math.round(petScale.value * 100))

async function startWindowDrag() {
  if (isResizing.value || !appWindow) {
    return
  }

  isDraggingWindow = true
  await appWindow.startDragging()
}

function beginPetGesture(event: PointerEvent) {
  if (isResizing.value) {
    return
  }

  const target = event.currentTarget as HTMLElement
  isContextMenuOpen.value = false
  petPointerStart = { x: event.clientX, y: event.clientY }
  petDragStarted = false
  target.setPointerCapture(event.pointerId)
}

function trackPetGesture(event: PointerEvent) {
  if (!petPointerStart || petDragStarted) {
    return
  }

  const distance = Math.hypot(
    event.clientX - petPointerStart.x,
    event.clientY - petPointerStart.y,
  )

  if (distance < dragThreshold) {
    return
  }

  petDragStarted = true
  void startWindowDrag()
}

function finishPetGesture(event: PointerEvent) {
  const target = event.currentTarget as HTMLElement

  if (!petPointerStart) {
    return
  }

  if (target.hasPointerCapture(event.pointerId)) {
    target.releasePointerCapture(event.pointerId)
  }

  petPointerStart = null
}

function cancelPetGesture() {
  petPointerStart = null
  petDragStarted = false
}

function openContextMenu(event: MouseEvent) {
  const avatar = petAvatar.value
  if (!avatar) {
    return
  }

  const rect = avatar.getBoundingClientRect()
  contextMenuPosition.value = {
    x: event.clientX - rect.left,
    y: event.clientY - rect.top,
  }
  isContextMenuOpen.value = true
}

function closeContextMenu() {
  isContextMenuOpen.value = false
}

async function closeApp() {
  await appWindow?.close()
}

async function resetWindowPosition() {
  closeContextMenu()
  await appWindow?.center()
}

function clampPetScale(scale: number) {
  return Math.min(maximumPetScale, Math.max(minimumPetScale, scale))
}

function updateResizeFrame() {
  const avatar = petAvatar.value
  const body = petBody.value
  if (!avatar || !body) {
    return
  }

  const avatarRect = avatar.getBoundingClientRect()
  const imageRect = body.getBoundingClientRect()
  resizeFrameStyle.value = {
    left: `${imageRect.left - avatarRect.left}px`,
    top: `${imageRect.top - avatarRect.top}px`,
    width: `${imageRect.width}px`,
    height: `${imageRect.height}px`,
  }
}

function openResizeMode() {
  closeContextMenu()
  isResizing.value = true
  void nextTick(updateResizeFrame)
}

function beginResize(event: PointerEvent) {
  const target = event.currentTarget as HTMLElement
  resizePointerStart = { x: event.clientX, y: event.clientY }
  resizeStartScale = petScale.value
  target.setPointerCapture(event.pointerId)
}

function resizePet(event: PointerEvent) {
  if (!resizePointerStart || !petBody.value) {
    return
  }

  const distance = event.clientX - resizePointerStart.x
  petScale.value = clampPetScale(resizeStartScale + distance / basePetWidth)
  void nextTick(updateResizeFrame)
}

function finishResize(event: PointerEvent) {
  const target = event.currentTarget as HTMLElement
  if (target.hasPointerCapture(event.pointerId)) {
    target.releasePointerCapture(event.pointerId)
  }

  resizePointerStart = null
}

function resetPetScale() {
  petScale.value = defaultPetScale
  void nextTick(updateResizeFrame)
}

function confirmPetScale() {
  confirmedPetScale.value = petScale.value
  localStorage.setItem(petScaleStorageKey, String(petScale.value))
  isResizing.value = false
}

function cancelResizeMode() {
  petScale.value = confirmedPetScale.value
  isResizing.value = false
}

function handleKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape' && isResizing.value) {
    cancelResizeMode()
  }
}

function showDragHandle() {
  if (dragHandleHideTimer) {
    clearTimeout(dragHandleHideTimer)
    dragHandleHideTimer = undefined
  }

  isDragHandleVisible.value = true
}

function scheduleDragHandleHide() {
  if (!isDragHandleVisible.value || dragHandleHideTimer) {
    return
  }

  dragHandleHideTimer = setTimeout(() => {
    isDragHandleVisible.value = false
    dragHandleHideTimer = undefined
  }, dragHandleHideDelay)
}

function preparePetAlphaMap() {
  const image = hitMaskImage.value

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
  const image = hitMaskImage.value
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
  if (isCheckingPointer || !alphaPixels || !appWindow) {
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
    const isPointerInInteractiveArea =
      isResizing.value ||
      isDraggingWindow ||
      isPointOnOpaquePetPixel(x, y) ||
      isPointInsideElement(dragHandle.value, x, y) ||
      isPointInsideElement(contextMenu.value, x, y) ||
      isPointInsideElement(resizeHandle.value, x, y) ||
      isPointInsideElement(resizeResetButton.value, x, y) ||
      isPointInsideElement(resizeConfirmButton.value, x, y)

    if (isPointerInInteractiveArea && !isDraggingWindow) {
      showDragHandle()
    } else if (!isDraggingWindow) {
      scheduleDragHandleHide()
    }

    const shouldIgnoreCursorEvents = !isPointerInInteractiveArea
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

  const storedScale = Number(localStorage.getItem(petScaleStorageKey))
  if (Number.isFinite(storedScale) && storedScale > 0) {
    petScale.value = clampPetScale(storedScale)
    confirmedPetScale.value = petScale.value
  }

  preparePetAlphaMap()
  window.addEventListener('keydown', handleKeydown)
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

  if (dragHandleHideTimer) {
    clearTimeout(dragHandleHideTimer)
  }

  window.removeEventListener('keydown', handleKeydown)
  window.removeEventListener('mouseup', stopWindowDrag)

  if (ignoresCursorEvents && appWindow) {
    void appWindow.setIgnoreCursorEvents(false)
  }
})
</script>

<template>
  <section
    ref="petAvatar"
    class="pet-avatar"
    :class="{ 'pet-avatar--interactive': isDragHandleVisible }"
    aria-label="桌面宠物"
  >
    <button
      v-if="!isResizing"
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

    <div
      ref="petBody"
      class="pet-avatar__body"
      :style="petBodyStyle"
      role="button"
      tabindex="0"
      aria-label="拖动角色可移动窗口，右键打开菜单"
      @dragstart.prevent
      @contextmenu.prevent="openContextMenu"
      @pointercancel="cancelPetGesture"
      @pointerdown.left="beginPetGesture"
      @pointermove="trackPetGesture"
      @pointerup.left="finishPetGesture"
    >
      <SpineAvatar
        animation="wait"
        :display-width="petDisplayWidth"
        :display-height="petDisplayHeight"
      />
      <img
        ref="hitMaskImage"
        :src="mamimiImage"
        class="pet-avatar__hit-mask"
        alt=""
        aria-hidden="true"
        draggable="false"
        @dragstart.prevent
        @load="preparePetAlphaMap"
      />
    </div>

    <div
      v-if="isResizing"
      class="pet-avatar__resize-frame"
      :style="resizeFrameStyle"
    >
      <output class="pet-avatar__resize-value" aria-live="polite">
        {{ petScalePercent }}%
      </output>
      <button
        ref="resizeResetButton"
        type="button"
        class="pet-avatar__resize-reset"
        aria-label="恢复默认大小"
        title="恢复默认大小"
        @click="resetPetScale"
      >
        ↺
      </button>
      <button
        ref="resizeConfirmButton"
        type="button"
        class="pet-avatar__resize-confirm"
        aria-label="确认保存缩放"
        title="确认保存缩放"
        @click="confirmPetScale"
      >
        ✓
      </button>
      <button
        ref="resizeHandle"
        type="button"
        class="pet-avatar__resize-handle"
        aria-label="拖动调整大小"
        title="拖动调整大小"
        @pointerdown.left.prevent="beginResize"
        @pointermove="resizePet"
        @pointerup.left="finishResize"
      ></button>
    </div>

    <menu
      v-if="isContextMenuOpen"
      ref="contextMenu"
      class="pet-avatar__context-menu"
      :style="{
        left: `${contextMenuPosition.x}px`,
        top: `${contextMenuPosition.y}px`,
      }"
    >
      <li><button type="button" @click="openResizeMode">调整大小</button></li>
      <li><button type="button" @click="resetWindowPosition">重置位置</button></li>
      <li><button type="button" @click="closeApp">退出</button></li>
    </menu>
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
.pet-avatar__drag-handle:focus-visible {
  opacity: 1;
  pointer-events: auto;
  visibility: visible;
  transform: translateY(0) scale(1);
  transition-delay: 180ms, 180ms, 180ms;
}

.pet-avatar__body {
  position: relative;
  border: 0;
  padding: 0;
  background: transparent;
  cursor: grab;
  line-height: 0;
  user-select: none;
  -webkit-user-select: none;
  -webkit-user-drag: none;
}

.pet-avatar__context-menu {
  position: absolute;
  z-index: 2;
  min-width: 8rem;
  margin: 0;
  padding: 0.35rem;
  border: 1px solid rgb(255 255 255 / 55%);
  border-radius: 0.65rem;
  background: rgb(35 22 48 / 92%);
  box-shadow: 0 0.5rem 1.25rem rgb(0 0 0 / 28%);
  list-style: none;
}

.pet-avatar__context-menu button {
  width: 100%;
  padding: 0.55rem 0.7rem;
  border: 0;
  border-radius: 0.4rem;
  background: transparent;
  color: #fff;
  cursor: pointer;
  font: inherit;
  text-align: left;
}

.pet-avatar__context-menu button:hover,
.pet-avatar__context-menu button:focus-visible {
  background: rgb(255 255 255 / 16%);
  outline: none;
}

.pet-avatar__hit-mask {
  position: absolute;
  inset: 0;
  display: block;
  width: 100%;
  height: 100%;
  object-fit: contain;
  opacity: 0;
  pointer-events: none;
  user-select: none;
  -webkit-user-select: none;
  -webkit-user-drag: none;
}

.pet-avatar__resize-frame {
  position: absolute;
  z-index: 3;
  border: 1px dashed rgb(255 255 255 / 85%);
  border-radius: 0.4rem;
  box-shadow: 0 0 0 1px rgb(35 22 48 / 55%);
  pointer-events: none;
}

.pet-avatar__resize-handle,
.pet-avatar__resize-confirm,
.pet-avatar__resize-reset {
  position: absolute;
  display: grid;
  width: 2rem;
  height: 2rem;
  padding: 0;
  border: 1px solid rgb(255 255 255 / 60%);
  border-radius: 0.5rem;
  place-items: center;
  background: rgb(35 22 48 / 92%);
  color: #fff;
  box-shadow: 0 0.25rem 0.75rem rgb(0 0 0 / 22%);
  font: inherit;
  pointer-events: auto;
}

.pet-avatar__resize-value {
  position: absolute;
  right: 0.35rem;
  bottom: 0.35rem;
  padding: 0.2rem 0.4rem;
  border-radius: 0.35rem;
  background: rgb(35 22 48 / 82%);
  color: #fff;
  font-size: 0.75rem;
  line-height: 1;
}

.pet-avatar__resize-handle {
  right: -0.55rem;
  bottom: -0.55rem;
  width: 1.1rem;
  height: 1.1rem;
  border-radius: 0.25rem;
  cursor: nwse-resize;
}

.pet-avatar__resize-confirm {
  right: -2.55rem;
  bottom: 0;
  cursor: pointer;
}

.pet-avatar__resize-reset {
  right: -2.55rem;
  bottom: 2.35rem;
  cursor: pointer;
}

.pet-avatar__resize-confirm:hover,
.pet-avatar__resize-confirm:focus-visible,
.pet-avatar__resize-reset:hover,
.pet-avatar__resize-reset:focus-visible {
  background: rgb(255 255 255 / 18%);
  outline: none;
}

.pet-avatar__body :deep(canvas) {
  user-select: none;
  -webkit-user-select: none;
  -webkit-user-drag: none;
  touch-action: none;
}

</style>
