<script setup>
import { ref, onMounted, watch } from 'vue'
import Navbar from './components/Navbar.vue'
import DataExtractor from './components/DataExtractor.vue'
import SqlQueryBuilder from './components/SqlQueryBuilder.vue'

const currentTab = ref('extractor')
const isDarkMode = ref(true) // Default state is Dark Mode

const toggleDarkMode = () => {
  isDarkMode.value = !isDarkMode.value
}

// Watch theme changes and update HTML class and localStorage
watch(isDarkMode, (newVal) => {
  if (newVal) {
    document.documentElement.classList.add('dark')
    localStorage.setItem('theme', 'dark')
  } else {
    document.documentElement.classList.remove('dark')
    localStorage.setItem('theme', 'light')
  }
}, { immediate: true })

onMounted(() => {
  const savedTheme = localStorage.getItem('theme')
  if (savedTheme) {
    isDarkMode.value = savedTheme === 'dark'
  } else {
    isDarkMode.value = true
  }
  
  if (isDarkMode.value) {
    document.documentElement.classList.add('dark')
  } else {
    document.documentElement.classList.remove('dark')
  }
})
</script>

<template>
  <div class="min-h-screen bg-slate-50 dark:bg-slate-950 text-slate-800 dark:text-slate-100 flex flex-col font-sans antialiased transition-colors duration-200">
    <!-- Navbar (Branding Header with Theme Toggle integration) -->
    <Navbar :isDarkMode="isDarkMode" @toggle-dark-mode="toggleDarkMode" />

    <!-- Menú Horizontal de Modos (Justo debajo del Header) -->
    <div class="bg-white dark:bg-slate-950 border-b border-slate-200/60 dark:border-slate-800/80 py-4 px-4 sm:px-6 shadow-sm transition-colors duration-200">
      <div class="max-w-7xl mx-auto flex items-center justify-center">
        <div class="flex items-center bg-slate-100 dark:bg-slate-900 p-1.5 rounded-2xl border border-slate-200/60 dark:border-slate-800 shadow-inner transition-colors duration-200">
          <!-- Pestaña 1: Extractor de Datos -->
          <button
            @click="currentTab = 'extractor'"
            :class="[
              'flex items-center gap-2.5 px-5 py-2.5 rounded-xl text-sm font-semibold tracking-wide transition-all duration-200 ease-in-out cursor-pointer select-none',
              currentTab === 'extractor'
                ? 'bg-white dark:bg-slate-800 text-indigo-650 dark:text-indigo-400 shadow-sm border border-slate-200/50 dark:border-slate-700/50 translate-y-[-1px]'
                : 'text-slate-500 dark:text-slate-400 hover:text-slate-800 dark:hover:text-slate-200 hover:bg-slate-200/40 dark:hover:bg-slate-800/40'
            ]"
          >
            <svg class="w-4.5 h-4.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
              <path stroke-linecap="round" stroke-linejoin="round" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4" />
            </svg>
            1. Extractor de Datos
          </button>

          <!-- Pestaña 2: Constructor de Queries SQL -->
          <button
            @click="currentTab = 'sql-builder'"
            :class="[
              'flex items-center gap-2.5 px-5 py-2.5 rounded-xl text-sm font-semibold tracking-wide transition-all duration-200 ease-in-out cursor-pointer select-none',
              currentTab === 'sql-builder'
                ? 'bg-white dark:bg-slate-800 text-indigo-650 dark:text-indigo-400 shadow-sm border border-slate-200/50 dark:border-slate-700/50 translate-y-[-1px]'
                : 'text-slate-500 dark:text-slate-400 hover:text-slate-800 dark:hover:text-slate-200 hover:bg-slate-200/40 dark:hover:bg-slate-800/40'
            ]"
          >
            <svg class="w-4.5 h-4.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
              <path stroke-linecap="round" stroke-linejoin="round" d="M10 20l4-16m4 4l4 4-4 4M6 16l-4-4 4-4" />
            </svg>
            2. Constructor de Queries SQL
          </button>
        </div>
      </div>
    </div>

    <!-- Main Content Area -->
    <main class="flex-grow bg-slate-50 dark:bg-slate-950 transition-colors duration-200">
      <div class="max-w-7xl mx-auto py-4">
        <!-- KeepAlive to preserve inputs and settings when switching between tabs -->
        <KeepAlive>
          <component :is="currentTab === 'extractor' ? DataExtractor : SqlQueryBuilder" />
        </KeepAlive>
      </div>
    </main>

    <!-- Footer Area -->
    <footer class="border-t border-slate-200/60 dark:border-slate-800/80 bg-white dark:bg-slate-950 py-6 text-slate-400 dark:text-slate-500 mt-auto transition-colors duration-200">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 flex flex-col sm:flex-row items-center justify-between gap-4 text-xs sm:text-sm">
        <div class="flex items-center gap-2">
          <div class="w-2 h-2 rounded-full bg-emerald-500 animate-pulse"></div>
          <p class="font-semibold tracking-wide text-slate-500 dark:text-slate-400">DataUtils Hub &copy; 2026. Todos los derechos reservados.</p>
        </div>
        <div class="flex items-center gap-3">
          <p class="font-medium text-slate-400 dark:text-slate-500">
            Desarrollado por <a href="https://kilamper.vercel.app/" target="_blank" rel="noopener noreferrer" class="text-indigo-500 dark:text-indigo-400 hover:text-indigo-650 dark:hover:text-indigo-300 hover:underline font-semibold transition-colors">Kilamper</a>
          </p>
          <span class="text-slate-200 dark:text-slate-800 select-none">|</span>
          <span class="text-[10px] tracking-wider uppercase text-slate-400 dark:text-slate-500 font-mono font-bold">
            Vue 3 Composition API
          </span>
        </div>
      </div>
    </footer>
  </div>
</template>

<style>
/* Base adjustments for the whole application scrollbar */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}
::-webkit-scrollbar-track {
  background: #f1f5f9;
}
::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 4px;
}
::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
}

/* Scrollbar adjustments for dark mode */
.dark ::-webkit-scrollbar-track {
  background: #020617;
}
.dark ::-webkit-scrollbar-thumb {
  background: #1e293b;
}
.dark ::-webkit-scrollbar-thumb:hover {
  background: #334155;
}
</style>
