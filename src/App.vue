<script setup lang="ts">
import { RouterView } from 'vue-router'
</script>

<template>
  <div class="min-h-screen bg-gradient-to-b from-gray-50 to-white">

    <main class="py-8">
      <RouterView />
    </main>
    <footer class="border-t border-gray-200 bg-gray-50 py-8 mt-12">
      <div class="max-w-7xl mx-auto px-6 lg:px-8">
        <div class="flex flex-col md:flex-row justify-between items-center">
          <div class="mb-4 md:mb-0">
            <p class="text-gray-600 text-sm">© 2025 Monetizely. All rights reserved.</p>
          </div>
          <div class="flex items-center space-x-4">
            <span class="text-gray-500 text-sm">Powered by</span>
            <a href="https://www.assemblyai.com" target="_blank" class="text-indigo-600 hover:text-indigo-500 font-semibold transition">AssemblyAI</a>
          </div>
        </div>
      </div>
    </footer>
  </div>
</template>

<style>
body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>