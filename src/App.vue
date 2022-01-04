<template>
  <div id="app">
    <machine></machine>
  </div>
</template>

<script>
import Machine from './components/Machine'

export default {
  name: 'app',
  components: {
    Machine
  }
}
</script>

<style>
#app{
  margin: 0 auto;
  width: 940px;
}
body {
  font-family: BlinkMacSystemFont,  sans-serif;
  background-image: linear-gradient(to right, #bdc2e8 0%, #bdc2e8 1%, #e6dee9 100%);
 background-blend-mode: screen, overlay;
}
</style>
