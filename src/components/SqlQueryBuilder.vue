<script setup>
import { ref, computed } from 'vue'

// Reactive state updated to support SCORE and CONCAT modes
const tableName = ref('ge_versio') // Default to 'ge_versio'
const operation = ref('SELECT') // SELECT, INSERT, UPDATE, DELETE, SCORE, CONCAT

const selectFields = ref(['id', 'nombre', 'email'])
const insertUpdatePairs = ref([
  { column: 'nombre', value: 'Carlos' },
  { column: 'email', value: 'carlos@example.com' },
  { column: 'activo', value: '1' }
])
const whereConditions = ref([
  { column: 'id', operator: '=', value: '12', logicalOperator: 'AND' }
])

// SCORE specific state
const scoreBaseFields = ref('ge_versio.*')
const scoreThresholdOperator = ref('>=')
const scoreThresholdValue = ref(0)
const scoreOrderDirection = ref('DESC')
const scoreRules = ref([
  { column: 'ge_versio_descri', operator: 'LIKE', value: '%MODELO%', weight: 4 },
  { column: 'ge_versio_estado', operator: '=', value: 'A', weight: 2 }
])

// CONCAT specific state
const concatAction = ref('UPDATE')
const concatSheetId = ref('DATOS1')
const concatSourceTable = ref('table')
const concatTargetTable = ref('SarixVerticales.table_name')
const concatTargetColumn = ref('column_name')
const concatTargetValue = ref("'value'")
const concatFilteredIds = ref('1, 2')

const copied = ref(false)

// Scoped row addition / removal methods
const addRow = (type) => {
  if (type === 'select') {
    selectFields.value.push('')
  } else if (type === 'pair') {
    insertUpdatePairs.value.push({ column: '', value: '' })
  } else if (type === 'where') {
    whereConditions.value.push({ column: '', operator: '=', value: '', logicalOperator: 'AND' })
  } else if (type === 'scoreRule') {
    scoreRules.value.push({ column: '', operator: 'LIKE', value: '', weight: 1 })
  }
}

const removeRow = (type, index) => {
  if (type === 'select') {
    selectFields.value.splice(index, 1)
  } else if (type === 'pair') {
    insertUpdatePairs.value.splice(index, 1)
  } else if (type === 'where') {
    whereConditions.value.splice(index, 1)
  } else if (type === 'scoreRule') {
    scoreRules.value.splice(index, 1)
  }
}

// Utility to format values: wraps strings in single quotes, keeps numbers/NULLs unwrapped
const formatValue = (val) => {
  if (val === undefined || val === null) return 'NULL'
  const trimmed = typeof val === 'string' ? val.trim() : String(val)
  if (trimmed === '') return "''"

  // Number validation (integers, decimals)
  if (/^-?\d+(\.\d+)?$/.test(trimmed)) {
    return trimmed
  }

  // NULL keyword check
  if (trimmed.toUpperCase() === 'NULL') {
    return 'NULL'
  }

  // Wrap strings and escape internal single quotes
  return `'${trimmed.replace(/'/g, "''")}'`
}

// Custom formatter for WHERE conditions to handle 'IN' operations nicely
const formatWhereValue = (val, operator) => {
  const trimmed = val.trim()
  if (operator.toUpperCase() === 'IN') {
    let cleanVal = trimmed
    // Strip external parentheses if the user typed them
    if (cleanVal.startsWith('(') && cleanVal.endsWith(')')) {
      cleanVal = cleanVal.slice(1, -1).trim()
    }

    if (cleanVal === '') return '()'

    const items = cleanVal.split(',').map(item => {
      const itemTrimmed = item.trim()
      if (/^-?\d+(\.\d+)?$/.test(itemTrimmed)) {
        return itemTrimmed
      }
      if (itemTrimmed.toUpperCase() === 'NULL') {
        return 'NULL'
      }
      // Strip outer quotes if manually written
      let stripped = itemTrimmed
      if ((stripped.startsWith("'") && stripped.endsWith("'")) || (stripped.startsWith('"') && stripped.endsWith('"'))) {
        stripped = stripped.slice(1, -1)
      }
      return `'${stripped.replace(/'/g, "''")}'`
    })

    return `(${items.join(', ')})`
  }

  return formatValue(val)
}

// Reusable WHERE clause builder that supports selective logical operators
const buildWhereClause = (activeWhere) => {
  if (activeWhere.length === 0) return ''
  let whereStr = ''
  activeWhere.forEach((w, idx) => {
    const formattedVal = formatWhereValue(w.value, w.operator)
    const condStr = `${w.column.trim()} ${w.operator} ${formattedVal}`
    if (idx === 0) {
      whereStr = condStr
    } else {
      const op = w.logicalOperator || 'AND'
      whereStr += `\n  ${op} ${condStr}`
    }
  })
  return `\nWHERE ${whereStr}`
}

