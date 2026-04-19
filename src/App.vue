<script setup lang="ts">
import Timetable from './components/Timetable.vue';
</script>

<template>
  <div class="app">
    <h1>Vue3 + TypeScript + Vite 课程表</h1>
    <Timetable />
  </div>
</template>

<style scoped>
.app {
  font-family: Arial, sans-serif;
  max-width: 1000px;
  margin: 0 auto;
  padding: 20px;
}

h1 {
  color: #42b983;
  text-align: center;
  margin-bottom: 30px;
}
</style>