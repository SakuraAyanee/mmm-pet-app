<script setup lang="ts">
import { computed, nextTick, onMounted, onUnmounted, ref } from 'vue'
import { isTauri } from '@tauri-apps/api/core'
import { cursorPosition, getCurrentWindow } from '@tauri-apps/api/window'
import SpineAvatar from './SpineAvatar.vue'

interface SpineHitMask {
  alpha: Uint8Array
  width: number
  height: number
}

interface ClickAnimationPreset {
  id: string
  label: string
  description: string
  mode: 'overlay' | 'authored'
  animation: string
  loopRepeats?: number
}

interface ExpressionOption {
  id: string
  label: string
}

const spineAvatar = ref<InstanceType<typeof SpineAvatar> | null>(null)
const petBody = ref<HTMLElement | null>(null)
const petAvatar = ref<HTMLElement | null>(null)
const dragHandle = ref<HTMLButtonElement | null>(null)
const contextMenu = ref<HTMLElement | null>(null)
const resizeHandle = ref<HTMLButtonElement | null>(null)
const resizeResetButton = ref<HTMLButtonElement | null>(null)
const resizeConfirmButton = ref<HTMLButtonElement | null>(null)
const animationWorkbench = ref<HTMLElement | null>(null)
const isDragHandleVisible = ref(false)
const isContextMenuOpen = ref(false)
const contextMenuPosition = ref({ x: 0, y: 0 })
const isResizing = ref(false)
const isAlwaysOnTop = ref(true)
const isUpdatingAlwaysOnTop = ref(false)
const isAnimationWorkbenchOpen = ref(false)
const petScale = ref(1)
const confirmedPetScale = ref(1)
const resizeFrameStyle = ref<Record<string, string>>({})
const animationWorkbenchPosition = ref({ x: 16, y: 16 })

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
const alwaysOnTopStorageKey = 'mmm-pet-always-on-top'
const clickAnimationStorageKey = 'mmm-pet-click-animation'
const clickExpressionStorageKey = 'mmm-pet-click-expression'
let alphaPixels: Uint8Array | null = null
let maskWidth = 0
let maskHeight = 0
let pointerTimer: ReturnType<typeof setInterval> | undefined
let isCheckingPointer = false
let ignoresCursorEvents = false
let isDraggingWindow = false
let petPointerStart: { x: number; y: number } | null = null
let petDragStarted = false
let dragHandleHideTimer: ReturnType<typeof setTimeout> | undefined
let resizePointerStart: { x: number; y: number } | null = null
let resizeStartScale = defaultPetScale
let workbenchPointerStart: {
  pointerId: number
  x: number
  y: number
  originX: number
  originY: number
} | null = null

