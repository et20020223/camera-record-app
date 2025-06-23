<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <div id="app">
    <nav class="main-nav">
      <RouterLink to="/">首頁</RouterLink>
      <RouterLink to="/camera">攝影機</RouterLink>
      <RouterLink to="/signature">簽名錄影</RouterLink>
    </nav>

    <main>
      <RouterView />
    </main>
  </div>
</template>

<style scoped>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  width: 100%;
}

.main-nav {
  background: #f8f9fa;
  padding: 1rem;
  text-align: center;
  border-bottom: 1px solid #dee2e6;
}

.main-nav a {
  display: inline-block;
  margin: 0 1rem;
  padding: 0.5rem 1rem;
  text-decoration: none;
  color: #495057;
  border-radius: 4px;
  transition: all 0.3s;
}

.main-nav a:hover {
  background: #e9ecef;
  color: #343a40;
}

.main-nav a.router-link-active {
  background: #007bff;
  color: white;
}

main {
  padding: 2rem;
}
</style>
