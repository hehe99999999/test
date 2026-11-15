<script setup>
defineProps({
  title: {
    type: String,
    required: true
  }
})
</script>

<template>
  <div class="placeholder-page">
    <div class="placeholder">
      <h1>{{ title }}页面</h1>
      <p>正在开发中，暂无具体内容...</p>
    </div>
  </div>
</template>
