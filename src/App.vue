<template>
  <div id="app">
    <Project v-if="false"/>
    <Map2 v-if="true"/>
  </div>
</template>

<script>

import Project from './components/project/Project'
import Map2 from './components/Mapnovo.vue';

export default {
  name: 'App',
  components: {
    Project,
    Map2
}
}
</script>

<style>
@import "~buefy/dist/buefy.css";
@import "~vuelayers/dist/vuelayers.css";

html, body, #app {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>
