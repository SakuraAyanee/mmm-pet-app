<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import * as PIXI from 'pixi.js'

interface LegacyPixiApplication {
  renderer: PIXI.SystemRenderer & {
    extract: {
      canvas(target?: PIXI.DisplayObject | PIXI.RenderTexture): HTMLCanvasElement
    }
  }
  stage: PIXI.Container
  view: HTMLCanvasElement
  destroy(removeView?: boolean): void
}

interface SpineHitMask {
  alpha: Uint8Array
  width: number
  height: number
}

interface AuthoredAnimationEvent {
  time?: number
  name: string
  string?: string
}

interface SpineJsonData {
  animations?: Record<string, { events?: AuthoredAnimationEvent[] }>
}

type PlaybackPhase = 'idle' | 'intro' | 'loop' | 'recovering'

interface PlaybackRequest {
  animation: string
  expression: string | null
  loopRepeats: number
  loopStart?: number
  relay?: string
}

interface LegacyPixiApplicationConstructor {
  new (
    width: number,
    height: number,
    options: {
      antialias: boolean
      transparent: boolean
      resolution: number
    },
  ): LegacyPixiApplication
}

const props = withDefaults(
  defineProps<{
    animation?: string
    loop?: boolean
    displayWidth?: number
    displayHeight?: number
  }>(),
  {
    animation: 'wait',
    loop: true,
    displayWidth: 384,
    displayHeight: 544,
  },
)

const emit = defineEmits<{
  hitMaskReady: [mask: SpineHitMask]
}>()

const canvasHost = ref<HTMLDivElement | null>(null)
const status = ref<'loading' | 'ready' | 'error'>('loading')
const errorMessage = ref('')

let app: LegacyPixiApplication | null = null
let avatar: PIXI.spine.Spine | null = null
let resizeObserver: ResizeObserver | null = null
let loader: PIXI.loaders.Loader | null = null
let isUnmounted = false
let resolutionFrame: number | null = null
let hitMaskFrame: number | null = null
let hitMaskTimer: ReturnType<typeof setTimeout> | undefined
let authoredAnimations: SpineJsonData['animations'] = {}
let playbackPhase: PlaybackPhase = 'idle'
let currentPlayback: PlaybackRequest | null = null
let pendingPlayback: PlaybackRequest | null = null
let playbackGeneration = 0
let genericRecoveryTimer: ReturnType<typeof setTimeout> | undefined

const modelUrl = `${import.meta.env.BASE_URL}spine/mamimi/data.json`
const padding = 12
const minimumResolution = 2
const maximumResolution = 3

const avatarStyle = computed(() => ({
  width: `${props.displayWidth}px`,
  height: `${props.displayHeight}px`,
}))

function getRenderResolution() {
  return Math.min(
    maximumResolution,
    Math.max(minimumResolution, window.devicePixelRatio || 1),
  )
}

function layoutAvatar() {
  const host = canvasHost.value
  if (!host || !app || !avatar) {
    return
  }

  const width = Math.max(1, props.displayWidth)
  const height = Math.max(1, props.displayHeight)
  app.renderer.resize(width, height)

  avatar.scale.set(1)
  const bounds = avatar.getLocalBounds()
  if (bounds.width <= 0 || bounds.height <= 0) {
    return
  }

  const availableWidth = Math.max(1, width - padding * 2)
  const availableHeight = Math.max(1, height - padding * 2)
  const scale = Math.min(
    availableWidth / bounds.width,
    availableHeight / bounds.height,
  )

  avatar.scale.set(scale)
  avatar.position.set(
    width / 2 - (bounds.x + bounds.width / 2) * scale,
    height - padding - (bounds.y + bounds.height) * scale,
  )
}

