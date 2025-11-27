<template>
  <div id="app">
    <CrosshairsViewer 
      :studyInstanceUID="studyInstanceUID"
      :seriesInstanceUID="seriesInstanceUID"
    />
  </div>
</template>

<script setup>
import { ref } from 'vue'
import CrosshairsViewer from './components/CrosshairsViewer.vue'

// DICOM 配置参数
const studyInstanceUID = ref('1.2.826.0.1.3680043.2.109.5.20220519102622656.1699758078.1')
const seriesInstanceUID = ref('1.3.12.2.1107.5.1.4.73336.30000022051900071431600050933')
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
}

#app {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
</style>

