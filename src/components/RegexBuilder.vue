<script setup>
import { ref, computed } from 'vue'

const emit = defineEmits(['use-regex'])

// Start and End Anchors (Global Options)
const startAnchor = ref(false)
const endAnchor = ref(false)

// Modifiers / Flags (Global Options)
const caseInsensitive = ref(true)
const globalFlag = ref(true)

// Reactive array of sequential blocks
const blocks = ref([
  {
    id: 1,
    type: 'alphanumeric', // letters, numbers, alphanumeric, whitespace, any, text_fixed, custom_class
    textFixed: '',
    customClass: '',
    lengthType: 'one', // one, optional, zero_or_more, one_or_more, exact, range
    exactLength: 1,
    minLength: 1,
    maxLength: 5
  }
])

let nextBlockId = 2

// Default sandbox text that matches the initial visual blocks
const sandboxText = ref('Ejemplos de códigos a buscar:\nINV-2026-X\nINV-1999_ABC\nINV-1234-YYY\n\nPrueba a modificar los bloques de la izquierda para ver cómo cambia el resaltado.')

const regexCopied = ref(false)
const usedInExtractor = ref(false)

// Add a new default block to the list
const addBlock = () => {
  blocks.value.push({
    id: nextBlockId++,
    type: 'alphanumeric',
    textFixed: '',
    customClass: '',
    lengthType: 'one',
    exactLength: 1,
    minLength: 1,
    maxLength: 5
  })
}

// Remove block by ID (keep at least one)
const removeBlock = (id) => {
  if (blocks.value.length === 1) return
  blocks.value = blocks.value.filter(b => b.id !== id)
}

// Move block up in sequence
const moveBlockUp = (index) => {
  if (index === 0) return
  const temp = blocks.value[index]
  blocks.value[index] = blocks.value[index - 1]
  blocks.value[index - 1] = temp
}

// Move block down in sequence
const moveBlockDown = (index) => {
  if (index === blocks.value.length - 1) return
  const temp = blocks.value[index]
  blocks.value[index] = blocks.value[index + 1]
  blocks.value[index + 1] = temp
}

// Helper to escape special regex characters for literal match blocks
function escapeRegExp(string) {
  return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}

// Compiled Regex computed property
const compiledRegex = computed(() => {
  let pattern = ''
  
  for (const block of blocks.value) {
    let charClass = ''
    let isGroupable = false
    
    if (block.type === 'letters') {
      charClass = '[A-Za-z]'
    } else if (block.type === 'numbers') {
      charClass = '[0-9]'
    } else if (block.type === 'alphanumeric') {
      charClass = '[A-Za-z0-9]'
    } else if (block.type === 'whitespace') {
      charClass = '\\s'
    } else if (block.type === 'any') {
      charClass = '.'
    } else if (block.type === 'text_fixed') {
      const escaped = escapeRegExp(block.textFixed || '')
      charClass = escaped
      if (escaped.length > 1) {
        isGroupable = true
      }
    } else if (block.type === 'custom_class') {
      charClass = `[${block.customClass || ''}]`
    }
    
    let quantifier = ''
    if (block.lengthType === 'one') {
      quantifier = ''
    } else if (block.lengthType === 'optional') {
      quantifier = '?'
    } else if (block.lengthType === 'zero_or_more') {
      quantifier = '*'
    } else if (block.lengthType === 'one_or_more') {
      quantifier = '+'
    } else if (block.lengthType === 'exact') {
      const n = block.exactLength !== null && block.exactLength !== undefined ? block.exactLength : 1
      quantifier = `{${n}}`
    } else if (block.lengthType === 'range') {
      const min = block.minLength !== null && block.minLength !== undefined ? block.minLength : 0
      const max = block.maxLength !== null && block.maxLength !== undefined ? block.maxLength : ''
      quantifier = `{${min},${max}}`
    }
    
    if (quantifier && isGroupable) {
      pattern += `(?:${charClass})${quantifier}`
    } else {
      pattern += `${charClass}${quantifier}`
    }
  }

  const finalPattern = `${startAnchor.value ? '^' : ''}${pattern}${endAnchor.value ? '$' : ''}`
  const flags = `${globalFlag.value ? 'g' : ''}${caseInsensitive.value ? 'i' : ''}`

  return {
    pattern: finalPattern,
    flags,
    full: `/${finalPattern}/${flags}`
  }
})

