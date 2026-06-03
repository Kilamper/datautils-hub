<script setup>
// Este componente representa la cabecera/header principal con el branding del toolkit y el toggle del tema.
defineProps({
  isDarkMode: {
    type: Boolean,
    required: true
  }
})
defineEmits(['toggle-dark-mode'])
</script>

<template>
  <header class="bg-white/80 dark:bg-slate-950/80 backdrop-blur-md border-b border-slate-200/80 dark:border-slate-800 px-4 sm:px-6 py-4 shadow-sm sticky top-0 z-50 transition-colors duration-200">
    <div class="max-w-7xl mx-auto flex items-center justify-between">
      <!-- Branding / Logo -->
      <div class="flex items-center gap-3 group select-none">
        <img
          src="/favicon.svg"
          alt="DataUtils Hub Logo"
          class="w-10 h-10 rounded-xl shadow-sm shadow-indigo-500/10 group-hover:scale-105 transition-transform duration-300"
        />
        <div>
          <span class="text-xl font-extrabold tracking-tight text-slate-900 dark:text-white">
            DataUtils Hub
          </span>
          <p class="text-[10px] text-slate-500 dark:text-slate-400 font-semibold tracking-wider uppercase -mt-0.5">Toolkit para Desarrolladores</p>
        </div>
      </div>

      <!-- Indicador de Estado Live & Toggle Theme -->
      <div class="flex items-center gap-3">
        <!-- Theme Toggle Button -->
        <button
          @click="$emit('toggle-dark-mode')"
          class="p-2.5 rounded-xl border border-slate-200 dark:border-slate-800 bg-slate-50 dark:bg-slate-900 text-slate-555 dark:text-slate-400 hover:bg-slate-100 dark:hover:bg-slate-800 hover:scale-[1.05] active:scale-95 transition-all duration-200 cursor-pointer flex items-center justify-center shadow-sm"
          title="Cambiar tema de color"
        >
          <!-- Sun Icon (visible in dark mode to switch to light) -->
          <svg v-if="isDarkMode" class="w-4.5 h-4.5 text-amber-500" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
            <path stroke-linecap="round" stroke-linejoin="round" d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364-6.364l-.707.707M6.343 17.657l-.707.707m12.728 0l-.707-.707M6.343 6.343l-.707-.707m2.828 9.9a5 5 0 117.072-7.072 5 5 0 01-7.072 7.072z" />
          </svg>
          <!-- Moon Icon (visible in light mode to switch to dark) -->
          <svg v-else class="w-4.5 h-4.5 text-indigo-600" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
            <path stroke-linecap="round" stroke-linejoin="round" d="M20.354 15.354A9 9 0 018.646 3.646 9.003 9.003 0 0012 21a9.003 9.003 0 008.354-5.646z" />
          </svg>
        </button>

        <!-- Indicador de Estado Live -->
        <div class="flex items-center gap-2 bg-slate-50 dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 px-3 py-1.5 rounded-xl shadow-sm">
          <span class="w-2 h-2 rounded-full bg-emerald-500 animate-pulse"></span>
          <span class="text-[10px] text-slate-500 dark:text-slate-400 font-bold uppercase tracking-widest font-mono">v1.1.0 Live</span>
        </div>
      </div>
    </div>
  </header>
</template>
