<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
import * as PIXI from 'pixi.js'

interface LegacyPixiApplication {
  renderer: PIXI.SystemRenderer
  stage: PIXI.Container
  view: HTMLCanvasElement
  destroy(removeView?: boolean): void
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
  }>(),
  {
    animation: 'wait',
    loop: true,
  },
)

const canvasHost = ref<HTMLDivElement | null>(null)
const status = ref<'loading' | 'ready' | 'error'>('loading')
const errorMessage = ref('')

let app: LegacyPixiApplication | null = null
let avatar: PIXI.spine.Spine | null = null
let resizeObserver: ResizeObserver | null = null
let loader: PIXI.loaders.Loader | null = null
let isUnmounted = false

const modelUrl = `${import.meta.env.BASE_URL}spine/mamimi/data.json`
const padding = 12

function layoutAvatar() {
  const host = canvasHost.value
  if (!host || !app || !avatar) {
    return
  }

  const width = Math.max(1, host.clientWidth)
  const height = Math.max(1, host.clientHeight)
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

function playAnimation(name: string, loop = true) {
  if (!avatar || !avatar.state.hasAnimation(name)) {
    return false
  }

  avatar.state.setAnimation(0, name, loop)
  return true
}

async function createRenderer() {
  const host = canvasHost.value
  if (!host) {
    return
  }

  try {
    // pixi-spine 1.4 是非模块化的旧运行时，需要先把 PIXI 暴露到全局。
    ;(globalThis as typeof globalThis & { PIXI: typeof PIXI }).PIXI = PIXI
    await import('pixi-spine')

    if (isUnmounted) {
      return
    }

    const Application = (
      PIXI as typeof PIXI & { Application: LegacyPixiApplicationConstructor }
    ).Application
    app = new Application(
      Math.max(1, host.clientWidth),
      Math.max(1, host.clientHeight),
      {
        antialias: true,
        transparent: true,
        resolution: window.devicePixelRatio || 1,
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

      avatar = new PIXI.spine.Spine(resource.spineData)
      app.stage.addChild(avatar)

      if (!playAnimation(props.animation, props.loop)) {
        status.value = 'error'
        errorMessage.value = `模型中不存在动画：${props.animation}`
        return
      }

      layoutAvatar()
      status.value = 'ready'
    })
    loader.on('error', (loadError: Error) => {
      status.value = 'error'
      errorMessage.value = loadError.message || 'Spine 资源加载失败。'
    })

    resizeObserver = new ResizeObserver(layoutAvatar)
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

onUnmounted(() => {
  isUnmounted = true
  resizeObserver?.disconnect()
  loader?.reset()
  app?.destroy(true)
  avatar = null
  loader = null
  app = null
})

defineExpose({ playAnimation })
</script>

<template>
  <section class="spine-avatar" aria-label="Spine 角色预览">
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
  width: min(72vw, 22rem);
  height: min(68vh, 31rem);
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
