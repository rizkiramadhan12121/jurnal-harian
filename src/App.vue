<script setup>
import JournalApp from './components/JournalApp.vue'
</script>

<template>
  <div class="logos">
    <a href="https://vite.dev" target="_blank">
      <img src="/vite.svg" class="logo" alt="Vite logo" />
    </a>
    <a href="https://vuejs.org/" target="_blank">
      <img src="./assets/vue.svg" class="logo vue" alt="Vue logo" />
    </a>
  </div>
  <JournalApp />
</template>

<style scoped>
.logos {
  display: flex;
  gap: 0.5rem;
  align-items: center;
  justify-content: center;
}
.logo {
  height: 3.5em;
  padding: 0.5em;
  will-change: filter;
  transition: filter 300ms;
  
}
.logo:hover {
  filter: drop-shadow(0 0 1.2em #646cff55);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 1.2em #42b88355);
}
</style>
