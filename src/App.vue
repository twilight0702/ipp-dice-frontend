<script setup lang="ts">
import { RouterView } from 'vue-router'
</script>

<template>
  <RouterView />
</template>

<style>
/* 全局样式重置，确保页面占满全屏 */
html, body {
  margin: 0;
  padding: 0;
  height: 100%;
}

#app {
  margin: 0;
  padding: 0;
  max-width: none;
  height: 100vh;
}
</style>
