<template>
  <h1>Tetris</h1>
  <tetris />
  <app-footer />
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import AppFooter from './components/AppFooter.vue'
import Tetris from './components/Tetris.vue'

export default defineComponent({
  name: 'App',

  components: {
    AppFooter,
    Tetris
  }
})
</script>

<style lang="scss">
body {
  min-height: 100vh;
}

#app {
  min-height: 100vh;
  position: relative;
  padding-bottom: 150px;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  line-height: 1.4;
  letter-spacing: 0.5px;
  color: #2c3e50;
  padding-top: 60px;
  background: lighten($color: #2c3e50, $amount: 50%);

  h1 {
    margin-bottom: 10px;
  }
}
</style>