// Real-time SQL Query generator
const generatedQuery = computed(() => {
  const table = tableName.value.trim() || 'mi_tabla'
  const activeWhere = whereConditions.value.filter(w => w.column.trim() !== '')

  switch (operation.value) {
    case 'SELECT': {
      const cols = selectFields.value.filter(f => f.trim() !== '').join(', ') || '*'
      let query = `SELECT ${cols}\nFROM ${table}`
      query += buildWhereClause(activeWhere)
      return query + ';'
    }

    case 'INSERT': {
      const activePairs = insertUpdatePairs.value.filter(p => p.column.trim() !== '')

      if (activePairs.length === 0) {
        return `INSERT INTO ${table} () VALUES ();`
      }

      const cols = activePairs.map(p => p.column.trim()).join(', ')
      const vals = activePairs.map(p => formatValue(p.value)).join(', ')

      return `INSERT INTO ${table} (${cols})\nVALUES (${vals});`
    }

    case 'UPDATE': {
      const activePairs = insertUpdatePairs.value.filter(p => p.column.trim() !== '')

      let query = `UPDATE ${table}`

      if (activePairs.length === 0) {
        query += `\nSET /* define columnas a actualizar */`
      } else {
        const setStr = activePairs
          .map(p => `${p.column.trim()} = ${formatValue(p.value)}`)
          .join(',\n    ')
        query += `\nSET ${setStr}`
      }

      const wherePart = buildWhereClause(activeWhere)
      query += wherePart || '\nWHERE /* id = 1 */'

      return query + ';'
    }

    case 'DELETE': {
      let query = `DELETE FROM ${table}`
      const wherePart = buildWhereClause(activeWhere)
      query += wherePart || '\nWHERE /* id = 1 */'

      return query + ';'
    }

    case 'SCORE': {
      const baseFields = scoreBaseFields.value.trim() || 'ge_versio.*'
      const activeRules = scoreRules.value.filter(r => r.column.trim() !== '')
      const threshOp = scoreThresholdOperator.value
      const threshVal = scoreThresholdValue.value !== '' ? scoreThresholdValue.value : '0'
      const orderDir = scoreOrderDirection.value

      let casesStr = ''
      if (activeRules.length === 0) {
        casesStr = '        0'
      } else {
        casesStr = activeRules.map((r, i) => {
          const formattedVal = formatValue(r.value)
          const plus = i < activeRules.length - 1 ? ' +' : ''
          return `        (CASE WHEN ${r.column.trim()} ${r.operator} ${formattedVal} THEN ${r.weight} ELSE 0 END)${plus}`
        }).join('\n')
      }

      return `SELECT \n    ${baseFields},\n    (\n${casesStr}\n    ) AS score\nFROM ${table}\nHAVING score ${threshOp} ${threshVal}\nORDER BY score ${orderDir};`
    }

    case 'CONCAT': {
      const action = concatAction.value
      const sheetId = concatSheetId.value.trim() || 'DATOS1'
      const sourceTable = concatSourceTable.value.trim() || 'table'
      const targetTable = concatTargetTable.value.trim() || 'SarixVerticales.table_name'
      const targetColumn = concatTargetColumn.value.trim() || 'column_name'
      const targetValue = concatTargetValue.value.trim() || "'value'"
      const filteredIds = concatFilteredIds.value.trim() || '1, 2'

      let concatInside = ''
      if (action === 'DELETE') {
        concatInside = `CONCAT("DELETE FROM ${targetTable} \\nWHERE id = ", tb.id)`
      } else if (action === 'INSERT') {
        concatInside = `CONCAT("INSERT INTO ${targetTable} (${targetColumn}) VALUES (${targetValue}) \\nWHERE id = ", tb.id)`
      } else { // UPDATE
        concatInside = `CONCAT("UPDATE ${targetTable} \\nSET ${targetColumn} = ${targetValue} \\nWHERE id = ", tb.id)`
      }

      return `SELECT\n    "${action}",\n    tb.id,\n    ${concatInside},\n    "${sheetId}"\nFROM ${sourceTable} tb\nWHERE tb.id IN (${filteredIds});`
    }

    default:
      return ''
  }
})

const copyQuery = async () => {
  if (!generatedQuery.value) return

  try {
    await navigator.clipboard.writeText(generatedQuery.value)
    copied.value = true
    setTimeout(() => {
      copied.value = false
    }, 2000)
  } catch (err) {
    console.error('Error al copiar query: ', err)
  }
}
</script>