// Highlighted regex output HTML representation for styling
const highlightedRegexHtml = computed(() => {
  let patternHtml = ''
  
  for (const block of blocks.value) {
    let charHtml = ''
    let isGroupable = false
    
    if (block.type === 'letters') {
      charHtml = '<span class="text-amber-500 dark:text-amber-400 font-semibold">[A-Za-z]</span>'
    } else if (block.type === 'numbers') {
      charHtml = '<span class="text-amber-500 dark:text-amber-400 font-semibold">[0-9]</span>'
    } else if (block.type === 'alphanumeric') {
      charHtml = '<span class="text-amber-500 dark:text-amber-400 font-semibold">[A-Za-z0-9]</span>'
    } else if (block.type === 'whitespace') {
      charHtml = '<span class="text-amber-500 dark:text-amber-400 font-semibold">\\s</span>'
    } else if (block.type === 'any') {
      charHtml = '<span class="text-amber-500 dark:text-amber-400 font-semibold">.</span>'
    } else if (block.type === 'text_fixed') {
      const escaped = escapeRegExp(block.textFixed || '')
      charHtml = `<span class="text-teal-400 dark:text-teal-300 font-semibold">${escapeHtml(escaped)}</span>`
      if (escaped.length > 1) {
        isGroupable = true
      }
    } else if (block.type === 'custom_class') {
      charHtml = `<span class="text-amber-500 dark:text-amber-400 font-semibold">[${escapeHtml(block.customClass || '')}]</span>`
    }
    
    let quantHtml = ''
    if (block.lengthType === 'one') {
      quantHtml = ''
    } else if (block.lengthType === 'optional') {
      quantHtml = '<span class="text-sky-500 dark:text-sky-400 font-semibold">?</span>'
    } else if (block.lengthType === 'zero_or_more') {
      quantHtml = '<span class="text-sky-500 dark:text-sky-400 font-semibold">*</span>'
    } else if (block.lengthType === 'one_or_more') {
      quantHtml = '<span class="text-sky-500 dark:text-sky-400 font-semibold">+</span>'
    } else if (block.lengthType === 'exact') {
      const n = block.exactLength !== null && block.exactLength !== undefined ? block.exactLength : 1
      quantHtml = `<span class="text-sky-500 dark:text-sky-400 font-semibold">{${n}}</span>`
    } else if (block.lengthType === 'range') {
      const min = block.minLength !== null && block.minLength !== undefined ? block.minLength : 0
      const max = block.maxLength !== null && block.maxLength !== undefined ? block.maxLength : ''
      quantHtml = `<span class="text-sky-500 dark:text-sky-400 font-semibold">{${min},${max}}</span>`
    }
    
    if (quantHtml && isGroupable) {
      patternHtml += `<span class="text-slate-500">(?:</span>${charHtml}<span class="text-slate-500">)</span>${quantHtml}`
    } else {
      patternHtml += `${charHtml}${quantHtml}`
    }
  }

  const startAnchorHtml = startAnchor.value ? '<span class="text-emerald-500 dark:text-emerald-400 font-semibold">^</span>' : ''
  const endAnchorHtml = endAnchor.value ? '<span class="text-emerald-500 dark:text-emerald-400 font-semibold">$</span>' : ''
  const flagsHtml = `<span class="text-pink-500 dark:text-pink-400 font-semibold">${globalFlag.value ? 'g' : ''}${caseInsensitive.value ? 'i' : ''}</span>`

  return `<span class="text-slate-500">/</span>${startAnchorHtml}${patternHtml}${endAnchorHtml}<span class="text-slate-500">/</span>${flagsHtml}`
})