const clickAnimationPresets: ClickAnimationPreset[] = [
  {
    id: 'gentle-yes',
    label: '点头回应',
    description: '在 wait 上只叠加作者提供的 yes 动作，不追加表情。',
    mode: 'overlay',
    animation: 'yes',
  },
  {
    id: 'serious-no',
    label: '摇头回应',
    description: '在 wait 上只叠加作者提供的 no 动作，不追加表情。',
    mode: 'overlay',
    animation: 'no',
  },
  {
    id: 'authored-anger',
    label: '生气（anger1）',
    description: '播放作者定义的 anger1 循环段，再衔接 arm_down_R。',
    mode: 'authored',
    animation: 'anger1',
    loopRepeats: 1,
  },
  {
    id: 'authored-hello',
    label: '打招呼（hello）',
    description: '播放作者定义的 hello 循环段，再衔接 arm_down_L。',
    mode: 'authored',
    animation: 'hello',
    loopRepeats: 1,
  },
  {
    id: 'authored-play',
    label: '玩耍（play）',
    description: '播放作者定义的 play 循环段，再衔接 arm_down_L。',
    mode: 'authored',
    animation: 'play',
    loopRepeats: 1,
  },
  {
    id: 'authored-sad',
    label: '难过（sad1）',
    description: '播放作者定义的 sad1 循环段，再衔接 arm_down。',
    mode: 'authored',
    animation: 'sad1',
    loopRepeats: 1,
  },
  {
    id: 'authored-shy',
    label: '害羞（shy1）',
    description: '播放作者定义的 shy1 循环段，再衔接 arm_down2。',
    mode: 'authored',
    animation: 'shy1',
    loopRepeats: 1,
  },
  {
    id: 'authored-sleep',
    label: '睡觉（sleep）',
    description: '播放作者定义的 sleep 循环段，再衔接 arm_down_L。',
    mode: 'authored',
    animation: 'sleep',
    loopRepeats: 1,
  },
  {
    id: 'authored-smile',
    label: '微笑（smile1）',
    description: '播放作者定义的 smile1 循环段，再衔接 arm_down_L。',
    mode: 'authored',
    animation: 'smile1',
    loopRepeats: 1,
  },
  {
    id: 'authored-surprise',
    label: '惊讶（surp1）',
    description: '播放作者定义的 surp1 循环段，再衔接 arm_down_R。',
    mode: 'authored',
    animation: 'surp1',
    loopRepeats: 1,
  },
  {
    id: 'authored-think',
    label: '思考（think）',
    description: '播放作者定义的 think 循环段，再衔接 arm_down_L。',
    mode: 'authored',
    animation: 'think',
    loopRepeats: 1,
  },
  {
    id: 'authored-touch',
    label: '完整触碰（touch）',
    description: '进入 touch 后重复作者标记的循环段两次，再衔接 arm_down2。',
    mode: 'authored',
    animation: 'touch',
    loopRepeats: 2,
  },
  {
    id: 'skill-action',
    label: '技能动作（独立）',
    description: 'skill1 没有作者定义的 relay，仅完整播放后恢复 wait。',
    mode: 'authored',
    animation: 'skill1',
  },
]

const expressionOptions: ExpressionOption[] = [
  { id: '', label: '跟随原动画（推荐）' },
  { id: 'face_wait', label: '默认表情' },
  { id: 'face_wait2', label: '待机表情 2' },
  { id: 'face_wait3', label: '待机表情 3' },
  { id: 'face_smile', label: '微笑' },
  { id: 'face_shy', label: '害羞' },
  { id: 'face_serious', label: '认真' },
  { id: 'face_sad', label: '难过' },
  { id: 'face_cry', label: '哭泣' },
  { id: 'face_anger', label: '生气' },
  { id: 'face_surp', label: '惊讶' },
  { id: 'face_close', label: '闭眼' },
  { id: 'face_close2', label: '闭眼 2' },
]

const legacyClickAnimationIds: Record<string, string> = {
  'shy-response': 'authored-shy',
  'full-touch': 'authored-touch',
}

const savedClickAnimationId = ref(clickAnimationPresets[0].id)
const draftClickAnimationId = ref(clickAnimationPresets[0].id)
const savedClickExpressionId = ref('')
const draftClickExpressionId = ref('')

const petDisplayWidth = computed(() => basePetWidth * petScale.value)
const petDisplayHeight = computed(() => basePetHeight * petScale.value)

const petBodyStyle = computed(() => ({
  width: `${petDisplayWidth.value}px`,
  height: `${petDisplayHeight.value}px`,
}))

const petScalePercent = computed(() => Math.round(petScale.value * 100))

const animationWorkbenchStyle = computed(() => ({
  left: `${animationWorkbenchPosition.value.x}px`,
  top: `${animationWorkbenchPosition.value.y}px`,
}))

const draftClickAnimation = computed(
  () =>
    clickAnimationPresets.find(
      (preset) => preset.id === draftClickAnimationId.value,
    ) ?? clickAnimationPresets[0],
)

function playClickAnimation(preset: ClickAnimationPreset, expression = '') {
  const renderer = spineAvatar.value
  if (!renderer) {
    return
  }

  if (preset.mode === 'overlay') {
    renderer.playOverlayAnimation(preset.animation, expression || null)
    return
  }

  if (preset.mode === 'authored') {
    renderer.playAuthoredAnimation(
      preset.animation,
      preset.loopRepeats ?? 1,
      expression || null,
    )
  }
}

