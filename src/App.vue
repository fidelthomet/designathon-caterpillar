<template>
  <div id="app">
    <Worm/>
  </div>
</template>

<script>
import Worm from './components/Worm.vue'

export default {
  name: 'app',
  components: {
    Worm
  }
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
}
#app {
  position: fixed;
  width: 100vw;
  height: 100vh;
  background: #211F42;
}
</style>
