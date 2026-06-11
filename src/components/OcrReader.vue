<script setup>
import { ref } from 'vue'
import Tesseract from 'tesseract.js'

const rawText = ref('')
const fileInput = ref(null)
const isDragging = ref(false)
const ocrProgress = ref(0)
const isOcrProcessing = ref(false)
const ocrStatusText = ref('')
const ocrError = ref('')
const ocrLanguage = ref('spa+eng')
const copied = ref(false)

function showOcrError(msg) {
  ocrError.value = msg
  setTimeout(() => {
    if (ocrError.value === msg) {
      ocrError.value = ''
    }
  }, 5000)
}

function handleFileSelect(event) {
  const file = event.target.files?.[0]
  if (file) {
    processImageFile(file)
  }
}

function handleDrop(event) {
  const file = event.dataTransfer?.files?.[0]
  if (file) {
    processImageFile(file)
  }
}

function triggerFileInput() {
  fileInput.value?.click()
}

async function processImageFile(file) {
  if (!file) return
  if (!['image/png', 'image/jpeg', 'image/jpg', 'image/webp'].includes(file.type)) {
    showOcrError('No se pudo procesar la imagen o el formato no es compatible')
    return
  }

  isOcrProcessing.value = true
  ocrProgress.value = 0
  ocrStatusText.value = 'Inicializando OCR...'
  ocrError.value = ''

  try {
    const reader = new FileReader()
    reader.onload = async (e) => {
      if (!isOcrProcessing.value) return
      const imageSrc = e.target.result
      try {
        const result = await Tesseract.recognize(
          imageSrc,
          ocrLanguage.value,
          {
            logger: (m) => {
              if (!isOcrProcessing.value) return
              if (m.status === 'recognizing text') {
                ocrProgress.value = Math.round(m.progress * 100)
                ocrStatusText.value = `Reconociendo texto: ${ocrProgress.value}%`
              } else {
                ocrStatusText.value = m.status === 'loading tesseract core' ? 'Cargando motor de OCR...' : 
                                      m.status === 'initializing api' ? 'Inicializando API...' : 
                                      m.status === 'loading language traineddata' ? 'Cargando datos de idioma...' : m.status
              }
            }
          }
        )
        
        if (!isOcrProcessing.value) return
        if (result && result.data && typeof result.data.text === 'string') {
          rawText.value = result.data.text
          ocrStatusText.value = '¡Extracción completada!'
        } else {
          throw new Error('No se detectó texto en la imagen')
        }
      } catch (err) {
        console.error('Tesseract error:', err)
        if (isOcrProcessing.value) {
          showOcrError('No se pudo procesar la imagen o el formato no es compatible')
        }
      } finally {
        isOcrProcessing.value = false
      }
    }
    
    reader.onerror = () => {
      if (isOcrProcessing.value) {
        showOcrError('Error al leer el archivo de imagen')
        isOcrProcessing.value = false
      }
    }
    
    reader.readAsDataURL(file)
  } catch (err) {
    console.error(err)
    if (isOcrProcessing.value) {
      showOcrError('No se pudo procesar la imagen o el formato no es compatible')
      isOcrProcessing.value = false
    }
  }
}

const copyToClipboard = async () => {
  if (!rawText.value) return

  try {
    await navigator.clipboard.writeText(rawText.value)
    copied.value = true
    setTimeout(() => {
      copied.value = false
    }, 2000)
  } catch (err) {
    console.error('Error al copiar: ', err)
  }
}
</script>

