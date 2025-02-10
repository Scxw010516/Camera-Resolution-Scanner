<template>
  <a-config-provider>
    <div class="container">
      <a-card class="app">
        <h1>摄像头分辨率扫描</h1>
        <CameraScanner />
      </a-card>
    </div>
  </a-config-provider>
</template>

<script setup>
import CameraScanner from "./components/CameraScanner.vue";
</script>

<style>
body {
  margin: 0;
  padding: 0;
  min-height: 100vh;
  background: #f5f7fa;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen, Ubuntu, Cantarell, "Open Sans",
    "Helvetica Neue", sans-serif;
}

.container {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  min-height: 100vh;
  padding: 20px;
}

.app {
  width: 100%;
  max-width: 1200px;
}

h1 {
  font-size: 24px;
  font-weight: 500;
  color: #000;
  margin: 0 0 24px;
  text-align: center;
}
</style>
