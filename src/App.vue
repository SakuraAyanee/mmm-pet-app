<script setup lang="ts">
import SpineAvatar from './components/SpineAvatar.vue'
</script>

<template>
  <main class="pet-app">
    <SpineAvatar animation="wait" />
  </main>
</template>

<style scoped>
.pet-app {
  display: grid;
  min-height: 100vh;
  place-items: center;
}
</style>
