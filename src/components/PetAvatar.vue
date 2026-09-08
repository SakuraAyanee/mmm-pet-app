<script setup lang="ts">
import { ref } from 'vue'
import type { PetState } from '../types/pet'

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
      class="pet-avatar__body"
      :style="{
        transform: `translate(${petState.x}px, ${petState.y}px)`,
        backgroundColor: petColor,
      }"
      aria-label="点击移动宠物"
      @click="movePet"
    >
      🐾
    </button>
    
    <p>{{ petName }}</p>
    <p>当前情绪：{{ petState.mood }}</p>

    <p>被摸次数：{{ affection }}</p>
  </section>
</template>

<style scoped>
.pet-avatar {
  display: grid;
  justify-items: center;
  gap: 0.75rem;
}

.pet-avatar__body {
  display: grid;
  width: 12rem;
  aspect-ratio: 1;
  place-items: center;
  border: 0;
  border-radius: 1.25rem;
  cursor: pointer;
  font-size: 5rem;
  transition: transform 300ms ease, background-color 300ms ease;
}

.pet-avatar__body:hover {
  filter: brightness(1.05);
}

p {
  margin: 0;
}
</style>
