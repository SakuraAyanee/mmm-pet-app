<script setup lang="ts">
import PetAvatar from './components/PetAvatar.vue'
</script>

<template>
  <main class="pet-app">
    <PetAvatar />
  </main>
</template>

<style scoped>
.pet-app {
  display: grid;
  min-height: 100vh;
  padding-bottom: 1rem;
  place-items: end center;
}
</style>