function captureHitMask() {
  hitMaskFrame = null
  if (!app || !avatar || isUnmounted) {
    return
  }

  try {
    app.renderer.render(app.stage)
    const snapshot = app.renderer.extract.canvas()
    const context = snapshot.getContext('2d', { willReadFrequently: true })
    if (!context) {
      return
    }

    const rgba = context.getImageData(
      0,
      0,
      snapshot.width,
      snapshot.height,
    ).data
    const alpha = new Uint8Array(snapshot.width * snapshot.height)

    for (let source = 3, target = 0; source < rgba.length; source += 4) {
      alpha[target] = rgba[source]
      target += 1
    }

    emit('hitMaskReady', {
      alpha,
      width: snapshot.width,
      height: snapshot.height,
    })
  } catch (error) {
    console.error('生成 Spine 默认命中蒙版失败：', error)
  }
}

function scheduleHitMaskCapture(delay = 180) {
  if (hitMaskTimer) {
    clearTimeout(hitMaskTimer)
  }

  if (hitMaskFrame !== null) {
    cancelAnimationFrame(hitMaskFrame)
    hitMaskFrame = null
  }

  hitMaskTimer = setTimeout(() => {
    hitMaskTimer = undefined
    hitMaskFrame = requestAnimationFrame(captureHitMask)
  }, delay)
}

function updateRendererLayout() {
  resolutionFrame = null
  if (!app) {
    return
  }

  const nextResolution = getRenderResolution()
  if (Math.abs(app.renderer.resolution - nextResolution) >= 0.01) {
    app.renderer.resolution = nextResolution
  }

  layoutAvatar()
}

function scheduleRendererLayout() {
  if (resolutionFrame !== null) {
    cancelAnimationFrame(resolutionFrame)
  }

  resolutionFrame = requestAnimationFrame(updateRendererLayout)
}

function playAnimation(name: string, loop = true) {
  if (!avatar || !avatar.state.hasAnimation(name)) {
    return false
  }

  avatar.state.setAnimation(0, name, loop)
  return true
}

function hasAnimations(names: string[]) {
  return Boolean(avatar && names.every((name) => avatar?.state.hasAnimation(name)))
}

function applyExpression(expression: string | null) {
  if (!avatar || !expression) {
    return
  }

  if (avatar.state.hasAnimation(expression)) {
    const animation = avatar.spineData.findAnimation(expression)
    avatar.state.setAnimation(2, expression, Boolean(animation?.duration))
  }
}

function restoreAuthoredExpression() {
  avatar?.state.setEmptyAnimation(2, 0.18)
}

function clearGenericRecoveryTimer() {
  if (genericRecoveryTimer) {
    clearTimeout(genericRecoveryTimer)
    genericRecoveryTimer = undefined
  }
}

function finishRecovery(generation: number) {
  if (generation !== playbackGeneration) {
    return
  }

  clearGenericRecoveryTimer()
  playbackPhase = 'idle'
  currentPlayback = null

  const nextPlayback = pendingPlayback
  pendingPlayback = null
  if (nextPlayback) {
    startAuthoredPlayback(nextPlayback)
  }
}

function scheduleGenericRecovery(generation: number) {
  clearGenericRecoveryTimer()
  genericRecoveryTimer = setTimeout(() => {
    genericRecoveryTimer = undefined
    finishRecovery(generation)
  }, 220)
}

function beginEarlyRecovery(generation: number) {
  if (
    !avatar ||
    generation !== playbackGeneration ||
    playbackPhase === 'recovering'
  ) {
    return
  }

  playbackGeneration += 1
  const recoveryGeneration = playbackGeneration
  const relay = currentPlayback?.relay

  playbackPhase = 'recovering'
  avatar.state.clearTrack(0)
  avatar.state.clearTrack(1)
  restoreAuthoredExpression()

  if (relay && avatar.state.hasAnimation(relay)) {
    const recovery = avatar.state.setAnimation(0, relay, false)
    avatar.state.addAnimation(0, props.animation, props.loop, 0)
    recovery.onComplete = () => finishRecovery(recoveryGeneration)
    return
  }

  const wait = avatar.state.setAnimation(0, props.animation, props.loop)
  wait.mixDuration = 0.22
  scheduleGenericRecovery(recoveryGeneration)
}