// Sandbox testing computed results with safe regex compilation
const sandboxResults = computed(() => {
  const raw = sandboxText.value
  if (!raw) {
    return {
      html: '<span class="text-slate-500 dark:text-slate-500 italic">Escribe algún texto arriba para ver el resultado...</span>',
      count: 0,
      isValid: true,
      error: ''
    }
  }

  try {
    // Force global flag for matches highlighting loop safety
    let flags = compiledRegex.value.flags
    if (!flags.includes('g')) {
      flags += 'g'
    }

    const regex = new RegExp(compiledRegex.value.pattern, flags)
    const matches = [...raw.matchAll(regex)]
    
    if (matches.length === 0) {
      return {
        html: escapeHtml(raw),
        count: 0,
        isValid: true,
        error: ''
      }
    }

    let resultHtml = ''
    let lastIndex = 0
    let matchCount = 0

    for (const match of matches) {
      const matchText = match[0]
      const index = match.index

      if (matchText.length === 0) {
        continue
      }

      matchCount++
      resultHtml += escapeHtml(raw.substring(lastIndex, index))
      resultHtml += `<mark class="bg-indigo-500/20 dark:bg-indigo-400/25 text-indigo-700 dark:text-indigo-350 border-b-2 border-indigo-500 dark:border-indigo-400 px-0.5 rounded transition-all font-mono font-semibold" title="Coincidencia #${matchCount}">${escapeHtml(matchText)}</mark>`
      lastIndex = index + matchText.length
    }

    resultHtml += escapeHtml(raw.substring(lastIndex))

    return {
      html: resultHtml,
      count: matchCount,
      isValid: true,
      error: ''
    }
  } catch (err) {
    return {
      html: escapeHtml(raw),
      count: 0,
      isValid: false,
      error: err.message
    }
  }
})

// Helper to escape HTML characters
function escapeHtml(text) {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;')
}

// Copy full regex to clipboard
const copyRegex = async () => {
  try {
    await navigator.clipboard.writeText(compiledRegex.value.full)
    regexCopied.value = true
    setTimeout(() => {
      regexCopied.value = false
    }, 2000)
  } catch (err) {
    console.error('Error al copiar: ', err)
  }
}

// Export raw pattern to clipboard and switch/inject to extractor tab
const useInExtractor = async () => {
  try {
    await navigator.clipboard.writeText(compiledRegex.value.pattern)
    usedInExtractor.value = true
    
    emit('use-regex', compiledRegex.value.pattern)

    setTimeout(() => {
      usedInExtractor.value = false
    }, 2000)
  } catch (err) {
    console.error('Error al exportar: ', err)
  }
}
</script>

