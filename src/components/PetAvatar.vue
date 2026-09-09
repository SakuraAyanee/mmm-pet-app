<script setup lang="ts">
import { ref } from 'vue'
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

function movePet() {
  const distance = 120

  petState.value.x = Math.round((Math.random() * 2 - 1) * distance)
  petState.value.y = Math.round((Math.random() * 2 - 1) * distance)
  petState.value.mood = 'happy'
  petColor.value = `hsl(${Math.floor(Math.random() * 360)} 75% 78%)`
  affection.value += 1
}
</script>

<template>
  <section class="pet-avatar" aria-label="桌面宠物">
    <button
      type="button"
      class="pet-avatar__drag-handle"
      data-tauri-drag-region="deep"
      aria-label="拖动窗口"
      title="拖动窗口"
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
      <img :src="mamimiImage" :alt="`${petName}，当前情绪：${petState.mood}`" />
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
  transform: translateY(-0.25rem);
  transition: opacity 160ms ease, transform 160ms ease;
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

.pet-avatar:hover .pet-avatar__drag-handle,
.pet-avatar:focus-within .pet-avatar__drag-handle {
  opacity: 1;
  pointer-events: auto;
  transform: translateY(0);
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
