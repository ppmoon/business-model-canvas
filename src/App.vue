<template>
  <BusinessModelCanvas />
</template>

<script>
import BusinessModelCanvas from './components/BusinessModelCanvas.vue'

export default {
  name: 'App',
  components: {
    BusinessModelCanvas
  }
}
</script>

<style>
html, body, #app {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
}
</style>