<template>
  <div class="grid grid-cols-1 lg:grid-cols-12 gap-8 max-w-7xl mx-auto px-4 sm:px-6 py-8">
    <!-- Entrada / Carga de Imagen (Col span 6) -->
    <div class="lg:col-span-6 flex flex-col gap-6">
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-6 transition-colors duration-200">
        <div class="flex items-center justify-between">
          <h2 class="text-xl font-bold text-slate-800 dark:text-slate-100 flex items-center gap-2">
            <span class="w-2.5 h-6 bg-indigo-600 rounded-full inline-block"></span>
            Subir Imagen / Documento
          </h2>
          <span class="text-xs text-slate-400 dark:text-slate-550 font-semibold uppercase tracking-wider">Lector OCR</span>
        </div>

        <!-- Drag-and-drop zone -->
        <div 
          @click="triggerFileInput"
          @dragover.prevent="isDragging = true"
          @dragenter.prevent="isDragging = true"
          @dragleave.prevent="isDragging = false"
          @dragend.prevent="isDragging = false"
          @drop.prevent="isDragging = false; handleDrop($event)"
          :class="[
            'border-dashed border-2 rounded-xl p-8 text-center cursor-pointer transition-all duration-200 flex flex-col items-center justify-center gap-4 select-none min-h-[260px]',
            isDragging 
              ? 'border-indigo-600 bg-indigo-50/50 dark:bg-indigo-950/20' 
              : 'border-slate-200 dark:border-slate-800 hover:border-indigo-500/50 dark:hover:border-indigo-550/30 bg-slate-50/50 dark:bg-slate-900/40 hover:bg-slate-100/30 dark:hover:bg-slate-850/20'
          ]"
        >
          <input 
            type="file" 
            ref="fileInput" 
            accept="image/png, image/jpeg, image/jpg, image/webp" 
            class="hidden" 
            @change="handleFileSelect" 
          />
          
          <!-- Upload Icon -->
          <div class="p-4 bg-white dark:bg-slate-800 rounded-2xl shadow-sm border border-slate-100 dark:border-slate-800 text-indigo-650 dark:text-indigo-400">
            <svg class="w-8 h-8 animate-pulse" fill="none" stroke="currentColor" stroke-width="2" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
            </svg>
          </div>
          
          <div class="flex flex-col gap-1 max-w-sm">
            <p class="text-base font-bold text-slate-700 dark:text-slate-200">
              Arrastra una imagen o haz clic para seleccionar
            </p>
            <p class="text-xs text-slate-400 dark:text-slate-500">
              Soporta archivos PNG, JPG, JPEG y WEBP
            </p>
          </div>
        </div>

        <!-- OCR Language Selector -->
        <div class="flex flex-col sm:flex-row items-start sm:items-center justify-between gap-3 text-xs border-b border-slate-100 dark:border-slate-850 pb-4">
          <span class="font-bold text-slate-500 dark:text-slate-450 uppercase tracking-wider">Idioma de Reconocimiento</span>
          <div class="flex gap-2 flex-wrap">
            <button 
              v-for="lang in [{ code: 'spa', name: 'Español' }, { code: 'eng', name: 'Inglés' }, { code: 'spa+eng', name: 'Ambos (ESP+ENG)' }]"
              :key="lang.code"
              @click="ocrLanguage = lang.code"
              type="button"
              :class="[
                'px-3.5 py-1.5 rounded-lg font-bold transition-all cursor-pointer text-[11px]',
                ocrLanguage === lang.code 
                  ? 'bg-indigo-650 text-white shadow-sm shadow-indigo-650/10' 
                  : 'bg-slate-100 dark:bg-slate-800 text-slate-600 dark:text-slate-450 hover:bg-slate-200 dark:hover:bg-slate-700'
              ]"
            >
              {{ lang.name }}
            </button>
          </div>
        </div>

        <!-- OCR Loading/Progress State -->
        <div 
          v-if="isOcrProcessing" 
          class="bg-indigo-50/30 dark:bg-indigo-950/10 border border-indigo-150 dark:border-indigo-900/50 rounded-2xl p-5 flex flex-col gap-4"
        >
          <div class="flex items-center justify-between">
            <div class="flex items-center gap-3 text-indigo-600 dark:text-indigo-400">
              <svg class="animate-spin h-5 w-5" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
              </svg>
              <span class="text-sm font-bold text-slate-750 dark:text-slate-305">
                Procesando imagen con OCR...
              </span>
            </div>
            <span class="text-xs font-mono font-extrabold text-indigo-650 dark:text-indigo-400 bg-indigo-50 dark:bg-indigo-950/30 px-2.5 py-1 rounded-md">
              {{ ocrProgress }}%
            </span>
          </div>
          
          <div class="w-full bg-slate-200 dark:bg-slate-800 rounded-full h-2.5 overflow-hidden">
            <div 
              class="bg-indigo-600 h-full transition-all duration-300 ease-out" 
              :style="{ width: `${ocrProgress}%` }"
            ></div>
          </div>
          
          <div class="flex justify-between items-center text-[11px] text-slate-400 dark:text-slate-500 font-semibold uppercase tracking-wider">
            <span>{{ ocrStatusText }}</span>
            <button 
              @click="isOcrProcessing = false" 
              type="button"
              class="text-red-500 hover:text-red-650 cursor-pointer font-bold transition-colors"
            >
              Cancelar
            </button>
          </div>
        </div>

        <!-- OCR Error Alert -->
        <div 
          v-if="ocrError" 
          class="bg-rose-500/10 border border-rose-200 dark:border-rose-900/50 rounded-2xl p-4 flex items-center justify-between text-rose-850 dark:text-rose-300"
        >
          <div class="flex items-center gap-3">
            <svg class="w-5 h-5 text-rose-500 flex-shrink-0" fill="none" stroke="currentColor" stroke-width="2" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
            </svg>
            <span class="text-sm font-semibold">{{ ocrError }}</span>
          </div>
          <button 
            @click="ocrError = ''" 
            type="button"
            class="text-rose-500 hover:text-rose-700 dark:hover:text-rose-400 font-extrabold text-sm p-1 cursor-pointer"
          >
            ✕
          </button>
        </div>
      </div>
    </div>

    <!-- Salida de Texto Extraído (Col span 6) -->
    <div class="lg:col-span-6 flex flex-col gap-6">
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-6 h-full min-h-[400px] transition-colors duration-200">
        <div class="flex items-center justify-between">
          <h2 class="text-xl font-bold text-slate-800 dark:text-slate-100 flex items-center gap-2">
            <span class="w-2.5 h-6 bg-emerald-500 rounded-full inline-block"></span>
            Texto Extractor Completo
          </h2>
          <button
            v-if="rawText"
            @click="rawText = ''"
            class="bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-slate-550 dark:text-slate-400 hover:text-slate-800 dark:hover:text-slate-205 px-2.5 py-1.5 rounded-lg text-xs font-bold transition-all duration-200 ease-in-out cursor-pointer active:scale-95 shadow-sm"
          >
            Limpiar Texto
          </button>
        </div>

        <!-- Output TextArea -->
        <div class="flex-grow flex flex-col relative min-h-[240px]">
          <textarea
            readonly
            :value="rawText"
            placeholder="El texto extraído de la imagen aparecerá aquí..."
            class="w-full flex-grow bg-slate-950 text-slate-200 border border-slate-950 rounded-2xl p-5 text-sm font-mono placeholder:text-slate-600 focus:outline-none focus:ring-0 resize-none h-full selection:bg-indigo-500/30 selection:text-white"
          ></textarea>

          <!-- Empty State Overlay -->
          <div
            v-if="!rawText"
            class="absolute inset-0 flex flex-col items-center justify-center gap-3.5 p-6 pointer-events-none text-center"
          >
            <div class="w-12 h-12 rounded-xl bg-slate-800 dark:bg-slate-900 border border-slate-950 flex items-center justify-center text-slate-400">
              <svg class="w-6 h-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
              </svg>
            </div>
            <div>
              <p class="text-sm font-bold text-slate-300">Esperando texto</p>
              <p class="text-xs text-slate-500 dark:text-slate-550 mt-1 max-w-[240px] mx-auto">Sube o arrastra una imagen en el panel de la izquierda para comenzar el escaneo.</p>
            </div>
          </div>
        </div>

        <!-- Copy Action Button -->
        <button
          @click="copyToClipboard"
          :disabled="!rawText"
          :class="[
            'w-full py-4 rounded-xl font-bold text-sm tracking-wide flex items-center justify-center gap-2 cursor-pointer shadow-md transition-all duration-200 ease-in-out active:scale-[0.98]',
            !rawText
              ? 'bg-slate-100 dark:bg-slate-800/40 text-slate-400 dark:text-slate-600 border border-slate-200 dark:border-slate-800 cursor-not-allowed shadow-none'
              : copied
                ? 'bg-emerald-600 text-white shadow-lg shadow-emerald-500/10'
                : 'bg-indigo-600 hover:bg-indigo-700 hover:scale-[1.01] text-white shadow-lg shadow-indigo-600/20'
          ]"
        >
          <!-- Copy icon -->
          <svg v-if="!copied" class="w-4.5 h-4.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
            <path stroke-linecap="round" stroke-linejoin="round" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3" />
          </svg>
          <!-- Success Check icon -->
          <svg v-else class="w-4.5 h-4.5 animate-bounce" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
            <path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7" />
          </svg>
          {{ copied ? '¡Texto Copiado!' : 'Copiar Texto Completo' }}
        </button>
      </div>
    </div>
  </div>
</template>
