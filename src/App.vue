<script setup>
// import ElevatorSimulator from './components/ElevatorSimulator.vue'
import ThreeBuilding from './components/ThreeBuilding.vue'
</script>

<template>
  <div class="app">
    <ThreeBuilding />
  </div>
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
  .app {
    min-height: 100vh;
    display: grid;
    place-items: start center;
    padding: 16px;
    background: #f6f7f9;
  }
</style>