function handlePlaybackBoundary(
  generation: number,
  isLastActionSection: boolean,
) {
  if (generation !== playbackGeneration) {
    return
  }

  if (pendingPlayback) {
    beginEarlyRecovery(generation)
    return
  }

  if (!isLastActionSection) {
    playbackPhase = 'loop'
    return
  }

  playbackPhase = 'recovering'
  restoreAuthoredExpression()

  if (!currentPlayback?.relay) {
    scheduleGenericRecovery(generation)
  }
}

function startAuthoredPlayback(request: PlaybackRequest) {
  if (!avatar) {
    return
  }

  clearGenericRecoveryTimer()
  playbackGeneration += 1
  const generation = playbackGeneration
  currentPlayback = request
  pendingPlayback = null
  avatar.state.clearTracks()

  const animation = avatar.spineData.findAnimation(request.animation)
  if (!animation) {
    playbackPhase = 'idle'
    currentPlayback = null
    return
  }

  const hasLoopSection =
    typeof request.loopStart === 'number' &&
    request.loopStart > 0 &&
    request.loopStart < animation.duration

  if (hasLoopSection) {
    playbackPhase = 'intro'
    const loopStart = request.loopStart as number
    const intro = avatar.state.setAnimation(0, request.animation, false)
    intro.animationStart = 0
    intro.animationEnd = loopStart

    const loopDuration = animation.duration - loopStart
    const repeatCount = Math.max(1, request.loopRepeats)
    let segmentDelay = loopStart

    intro.onComplete = () => handlePlaybackBoundary(generation, false)

    for (let index = 0; index < repeatCount; index += 1) {
      const repeatedSection = avatar.state.addAnimation(
        0,
        request.animation,
        false,
        segmentDelay,
      )
      repeatedSection.animationStart = loopStart
      repeatedSection.animationEnd = animation.duration
      repeatedSection.mixDuration = 0
      repeatedSection.onComplete = () =>
        handlePlaybackBoundary(generation, index === repeatCount - 1)
      segmentDelay = loopDuration
    }
  } else {
    playbackPhase = 'loop'
    const action = avatar.state.setAnimation(0, request.animation, false)
    action.onComplete = () => handlePlaybackBoundary(generation, true)
  }

  if (request.relay && avatar.state.hasAnimation(request.relay)) {
    const recovery = avatar.state.addAnimation(0, request.relay, false, 0)
    recovery.onComplete = () => finishRecovery(generation)
  }

  avatar.state.addAnimation(0, props.animation, props.loop, 0)
  applyExpression(request.expression)
}

function playOverlayAnimation(action: string, expression: string | null = null) {
  if (!avatar || !hasAnimations([action])) {
    return false
  }

  avatar.state.clearTrack(1)
  avatar.state.clearTrack(2)
  const entry = avatar.state.setAnimation(1, action, false)
  applyExpression(expression)
  entry.onComplete = () => {
    avatar?.state.setEmptyAnimation(1, 0.18)
    restoreAuthoredExpression()
  }

  return true
}

function playAuthoredAnimation(
  name: string,
  loopRepeats = 1,
  expression: string | null = null,
) {
  if (!avatar || !hasAnimations([name, props.animation])) {
    return false
  }

  const animation = avatar.spineData.findAnimation(name)
  if (!animation) {
    return false
  }

  const events = authoredAnimations?.[name]?.events ?? []
  const loopStart = events.find((event) => event.name === 'loop_start')?.time
  const authoredRelay = events.find((event) => event.name === 'relay')?.string
  const relay =
    authoredRelay && avatar.state.hasAnimation(authoredRelay)
      ? authoredRelay
      : undefined

  const request: PlaybackRequest = {
    animation: name,
    expression,
    loopRepeats,
    loopStart,
    relay,
  }

  if (playbackPhase === 'idle') {
    startAuthoredPlayback(request)
  } else {
    // 连续输入只保留最后一次请求，防止长动画把点击无限堆入队列。
    pendingPlayback = request
  }

  return true
}

