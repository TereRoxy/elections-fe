<script setup>
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <div class="app-container">
    <header>
      <nav class="navbar">
        <RouterLink to="/" class="nav-link">Candidates</RouterLink>
        <!-- Add more links if needed -->
      </nav>
    </header>

    <main class="main-content">
      <RouterView />
    </main>
  </div>
</template>

<style scoped>

</style>
