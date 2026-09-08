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
  place-items: center;
}
</style>