<template>
  <div class="app">
    <h1>Weighted Overtime Calculator</h1>
    <p>Welcome to your Vue app built with SaaSClaw Studio.</p>
    <button @click="count++">Clicked {{ count }} times</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'
const count = ref(0)
</script>

<style scoped>
.app {
  max-width: 800px;
  margin: 0 auto;
  padding: 40px;
  text-align: center;
  font-family: system-ui, sans-serif;
}
button {
  padding: 10px 24px;
  font-size: 1rem;
  border: none;
  border-radius: 8px;
  background: #42b883;
  color: white;
  cursor: pointer;
}
</style>