<template>
  <div class="grid grid-cols-1 lg:grid-cols-12 gap-8 max-w-7xl mx-auto px-4 sm:px-6 py-8 animate-fade-in">
    <!-- Configuration Panel (Col span 7) -->
    <div class="lg:col-span-7 flex flex-col gap-6">
      
      <!-- Card 1: Constructor por Bloques -->
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-6 transition-colors duration-200">
        <div class="flex items-center justify-between">
          <h2 class="text-xl font-bold text-slate-800 dark:text-slate-100 flex items-center gap-2">
            <span class="w-2.5 h-6 bg-indigo-600 rounded-full inline-block"></span>
            Constructor por Bloques
          </h2>
          <span class="text-xs text-slate-400 dark:text-slate-550 font-semibold uppercase tracking-wider">Paso 1: Secuencia</span>
        </div>

        <!-- Block List -->
        <div class="flex flex-col gap-4">
          <div 
            v-for="(block, index) in blocks" 
            :key="block.id"
            class="bg-slate-50/50 dark:bg-slate-950/40 p-4 sm:p-5 rounded-2xl border border-slate-200 dark:border-slate-800/80 flex flex-col gap-4 relative group hover:border-slate-300 dark:hover:border-slate-700 transition-all"
          >
            <!-- Block Header with controls -->
            <div class="flex items-center justify-between border-b border-slate-150 dark:border-slate-800/80 pb-2.5">
              <span class="text-xs font-extrabold text-slate-500 dark:text-slate-400 uppercase tracking-wide flex items-center gap-1.5">
                <span class="w-2 h-2 rounded-full bg-indigo-500"></span>
                Bloque #{{ index + 1 }}
              </span>
              
              <!-- Reordering and deleting buttons -->
              <div class="flex items-center gap-1">
                <!-- Move Up -->
                <button 
                  type="button"
                  @click="moveBlockUp(index)"
                  :disabled="index === 0"
                  class="p-1.5 rounded-lg border border-slate-200 dark:border-slate-800 bg-white dark:bg-slate-900 hover:bg-slate-100 dark:hover:bg-slate-800 text-slate-500 dark:text-slate-400 disabled:opacity-30 disabled:cursor-not-allowed transition cursor-pointer"
                  title="Subir bloque"
                >
                  <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M5 15l7-7 7 7" />
                  </svg>
                </button>

                <!-- Move Down -->
                <button 
                  type="button"
                  @click="moveBlockDown(index)"
                  :disabled="index === blocks.length - 1"
                  class="p-1.5 rounded-lg border border-slate-200 dark:border-slate-800 bg-white dark:bg-slate-900 hover:bg-slate-100 dark:hover:bg-slate-800 text-slate-500 dark:text-slate-400 disabled:opacity-30 disabled:cursor-not-allowed transition cursor-pointer"
                  title="Bajar bloque"
                >
                  <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M19 9l-7 7-7-7" />
                  </svg>
                </button>

                <!-- Delete Block -->
                <button 
                  type="button"
                  @click="removeBlock(block.id)"
                  :disabled="blocks.length === 1"
                  class="p-1.5 rounded-lg border border-red-200/50 dark:border-red-900/35 bg-white dark:bg-slate-900 hover:bg-red-50 dark:hover:bg-red-950/20 text-red-500 disabled:opacity-30 disabled:cursor-not-allowed transition cursor-pointer ml-1"
                  title="Eliminar bloque"
                >
                  <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                  </svg>
                </button>
              </div>
            </div>

            <!-- Block Controls Grid -->
            <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
              <!-- Type Dropdown -->
              <div class="flex flex-col gap-1.5">
                <label class="text-[10px] font-bold text-slate-400 dark:text-slate-500 uppercase tracking-wider">Tipo de coincidencia</label>
                <select 
                  v-model="block.type"
                  class="w-full bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-3 py-2 text-sm font-bold focus:outline-none focus:ring-2 focus:ring-indigo-500/20 text-slate-800 dark:text-slate-200 cursor-pointer"
                >
                  <option value="alphanumeric">Alfanumérico (Letras y números)</option>
                  <option value="letters">Letras (A-Z, a-z)</option>
                  <option value="numbers">Números (0-9)</option>
                  <option value="whitespace">Espacio en blanco (\s)</option>
                  <option value="any">Cualquier Carácter (.)</option>
                  <option value="text_fixed">Texto Fijo / Literal</option>
                  <option value="custom_class">Grupo Personalizado [abc]</option>
                </select>
              </div>

              <!-- Repetition Dropdown -->
              <div class="flex flex-col gap-1.5">
                <label class="text-[10px] font-bold text-slate-400 dark:text-slate-500 uppercase tracking-wider">Repetición / Longitud</label>
                <select 
                  v-model="block.lengthType"
                  class="w-full bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-3 py-2 text-sm font-bold focus:outline-none focus:ring-2 focus:ring-indigo-500/20 text-slate-800 dark:text-slate-200 cursor-pointer"
                >
                  <option value="one">Exactamente 1 vez</option>
                  <option value="optional">Opcional (0 o 1 vez) (?)</option>
                  <option value="zero_or_more">Cero o más veces (*)</option>
                  <option value="one_or_more">Una o más veces (+)</option>
                  <option value="exact">Longitud exacta (N)</option>
                  <option value="range">Rango de longitud (N a M)</option>
                </select>
              </div>
            </div>

            <!-- Conditional Controls Row -->
            <div class="flex flex-wrap items-center gap-4 bg-white/40 dark:bg-slate-900/30 border border-slate-200/50 dark:border-slate-800/60 rounded-xl p-3">
              <!-- Text Fixed Input -->
              <div v-if="block.type === 'text_fixed'" class="flex items-center gap-2 flex-grow">
                <span class="text-xs font-bold text-slate-500 dark:text-slate-400 uppercase tracking-wider shrink-0">Texto literal:</span>
                <input 
                  v-model="block.textFixed"
                  type="text"
                  placeholder="Ej. INV-"
                  class="flex-grow bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-3 py-1.5 text-sm font-semibold focus:outline-none focus:ring-2 focus:ring-indigo-500/20 text-slate-800 dark:text-slate-200"
                />
              </div>

              <!-- Custom Class Input -->
              <div v-if="block.type === 'custom_class'" class="flex items-center gap-2 flex-grow">
                <span class="text-xs font-bold text-slate-500 dark:text-slate-400 uppercase tracking-wider shrink-0">Caracteres permitidos:</span>
                <input 
                  v-model="block.customClass"
                  type="text"
                  placeholder="Ej. a-z-_"
                  class="flex-grow bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-3 py-1.5 text-sm font-semibold focus:outline-none focus:ring-2 focus:ring-indigo-500/20 text-slate-800 dark:text-slate-200"
                />
              </div>

              <!-- Quantifier Exact Inputs -->
              <div v-if="block.lengthType === 'exact'" class="flex items-center gap-2 flex-grow">
                <span class="text-xs font-bold text-slate-500 dark:text-slate-400 uppercase tracking-wider shrink-0">Veces a repetir:</span>
                <input 
                  v-model.number="block.exactLength"
                  type="number"
                  min="1"
                  class="w-20 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-2.5 py-1.5 text-sm font-bold text-center focus:outline-none text-slate-800 dark:text-slate-200"
                />
              </div>

              <!-- Quantifier Range Inputs -->
              <div v-if="block.lengthType === 'range'" class="flex items-center gap-2 flex-grow">
                <span class="text-xs font-bold text-slate-550 dark:text-slate-400 uppercase tracking-wider shrink-0">Repetir de</span>
                <input 
                  v-model.number="block.minLength"
                  type="number"
                  min="0"
                  placeholder="Mín"
                  class="w-16 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-2 py-1.5 text-sm font-bold text-center focus:outline-none text-slate-800 dark:text-slate-200"
                />
                <span class="text-xs font-bold text-slate-550 dark:text-slate-400 uppercase tracking-wider">a</span>
                <input 
                  v-model.number="block.maxLength"
                  type="number"
                  min="1"
                  placeholder="Máx"
                  class="w-16 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-2 py-1.5 text-sm font-bold text-center focus:outline-none text-slate-800 dark:text-slate-200"
                />
                <span class="text-xs font-bold text-slate-550 dark:text-slate-400 uppercase tracking-wider">veces</span>
              </div>

              <!-- General Info label -->
              <div class="text-[10px] text-slate-400 dark:text-slate-500 font-bold uppercase tracking-widest ml-auto shrink-0 font-mono">
                {{ block.type === 'text_fixed' ? 'Literal' : block.type === 'custom_class' ? 'Clase' : 'Estándar' }}
              </div>
            </div>
          </div>
        </div>

        <!-- Add block button -->
        <button 
          type="button"
          @click="addBlock"
          class="w-full py-4 border-2 border-dashed border-slate-200 dark:border-slate-800 hover:border-indigo-500 dark:hover:border-indigo-500/60 rounded-2xl flex items-center justify-center gap-2 text-sm font-bold text-slate-500 dark:text-slate-400 hover:text-indigo-600 dark:hover:text-indigo-400 bg-slate-50/30 hover:bg-indigo-50/10 dark:hover:bg-indigo-950/10 cursor-pointer transition-all"
        >
          <svg class="w-4.5 h-4.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
            <path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4" />
          </svg>
          Añadir nuevo bloque
        </button>
      </div>

      <!-- Card 2: Opciones Globales -->
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-6 transition-colors duration-200">
        <div class="flex items-center justify-between">
          <h2 class="text-xl font-bold text-slate-800 dark:text-slate-100 flex items-center gap-2">
            <span class="w-2.5 h-6 bg-slate-400 rounded-full inline-block"></span>
            Opciones Globales
          </h2>
          <span class="text-xs text-slate-400 dark:text-slate-550 font-semibold uppercase tracking-wider">Parámetros</span>
        </div>

        <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
          <!-- Debe empezar al principio -->
          <label class="inline-flex items-center justify-between cursor-pointer group select-none bg-slate-50 dark:bg-slate-950 border border-slate-200/60 dark:border-slate-800 rounded-2xl px-4 py-3.5 transition-all hover:border-slate-300 dark:hover:border-slate-700">
            <div>
              <span class="text-xs font-bold text-slate-750 dark:text-slate-200">Anclar al Inicio (^)</span>
              <p class="text-[9px] text-slate-500 dark:text-slate-450 font-medium">El texto debe empezar así</p>
            </div>
            <div class="relative">
              <input type="checkbox" v-model="startAnchor" class="sr-only peer" />
              <div class="w-9 h-5 bg-slate-200 dark:bg-slate-800 border border-slate-300 dark:border-slate-700 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-slate-300 dark:after:border-slate-600 after:border after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:bg-indigo-600 relative transition-colors duration-300"></div>
            </div>
          </label>

          <!-- Debe terminar al final -->
          <label class="inline-flex items-center justify-between cursor-pointer group select-none bg-slate-50 dark:bg-slate-950 border border-slate-200/60 dark:border-slate-800 rounded-2xl px-4 py-3.5 transition-all hover:border-slate-300 dark:hover:border-slate-700">
            <div>
              <span class="text-xs font-bold text-slate-750 dark:text-slate-200">Anclar al Final ($)</span>
              <p class="text-[9px] text-slate-500 dark:text-slate-450 font-medium">El texto debe terminar así</p>
            </div>
            <div class="relative">
              <input type="checkbox" v-model="endAnchor" class="sr-only peer" />
              <div class="w-9 h-5 bg-slate-200 dark:bg-slate-800 border border-slate-300 dark:border-slate-700 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-slate-300 dark:after:border-slate-600 after:border after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:bg-indigo-600 relative transition-colors duration-300"></div>
            </div>
          </label>

          <!-- Ignore Case -->
          <label class="inline-flex items-center justify-between cursor-pointer group select-none bg-slate-50 dark:bg-slate-950 border border-slate-200/60 dark:border-slate-800 rounded-2xl px-4 py-3.5 transition-all hover:border-slate-300 dark:hover:border-slate-700">
            <div>
              <span class="text-xs font-bold text-slate-750 dark:text-slate-200">Ignorar mayúsculas (Flag i)</span>
              <p class="text-[9px] text-slate-500 dark:text-slate-450 font-medium">Insensible a mayús/minús</p>
            </div>
            <div class="relative">
              <input type="checkbox" v-model="caseInsensitive" class="sr-only peer" />
              <div class="w-9 h-5 bg-slate-200 dark:bg-slate-800 border border-slate-300 dark:border-slate-700 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-slate-300 dark:after:border-slate-600 after:border after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:bg-indigo-600 relative transition-colors duration-300"></div>
            </div>
          </label>

          <!-- Global -->
          <label class="inline-flex items-center justify-between cursor-pointer group select-none bg-slate-50 dark:bg-slate-950 border border-slate-200/60 dark:border-slate-800 rounded-2xl px-4 py-3.5 transition-all hover:border-slate-300 dark:hover:border-slate-700">
            <div>
              <span class="text-xs font-bold text-slate-750 dark:text-slate-200">Búsqueda global (Flag g)</span>
              <p class="text-[9px] text-slate-500 dark:text-slate-450 font-medium">Busca todas las coincidencias</p>
            </div>
            <div class="relative">
              <input type="checkbox" v-model="globalFlag" class="sr-only peer" />
              <div class="w-9 h-5 bg-slate-200 dark:bg-slate-800 border border-slate-300 dark:border-slate-700 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-slate-300 dark:after:border-slate-600 after:border after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:bg-indigo-600 relative transition-colors duration-300"></div>
            </div>
          </label>
        </div>
      </div>

      <!-- Card 3: Patrón Generado -->
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-5 transition-colors duration-200">
        <div class="flex items-center justify-between">
          <h2 class="text-xl font-bold text-slate-800 dark:text-slate-100 flex items-center gap-2">
            <span class="w-2.5 h-6 bg-emerald-500 rounded-full inline-block"></span>
            Patrón Generado
          </h2>
          <span class="text-xs text-slate-400 dark:text-slate-550 font-semibold uppercase tracking-wider">Paso 2: Compilador</span>
        </div>

        <!-- Output display editor block -->
        <div class="bg-slate-950 border border-slate-950 rounded-2xl p-5 shadow-inner flex items-center justify-between gap-4 overflow-hidden relative group">
          <div v-html="highlightedRegexHtml" class="font-mono text-base sm:text-lg select-all truncate selection:bg-indigo-500/30 selection:text-white" dir="ltr"></div>
          
          <div class="text-[10px] uppercase font-bold text-slate-600 tracking-wider select-none shrink-0 border border-slate-800 rounded px-1.5 py-0.5 bg-slate-900">
            Regex
          </div>
        </div>

        <!-- Action buttons grid -->
        <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
          <!-- Copiar Regex -->
          <button
            @click="copyRegex"
            :class="[
              'w-full py-3.5 rounded-xl font-bold text-sm tracking-wide flex items-center justify-center gap-2 cursor-pointer shadow-md transition-all duration-200 ease-in-out active:scale-[0.98]',
              regexCopied
                ? 'bg-emerald-600 text-white shadow-lg shadow-emerald-500/10'
                : 'bg-slate-100 hover:bg-slate-200 dark:bg-slate-800 dark:hover:bg-slate-700 text-slate-700 dark:text-slate-300 border border-slate-200/50 dark:border-slate-700/50'
            ]"
          >
            <!-- Clipboard Icon / Success Icon -->
            <svg v-if="!regexCopied" class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
              <path stroke-linecap="round" stroke-linejoin="round" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3" />
            </svg>
            <svg v-else class="w-4 h-4 animate-bounce" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
              <path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7" />
            </svg>
            {{ regexCopied ? '¡Copiado!' : 'Copiar Regex' }}
          </button>

          <!-- Usar en Extractor -->
          <button
            @click="useInExtractor"
            :class="[
              'w-full py-3.5 rounded-xl font-bold text-sm tracking-wide flex items-center justify-center gap-2 cursor-pointer shadow-md transition-all duration-200 ease-in-out active:scale-[0.98] text-white',
              usedInExtractor
                ? 'bg-emerald-600 shadow-lg shadow-emerald-500/10'
                : 'bg-indigo-650 hover:bg-indigo-700 hover:scale-[1.01] shadow-lg shadow-indigo-650/20'
            ]"
          >
            <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
              <path stroke-linecap="round" stroke-linejoin="round" d="M8.684 10.742L12 8l3.316 2.742m0 2.516L12 16l-3.316-2.742" />
            </svg>
            {{ usedInExtractor ? '¡Exportado!' : 'Usar en Extractor' }}
          </button>
        </div>
      </div>
    </div>

    <!-- Sandbox Testing Panel (Col span 5) -->
    <div class="lg:col-span-5 flex flex-col gap-6">
      <!-- Card 3: Área de Pruebas (The Sandbox) -->
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-6 h-full transition-colors duration-200">
        <div class="flex items-center justify-between">
          <h2 class="text-xl font-bold text-slate-800 dark:text-slate-100 flex items-center gap-2">
            <span class="w-2.5 h-6 bg-sky-500 rounded-full inline-block"></span>
            Área de Pruebas
          </h2>
          <!-- Matches Counter Badge -->
          <div :class="[
            'flex items-center gap-1.5 border px-3 py-1.5 rounded-full select-none transition-colors',
            !sandboxResults.isValid 
              ? 'bg-red-500/10 border-red-500/25 text-red-550 dark:text-red-400' 
              : sandboxResults.count > 0 
                ? 'bg-emerald-500/10 border-emerald-500/20 text-emerald-600 dark:text-emerald-450' 
                : 'bg-slate-50 dark:bg-slate-800/60 border-slate-200 dark:border-slate-700 text-slate-500'
          ]">
            <span class="text-[10px] font-bold uppercase tracking-wider">Coincidencias:</span>
            <span class="text-xs font-extrabold font-mono">{{ sandboxResults.count }}</span>
          </div>
        </div>

        <!-- Raw Text Testarea -->
        <div class="relative flex flex-col">
          <textarea
            v-model="sandboxText"
            rows="6"
            placeholder="Escribe o pega texto aquí para probar tu expresión regular..."
            class="w-full bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-2xl p-4 text-sm font-mono placeholder:text-slate-400 dark:placeholder:text-slate-650 focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out resize-y"
          ></textarea>
          
          <button
            v-if="sandboxText"
            @click="sandboxText = ''"
            class="absolute top-4 right-4 bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-slate-500 dark:text-slate-400 hover:text-slate-800 dark:hover:text-slate-200 px-2.5 py-1.5 rounded-lg text-xs font-bold transition-all duration-200 ease-in-out cursor-pointer active:scale-95 shadow-sm"
          >
            Limpiar
          </button>
        </div>

        <!-- Real-Time Viewer Output -->
        <div class="flex-grow flex flex-col min-h-[220px] relative mt-1">
          <label class="text-xs font-bold text-slate-500 dark:text-slate-400 uppercase tracking-wider mb-2">Visor de Coincidencias</label>
          <div 
            class="w-full flex-grow bg-slate-950 text-slate-350 border border-slate-950 rounded-2xl p-4 text-sm font-mono whitespace-pre-wrap break-all min-h-[160px] leading-relaxed selection:bg-indigo-500/30 select-text overflow-y-auto font-bold"
            v-html="sandboxResults.html"
          ></div>

          <!-- Compilation Error State -->
          <div 
            v-if="!sandboxResults.isValid" 
            class="bg-red-500/10 border border-red-500/20 text-red-500 text-xs rounded-xl p-3.5 font-semibold mt-3 flex items-start gap-2.5 animate-fade-in"
          >
            <svg class="w-4 h-4 shrink-0 mt-0.5 text-red-500" fill="none" stroke="currentColor" stroke-width="2" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
            </svg>
            <div>
              <p class="font-bold">Error de compilación de la Regex</p>
              <p class="text-[11px] opacity-80 mt-0.5 font-mono leading-normal">{{ sandboxResults.error }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Scoped fade in transition */
.animate-fade-in {
  animation: fadeIn 0.25s ease-out forwards;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(4px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