<template>
  <div class="grid grid-cols-1 lg:grid-cols-12 gap-8 max-w-7xl mx-auto px-4 sm:px-6 py-8">
    
    <!-- Left Configuration Panel: Col span 7 -->
    <div class="lg:col-span-7 flex flex-col gap-6">
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-6 transition-colors duration-200">
        
        <!-- Header & Grouped Selector -->
        <div class="flex flex-col sm:flex-row sm:items-center justify-between gap-4 border-b border-slate-100 pb-4">
          <h2 class="text-xl font-bold text-slate-800 dark:text-slate-100 flex items-center gap-2">
            <span class="w-2.5 h-6 bg-indigo-600 rounded-full inline-block"></span>
            Estructurador Visual
          </h2>
          
          <!-- Modo de Consulta Select dropdown grouped in two categories -->
          <div class="flex flex-col gap-1 w-full sm:w-60">
            <label class="text-[9px] text-slate-500 dark:text-slate-400 font-bold uppercase tracking-wider font-mono select-none">Modo de Consulta</label>
            <div class="relative">
              <select
                v-model="operation"
                class="w-full bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-2.5 text-xs font-mono font-bold uppercase tracking-wider focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out appearance-none cursor-pointer"
              >
                <optgroup label="Sentencias Clásicas" class="bg-white dark:bg-slate-900 text-indigo-700 dark:text-indigo-400 font-bold font-mono">
                  <option value="SELECT">SELECT (Consultar)</option>
                  <option value="INSERT">INSERT (Insertar)</option>
                  <option value="UPDATE">UPDATE (Actualizar)</option>
                  <option value="DELETE">DELETE (Eliminar)</option>
                </optgroup>
                <optgroup label="Plantillas Especiales" class="bg-white dark:bg-slate-900 text-emerald-600 dark:text-emerald-450 font-bold font-mono">
                  <option value="SCORE">SCORE (Scoring Query)</option>
                  <option value="CONCAT">CONCAT (Excel Log Generator)</option>
                </optgroup>
              </select>
              <div class="absolute inset-y-0 right-4 flex items-center pointer-events-none text-slate-400">
                <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M19 9l-7 7-7-7" /></svg>
              </div>
            </div>
          </div>
        </div>

        <!-- DYNAMIC LAYOUTS BASED ON OPERATION -->
        <!-- 1. SELECT LAYOUT -->
        <div v-if="operation === 'SELECT'" class="flex flex-col gap-6 animate-fade-in">
          
          <!-- SELECT Row -->
          <div class="flex flex-col gap-2.5">
            <span class="text-indigo-650 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">SELECT</span>
            
            <div class="flex flex-wrap items-center gap-2 bg-slate-50 dark:bg-slate-800/40 p-4 rounded-2xl border border-slate-200 dark:border-slate-800">
              <div v-for="(field, idx) in selectFields" :key="idx" class="flex items-center gap-1 bg-white dark:bg-slate-900 p-1.5 rounded-lg border border-slate-200 dark:border-slate-800 shadow-sm focus-within:ring-2 focus-within:ring-indigo-500/20 focus-within:border-indigo-500 transition-all duration-200 ease-in-out">
                <input
                  v-model="selectFields[idx]"
                  type="text"
                  placeholder="campo"
                  class="bg-transparent text-slate-800 dark:text-slate-200 border-none font-mono text-sm focus:outline-none w-20 sm:w-24 placeholder:text-slate-400 dark:placeholder:text-slate-600 font-medium"
                />
                <button
                  v-if="selectFields.length > 1"
                  @click="removeRow('select', idx)"
                  class="text-slate-400 dark:text-slate-550 hover:text-red-500 dark:hover:text-red-400 p-0.5 rounded transition-all duration-200 ease-in-out cursor-pointer active:scale-90"
                >
                  <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5"><path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" /></svg>
                </button>
              </div>

              <!-- Add column button -->
              <button
                @click="addRow('select')"
                class="bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-indigo-600 dark:text-indigo-400 p-2 rounded-lg text-xs font-bold flex items-center justify-center cursor-pointer transition-all duration-200 ease-in-out hover:scale-105 active:scale-95 shadow-sm"
                title="Añadir columna de selección"
              >
                <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5"><path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4" /></svg>
              </button>
            </div>
          </div>

          <!-- FROM Row -->
          <div class="flex items-center gap-3">
            <span class="text-indigo-650 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none w-14">FROM</span>
            <input
              v-model="tableName"
              type="text"
              placeholder="nombre_tabla"
              class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out flex-grow sm:flex-grow-0 sm:w-72 font-semibold"
            />
          </div>

          <!-- WHERE Row -->
          <div class="flex flex-col gap-2.5">
            <span class="text-indigo-650 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">WHERE</span>
            
            <div class="flex flex-col gap-3">
              <div v-for="(cond, idx) in whereConditions" :key="idx" class="flex flex-col gap-3">
                
                <!-- Selectable Logical Operator (AND / OR) between conditions -->
                <div v-if="idx > 0" class="self-center flex items-center">
                  <div class="relative flex items-center">
                    <select
                      v-model="cond.logicalOperator"
                      class="bg-indigo-50 dark:bg-indigo-950/60 hover:bg-indigo-100 dark:hover:bg-indigo-900/60 text-indigo-700 dark:text-indigo-400 border border-indigo-200 dark:border-indigo-800 rounded-xl px-5 py-1 text-xs font-mono font-bold tracking-widest cursor-pointer focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500 appearance-none text-center pr-8 select-none transition-all duration-200 ease-in-out"
                    >
                      <option value="AND">AND</option>
                      <option value="OR">OR</option>
                    </select>
                    <div class="absolute inset-y-0 right-2.5 flex items-center pointer-events-none text-indigo-600">
                      <svg class="w-3 h-3" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M19 9l-7 7-7-7" /></svg>
                    </div>
                  </div>
                </div>
                
                <div class="flex flex-wrap sm:flex-nowrap items-center gap-2 bg-slate-50 dark:bg-slate-800/40 p-3 rounded-2xl border border-slate-200 dark:border-slate-700">
                  <input
                    v-model="cond.column"
                    type="text"
                    placeholder="columna"
                    class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out"
                  />
                  
                  <select
                    v-model="cond.operator"
                    class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none cursor-pointer w-20 appearance-none text-center transition-all duration-200 ease-in-out"
                  >
                    <option value="=">=</option>
                    <option value="!=">!=</option>
                    <option value="LIKE">LIKE</option>
                    <option value="IN">IN</option>
                  </select>
                  
                  <input
                    v-model="cond.value"
                    type="text"
                    placeholder="valor (ej. Carlos, 12, null)"
                    class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out"
                  />
                  
                  <button
                    @click="removeRow('where', idx)"
                    class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-950/30 border border-slate-200 dark:border-slate-700 hover:border-red-200 dark:hover:border-red-800 text-slate-400 dark:text-slate-500 hover:text-red-550 dark:hover:text-red-400 p-2 rounded-lg transition-all duration-200 ease-in-out cursor-pointer hover:scale-105 active:scale-95"
                    title="Eliminar condición"
                  >
                    <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                  </button>
                </div>
              </div>

              <!-- Add condition button -->
              <button
                @click="addRow('where')"
                class="bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-indigo-600 dark:text-indigo-400 px-3 py-2 rounded-xl text-xs font-bold self-start cursor-pointer flex items-center gap-1.5 transition-all duration-200 ease-in-out hover:scale-105 active:scale-95 shadow-sm"
              >
                <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5"><path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4"/></svg>
                Añadir Condición
              </button>
            </div>
          </div>
        </div>

        <!-- 2. INSERT LAYOUT -->
        <div v-else-if="operation === 'INSERT'" class="flex flex-col gap-6 animate-fade-in">
          
          <!-- INSERT INTO [ __table_name__ ] -->
          <div class="flex items-center gap-3">
            <span class="text-indigo-650 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">INSERT INTO</span>
            <input
              v-model="tableName"
              type="text"
              placeholder="nombre_tabla"
              class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out flex-grow sm:flex-grow-0 sm:w-72 font-semibold"
            />
          </div>

          <!-- Dynamic List of Columns and Values -->
          <div class="flex flex-col gap-3">
            <span class="text-xs font-bold text-slate-500 uppercase tracking-wider">Pares de Inserción (Columna / Valor)</span>
            
            <div class="flex flex-col gap-3">
              <div
                v-for="(pair, idx) in insertUpdatePairs"
                :key="idx"
                class="flex items-center gap-3 bg-slate-50 dark:bg-slate-800/40 p-3 rounded-2xl border border-slate-200 dark:border-slate-700"
              >
                <!-- Paired input elements -->
                <div class="grid grid-cols-1 sm:grid-cols-2 gap-3 flex-1">
                  <div class="flex flex-col gap-1">
                    <label class="text-[10px] text-slate-500 font-bold uppercase tracking-wider font-mono">Columna</label>
                    <input
                      v-model="pair.column"
                      type="text"
                      placeholder="nombre_columna"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-1.5 text-sm font-mono focus:outline-none transition-all duration-200 ease-in-out"
                    />
                  </div>
                  <div class="flex flex-col gap-1">
                    <label class="text-[10px] text-slate-500 font-bold uppercase tracking-wider font-mono">Valor asignado</label>
                    <input
                      v-model="pair.value"
                      type="text"
                      placeholder="Valor del inserto"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-1.5 text-sm font-mono focus:outline-none transition-all duration-200 ease-in-out"
                    />
                  </div>
                </div>
                
                <!-- Remove Pair Button -->
                <button
                  v-if="insertUpdatePairs.length > 1"
                  @click="removeRow('pair', idx)"
                  class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-950/30 border border-slate-200 dark:border-slate-700 hover:border-red-200 dark:hover:border-red-800 text-slate-400 dark:text-slate-500 hover:text-red-550 dark:hover:text-red-400 p-2 rounded-lg transition-all duration-200 ease-in-out cursor-pointer self-end mb-[2px] hover:scale-105 active:scale-95"
                  title="Eliminar fila"
                >
                  <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                </button>
              </div>

              <!-- Add Pair Button -->
              <button
                @click="addRow('pair')"
                class="bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-indigo-600 dark:text-indigo-400 px-3 py-2 rounded-xl text-xs font-bold self-start cursor-pointer flex items-center gap-1.5 transition-all duration-200 ease-in-out hover:scale-105 active:scale-95 shadow-sm"
              >
                <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5"><path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4"/></svg>
                Añadir Fila de Inserción
              </button>
            </div>
          </div>

          <!-- UI preview visually grouping Columns and Values inside Parentheses -->
          <div class="mt-4 p-4 bg-slate-900 rounded-2xl border border-slate-950 font-mono text-xs sm:text-sm flex flex-col gap-2 shadow-lg text-slate-100">
            <div class="text-[9px] text-slate-500 uppercase tracking-wider font-bold border-b border-slate-800 pb-1.5 mb-1 select-none">Estructuración visual de los datos</div>
            <div class="flex items-center gap-2">
              <span class="text-indigo-400 font-extrabold select-none">INSERT INTO</span>
              <span class="text-slate-350 font-semibold">{{ tableName || 'mi_tabla' }}</span>
            </div>
            <!-- Columns (grouped in parentheses) -->
            <div class="flex items-start gap-1 pl-4">
              <span class="text-indigo-450 font-bold select-none">(</span>
              <div class="flex flex-wrap items-center gap-1 text-emerald-400">
                <template v-for="(p, i) in insertUpdatePairs.filter(p => p.column.trim())" :key="'col-' + i">
                  <span class="font-semibold">{{ p.column }}</span>
                  <span v-if="i < insertUpdatePairs.filter(p => p.column.trim()).length - 1" class="text-slate-650 select-none">,</span>
                </template>
                <span v-if="insertUpdatePairs.filter(p => p.column.trim()).length === 0" class="text-slate-700 italic select-none">columnas...</span>
              </div>
              <span class="text-indigo-450 font-bold select-none">)</span>
            </div>
            <!-- VALUES label -->
            <div class="text-indigo-400 font-extrabold pl-2 select-none">VALUES</div>
            <!-- Values (grouped in separate parentheses) -->
            <div class="flex items-start gap-1 pl-4">
              <span class="text-indigo-450 font-bold select-none">(</span>
              <div class="flex flex-wrap items-center gap-1 text-amber-300">
                <template v-for="(p, i) in insertUpdatePairs.filter(p => p.column.trim())" :key="'val-' + i">
                  <span class="font-semibold">{{ formatValue(p.value) }}</span>
                  <span v-if="i < insertUpdatePairs.filter(p => p.column.trim()).length - 1" class="text-slate-650 select-none">,</span>
                </template>
                <span v-if="insertUpdatePairs.filter(p => p.column.trim()).length === 0" class="text-slate-700 italic select-none">valores...</span>
              </div>
              <span class="text-indigo-450 font-bold select-none">)</span>
            </div>
          </div>
        </div>
              <!-- 3. UPDATE LAYOUT -->
        <div v-else-if="operation === 'UPDATE'" class="flex flex-col gap-6 animate-fade-in">
          
          <!-- UPDATE [ __table_name__ ] -->
          <div class="flex items-center gap-3">
            <span class="text-indigo-650 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">UPDATE</span>
            <input
              v-model="tableName"
              type="text"
              placeholder="nombre_tabla"
              class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out flex-grow sm:flex-grow-0 sm:w-72 font-semibold"
            />
          </div>

          <!-- SET [ __set_pairs__ ] -->
          <div class="flex flex-col gap-2.5">
            <span class="text-indigo-650 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">SET</span>
            
            <div class="flex flex-col gap-3">
              <div
                v-for="(pair, idx) in insertUpdatePairs"
                :key="idx"
                class="flex items-center gap-2 bg-slate-50 dark:bg-slate-800/40 p-3 rounded-2xl border border-slate-200 dark:border-slate-800"
              >
                <input
                  v-model="pair.column"
                  type="text"
                  placeholder="campo"
                  class="bg-white text-slate-800 border border-slate-200 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-1.5 text-sm font-mono focus:outline-none flex-1 transition-all duration-200 ease-in-out"
                />
                <span class="text-slate-400 font-mono font-extrabold text-sm select-none">=</span>
                <input
                  v-model="pair.value"
                  type="text"
                  placeholder="valor"
                  class="bg-white text-slate-800 border border-slate-200 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-1.5 text-sm font-mono focus:outline-none flex-1 transition-all duration-200 ease-in-out"
                />
                
                <button
                  v-if="insertUpdatePairs.length > 1"
                  @click="removeRow('pair', idx)"
                  class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-950/30 border border-slate-200 dark:border-slate-700 hover:border-red-200 dark:hover:border-red-800 text-slate-400 dark:text-slate-500 hover:text-red-550 dark:hover:text-red-400 p-2 rounded-lg transition-all duration-200 ease-in-out cursor-pointer hover:scale-105 active:scale-95"
                  title="Eliminar fila"
                >
                  <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                </button>
              </div>

              <!-- Add Pair Button -->
              <button
                @click="addRow('pair')"
                class="bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-indigo-600 dark:text-indigo-400 px-3 py-2 rounded-xl text-xs font-bold self-start cursor-pointer flex items-center gap-1.5 transition-all duration-200 ease-in-out hover:scale-105 active:scale-95 shadow-sm"
              >
                <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5"><path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4"/></svg>
                Añadir Par de Actualización
              </button>
            </div>
          </div>

          <!-- WHERE [ __where_conditions__ ] -->
          <div class="flex flex-col gap-2.5">
            <span class="text-indigo-650 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">WHERE</span>
            
            <div class="flex flex-col gap-3">
              <div v-for="(cond, idx) in whereConditions" :key="idx" class="flex flex-col gap-3">
                
                <!-- Selectable Logical Operator (AND / OR) between conditions -->
                <div v-if="idx > 0" class="self-center flex items-center">
                  <div class="relative flex items-center">
                    <select
                      v-model="cond.logicalOperator"
                      class="bg-indigo-50 hover:bg-indigo-100 text-indigo-700 border border-indigo-200 rounded-xl px-5 py-1 text-xs font-mono font-bold tracking-widest cursor-pointer focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-550 appearance-none text-center pr-8 select-none transition-all duration-200 ease-in-out"
                    >
                      <option value="AND">AND</option>
                      <option value="OR">OR</option>
                    </select>
                    <div class="absolute inset-y-0 right-2.5 flex items-center pointer-events-none text-indigo-600">
                      <svg class="w-3 h-3" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M19 9l-7 7-7-7" /></svg>
                    </div>
                  </div>
                </div>
                
                <div class="flex flex-wrap sm:flex-nowrap items-center gap-2 bg-slate-50 dark:bg-slate-800/40 p-3 rounded-2xl border border-slate-200 dark:border-slate-700">
                  <input
                    v-model="cond.column"
                    type="text"
                    placeholder="columna"
                    class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out"
                  />
                  
                  <select
                    v-model="cond.operator"
                    class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none cursor-pointer w-20 appearance-none text-center transition-all duration-200 ease-in-out"
                  >
                    <option value="=">=</option>
                    <option value="!=">!=</option>
                    <option value="LIKE">LIKE</option>
                    <option value="IN">IN</option>
                  </select>
                  
                  <input
                    v-model="cond.value"
                    type="text"
                    placeholder="valor"
                    class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out"
                  />
                  
                  <button
                    @click="removeRow('where', idx)"
                    class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-950/30 border border-slate-200 dark:border-slate-700 hover:border-red-200 dark:hover:border-red-800 text-slate-400 dark:text-slate-500 hover:text-red-550 dark:hover:text-red-400 p-2 rounded-lg transition-all duration-200 ease-in-out cursor-pointer hover:scale-105 active:scale-95"
                    title="Eliminar condición"
                  >
                    <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                  </button>
                </div>
              </div>

              <!-- Add condition button -->
              <button
                @click="addRow('where')"
                class="bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-indigo-600 dark:text-indigo-400 px-3 py-2 rounded-xl text-xs font-bold self-start cursor-pointer flex items-center gap-1.5 transition-all duration-200 ease-in-out hover:scale-105 active:scale-95 shadow-sm"
              >
                <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5"><path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4"/></svg>
                Añadir Condición
              </button>
            </div>
          </div>
        </div>

        <!-- 4. DELETE LAYOUT -->
        <div v-else-if="operation === 'DELETE'" class="flex flex-col gap-6 animate-fade-in">
          
          <!-- DELETE FROM [ __table_name__ ] -->
          <div class="flex items-center gap-3">
            <span class="text-indigo-650 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">DELETE FROM</span>
            <input
              v-model="tableName"
              type="text"
              placeholder="nombre_tabla"
              class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out flex-grow sm:flex-grow-0 sm:w-72 font-semibold"
            />
          </div>

          <!-- WHERE [ __where_conditions__ ] -->
          <div class="flex flex-col gap-2.5">
            <span class="text-indigo-650 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">WHERE</span>
            
            <div class="flex flex-col gap-3">
              <div v-for="(cond, idx) in whereConditions" :key="idx" class="flex flex-col gap-3">
                
                <!-- Selectable Logical Operator (AND / OR) between conditions -->
                <div v-if="idx > 0" class="self-center flex items-center">
                  <div class="relative flex items-center">
                    <select
                      v-model="cond.logicalOperator"
                      class="bg-indigo-50 hover:bg-indigo-100 text-indigo-750 border border-indigo-200 rounded-xl px-5 py-1 text-xs font-mono font-bold tracking-widest cursor-pointer focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500 appearance-none text-center pr-8 select-none transition-all duration-200 ease-in-out"
                    >
                      <option value="AND">AND</option>
                      <option value="OR">OR</option>
                    </select>
                    <div class="absolute inset-y-0 right-2.5 flex items-center pointer-events-none text-indigo-600">
                      <svg class="w-3 h-3" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M19 9l-7 7-7-7" /></svg>
                    </div>
                  </div>
                </div>
                
                <div class="flex flex-wrap sm:flex-nowrap items-center gap-2 bg-slate-50 dark:bg-slate-800/40 p-3 rounded-2xl border border-slate-200 dark:border-slate-700">
                  <input
                    v-model="cond.column"
                    type="text"
                    placeholder="columna"
                    class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out"
                  />
                  
                  <select
                    v-model="cond.operator"
                    class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none cursor-pointer w-20 appearance-none text-center transition-all duration-200 ease-in-out"
                  >
                    <option value="=">=</option>
                    <option value="!=">!=</option>
                    <option value="LIKE">LIKE</option>
                    <option value="IN">IN</option>
                  </select>
                  
                  <input
                    v-model="cond.value"
                    type="text"
                    placeholder="valor"
                    class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out"
                  />
                  
                  <button
                    @click="removeRow('where', idx)"
                    class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-950/30 border border-slate-200 dark:border-slate-700 hover:border-red-200 dark:hover:border-red-800 text-slate-400 dark:text-slate-500 hover:text-red-550 dark:hover:text-red-400 p-2 rounded-lg transition-all duration-200 ease-in-out cursor-pointer hover:scale-105 active:scale-95"
                    title="Eliminar condición"
                  >
                    <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                  </button>
                </div>
              </div>

              <!-- Add condition button -->
              <button
                @click="addRow('where')"
                class="bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-indigo-600 dark:text-indigo-400 px-3 py-2 rounded-xl text-xs font-bold self-start cursor-pointer flex items-center gap-1.5 transition-all duration-200 ease-in-out hover:scale-105 active:scale-95 shadow-sm"
              >
                <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5"><path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4"/></svg>
                Añadir Condición
              </button>
            </div>
          </div>
        </div>

        <!-- 5. SCORE LAYOUT (CUSTOM TEMPLATE) -->
        <div v-else-if="operation === 'SCORE'" class="flex flex-col gap-6 animate-fade-in bg-amber-50/80 dark:bg-amber-950/15 border border-amber-200 dark:border-amber-900/40 p-6 rounded-3xl shadow-inner text-amber-900 dark:text-amber-200">
          
          <!-- Global Configuration Parameters inside visual box -->
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 bg-white/80 dark:bg-slate-900/80 p-5 rounded-2xl border border-amber-200/50 dark:border-amber-900/30 shadow-sm">
            <!-- Table Name input -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-amber-800 font-extrabold uppercase tracking-wider font-mono select-none">TABLA...</label>
              <input
                v-model="tableName"
                type="text"
                placeholder="ge_versio"
                class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-amber-200 dark:border-amber-800/60 focus:border-amber-500 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-amber-500/20 transition-all duration-200 ease-in-out"
              />
            </div>
            
            <!-- Campos Base to retrieve -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-amber-800 font-extrabold uppercase tracking-wider font-mono select-none">CAMPOS BASE...</label>
              <input
                v-model="scoreBaseFields"
                type="text"
                placeholder="ge_versio.*"
                class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-amber-200 dark:border-amber-800/60 focus:border-amber-500 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-amber-500/20 transition-all duration-200 ease-in-out"
              />
            </div>
            
            <!-- Threshold (HAVING Operator and Value) -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-amber-800 font-extrabold uppercase tracking-wider font-mono select-none">FILTRAR SCORE (HAVING)...</label>
              <div class="flex gap-2">
                <select
                  v-model="scoreThresholdOperator"
                  class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-amber-200 dark:border-amber-800/60 focus:border-amber-500 rounded-xl px-3 py-2 text-sm font-mono focus:outline-none cursor-pointer w-20 appearance-none text-center focus:ring-2 focus:ring-amber-500/20 transition-all duration-200 ease-in-out"
                >
                  <option value=">=">&gt;=</option>
                  <option value=">">&gt;</option>
                  <option value="=">=</option>
                  <option value="!=">!=</option>
                </select>
                <input
                  v-model.number="scoreThresholdValue"
                  type="number"
                  placeholder="0"
                  class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-amber-200 dark:border-amber-800/60 focus:border-amber-500 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none flex-grow focus:ring-2 focus:ring-amber-500/20 transition-all duration-200 ease-in-out"
                />
              </div>
            </div>
            
            <!-- Order direction (DESC / ASC) -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-amber-800 font-extrabold uppercase tracking-wider font-mono select-none">DIRECCIÓN DE ORDEN...</label>
              <div class="relative">
                <select
                  v-model="scoreOrderDirection"
                  class="w-full bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-amber-200 dark:border-amber-800/60 focus:border-amber-500 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none cursor-pointer appearance-none focus:ring-2 focus:ring-amber-550/20 transition-all duration-200 ease-in-out"
                >
                  <option value="DESC">DESC (Descendente)</option>
                  <option value="ASC">ASC (Ascendente)</option>
                </select>
                <div class="absolute inset-y-0 right-4 flex items-center pointer-events-none text-amber-500">
                  <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" /></svg>
                </div>
              </div>
            </div>
          </div>

          <!-- Case Rules list (Dynamic Rows) -->
          <div class="flex flex-col gap-3">
            <span class="text-xs font-bold text-amber-800 uppercase tracking-wider font-mono select-none">PUNTUAR SI... (CASE WHEN)</span>
            
            <div class="flex flex-col gap-3">
              <div
                v-for="(rule, idx) in scoreRules"
                :key="idx"
                class="flex flex-col gap-3"
              >
                <!-- Symbolism: + badge representing summation between dynamic rules -->
                <div v-if="idx > 0" class="self-center">
                  <span class="bg-amber-100 border border-amber-200 text-amber-700 font-bold font-mono text-sm w-7 h-7 flex items-center justify-center rounded-full select-none shadow-sm shadow-amber-200/30">+</span>
                </div>

                <div class="flex flex-col sm:flex-row items-stretch sm:items-center gap-2 bg-white/80 p-3 rounded-2xl border border-amber-200/50 shadow-sm">
                  <!-- Column Name to evaluate -->
                  <input
                    v-model="rule.column"
                    type="text"
                    placeholder="campo a evaluar"
                    class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-amber-800/60 focus:border-amber-500 focus:ring-2 focus:ring-amber-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out"
                    title="Campo a evaluar"
                  />
                  
                  <!-- Operator select -->
                  <select
                    v-model="rule.operator"
                    class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-amber-800/60 focus:border-amber-500 focus:ring-2 focus:ring-amber-500/20 rounded-lg px-2 py-2 text-xs sm:text-sm font-mono focus:outline-none cursor-pointer w-20 appearance-none text-center transition-all duration-200 ease-in-out"
                  >
                    <option value="LIKE">LIKE</option>
                    <option value="=">=</option>
                    <option value="!=">!=</option>
                    <option value=">">&gt;</option>
                    <option value="<">&lt;</option>
                  </select>
                  
                  <!-- Value/Pattern input -->
                  <input
                    v-model="rule.value"
                    type="text"
                    placeholder="valor / patrón"
                    class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-amber-800/60 focus:border-amber-500 focus:ring-2 focus:ring-amber-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out"
                    title="Patrón de evaluación"
                  />
                  
                  <!-- Weight Input -->
                  <div class="flex items-center gap-1.5 bg-slate-50 dark:bg-slate-800/40 border border-slate-200 dark:border-slate-700 rounded-lg px-2.5 py-1.5 w-full sm:w-28 focus-within:ring-2 focus-within:ring-amber-500/20 focus-within:border-amber-500 transition-all duration-200 ease-in-out">
                    <span class="text-[9px] text-amber-800/70 font-bold uppercase tracking-wider font-mono select-none">PESO:</span>
                    <input
                      v-model.number="rule.weight"
                      type="number"
                      placeholder="1"
                      class="bg-transparent text-slate-800 border-none font-mono text-xs focus:outline-none w-full font-semibold"
                      title="Peso en score"
                    />
                  </div>
                  
                  <!-- Remove Rule Button -->
                  <button
                    v-if="scoreRules.length > 1"
                    @click="removeRow('scoreRule', idx)"
                    class="bg-white hover:bg-red-50 border border-slate-200 hover:border-red-200 text-slate-400 hover:text-red-500 p-2.5 rounded-lg transition-all duration-200 ease-in-out cursor-pointer self-end sm:self-center hover:scale-105 active:scale-95"
                    title="Eliminar regla de scoring"
                  >
                    <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                  </button>
                </div>
              </div>

              <!-- Add Rule Button -->
              <button
                @click="addRow('scoreRule')"
                class="bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-amber-200 dark:border-amber-700 text-amber-700 dark:text-amber-300 px-3 py-2 rounded-xl text-xs font-bold self-start cursor-pointer flex items-center gap-1.5 transition-all duration-200 ease-in-out hover:scale-105 active:scale-95 shadow-sm"
              >
                <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5"><path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4"/></svg>
                Añadir Regla de Scoring
              </button>
            </div>
          </div>
        </div>
                  <!-- 6. CONCAT LAYOUT (EXCEL LOG GENERATOR CUSTOM TEMPLATE) -->
        <div v-else-if="operation === 'CONCAT'" class="flex flex-col gap-6 animate-fade-in text-slate-800">
          
          <!-- Highlighted Top Row: Excel/Log Meta-data with subtle green/teal background accent -->
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 bg-emerald-50/80 dark:bg-emerald-950/15 border border-emerald-200 dark:border-emerald-900/40 p-5 rounded-2xl shadow-inner">
            <!-- Acción dropdown -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-emerald-800 font-bold uppercase tracking-wider font-mono select-none">Tipo de Acción (Acción)</label>
              <div class="relative">
                <select
                  v-model="concatAction"
                  class="w-full bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-emerald-200 dark:border-emerald-800 focus:border-emerald-500 rounded-xl px-4 py-2.5 text-xs font-mono font-bold uppercase tracking-wider focus:outline-none focus:ring-2 focus:ring-emerald-500/20 transition-all duration-200 ease-in-out cursor-pointer appearance-none"
                >
                  <option value="UPDATE">UPDATE</option>
                  <option value="INSERT">INSERT</option>
                  <option value="DELETE">DELETE</option>
                </select>
                <div class="absolute inset-y-0 right-4 flex items-center pointer-events-none text-emerald-600">
                  <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" d="M19 9l-7 7-7-7" /></svg>
                </div>
              </div>
            </div>
            
            <!-- Identificador de Hoja -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-emerald-800 font-bold uppercase tracking-wider font-mono select-none">Identificador de Hoja (Excel)</label>
              <input
                v-model="concatSheetId"
                type="text"
                placeholder="DATOS1"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-emerald-200 dark:border-emerald-800 focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none font-semibold transition-all duration-200 ease-in-out"
              />
            </div>
          </div>

          <!-- Standard database structural inputs in a grid below -->
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 bg-slate-55 dark:bg-slate-800/40 p-5 rounded-2xl border border-slate-200 dark:border-slate-800">
            <!-- Tabla Origen -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-slate-500 font-bold uppercase tracking-wider font-mono select-none">Tabla Origen (tb)</label>
              <input
                v-model="concatSourceTable"
                type="text"
                placeholder="table"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none transition-all duration-200 ease-in-out"
              />
            </div>
            
            <!-- Tabla a Modificar -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-slate-500 font-bold uppercase tracking-wider font-mono select-none">Tabla a Modificar (dentro de CONCAT)</label>
              <input
                v-model="concatTargetTable"
                type="text"
                placeholder="SarixVerticales.table_name"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none transition-all duration-200 ease-in-out"
              />
            </div>
            
            <!-- Columna a Modificar (Disabled if DELETE is chosen) -->
            <div class="flex flex-col gap-1.5">
              <label
                class="text-[10px] font-bold uppercase tracking-wider font-mono select-none transition-colors duration-300"
                :class="concatAction === 'DELETE' ? 'text-slate-400' : 'text-slate-500'"
              >
                Columna a Modificar
              </label>
              <input
                v-model="concatTargetColumn"
                type="text"
                placeholder="column_name"
                :disabled="concatAction === 'DELETE'"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none disabled:bg-slate-100 dark:disabled:bg-slate-800/60 disabled:opacity-40 disabled:cursor-not-allowed transition-all duration-300"
              />
            </div>
            
            <!-- Valor a Asignar (Disabled if DELETE is chosen) -->
            <div class="flex flex-col gap-1.5">
              <label
                class="text-[10px] font-bold uppercase tracking-wider font-mono select-none transition-colors duration-300"
                :class="concatAction === 'DELETE' ? 'text-slate-400' : 'text-slate-500'"
              >
                Valor a Asignar
              </label>
              <input
                v-model="concatTargetValue"
                type="text"
                placeholder="'value'"
                :disabled="concatAction === 'DELETE'"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none disabled:bg-slate-100 dark:disabled:bg-slate-800/60 disabled:opacity-40 disabled:cursor-not-allowed transition-all duration-300"
              />
            </div>
          </div>

          <!-- Scope filter WHERE (IDs) -->
          <div class="flex flex-col gap-1.5 bg-slate-50 dark:bg-slate-800/40 p-5 rounded-2xl border border-slate-200 dark:border-slate-800">
            <label class="text-[10px] text-slate-500 font-bold uppercase tracking-wider font-mono select-none">Filtro de IDs (Separados por Coma)</label>
            <input
              v-model="concatFilteredIds"
              type="text"
              placeholder="1, 2"
              class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none transition-all duration-200 ease-in-out"
            />
          </div>
        </div>
      </div>
    </div>

    <!-- Right Preview & Output Panel: Col span 5 -->
    <div class="lg:col-span-5 flex flex-col gap-6">
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-6 h-full transition-colors duration-200">
        
        <div class="flex items-center justify-between">
          <h2 class="text-xl font-bold text-slate-800 dark:text-slate-100 flex items-center gap-2">
            <span class="w-2.5 h-6 bg-emerald-500 rounded-full inline-block"></span>
            Query SQL Generada
          </h2>
          <span class="text-[10px] bg-slate-100 dark:bg-slate-800 text-indigo-600 dark:text-indigo-400 font-bold border border-slate-200 dark:border-slate-700 px-2 py-0.5 rounded-md uppercase tracking-wider font-mono select-none">
            {{ operation }}
          </span>
        </div>

        <!-- Query Output Window -->
        <div class="flex-grow flex flex-col min-h-[280px] relative">
          <!-- Textarea container (read only - Dark theme for code views) -->
          <pre class="w-full flex-grow bg-slate-950 text-emerald-400 border border-slate-950 dark:border-slate-950/80 rounded-2xl p-4 text-sm font-mono overflow-x-auto overflow-y-auto whitespace-pre h-full selection:bg-indigo-500/30 selection:text-white">{{ generatedQuery }}</pre>

          <!-- Tips and Info Panel -->
          <div class="mt-4 p-3 bg-slate-955/50 dark:bg-slate-950/80 border border-slate-950 dark:border-slate-950/50 rounded-xl">
            <h4 class="text-xs font-bold text-indigo-400 mb-1 flex items-center gap-1.5 select-none font-mono uppercase tracking-wider">
              <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" /></svg>
              Reglas de Formateo Inteligente:
            </h4>
            <ul class="text-[10px] sm:text-[11px] text-slate-400 list-disc list-inside space-y-1">
              <li v-if="operation !== 'SCORE' && operation !== 'CONCAT'">Los textos en valores se envuelven en comillas simples (<code class="text-[10px] text-amber-300 font-mono">'valor'</code>).</li>
              <li v-if="operation === 'SCORE'">Las condiciones en los <code class="text-[10px] text-indigo-300 font-mono">CASE WHEN</code> ponderan cada resultado sumando los pesos correspondientes.</li>
              <li v-if="operation === 'CONCAT'">Genera sentencias de registro utilizando retornos de carro (<code class="text-[10px] text-indigo-300 font-mono">\n</code>) dentro de la función <code class="text-[10px] text-indigo-300 font-mono">CONCAT</code> para un formateo estructurado de logs Excel.</li>
              <li v-if="operation === 'CONCAT' && concatAction === 'DELETE'">Lógica de Mutación Activa: Al elegir <code class="text-[10px] text-red-400 font-mono">DELETE</code>, el CONCAT interno se ajusta para omitir el SET.</li>
              <li>Los números y la palabra clave <code class="text-[10px] text-emerald-300 font-mono">NULL</code> quedan sin comillas.</li>
              <li v-if="operation !== 'SCORE' && operation !== 'CONCAT'">Para el operador <code class="text-[10px] text-indigo-300 font-mono">IN</code>, separa los valores por coma para auto-formatear a lista.</li>
            </ul>
          </div>
        </div>

        <!-- Copy Action Button -->
        <button
          @click="copyQuery"
          :class="[
            'w-full py-3.5 rounded-xl font-bold text-sm tracking-wide flex items-center justify-center gap-2 cursor-pointer shadow-md transition-all duration-200 ease-in-out active:scale-[0.98]',
            copied
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
          {{ copied ? '¡Query Copiada!' : 'Copiar Query SQL' }}
        </button>

      </div>
    </div>

  </div>
</template>

<style scoped>
/* Custom animations */
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
