<script setup>
import { ref, computed } from 'vue'
import Tesseract from 'tesseract.js'

const rawText = ref('')
const dataType = ref('vin') // vin, matricula, id_spain, custom
const customRegex = ref('')
const outputFormat = ref('list') // list, sql
const removeDuplicates = ref(true)
const copied = ref(false)

const ocrProgress = ref(0)
const isOcrProcessing = ref(false)
const ocrError = ref('')
const ocrLanguage = ref('spa+eng')

// Function to validate Spanish IDs (NIF, NIE, CIF)
function isValidSpanishID(token) {
  if (typeof token !== 'string') return false
  const clean = token.toUpperCase().trim()
  if (clean.length !== 9) return false

  // 1. DNI/NIF: 8 digits + 1 letter
  if (/^\d{8}[A-Z]$/.test(clean)) {
    const num = parseInt(clean.substring(0, 8), 10)
    const letter = clean[8]
    const validLetters = 'TRWAGMYFPDXBNJZSQVHLCKE'
    return letter === validLetters[num % 23]
  }

  // 2. NIE: 1 starting letter (X, Y, Z) + 7 digits + 1 letter
  if (/^[XYZ]\d{7}[A-Z]$/.test(clean)) {
    const first = clean[0]
    const prefix = first === 'X' ? '0' : first === 'Y' ? '1' : '2'
    const numStr = prefix + clean.substring(1, 8)
    const num = parseInt(numStr, 10)
    const letter = clean[8]
    const validLetters = 'TRWAGMYFPDXBNJZSQVHLCKE'
    return letter === validLetters[num % 23]
  }

  // 3. CIF: 1 organization type letter + 7 digits + 1 control digit/letter
  if (/^[A-Z]\d{7}[A-Z0-9]$/.test(clean)) {
    const firstChar = clean[0]
    const digits = clean.substring(1, 8)
    const lastChar = clean[8]

    let evenSum = 0
    let oddSum = 0

    for (let i = 0; i < 7; i++) {
      const d = parseInt(digits[i], 10)
      if (i % 2 === 1) {
        evenSum += d
      } else {
        let doubled = d * 2
        if (doubled > 9) {
          doubled = Math.floor(doubled / 10) + (doubled % 10)
        }
        oddSum += doubled
      }
    }

    const totalSum = evenSum + oddSum
    const lastDigitOfSum = totalSum % 10
    const controlDigitVal = (10 - lastDigitOfSum) % 10
    const controlDigit = controlDigitVal.toString()
    const controlLetter = 'JABCDEFGHI'[controlDigitVal]

    if (['P', 'Q', 'R', 'S', 'W'].includes(firstChar)) {
      return lastChar === controlLetter
    } else if (['A', 'B', 'E', 'H'].includes(firstChar)) {
      return lastChar === controlDigit
    } else {
      return lastChar === controlDigit || lastChar === controlLetter
    }
  }

  return false
}