function playSavedClickAnimation() {
  if (isAnimationWorkbenchOpen.value) {
    playClickAnimation(draftClickAnimation.value, draftClickExpressionId.value)
    return
  }

  const preset =
    clickAnimationPresets.find(
      (candidate) => candidate.id === savedClickAnimationId.value,
    ) ?? clickAnimationPresets[0]

  playClickAnimation(preset, savedClickExpressionId.value)
}

function openAnimationWorkbench() {
  closeContextMenu()
  draftClickAnimationId.value = savedClickAnimationId.value
  draftClickExpressionId.value = savedClickExpressionId.value
  isAnimationWorkbenchOpen.value = true
  void nextTick(() =>
    moveWorkbenchIntoViewport(
      animationWorkbenchPosition.value.x,
      animationWorkbenchPosition.value.y,
    ),
  )
}

function previewDraftClickAnimation() {
  playClickAnimation(draftClickAnimation.value, draftClickExpressionId.value)
}

function saveDraftClickAnimation() {
  savedClickAnimationId.value = draftClickAnimation.value.id
  savedClickExpressionId.value = draftClickExpressionId.value
  localStorage.setItem(clickAnimationStorageKey, draftClickAnimation.value.id)
  localStorage.setItem(clickExpressionStorageKey, draftClickExpressionId.value)
  isAnimationWorkbenchOpen.value = false
}

function closeAnimationWorkbench() {
  isAnimationWorkbenchOpen.value = false
  workbenchPointerStart = null
}

function moveWorkbenchIntoViewport(x: number, y: number) {
  const panel = animationWorkbench.value
  const width = panel?.offsetWidth ?? 304
  const height = panel?.offsetHeight ?? 220

  animationWorkbenchPosition.value = {
    x: Math.min(Math.max(8, x), Math.max(8, window.innerWidth - width - 8)),
    y: Math.min(Math.max(8, y), Math.max(8, window.innerHeight - height - 8)),
  }
}

function beginWorkbenchDrag(event: PointerEvent) {
  const target = event.target as HTMLElement
  if (target.closest('button, select, input')) {
    return
  }

  const handle = event.currentTarget as HTMLElement
  workbenchPointerStart = {
    pointerId: event.pointerId,
    x: event.clientX,
    y: event.clientY,
    originX: animationWorkbenchPosition.value.x,
    originY: animationWorkbenchPosition.value.y,
  }
  handle.setPointerCapture(event.pointerId)
}

function dragAnimationWorkbench(event: PointerEvent) {
  if (
    !workbenchPointerStart ||
    workbenchPointerStart.pointerId !== event.pointerId
  ) {
    return
  }

  moveWorkbenchIntoViewport(
    workbenchPointerStart.originX + event.clientX - workbenchPointerStart.x,
    workbenchPointerStart.originY + event.clientY - workbenchPointerStart.y,
  )
}

function finishWorkbenchDrag(event: PointerEvent) {
  const handle = event.currentTarget as HTMLElement
  if (handle.hasPointerCapture(event.pointerId)) {
    handle.releasePointerCapture(event.pointerId)
  }

  workbenchPointerStart = null
}

async function startWindowDrag() {
  if (!appWindow) {
    return
  }

  isDraggingWindow = true
  await appWindow.startDragging()
}

