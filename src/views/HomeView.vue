<script setup lang="ts">
import Card from 'primevue/card'
</script>

<template>
  <div class="surface-ground p-4">
    <div class="grid justify-content-center">
      <div class="col-12 md:col-8">
        <Card>
          <template #title>首页</template>
          <template #content>
            <p class="text-700">欢迎使用基于 PrimeVue 的骰子游戏前端。</p>
          </template>
        </Card>
      </div>
    </div>
  </div>
</template>