// Function to validate and optionally correct Spanish IDs (NIF, NIE, CIF) with OCR error recovery
function validateAndCorrectSpanishID(token) {
  if (typeof token !== 'string') return null
  let clean = token.toUpperCase().trim()
  // Remove any separators
  clean = clean.replace(/[^A-Z0-9]/g, '')
  if (clean.length !== 9) return null

  // Map of common OCR letter-to-digit substitutions
  const letterToDigit = {
    'O': '0', 'I': '1', 'L': '1', 'Z': '2', 'S': '5', 'G': '6', 'T': '7', 'B': '8', 'U': '0', 'V': '0'
  }

  function getDigit(char) {
    if (/\d/.test(char)) return char
    return letterToDigit[char] || char
  }

  // 1. Try validating as-is first
  if (isValidSpanishID(clean)) {
    return clean
  }

  // 2. Try DNI/NIF correction: 8 digits + 1 letter
  let dniCandidate = ''
  for (let i = 0; i < 8; i++) {
    dniCandidate += getDigit(clean[i])
  }
  const dniLastChar = clean[8]
  if (/^\d{8}$/.test(dniCandidate) && /[A-Z]/.test(dniLastChar)) {
    const candidate = dniCandidate + dniLastChar
    if (isValidSpanishID(candidate)) {
      return candidate
    }
  }

  // 3. Try NIE correction: X/Y/Z + 7 digits + 1 letter
  const nieFirstChar = clean[0]
  let nieDigitsCandidate = ''
  for (let i = 1; i < 8; i++) {
    nieDigitsCandidate += getDigit(clean[i])
  }
  const nieLastChar = clean[8]
  if (/^[XYZ]/.test(nieFirstChar) && /^\d{7}$/.test(nieDigitsCandidate) && /[A-Z]/.test(nieLastChar)) {
    const candidate = nieFirstChar + nieDigitsCandidate + nieLastChar
    if (isValidSpanishID(candidate)) {
      return candidate
    }
  }

  // 4. Try CIF correction: 1 organization letter + 7 digits + 1 control digit/letter
  const cifFirstChar = clean[0]
  let cifDigitsCandidate = ''
  for (let i = 1; i < 8; i++) {
    cifDigitsCandidate += getDigit(clean[i])
  }
  const cifLastChar = clean[8]
  if (/^[A-Z]/.test(cifFirstChar) && /^\d{7}$/.test(cifDigitsCandidate)) {
    // Try last char as is
    let candidate = cifFirstChar + cifDigitsCandidate + cifLastChar
    if (isValidSpanishID(candidate)) {
      return candidate
    }
    // Try last char converted to digit
    const lastDigit = getDigit(cifLastChar)
    candidate = cifFirstChar + cifDigitsCandidate + lastDigit
    if (isValidSpanishID(candidate)) {
      return candidate
    }
  }

  return null
}

// Regex definitions
const regexMap = {
  vin: /[A-HJ-NPR-Z0-9]{17}/gi,
  matricula: /\d{4}[\s-]?[BCDFGHJKLMNPRSTVWXYZ]{3}/gi,
  id_spain: /(?:[A-Z0-9][\s\-\.\/|]?){8}[A-Z0-9]/gi
}

const regexErrorMessage = ref('')

const results = computed(() => {
  if (!rawText.value) return []

  let processed = []
  regexErrorMessage.value = ''

  if (dataType.value === 'vin') {
    // Custom smart extraction for VINs
    // 1. Find all blocks of contiguous alphanumeric characters that are valid for a VIN
    // Note: A VIN character can be A-Z (excluding I, O, Q) or 0-9.
    const blocks = rawText.value.match(/[A-HJ-NPR-Z0-9]+/gi) || []
    const extractedVins = []

    for (const block of blocks) {
      if (block.length < 17) continue

      const candidates = []
      for (let i = 0; i <= block.length - 17; i++) {
        const str = block.substring(i, i + 17).toUpperCase()
        if (/[IOQ]/i.test(str)) continue
        if (!/[A-Z1-9]/i.test(str[0])) continue
        if (!/\d{4}$/.test(str)) continue

        const trailingDigitsMatch = str.match(/\d+$/)
        const score = trailingDigitsMatch ? trailingDigitsMatch[0].length : 0

        candidates.push({
          str,
          start: i,
          end: i + 17,
          score,
          isLetterStart: /[A-Z]/i.test(str[0])
        })
      }

      // Sort candidates by start index to process left-to-right
      candidates.sort((a, b) => a.start - b.start)

      const selected = []
      for (const cand of candidates) {
        const overlapIdx = selected.findIndex(sel => cand.start < sel.end && cand.end > sel.start)
        if (overlapIdx !== -1) {
          const existing = selected[overlapIdx]
          let replace = false
          if (cand.score > existing.score) {
            replace = true
          } else if (cand.score === existing.score) {
            if (cand.isLetterStart && !existing.isLetterStart) {
              replace = true
            }
          }
          if (replace) {
            selected[overlapIdx] = cand
          }
        } else {
          selected.push(cand)
        }
      }

      for (const sel of selected) {
        extractedVins.push(sel.str)
      }
    }
    processed = extractedVins
  } else if (dataType.value === 'id_spain') {
    // Custom smart extraction for Spanish IDs (DNI, NIE, CIF)
    // Allows extraction even when merged with other text (e.g. "NIEsválidosX1234567L")
    const blocks = rawText.value.match(/[A-Z0-9\-\.\/|]+/gi) || []
    const extractedIds = []

    for (const block of blocks) {
      const cleanBlock = block.replace(/[^A-Z0-9]/g, '').toUpperCase()
      if (cleanBlock.length < 9) continue

      for (let i = 0; i <= cleanBlock.length - 9; i++) {
        const windowStr = cleanBlock.substring(i, i + 9)
        const corrected = validateAndCorrectSpanishID(windowStr)
        if (corrected) {
          extractedIds.push(corrected)
        }
      }
    }
    processed = extractedIds
  } else {
    let regex
    if (dataType.value === 'custom') {
      if (!customRegex.value) return []
      try {
        regex = new RegExp(customRegex.value, 'gi')
      } catch (e) {
        regexErrorMessage.value = 'Expresión regular no válida.'
        return []
      }
    } else {
      regex = regexMap[dataType.value]
    }

    const matches = rawText.value.match(regex)
    if (!matches) return []

    // Standardize values (e.g. uppercase, trim spaces/hyphens)
    processed = matches.map(match => {
      let val = match.trim().toUpperCase()
      if (dataType.value === 'matricula') {
        val = val.replace(/[\s-]+/g, '')
      }
      return val
    })
  }

  if (removeDuplicates.value) {
    processed = [...new Set(processed)]
  }

  return processed
})

