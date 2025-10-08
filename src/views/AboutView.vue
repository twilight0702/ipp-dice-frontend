<script setup lang="ts">
import Card from 'primevue/card'
</script>

<template>
  <div class="surface-ground p-4">
    <div class="grid justify-content-center">
      <div class="col-12 md:col-8">
        <Card>
          <template #title>关于</template>
          <template #content>
            <p class="text-700">这是一个基于 PrimeVue 的页面示例。</p>
          </template>
        </Card>
      </div>
    </div>
  </div>
</template>

<style scoped>
</style>