function beginPetGesture(event: PointerEvent) {
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

  if (!petDragStarted && !isResizing.value) {
    playSavedClickAnimation()
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
  isAnimationWorkbenchOpen.value = false
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

async function restoreAlwaysOnTop() {
  if (!appWindow) {
    return
  }

  const storedValue = localStorage.getItem(alwaysOnTopStorageKey)
  const shouldStayOnTop = storedValue === null ? true : storedValue === 'true'

  try {
    await appWindow.setAlwaysOnTop(shouldStayOnTop)
    isAlwaysOnTop.value = shouldStayOnTop
  } catch (error) {
    console.error('恢复窗口置顶状态失败：', error)
  }
}

async function toggleAlwaysOnTop() {
  if (!appWindow || isUpdatingAlwaysOnTop.value) {
    return
  }

  isUpdatingAlwaysOnTop.value = true
  const nextValue = !isAlwaysOnTop.value

  try {
    await appWindow.setAlwaysOnTop(nextValue)
    isAlwaysOnTop.value = nextValue
    localStorage.setItem(alwaysOnTopStorageKey, String(nextValue))
    closeContextMenu()
  } catch (error) {
    console.error('切换窗口置顶状态失败：', error)
  } finally {
    isUpdatingAlwaysOnTop.value = false
  }
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
  closeAnimationWorkbench()
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

function applySpineHitMask(mask: SpineHitMask) {
  alphaPixels = mask.alpha
  maskWidth = mask.width
  maskHeight = mask.height
  void updateCursorEventMode()
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
  const body = petBody.value
  if (!body || !alphaPixels || maskWidth === 0 || maskHeight === 0) {
    return false
  }

  const rect = body.getBoundingClientRect()

  if (
    x < rect.left ||
    x >= rect.right ||
    y < rect.top ||
    y >= rect.bottom
  ) {
    return false
  }

  const pixelX = Math.min(
    maskWidth - 1,
    Math.floor(((x - rect.left) / rect.width) * maskWidth),
  )
  const pixelY = Math.min(
    maskHeight - 1,
    Math.floor(((y - rect.top) / rect.height) * maskHeight),
  )
  const alphaIndex = pixelY * maskWidth + pixelX

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
      isPointInsideElement(resizeConfirmButton.value, x, y) ||
      isPointInsideElement(animationWorkbench.value, x, y)

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
  const storedClickAnimationId = localStorage.getItem(clickAnimationStorageKey)
  const resolvedClickAnimationId = storedClickAnimationId
    ? (legacyClickAnimationIds[storedClickAnimationId] ?? storedClickAnimationId)
    : null
  const storedClickExpressionId = localStorage.getItem(clickExpressionStorageKey)
  if (
    resolvedClickAnimationId &&
    clickAnimationPresets.some(
      (preset) => preset.id === resolvedClickAnimationId,
    )
  ) {
    savedClickAnimationId.value = resolvedClickAnimationId
    draftClickAnimationId.value = resolvedClickAnimationId
  }

  if (
    storedClickExpressionId !== null &&
    expressionOptions.some((option) => option.id === storedClickExpressionId)
  ) {
    savedClickExpressionId.value = storedClickExpressionId
    draftClickExpressionId.value = storedClickExpressionId
  }

  if (!isTauri()) {
    return
  }

  const storedScale = Number(localStorage.getItem(petScaleStorageKey))
  if (Number.isFinite(storedScale) && storedScale > 0) {
    petScale.value = clampPetScale(storedScale)
    confirmedPetScale.value = petScale.value
  }

  void restoreAlwaysOnTop()
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
        ref="spineAvatar"
        animation="wait"
        :display-width="petDisplayWidth"
        :display-height="petDisplayHeight"
        @hit-mask-ready="applySpineHitMask"
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
      <li>
        <button
          type="button"
          :disabled="isUpdatingAlwaysOnTop"
          @click="toggleAlwaysOnTop"
        >
          {{ isAlwaysOnTop ? '取消总在最前' : '总在最前' }}
        </button>
      </li>
      <li>
        <button type="button" @click="openAnimationWorkbench">
          点击动画测试
        </button>
      </li>
      <li><button type="button" @click="openResizeMode">调整大小</button></li>
      <li><button type="button" @click="resetWindowPosition">重置位置</button></li>
      <li><button type="button" @click="closeApp">退出</button></li>
    </menu>

    <aside
      v-if="isAnimationWorkbenchOpen"
      ref="animationWorkbench"
      class="pet-avatar__animation-workbench"
      :style="animationWorkbenchStyle"
      aria-label="点击动画测试工作台"
      @contextmenu.prevent
      @pointerdown.stop
    >
      <header
        class="pet-avatar__animation-workbench-header"
        title="拖动测试面板"
        @pointercancel="finishWorkbenchDrag"
        @pointerdown.left.prevent="beginWorkbenchDrag"
        @pointermove="dragAnimationWorkbench"
        @pointerup.left="finishWorkbenchDrag"
      >
        <strong>点击动画测试</strong>
        <button
          type="button"
          class="pet-avatar__animation-workbench-close"
          aria-label="关闭动画测试"
          @click="closeAnimationWorkbench"
        >
          ×
        </button>
      </header>

      <label class="pet-avatar__animation-field">
        <span>预设组合</span>
        <select
          v-model="draftClickAnimationId"
          @change="previewDraftClickAnimation"
        >
          <option
            v-for="preset in clickAnimationPresets"
            :key="preset.id"
            :value="preset.id"
          >
            {{ preset.label }}
          </option>
        </select>
      </label>

      <p class="pet-avatar__animation-description">
        {{ draftClickAnimation.description }}
      </p>

      <label class="pet-avatar__animation-field">
        <span>表情覆盖（可选）</span>
        <select
          v-model="draftClickExpressionId"
          @change="previewDraftClickAnimation"
        >
          <option
            v-for="expression in expressionOptions"
            :key="expression.id"
            :value="expression.id"
          >
            {{ expression.label }}
          </option>
        </select>
      </label>

      <p class="pet-avatar__animation-hint">
        默认保留作者为动作制作的表情；手动选择时只在本次动作期间覆盖，结束后自动恢复。
      </p>

      <div class="pet-avatar__animation-actions">
        <button type="button" @click="previewDraftClickAnimation">
          预览
        </button>
        <button type="button" @click="saveDraftClickAnimation">
          设为点击动画
        </button>
      </div>
    </aside>
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

.pet-avatar__body:active {
  cursor: grabbing;
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

.pet-avatar__context-menu button:disabled {
  cursor: wait;
  opacity: 0.6;
}

.pet-avatar__animation-workbench {
  position: absolute;
  z-index: 4;
  width: min(19rem, calc(100vw - 1.5rem));
  padding: 0.85rem;
  border: 1px solid rgb(255 255 255 / 55%);
  border-radius: 0.75rem;
  background: rgb(35 22 48 / 94%);
  box-shadow: 0 0.75rem 1.75rem rgb(0 0 0 / 32%);
  color: #fff;
  line-height: 1.4;
}

.pet-avatar__animation-workbench-header,
.pet-avatar__animation-actions {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.6rem;
}

.pet-avatar__animation-workbench-header {
  cursor: grab;
  user-select: none;
  -webkit-user-select: none;
}

.pet-avatar__animation-workbench-header:active {
  cursor: grabbing;
}

.pet-avatar__animation-workbench-close {
  width: 1.75rem;
  height: 1.75rem;
  padding: 0;
  border: 0;
  border-radius: 0.4rem;
  background: transparent;
  color: #fff;
  cursor: pointer;
  font: inherit;
  font-size: 1.25rem;
}

.pet-avatar__animation-workbench-close:hover,
.pet-avatar__animation-workbench-close:focus-visible {
  background: rgb(255 255 255 / 16%);
  outline: none;
}

.pet-avatar__animation-field {
  display: grid;
  gap: 0.35rem;
  margin-top: 0.7rem;
  font-size: 0.8rem;
}

.pet-avatar__animation-field select {
  width: 100%;
  padding: 0.5rem 0.6rem;
  border: 1px solid rgb(255 255 255 / 40%);
  border-radius: 0.45rem;
  background: rgb(17 11 24 / 92%);
  color: #fff;
  font: inherit;
}

.pet-avatar__animation-description {
  min-height: 2.8em;
  margin: 0.65rem 0;
  color: rgb(255 255 255 / 78%);
  font-size: 0.78rem;
}

.pet-avatar__animation-hint {
  margin: 0.45rem 0 0.7rem;
  color: rgb(255 255 255 / 62%);
  font-size: 0.72rem;
}

.pet-avatar__animation-actions {
  justify-content: flex-end;
}

.pet-avatar__animation-actions button {
  padding: 0.45rem 0.7rem;
  border: 1px solid rgb(255 255 255 / 45%);
  border-radius: 0.45rem;
  background: rgb(255 255 255 / 10%);
  color: #fff;
  cursor: pointer;
  font: inherit;
}

.pet-avatar__animation-actions button:hover,
.pet-avatar__animation-actions button:focus-visible {
  background: rgb(255 255 255 / 20%);
  outline: none;
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