const formattedOutput = computed(() => {
  if (results.value.length === 0) return ''

  if (outputFormat.value === 'sql') {
    const escaped = results.value.map(val => `'${val.replace(/'/g, "''")}'`)
    return `(${escaped.join(', ')})`
  }

  return results.value.join('\n')
})

const copyToClipboard = async () => {
  if (!formattedOutput.value) return

  try {
    await navigator.clipboard.writeText(formattedOutput.value)
    copied.value = true
    setTimeout(() => {
      copied.value = false
    }, 2000)
  } catch (err) {
    console.error('Error al copiar: ', err)
  }
}

function showOcrError(msg) {
  ocrError.value = msg
  setTimeout(() => {
    if (ocrError.value === msg) {
      ocrError.value = ''
    }
  }, 5000)
}

async function processImageFile(file) {
  if (!file) return
  if (!['image/png', 'image/jpeg', 'image/jpg', 'image/webp'].includes(file.type)) {
    showOcrError('No se pudo procesar la imagen o el formato no es compatible')
    return
  }

  isOcrProcessing.value = true
  ocrProgress.value = 0
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
              }
            }
          }
        )
        
        if (!isOcrProcessing.value) return
        if (result && result.data && typeof result.data.text === 'string') {
          if (rawText.value) {
            rawText.value += '\n' + result.data.text
          } else {
            rawText.value = result.data.text
          }
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

const handlePaste = (event) => {
  const items = event.clipboardData?.items
  if (!items) return
  for (const item of items) {
    if (item.type.indexOf('image') !== -1) {
      const file = item.getAsFile()
      if (file) {
        processImageFile(file)
        event.preventDefault()
        break
      }
    }
  }
}
</script>
<template>
  <div class="grid grid-cols-1 lg:grid-cols-12 gap-8 max-w-7xl mx-auto px-4 sm:px-6 py-8">
       <!-- Input Section: Left Side (Col span 7) -->
    <div class="lg:col-span-7 flex flex-col gap-6">
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-6 transition-colors duration-200">
        <div class="flex items-center justify-between">
          <h2 class="text-xl font-bold text-slate-800 dark:text-slate-100 flex items-center gap-2">
            <span class="w-2.5 h-6 bg-indigo-600 rounded-full inline-block"></span>
            Entrada de Texto
          </h2>
          <span class="text-xs text-slate-400 dark:text-slate-550 font-semibold uppercase tracking-wider">Origen de datos</span>
        </div>

        <!-- Raw Text Textarea -->
        <div class="relative flex flex-col">
          <textarea
            v-model="rawText"
            @paste="handlePaste"
            rows="10"
            placeholder="Introduce o pega el texto que contiene los datos que deseas extraer (o pega una captura de pantalla)..."
            class="w-full bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-2xl p-4 pb-12 text-sm font-mono placeholder:text-slate-400 dark:placeholder:text-slate-600 focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out resize-y"
          ></textarea>
          
          <!-- Clear Button -->
          <button
            v-if="rawText"
            @click="rawText = ''"
            class="absolute top-4 right-4 bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-slate-500 dark:text-slate-400 hover:text-slate-800 dark:hover:text-slate-200 px-2.5 py-1.5 rounded-lg text-xs font-bold transition-all duration-200 ease-in-out cursor-pointer active:scale-95 shadow-sm"
          >
            Limpiar
          </button>

          <!-- Subtle Loading / Error State Overlay inside the Textarea -->
          <div 
            v-if="isOcrProcessing || ocrError"
            class="absolute bottom-3 left-4 right-4 flex items-center justify-between text-xs bg-slate-100/90 dark:bg-slate-900/90 backdrop-blur-sm px-3 py-2 rounded-xl border border-slate-200/50 dark:border-slate-800/50 pointer-events-auto shadow-sm"
          >
            <div class="flex items-center gap-2">
              <svg v-if="isOcrProcessing" class="animate-spin h-3.5 w-3.5 text-indigo-600 dark:text-indigo-400" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
              </svg>
              <svg v-else class="h-3.5 w-3.5 text-rose-500" fill="none" stroke="currentColor" stroke-width="2" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
              </svg>

              <span class="font-medium text-slate-700 dark:text-slate-300">
                {{ isOcrProcessing ? `Analizando captura de pantalla... (${ocrProgress}%)` : ocrError }}
              </span>
            </div>

            <button 
              v-if="isOcrProcessing"
              @click="isOcrProcessing = false"
              type="button"
              class="text-red-500 hover:text-red-650 cursor-pointer font-bold transition-colors"
            >
              Cancelar
            </button>
            <button
              v-else
              @click="ocrError = ''"
              type="button"
              class="text-slate-400 hover:text-slate-600 dark:hover:text-slate-300 font-bold transition-colors cursor-pointer"
            >
              ✕
            </button>
          </div>
        </div>

        <!-- Extraction Settings Grid -->
        <div class="grid grid-cols-1 sm:grid-cols-2 gap-5">
          <!-- Data Type Dropdown -->
          <div class="flex flex-col gap-2">
            <label class="text-xs font-bold text-slate-500 dark:text-slate-400 uppercase tracking-wider">Tipo de Dato a Buscar</label>
            <div class="relative">
              <select
                v-model="dataType"
                class="w-full bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-3 text-sm font-bold focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out appearance-none cursor-pointer"
              >
                <option value="vin">Chasis (VIN)</option>
                <option value="matricula">Matrículas (España)</option>
                <option value="id_spain">Documentos (NIF/NIE/CIF)</option>
                <option value="custom">Personalizado (Regex)</option>
              </select>
              <div class="absolute inset-y-0 right-4 flex items-center pointer-events-none text-slate-400 dark:text-slate-500">
                <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M19 9l-7 7-7-7" /></svg>
              </div>
            </div>
          </div>

          <!-- Output Format Dropdown -->
          <div class="flex flex-col gap-2">
            <label class="text-xs font-bold text-slate-500 dark:text-slate-400 uppercase tracking-wider">Formato de Salida</label>
            <div class="relative">
              <select
                v-model="outputFormat"
                class="w-full bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-3 text-sm font-bold focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out appearance-none cursor-pointer"
              >
                <option value="list">Buscador Múltiple (Líneas)</option>
                <option value="sql">Query SQL - Lista IN ('x', 'y')</option>
              </select>
              <div class="absolute inset-y-0 right-4 flex items-center pointer-events-none text-slate-400 dark:text-slate-500">
                <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M19 9l-7 7-7-7" /></svg>
              </div>
            </div>
          </div>
        </div>

        <!-- Custom Regex Input (Conditional) -->
        <div v-if="dataType === 'custom'" class="flex flex-col gap-2 animate-fade-in">
          <label class="text-xs font-bold text-slate-500 dark:text-slate-400 uppercase tracking-wider flex items-center gap-1.5">
            Expresión Regular Personalizada
            <span class="text-[10px] text-slate-400 dark:text-slate-500 lowercase normal-case">(g, i añadidas automáticamente)</span>
          </label>
          <input
            v-model="customRegex"
            type="text"
            placeholder="Ejemplo: [A-Z]{3}-\d{4}"
            class="w-full bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-3 text-sm font-mono placeholder:text-slate-400 dark:placeholder:text-slate-600 focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out"
          />
          <p v-if="regexErrorMessage" class="text-xs text-red-500 font-semibold mt-1">{{ regexErrorMessage }}</p>
        </div>

        <!-- Extra Controls -->
        <div class="flex items-center gap-3 pt-4 border-t border-slate-100 dark:border-slate-800">
          <label class="inline-flex items-center gap-3 cursor-pointer group select-none">
            <input
              type="checkbox"
              v-model="removeDuplicates"
              class="sr-only peer"
            />
            <div class="w-9 h-5 bg-slate-200 dark:bg-slate-800 border border-slate-300 dark:border-slate-700 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-slate-300 dark:after:border-slate-600 after:border after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:bg-emerald-500 relative transition-colors duration-300"></div>
            <span class="text-xs font-bold text-slate-600 dark:text-slate-400 group-hover:text-slate-800 dark:group-hover:text-slate-200 transition-colors">Eliminar duplicados automáticamente</span>
          </label>
        </div>
      </div>
    </div>

    <!-- Output Section: Right Side (Col span 5) -->
    <div class="lg:col-span-5 flex flex-col gap-6">
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-6 h-full transition-colors duration-200">
        <div class="flex items-center justify-between">
          <h2 class="text-xl font-bold text-slate-800 dark:text-slate-100 flex items-center gap-2">
            <span class="w-2.5 h-6 bg-emerald-500 rounded-full inline-block"></span>
            Resultados
          </h2>
          <!-- Matches Counter Badge -->
          <div class="flex items-center gap-1.5 bg-slate-5-0 dark:bg-slate-800/60 border border-slate-200 dark:border-slate-700 px-3 py-1.5 rounded-full select-none">
            <span class="text-[10px] text-slate-500 dark:text-slate-400 font-bold uppercase tracking-wider">Total:</span>
            <span class="text-xs font-extrabold text-emerald-600 dark:text-emerald-400 font-mono">{{ results.length }}</span>
          </div>
        </div>

        <!-- Result Content -->
        <div class="flex-grow flex flex-col min-h-[280px] relative">
          <!-- IDE code view (dark theme for output) -->
          <textarea
            readonly
            :value="formattedOutput"
            placeholder="Aquí se mostrarán los datos extraídos listos para su uso..."
            class="w-full flex-grow bg-slate-950 text-emerald-400 border border-slate-950 rounded-2xl p-4 text-sm font-mono placeholder:text-slate-600 focus:outline-none focus:ring-0 resize-none h-full selection:bg-indigo-500/30 selection:text-white"
          ></textarea>

          <!-- Empty State Overlay (Visible when no results) -->
          <div
            v-if="results.length === 0"
            class="absolute inset-0 flex flex-col items-center justify-center gap-3.5 p-6 pointer-events-none text-center"
          >
            <div class="w-12 h-12 rounded-xl bg-slate-800 dark:bg-slate-900 border border-slate-950 flex items-center justify-center text-slate-400">
              <svg class="w-6 h-6" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" /></svg>
            </div>
            <div>
              <p class="text-sm font-bold text-slate-200">Sin datos extraídos</p>
              <p class="text-xs text-slate-500 dark:text-slate-550 mt-1 max-w-[200px] mx-auto">Introduce texto a la izquierda para iniciar la extracción en tiempo real.</p>
            </div>
          </div>
        </div>

        <!-- Copy Action Button with active click down and hover states -->
        <button
          @click="copyToClipboard"
          :disabled="results.length === 0"
          :class="[
            'w-full py-3.5 rounded-xl font-bold text-sm tracking-wide flex items-center justify-center gap-2 cursor-pointer shadow-md transition-all duration-200 ease-in-out active:scale-[0.98]',
            results.length === 0
              ? 'bg-slate-100 dark:bg-slate-800/40 text-slate-400 dark:text-slate-600 border border-slate-200 dark:border-slate-800 cursor-not-allowed shadow-none'
              : copied
                ? 'bg-emerald-600 text-white shadow-lg shadow-emerald-500/10'
                : 'bg-indigo-600 hover:bg-indigo-700 hover:scale-[1.01] text-white shadow-lg shadow-indigo-600/20'
          ]"
        >
          <!-- Copy icon -->
          <svg v-if="!copied" class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
            <path stroke-linecap="round" stroke-linejoin="round" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3" />
          </svg>
          <!-- Success Check icon -->
          <svg v-else class="w-4 h-4 animate-bounce" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
            <path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7" />
          </svg>
          {{ copied ? '¡Copiado al Portapapeles!' : 'Copiar Resultados' }}
        </button>
      </div>
    </div>

  </div>
</template>
