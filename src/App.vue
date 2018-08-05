<template>
  <div id="app">
    <Caterpillar/>
  </div>
</template>

<script>
import Caterpillar from './components/Caterpillar.vue'

export default {
  components: {
    Caterpillar
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