async function createRenderer() {
  const host = canvasHost.value
  if (!host) {
    return
  }

  try {
    // pixi-spine 1.x 是非模块化的旧运行时，需要先把 PIXI 暴露到全局。
    ;(globalThis as typeof globalThis & { PIXI: typeof PIXI }).PIXI = PIXI
    await import('pixi-spine')

    if (isUnmounted) {
      return
    }

    const Application = (
      PIXI as typeof PIXI & { Application: LegacyPixiApplicationConstructor }
    ).Application
    app = new Application(
      Math.max(1, props.displayWidth),
      Math.max(1, props.displayHeight),
      {
        antialias: true,
        transparent: true,
        resolution: getRenderResolution(),
      },
    )
    app.renderer.autoResize = true
    app.view.classList.add('spine-avatar__canvas')
    host.appendChild(app.view)

    loader = new PIXI.loaders.Loader()
    loader.add('mamimi', modelUrl)
    loader.load((_currentLoader, resources) => {
      if (isUnmounted || !app) {
        return
      }

      const resource = resources.mamimi
      if (!resource?.spineData) {
        status.value = 'error'
        errorMessage.value = '模型已加载，但没有解析出 Spine 骨骼数据。'
        return
      }

      authoredAnimations = (resource.data as SpineJsonData).animations ?? {}
      avatar = new PIXI.spine.Spine(resource.spineData)
      avatar.stateData.defaultMix = 0.18
      app.stage.addChild(avatar)

      if (!playAnimation(props.animation, props.loop)) {
        status.value = 'error'
        errorMessage.value = `模型中不存在动画：${props.animation}`
        return
      }

      layoutAvatar()
      status.value = 'ready'
      scheduleHitMaskCapture(0)
    })
    loader.on('error', (loadError: Error) => {
      status.value = 'error'
      errorMessage.value = loadError.message || 'Spine 资源加载失败。'
    })

    resizeObserver = new ResizeObserver(scheduleRendererLayout)
    resizeObserver.observe(host)
  } catch (error) {
    status.value = 'error'
    errorMessage.value =
      error instanceof Error ? error.message : 'Spine 渲染器初始化失败。'
  }
}

onMounted(() => {
  void createRenderer()
})

watch(
  [() => props.displayWidth, () => props.displayHeight],
  scheduleRendererLayout,
  { flush: 'post' },
)

onUnmounted(() => {
  isUnmounted = true
  if (resolutionFrame !== null) {
    cancelAnimationFrame(resolutionFrame)
  }
  if (hitMaskFrame !== null) {
    cancelAnimationFrame(hitMaskFrame)
  }
  if (hitMaskTimer) {
    clearTimeout(hitMaskTimer)
  }
  clearGenericRecoveryTimer()
  resizeObserver?.disconnect()
  loader?.reset()
  app?.destroy(true)
  avatar = null
  loader = null
  app = null
  authoredAnimations = {}
  currentPlayback = null
  pendingPlayback = null
  playbackPhase = 'idle'
})

defineExpose({
  playAnimation,
  playOverlayAnimation,
  playAuthoredAnimation,
  refreshHitMask: scheduleHitMaskCapture,
})
</script>

<template>
  <section
    class="spine-avatar"
    :style="avatarStyle"
    aria-label="Spine 角色预览"
  >
    <div ref="canvasHost" class="spine-avatar__canvas-host"></div>
    <p v-if="status === 'loading'" class="spine-avatar__status">模型加载中…</p>
    <p v-else-if="status === 'error'" class="spine-avatar__status spine-avatar__status--error">
      {{ errorMessage }}
    </p>
  </section>
</template>

<style scoped>
.spine-avatar {
  position: relative;
}

.spine-avatar__canvas-host {
  width: 100%;
  height: 100%;
}

.spine-avatar__canvas-host :deep(.spine-avatar__canvas) {
  display: block;
  width: 100%;
  height: 100%;
}

.spine-avatar__status {
  position: absolute;
  inset: 50% auto auto 50%;
  margin: 0;
  padding: 0.5rem 0.75rem;
  border-radius: 0.5rem;
  background: rgb(35 22 48 / 80%);
  color: #fff;
  font-size: 0.875rem;
  line-height: 1.4;
  text-align: center;
  transform: translate(-50%, -50%);
}

.spine-avatar__status--error {
  width: min(18rem, 80vw);
  background: rgb(105 24 39 / 88%);
}
</style>
