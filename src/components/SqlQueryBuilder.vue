<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  depth: {
    type: Number,
    default: 0
  },
  modelValue: {
    type: Object,
    default: null
  }
})

const emit = defineEmits(['update:modelValue'])

// Helper to create the default structure of a query state
const createDefaultQueryState = () => ({
  tableName: 'table_name',
  operation: 'SELECT', // SELECT, INSERT, UPDATE, DELETE, SCORE, CONCAT
  fromType: 'table', // 'table' or 'subquery'
  subqueryFromData: null,
  subqueryFromAlias: 'sub_tabla',
  selectFields: ['id', 'nombre', 'email'],
  insertUpdatePairs: [
    { column: 'nombre', value: 'Carlos' },
    { column: 'email', value: 'carlos@example.com' },
    { column: 'activo', value: '1' }
  ],
  whereConditions: [
    { column: 'id', operator: '=', valueType: 'literal', value: '12', subqueryData: null, logicalOperator: 'AND' }
  ],
  scoreBaseFields: 'table_name.*',
  scoreThresholdOperator: '>=',
  scoreThresholdValue: 0,
  scoreOrderDirection: 'DESC',
  scoreRules: [
    { column: 'desc_column', operator: 'LIKE', value: '%value%', weight: 4 },
    { column: 'status_column', operator: '=', value: 'A', weight: 2 }
  ],
  concatAction: 'UPDATE',
  concatSheetId: 'DATOS1',
  concatSourceTable: 'table',
  concatTargetTable: 'table_name',
  concatTargetColumn: 'column_name',
  concatTargetValue: "'value'",
  concatFilteredIds: '1, 2',
  concatOperator: 'IN' // IN or NOT IN
})

// Local state for the root query builder
const localState = ref(null)
if (!props.modelValue) {
  localState.value = createDefaultQueryState()
}

// Compute the active state: either props.modelValue (nested) or localState (root)
const state = computed({
  get: () => props.modelValue || localState.value,
  set: (val) => {
    if (props.modelValue) {
      emit('update:modelValue', val)
    } else {
      localState.value = val
    }
  }
})

// Proxy computed properties to allow seamless integration with the existing template
const tableName = computed({
  get: () => state.value?.tableName ?? '',
  set: (val) => { if (state.value) state.value.tableName = val }
})

const operation = computed({
  get: () => state.value?.operation ?? 'SELECT',
  set: (val) => { if (state.value) state.value.operation = val }
})

const selectFields = computed({
  get: () => state.value?.selectFields ?? [],
  set: (val) => { if (state.value) state.value.selectFields = val }
})

const insertUpdatePairs = computed({
  get: () => state.value?.insertUpdatePairs ?? [],
  set: (val) => { if (state.value) state.value.insertUpdatePairs = val }
})

const whereConditions = computed({
  get: () => state.value?.whereConditions ?? [],
  set: (val) => { if (state.value) state.value.whereConditions = val }
})

const scoreBaseFields = computed({
  get: () => state.value?.scoreBaseFields ?? '',
  set: (val) => { if (state.value) state.value.scoreBaseFields = val }
})

const scoreThresholdOperator = computed({
  get: () => state.value?.scoreThresholdOperator ?? '>=',
  set: (val) => { if (state.value) state.value.scoreThresholdOperator = val }
})

const scoreThresholdValue = computed({
  get: () => state.value?.scoreThresholdValue ?? 0,
  set: (val) => { if (state.value) state.value.scoreThresholdValue = val }
})

const scoreOrderDirection = computed({
  get: () => state.value?.scoreOrderDirection ?? 'DESC',
  set: (val) => { if (state.value) state.value.scoreOrderDirection = val }
})

const scoreRules = computed({
  get: () => state.value?.scoreRules ?? [],
  set: (val) => { if (state.value) state.value.scoreRules = val }
})

const concatAction = computed({
  get: () => state.value?.concatAction ?? 'UPDATE',
  set: (val) => { if (state.value) state.value.concatAction = val }
})

const concatSheetId = computed({
  get: () => state.value?.concatSheetId ?? '',
  set: (val) => { if (state.value) state.value.concatSheetId = val }
})

const concatSourceTable = computed({
  get: () => state.value?.concatSourceTable ?? '',
  set: (val) => { if (state.value) state.value.concatSourceTable = val }
})

const concatTargetTable = computed({
  get: () => state.value?.concatTargetTable ?? '',
  set: (val) => { if (state.value) state.value.concatTargetTable = val }
})

const concatTargetColumn = computed({
  get: () => state.value?.concatTargetColumn ?? '',
  set: (val) => { if (state.value) state.value.concatTargetColumn = val }
})

const concatTargetValue = computed({
  get: () => state.value?.concatTargetValue ?? '',
  set: (val) => { if (state.value) state.value.concatTargetValue = val }
})

const concatFilteredIds = computed({
  get: () => state.value?.concatFilteredIds ?? '',
  set: (val) => { if (state.value) state.value.concatFilteredIds = val }
})

const concatOperator = computed({
  get: () => state.value?.concatOperator ?? 'IN',
  set: (val) => { if (state.value) state.value.concatOperator = val }
})

const copied = ref(false)

// Scoped row addition / removal methods updated to use current state
const addRow = (type) => {
  if (!state.value) return
  if (type === 'select') {
    state.value.selectFields.push('')
  } else if (type === 'pair') {
    state.value.insertUpdatePairs.push({ column: '', value: '' })
  } else if (type === 'where') {
    state.value.whereConditions.push({
      column: '',
      operator: '=',
      valueType: 'literal',
      value: '',
      subqueryData: null,
      logicalOperator: 'AND'
    })
  } else if (type === 'scoreRule') {
    state.value.scoreRules.push({ column: '', operator: 'LIKE', value: '', weight: 1 })
  }
}

const removeRow = (type, index) => {
  if (!state.value) return
  if (type === 'select') {
    state.value.selectFields.splice(index, 1)
  } else if (type === 'pair') {
    state.value.insertUpdatePairs.splice(index, 1)
  } else if (type === 'where') {
    state.value.whereConditions.splice(index, 1)
  } else if (type === 'scoreRule') {
    state.value.scoreRules.splice(index, 1)
  }
}

// Handlers for nested state initialization
const handleValueTypeChange = (cond) => {
  if (cond.valueType === 'subquery' && !cond.subqueryData) {
    cond.subqueryData = createDefaultQueryState()
    cond.subqueryData.operation = 'SELECT'
  }
}

const handleFromTypeChange = () => {
  if (state.value && state.value.fromType === 'subquery' && !state.value.subqueryFromData) {
    state.value.subqueryFromData = createDefaultQueryState()
    state.value.subqueryFromData.operation = 'SELECT'
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

// Custom formatter for WHERE conditions to handle 'IN' / 'NOT IN' operations nicely
const formatWhereValue = (val, operator) => {
  if (!val) return "''"
  const trimmed = val.trim()
  const isListOperator = operator.toUpperCase() === 'IN' || operator.toUpperCase() === 'NOT IN'
  
  if (isListOperator) {
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

// Placeholder helper for standard inputs vs IN/NOT IN list inputs
const getPlaceholderForValue = (operator, valueType) => {
  if (valueType === 'subquery') return ''
  const opUpper = operator ? operator.toUpperCase() : ''
  if (opUpper === 'IN' || opUpper === 'NOT IN') {
    return "Valores: 1, 2, 3 o 'activo', 'pendiente'"
  }
  return "valor (ej. Carlos, 12, null)"
}

// Visual depth/theme helper for nested layouts
const bgClass = computed(() => {
  if (props.depth === 1) {
    return 'bg-slate-50 dark:bg-slate-900/50 border border-slate-200 dark:border-slate-800'
  }
  return 'bg-slate-100/70 dark:bg-slate-950/80 border border-slate-200 dark:border-slate-800/60'
})

// Recursive query compilation logic
const compileQuery = (stateData, indentLevel = 0) => {
  if (!stateData) return ''
  
  const indent = '  '.repeat(indentLevel)
  const innerIndent = '  '.repeat(indentLevel + 1)
  const op = stateData.operation || 'SELECT'

  switch (op) {
    case 'SELECT': {
      const cols = (stateData.selectFields || [])
        .filter(f => f.trim() !== '')
        .join(', ') || '*'
      
      let fromPart = ''
      if (stateData.fromType === 'subquery' && stateData.subqueryFromData) {
        const compiledSub = compileQuery(stateData.subqueryFromData, indentLevel + 2)
        const alias = stateData.subqueryFromAlias ? stateData.subqueryFromAlias.trim() : 'sub_tabla'
        fromPart = `(\n${compiledSub}\n${innerIndent}) AS ${alias}`
      } else {
        fromPart = (stateData.tableName || '').trim() || 'mi_tabla'
      }

      let query = `${indent}SELECT ${cols}\n${indent}FROM ${fromPart}`

      const activeWhere = (stateData.whereConditions || []).filter(w => w.column.trim() !== '')
      if (activeWhere.length > 0) {
        query += `\n${indent}WHERE `
        activeWhere.forEach((w, idx) => {
          let condStr = ''
          const colName = w.column.trim()
          const operator = w.operator || '='
          
          if (w.valueType === 'subquery' && w.subqueryData) {
            const compiledSub = compileQuery(w.subqueryData, indentLevel + 2)
            condStr = `${colName} ${operator} (\n${compiledSub}\n${innerIndent})`
          } else {
            const formattedVal = formatWhereValue(w.value, operator)
            condStr = `${colName} ${operator} ${formattedVal}`
          }

          if (idx === 0) {
            query += condStr
          } else {
            const logicalOp = w.logicalOperator || 'AND'
            query += `\n${innerIndent}${logicalOp} ${condStr}`
          }
        })
      }
      return query
    }

    case 'INSERT': {
      const table = (stateData.tableName || '').trim() || 'mi_tabla'
      const activePairs = (stateData.insertUpdatePairs || []).filter(p => p.column.trim() !== '')

      if (activePairs.length === 0) {
        return `${indent}INSERT INTO ${table} () VALUES ()`
      }

      const cols = activePairs.map(p => p.column.trim()).join(', ')
      const vals = activePairs.map(p => formatValue(p.value)).join(', ')

      return `${indent}INSERT INTO ${table} (${cols})\n${indent}VALUES (${vals})`
    }

    case 'UPDATE': {
      const table = (stateData.tableName || '').trim() || 'mi_tabla'
      const activePairs = (stateData.insertUpdatePairs || []).filter(p => p.column.trim() !== '')

      let query = `${indent}UPDATE ${table}`

      if (activePairs.length === 0) {
        query += `\n${indent}SET /* define columnas a actualizar */`
      } else {
        const setStr = activePairs
          .map(p => `${p.column.trim()} = ${formatValue(p.value)}`)
          .join(`,\n${indent}    `)
        query += `\n${indent}SET ${setStr}`
      }

      const activeWhere = (stateData.whereConditions || []).filter(w => w.column.trim() !== '')
      if (activeWhere.length > 0) {
        let whereStr = ''
        activeWhere.forEach((w, idx) => {
          let condStr = ''
          const colName = w.column.trim()
          const operator = w.operator || '='
          
          if (w.valueType === 'subquery' && w.subqueryData) {
            const compiledSub = compileQuery(w.subqueryData, indentLevel + 2)
            condStr = `${colName} ${operator} (\n${compiledSub}\n${innerIndent})`
          } else {
            const formattedVal = formatWhereValue(w.value, operator)
            condStr = `${colName} ${operator} ${formattedVal}`
          }

          if (idx === 0) {
            whereStr = condStr
          } else {
            const logicalOp = w.logicalOperator || 'AND'
            whereStr += `\n${indent}  ${logicalOp} ${condStr}`
          }
        })
        query += `\n${indent}WHERE ${whereStr}`
      } else {
        query += `\n${indent}WHERE /* id = 1 */`
      }

      return query
    }

    case 'DELETE': {
      const table = (stateData.tableName || '').trim() || 'mi_tabla'
      let query = `${indent}DELETE FROM ${table}`

      const activeWhere = (stateData.whereConditions || []).filter(w => w.column.trim() !== '')
      if (activeWhere.length > 0) {
        let whereStr = ''
        activeWhere.forEach((w, idx) => {
          let condStr = ''
          const colName = w.column.trim()
          const operator = w.operator || '='
          
          if (w.valueType === 'subquery' && w.subqueryData) {
            const compiledSub = compileQuery(w.subqueryData, indentLevel + 2)
            condStr = `${colName} ${operator} (\n${compiledSub}\n${innerIndent})`
          } else {
            const formattedVal = formatWhereValue(w.value, operator)
            condStr = `${colName} ${operator} ${formattedVal}`
          }

          if (idx === 0) {
            whereStr = condStr
          } else {
            const logicalOp = w.logicalOperator || 'AND'
            whereStr += `\n${indent}  ${logicalOp} ${condStr}`
          }
        })
        query += `\n${indent}WHERE ${whereStr}`
      } else {
        query += `\n${indent}WHERE /* id = 1 */`
      }

      return query
    }

    case 'SCORE': {
      const table = (stateData.tableName || '').trim() || 'mi_tabla'
      const baseFields = (stateData.scoreBaseFields || '').trim() || 'table_name.*'
      const activeRules = (stateData.scoreRules || []).filter(r => r.column.trim() !== '')
      const threshOp = stateData.scoreThresholdOperator || '>='
      const threshVal = stateData.scoreThresholdValue !== '' ? stateData.scoreThresholdValue : '0'
      const orderDir = stateData.scoreOrderDirection || 'DESC'

      let casesStr = ''
      if (activeRules.length === 0) {
        casesStr = `${indent}        0`
      } else {
        casesStr = activeRules.map((r, i) => {
          const formattedVal = formatWhereValue(r.value, r.operator)
          const plus = i < activeRules.length - 1 ? ' +' : ''
          return `${indent}        (CASE WHEN ${r.column.trim()} ${r.operator} ${formattedVal} THEN ${r.weight} ELSE 0 END)${plus}`
        }).join('\n')
      }

      return `${indent}SELECT \n${indent}    ${baseFields},\n${indent}    (\n${casesStr}\n${indent}    ) AS score\n${indent}FROM ${table}\n${indent}HAVING score ${threshOp} ${threshVal}\n${indent}ORDER BY score ${orderDir}`
    }

    case 'CONCAT': {
      const action = stateData.concatAction || 'UPDATE'
      const sheetId = (stateData.concatSheetId || '').trim() || 'DATOS1'
      const sourceTable = (stateData.concatSourceTable || '').trim() || 'table'
      const targetTable = (stateData.concatTargetTable || '').trim() || 'table_name'
      const targetColumn = (stateData.concatTargetColumn || '').trim() || 'column_name'
      const targetValue = (stateData.concatTargetValue || '').trim() || "'value'"
      const filteredIds = (stateData.concatFilteredIds || '').trim() || '1, 2'
      const concatOp = stateData.concatOperator || 'IN'

      let concatInside = ''
      if (action === 'DELETE') {
        concatInside = `CONCAT("DELETE FROM ${targetTable} \\nWHERE id = ", tb.id)`
      } else if (action === 'INSERT') {
        concatInside = `CONCAT("INSERT INTO ${targetTable} (${targetColumn}) VALUES (${targetValue}) \\nWHERE id = ", tb.id)`
      } else { // UPDATE
        concatInside = `CONCAT("UPDATE ${targetTable} \\nSET ${targetColumn} = ${targetValue} \\nWHERE id = ", tb.id)`
      }

      return `${indent}SELECT\n${indent}    "${action}",\n${indent}    tb.id,\n${indent}    ${concatInside},\n${indent}    "${sheetId}"\n${indent}FROM ${sourceTable} tb\n${indent}WHERE tb.id ${concatOp} (${filteredIds})`
    }

    default:
      return ''
  }
}

// Real-time SQL Query generator wrapper
const generatedQuery = computed(() => {
  if (!state.value) return ''
  const compiled = compileQuery(state.value, 0)
  if (!compiled) return ''
  let result = compiled
  if (result && !result.endsWith(';')) {
    result += ';'
  }
  return result
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
  <!-- Main layout if root (depth === 0) -->
  <div v-if="depth === 0" class="grid grid-cols-1 lg:grid-cols-12 gap-8 max-w-7xl mx-auto px-4 sm:px-6 py-8">
    
    <!-- Left Configuration Panel: Col span 7 -->
    <div class="lg:col-span-7 flex flex-col gap-6">
      <div class="bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 rounded-3xl p-6 sm:p-8 shadow-md dark:shadow-none flex flex-col gap-6 transition-colors duration-200">
        
        <!-- Header & Grouped Selector -->
        <div class="flex flex-col sm:flex-row sm:items-center justify-between gap-4 border-b border-slate-150 dark:border-slate-800 pb-4">
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
                <optgroup label="Sentencias Clásicas" class="bg-white dark:bg-slate-900 text-indigo-755 dark:text-indigo-400 font-bold font-mono">
                  <option value="SELECT">SELECT (Consultar)</option>
                  <option value="INSERT">INSERT (Insertar)</option>
                  <option value="UPDATE">UPDATE (Actualizar)</option>
                  <option value="DELETE">DELETE (Eliminar)</option>
                </optgroup>
                <optgroup label="Plantillas Especiales" class="bg-white dark:bg-slate-900 text-emerald-600 dark:text-emerald-400 font-bold font-mono">
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
            <span class="text-indigo-650 dark:text-indigo-400 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">SELECT</span>
            
            <div class="flex flex-wrap items-center gap-2 bg-slate-50 dark:bg-slate-800/40 p-4 rounded-2xl border border-slate-200 dark:border-slate-800">
              <div v-for="(field, idx) in selectFields" :key="idx" class="flex items-center gap-1 bg-white dark:bg-slate-900 p-1.5 rounded-lg border border-slate-200 dark:border-slate-800 shadow-sm focus-within:ring-2 focus-within:ring-indigo-500/20 focus-within:border-indigo-500 transition-all duration-200 ease-in-out">
                <input
                  v-model="selectFields[idx]"
                  type="text"
                  placeholder="campo"
                  class="bg-transparent text-slate-800 dark:text-slate-200 border-none font-mono text-sm focus:outline-none w-20 sm:w-24 placeholder:text-slate-450 dark:placeholder:text-slate-600 font-medium"
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
          <div class="flex flex-col gap-3">
            <div class="flex flex-wrap items-center gap-3">
              <span class="text-indigo-600 dark:text-indigo-400 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none w-14">FROM</span>
              
              <!-- Source type selector toggle -->
              <div class="flex items-center bg-slate-100 dark:bg-slate-800 p-1 rounded-xl border border-slate-200 dark:border-slate-700 shadow-inner">
                <button
                  type="button"
                  @click="state.fromType = 'table'"
                  :class="[
                    'px-3 py-1.5 rounded-lg text-xs font-bold font-mono uppercase transition-all duration-200 cursor-pointer',
                    state.fromType !== 'subquery'
                      ? 'bg-white dark:bg-slate-900 text-indigo-600 dark:text-indigo-400 shadow-sm'
                      : 'text-slate-500 hover:text-slate-700 dark:text-slate-400'
                  ]"
                >
                  Tabla
                </button>
                <button
                  type="button"
                  @click="state.fromType = 'subquery'; handleFromTypeChange()"
                  :class="[
                    'px-3 py-1.5 rounded-lg text-xs font-bold font-mono uppercase transition-all duration-200 cursor-pointer',
                    state.fromType === 'subquery'
                      ? 'bg-white dark:bg-slate-900 text-indigo-600 dark:text-indigo-400 shadow-sm'
                      : 'text-slate-500 hover:text-slate-700 dark:text-slate-400'
                  ]"
                >
                  Subquery
                </button>
              </div>

              <input
                v-if="state.fromType !== 'subquery'"
                v-model="tableName"
                type="text"
                placeholder="nombre_tabla"
                class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out flex-grow sm:flex-grow-0 sm:w-72 font-semibold"
              />
            </div>

            <!-- Nested subquery for FROM -->
            <div v-if="state.fromType === 'subquery'" class="border-l-4 border-indigo-500/50 pl-4 bg-slate-50 dark:bg-slate-900/55 rounded-r-2xl p-4 shadow-inner flex flex-col gap-4">
              <div class="flex items-center gap-3">
                <span class="text-xs font-bold text-slate-500 dark:text-slate-400 font-mono uppercase">Alias del Subquery:</span>
                <input
                  v-model="state.subqueryFromAlias"
                  type="text"
                  placeholder="sub_tabla"
                  class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-3 py-1.5 text-xs font-mono focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 w-44 font-semibold"
                />
              </div>
              <div v-if="depth < 2">
                <div class="text-[10px] text-indigo-550 dark:text-indigo-400 font-bold uppercase tracking-wider mb-2">Subconsulta FROM (Nivel {{ depth + 1 }})</div>
                <SqlQueryBuilder v-model="state.subqueryFromData" :depth="depth + 1" />
              </div>
              <div v-else class="text-xs text-red-500 font-semibold p-3 bg-red-50/50 dark:bg-red-950/20 border border-red-200 dark:border-red-900/50 rounded-xl">
                Límite de anidación alcanzado (máximo 2 niveles de subqueries).
              </div>
            </div>
          </div>

          <!-- WHERE Row -->
          <div class="flex flex-col gap-2.5">
            <span class="text-indigo-600 dark:text-indigo-400 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">WHERE</span>
            
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
                
                <!-- WHERE condition inputs container -->
                <div class="flex flex-col gap-3 p-3 bg-slate-50 dark:bg-slate-800/40 rounded-2xl border border-slate-200 dark:border-slate-700">
                  <div class="flex flex-wrap sm:flex-nowrap items-center gap-2">
                    <input
                      v-model="cond.column"
                      type="text"
                      placeholder="columna"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out font-semibold"
                    />
                    
                    <select
                      v-model="cond.operator"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none cursor-pointer w-24 appearance-none text-center transition-all duration-200 ease-in-out font-semibold"
                    >
                      <option value="=">=</option>
                      <option value="!=">!=</option>
                      <option value="LIKE">LIKE</option>
                      <option value="IN">IN</option>
                      <option value="NOT IN">NOT IN</option>
                    </select>
                    
                    <select
                      v-model="cond.valueType"
                      @change="handleValueTypeChange(cond)"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-2.5 py-2 text-xs font-mono focus:outline-none cursor-pointer w-28 appearance-none text-center transition-all duration-200 ease-in-out font-semibold"
                    >
                      <option value="literal">Literal</option>
                      <option value="subquery">Subquery</option>
                    </select>
                    
                    <input
                      v-if="cond.valueType === 'literal'"
                      v-model="cond.value"
                      type="text"
                      :placeholder="getPlaceholderForValue(cond.operator, cond.valueType)"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-grow min-w-[120px] transition-all duration-200 ease-in-out font-semibold"
                    />
                    
                    <button
                      @click="removeRow('where', idx)"
                      class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-950/30 border border-slate-200 dark:border-slate-700 hover:border-red-200 dark:hover:border-red-800 text-slate-400 dark:text-slate-500 hover:text-red-500 p-2 rounded-lg transition-all duration-200 ease-in-out cursor-pointer hover:scale-105 active:scale-95"
                      title="Eliminar condición"
                    >
                      <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                    </button>
                  </div>
                  
                  <!-- Nested Subquery inside WHERE -->
                  <div v-if="cond.valueType === 'subquery'" class="mt-2 border-l-4 border-indigo-500/50 pl-4 bg-slate-50 dark:bg-slate-900/55 rounded-r-2xl p-4 shadow-inner">
                    <div v-if="depth < 2">
                      <div class="text-[10px] text-indigo-550 dark:text-indigo-400 font-bold uppercase tracking-wider mb-2">Subconsulta WHERE (Nivel {{ depth + 1 }})</div>
                      <SqlQueryBuilder v-model="cond.subqueryData" :depth="depth + 1" />
                    </div>
                    <div v-else class="text-xs text-red-500 font-semibold p-3 bg-red-50/50 dark:bg-red-950/20 border border-red-200 dark:border-red-900/50 rounded-xl">
                      Límite de anidación alcanzado (máximo 2 niveles de subqueries).
                    </div>
                  </div>
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
            <span class="text-indigo-600 dark:text-indigo-400 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">INSERT INTO</span>
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
                  class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-950/30 border border-slate-200 dark:border-slate-700 hover:border-red-200 dark:hover:border-red-800 text-slate-400 dark:text-slate-500 hover:text-red-500 p-2 rounded-lg transition-all duration-200 ease-in-out cursor-pointer self-end mb-[2px] hover:scale-105 active:scale-95"
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
              <span class="text-indigo-400 font-bold select-none">(</span>
              <div class="flex flex-wrap items-center gap-1 text-emerald-400">
                <template v-for="(p, i) in insertUpdatePairs.filter(p => p.column.trim())" :key="'col-' + i">
                  <span class="font-semibold">{{ p.column }}</span>
                  <span v-if="i < insertUpdatePairs.filter(p => p.column.trim()).length - 1" class="text-slate-600 select-none">,</span>
                </template>
                <span v-if="insertUpdatePairs.filter(p => p.column.trim()).length === 0" class="text-slate-700 italic select-none">columnas...</span>
              </div>
              <span class="text-indigo-400 font-bold select-none">)</span>
            </div>
            <!-- VALUES label -->
            <div class="text-indigo-400 font-extrabold pl-2 select-none">VALUES</div>
            <!-- Values (grouped in separate parentheses) -->
            <div class="flex items-start gap-1 pl-4">
              <span class="text-indigo-400 font-bold select-none">(</span>
              <div class="flex flex-wrap items-center gap-1 text-amber-300">
                <template v-for="(p, i) in insertUpdatePairs.filter(p => p.column.trim())" :key="'val-' + i">
                  <span class="font-semibold">{{ formatValue(p.value) }}</span>
                  <span v-if="i < insertUpdatePairs.filter(p => p.column.trim()).length - 1" class="text-slate-605 select-none">,</span>
                </template>
                <span v-if="insertUpdatePairs.filter(p => p.column.trim()).length === 0" class="text-slate-700 italic select-none">valores...</span>
              </div>
              <span class="text-indigo-400 font-bold select-none">)</span>
            </div>
          </div>
        </div>

        <!-- 3. UPDATE LAYOUT -->
        <div v-else-if="operation === 'UPDATE'" class="flex flex-col gap-6 animate-fade-in">
          
          <!-- UPDATE [ __table_name__ ] -->
          <div class="flex items-center gap-3">
            <span class="text-indigo-650 dark:text-indigo-400 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">UPDATE</span>
            <input
              v-model="tableName"
              type="text"
              placeholder="nombre_tabla"
              class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out flex-grow sm:flex-grow-0 sm:w-72 font-semibold"
            />
          </div>

          <!-- SET [ __set_pairs__ ] -->
          <div class="flex flex-col gap-2.5">
            <span class="text-indigo-650 dark:text-indigo-400 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">SET</span>
            
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
                  class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-205 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-1.5 text-sm font-mono focus:outline-none flex-1 transition-all duration-200 ease-in-out"
                />
                <span class="text-slate-400 font-mono font-extrabold text-sm select-none">=</span>
                <input
                  v-model="pair.value"
                  type="text"
                  placeholder="valor"
                  class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-205 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-1.5 text-sm font-mono focus:outline-none flex-1 transition-all duration-200 ease-in-out"
                />
                
                <button
                  v-if="insertUpdatePairs.length > 1"
                  @click="removeRow('pair', idx)"
                  class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-955/30 border border-slate-200 dark:border-slate-700 hover:border-red-200 dark:hover:border-red-800 text-slate-400 dark:text-slate-500 hover:text-red-500 p-2 rounded-lg transition-all duration-200 ease-in-out cursor-pointer hover:scale-105 active:scale-95"
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
            <span class="text-indigo-650 dark:text-indigo-400 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">WHERE</span>
            
            <div class="flex flex-col gap-3">
              <div v-for="(cond, idx) in whereConditions" :key="idx" class="flex flex-col gap-3">
                
                <!-- Selectable Logical Operator (AND / OR) between conditions -->
                <div v-if="idx > 0" class="self-center flex items-center">
                  <div class="relative flex items-center">
                    <select
                      v-model="cond.logicalOperator"
                      class="bg-indigo-50 dark:bg-indigo-955/60 hover:bg-indigo-100 dark:hover:bg-indigo-900/60 text-indigo-700 dark:text-indigo-400 border border-indigo-200 dark:border-indigo-800 rounded-xl px-5 py-1 text-xs font-mono font-bold tracking-widest cursor-pointer focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-550 appearance-none text-center pr-8 select-none transition-all duration-200 ease-in-out"
                    >
                      <option value="AND">AND</option>
                      <option value="OR">OR</option>
                    </select>
                    <div class="absolute inset-y-0 right-2.5 flex items-center pointer-events-none text-indigo-600">
                      <svg class="w-3 h-3" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M19 9l-7 7-7-7" /></svg>
                    </div>
                  </div>
                </div>
                
                <div class="flex flex-col gap-3 p-3 bg-slate-55 dark:bg-slate-800/40 rounded-2xl border border-slate-200 dark:border-slate-700">
                  <div class="flex flex-wrap sm:flex-nowrap items-center gap-2">
                    <input
                      v-model="cond.column"
                      type="text"
                      placeholder="columna"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out font-semibold"
                    />
                    
                    <select
                      v-model="cond.operator"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none cursor-pointer w-24 appearance-none text-center transition-all duration-200 ease-in-out font-semibold"
                    >
                      <option value="=">=</option>
                      <option value="!=">!=</option>
                      <option value="LIKE">LIKE</option>
                      <option value="IN">IN</option>
                      <option value="NOT IN">NOT IN</option>
                    </select>
                    
                    <select
                      v-model="cond.valueType"
                      @change="handleValueTypeChange(cond)"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-2.5 py-2 text-xs font-mono focus:outline-none cursor-pointer w-28 appearance-none text-center transition-all duration-200 ease-in-out font-semibold"
                    >
                      <option value="literal">Literal</option>
                      <option value="subquery">Subquery</option>
                    </select>
                    
                    <input
                      v-if="cond.valueType === 'literal'"
                      v-model="cond.value"
                      type="text"
                      :placeholder="getPlaceholderForValue(cond.operator, cond.valueType)"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-grow min-w-[120px] transition-all duration-200 ease-in-out font-semibold"
                    />
                    
                    <button
                      @click="removeRow('where', idx)"
                      class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-955/30 border border-slate-200 dark:border-slate-700 hover:border-red-200 dark:hover:border-red-800 text-slate-400 dark:text-slate-550 hover:text-red-500 p-2 rounded-lg transition-all duration-200 ease-in-out cursor-pointer hover:scale-105 active:scale-95"
                      title="Eliminar condición"
                    >
                      <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                    </button>
                  </div>
                  
                  <!-- Nested Subquery inside WHERE -->
                  <div v-if="cond.valueType === 'subquery'" class="mt-2 border-l-4 border-indigo-500/50 pl-4 bg-slate-50 dark:bg-slate-900/55 rounded-r-2xl p-4 shadow-inner">
                    <div v-if="depth < 2">
                      <div class="text-[10px] text-indigo-555 dark:text-indigo-400 font-bold uppercase tracking-wider mb-2">Subconsulta WHERE (Nivel {{ depth + 1 }})</div>
                      <SqlQueryBuilder v-model="cond.subqueryData" :depth="depth + 1" />
                    </div>
                    <div v-else class="text-xs text-red-500 font-semibold p-3 bg-red-50/50 dark:bg-red-955/20 border border-red-200 dark:border-red-900/50 rounded-xl">
                      Límite de anidación alcanzado (máximo 2 niveles de subqueries).
                    </div>
                  </div>
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
            <span class="text-indigo-650 dark:text-indigo-400 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">DELETE FROM</span>
            <input
              v-model="tableName"
              type="text"
              placeholder="nombre_tabla"
              class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 ease-in-out flex-grow sm:flex-grow-0 sm:w-72 font-semibold"
            />
          </div>

          <!-- WHERE [ __where_conditions__ ] -->
          <div class="flex flex-col gap-2.5">
            <span class="text-indigo-650 dark:text-indigo-400 font-mono font-extrabold text-sm sm:text-base tracking-wider uppercase select-none">WHERE</span>
            
            <div class="flex flex-col gap-3">
              <div v-for="(cond, idx) in whereConditions" :key="idx" class="flex flex-col gap-3">
                
                <!-- Selectable Logical Operator (AND / OR) between conditions -->
                <div v-if="idx > 0" class="self-center flex items-center">
                  <div class="relative flex items-center">
                    <select
                      v-model="cond.logicalOperator"
                      class="bg-indigo-55 hover:bg-indigo-100 text-indigo-750 border border-indigo-200 rounded-xl px-5 py-1 text-xs font-mono font-bold tracking-widest cursor-pointer focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500 appearance-none text-center pr-8 select-none transition-all duration-200 ease-in-out"
                    >
                      <option value="AND">AND</option>
                      <option value="OR">OR</option>
                    </select>
                    <div class="absolute inset-y-0 right-2.5 flex items-center pointer-events-none text-indigo-600">
                      <svg class="w-3 h-3" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M19 9l-7 7-7-7" /></svg>
                    </div>
                  </div>
                </div>
                
                <div class="flex flex-col gap-3 p-3 bg-slate-50 dark:bg-slate-800/40 rounded-2xl border border-slate-200 dark:border-slate-700">
                  <div class="flex flex-wrap sm:flex-nowrap items-center gap-2">
                    <input
                      v-model="cond.column"
                      type="text"
                      placeholder="columna"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-1 min-w-[120px] transition-all duration-200 ease-in-out font-semibold"
                    />
                    
                    <select
                      v-model="cond.operator"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none cursor-pointer w-24 appearance-none text-center transition-all duration-200 ease-in-out font-semibold"
                    >
                      <option value="=">=</option>
                      <option value="!=">!=</option>
                      <option value="LIKE">LIKE</option>
                      <option value="IN">IN</option>
                      <option value="NOT IN">NOT IN</option>
                    </select>
                    
                    <select
                      v-model="cond.valueType"
                      @change="handleValueTypeChange(cond)"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-2.5 py-2 text-xs font-mono focus:outline-none cursor-pointer w-28 appearance-none text-center transition-all duration-200 ease-in-out font-semibold"
                    >
                      <option value="literal">Literal</option>
                      <option value="subquery">Subquery</option>
                    </select>
                    
                    <input
                      v-if="cond.valueType === 'literal'"
                      v-model="cond.value"
                      type="text"
                      :placeholder="getPlaceholderForValue(cond.operator, cond.valueType)"
                      class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-lg px-3 py-2 text-xs sm:text-sm font-mono focus:outline-none flex-grow min-w-[120px] transition-all duration-200 ease-in-out font-semibold"
                    />
                    
                    <button
                      @click="removeRow('where', idx)"
                      class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-955/30 border border-slate-200 dark:border-slate-700 hover:border-red-200 dark:hover:border-red-800 text-slate-400 dark:text-slate-500 hover:text-red-500 p-2 rounded-lg transition-all duration-200 ease-in-out cursor-pointer hover:scale-105 active:scale-95"
                      title="Eliminar condición"
                    >
                      <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                    </button>
                  </div>
                  
                  <!-- Nested Subquery inside WHERE -->
                  <div v-if="cond.valueType === 'subquery'" class="mt-2 border-l-4 border-indigo-500/50 pl-4 bg-slate-50 dark:bg-slate-900/55 rounded-r-2xl p-4 shadow-inner">
                    <div v-if="depth < 2">
                      <div class="text-[10px] text-indigo-550 dark:text-indigo-400 font-bold uppercase tracking-wider mb-2">Subconsulta WHERE (Nivel {{ depth + 1 }})</div>
                      <SqlQueryBuilder v-model="cond.subqueryData" :depth="depth + 1" />
                    </div>
                    <div v-else class="text-xs text-red-500 font-semibold p-3 bg-red-55/50 dark:bg-red-955/20 border border-red-200 dark:border-red-900/50 rounded-xl">
                      Límite de anidación alcanzado (máximo 2 niveles de subqueries).
                    </div>
                  </div>
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
              <label class="text-[10px] text-amber-800 dark:text-amber-400 font-extrabold uppercase tracking-wider font-mono select-none">TABLA...</label>
              <input
                v-model="tableName"
                type="text"
                placeholder="table_name"
                class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-amber-200 dark:border-amber-800/60 focus:border-amber-500 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-amber-500/20 transition-all duration-200 ease-in-out"
              />
            </div>
            
            <!-- Campos Base to retrieve -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-amber-800 dark:text-amber-400 font-extrabold uppercase tracking-wider font-mono select-none">CAMPOS BASE...</label>
              <input
                v-model="scoreBaseFields"
                type="text"
                placeholder="table_name.*"
                class="bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-amber-200 dark:border-amber-800/60 focus:border-amber-500 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-amber-500/20 transition-all duration-200 ease-in-out"
              />
            </div>
            
            <!-- Threshold (HAVING Operator and Value) -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-amber-800 dark:text-amber-400 font-extrabold uppercase tracking-wider font-mono select-none">FILTRAR SCORE (HAVING)...</label>
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
              <label class="text-[10px] text-amber-800 dark:text-amber-400 font-extrabold uppercase tracking-wider font-mono select-none">DIRECCIÓN DE ORDEN...</label>
              <div class="relative">
                <select
                  v-model="scoreOrderDirection"
                  class="w-full bg-slate-50 dark:bg-slate-800/40 text-slate-800 dark:text-slate-200 border border-amber-200 dark:border-amber-800/60 focus:border-amber-500 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none cursor-pointer appearance-none focus:ring-2 focus:ring-amber-550/20 transition-all duration-200 ease-in-out"
                >
                  <option value="DESC">DESC (Descendente)</option>
                  <option value="ASC">ASC (Ascendente)</option>
                </select>
                <div class="absolute inset-y-0 right-4 flex items-center pointer-events-none text-amber-500">
                  <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" d="M19 9l-7 7-7-7" /></svg>
                </div>
              </div>
            </div>
          </div>

          <!-- Case Rules list (Dynamic Rows) -->
          <div class="flex flex-col gap-3">
            <span class="text-xs font-bold text-amber-800 dark:text-amber-400 uppercase tracking-wider font-mono select-none">PUNTUAR SI... (CASE WHEN)</span>
            
            <div class="flex flex-col gap-3">
              <div
                v-for="(rule, idx) in scoreRules"
                :key="idx"
                class="flex flex-col gap-3"
              >
                <!-- Symbolism: + badge representing summation between dynamic rules -->
                <div v-if="idx > 0" class="self-center">
                  <span class="bg-amber-105 border border-amber-200 dark:border-amber-900 text-amber-700 dark:text-amber-400 font-bold font-mono text-sm w-7 h-7 flex items-center justify-center rounded-full select-none shadow-sm shadow-amber-200/30">+</span>
                </div>

                <div class="flex flex-col sm:flex-row items-stretch sm:items-center gap-2 bg-white/80 dark:bg-slate-900/80 p-3 rounded-2xl border border-amber-200/50 dark:border-amber-900/30 shadow-sm">
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
                    <option value="IN">IN</option>
                    <option value="NOT IN">NOT IN</option>
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
                  <div class="flex items-center gap-1.5 bg-slate-55 dark:bg-slate-800/40 border border-slate-200 dark:border-slate-700 rounded-lg px-2.5 py-1.5 w-full sm:w-28 focus-within:ring-2 focus-within:ring-amber-500/20 focus-within:border-amber-500 transition-all duration-200 ease-in-out">
                    <span class="text-[9px] text-amber-800/70 font-bold uppercase tracking-wider font-mono select-none">PESO:</span>
                    <input
                      v-model.number="rule.weight"
                      type="number"
                      placeholder="1"
                      class="bg-transparent text-slate-800 dark:text-slate-200 border-none font-mono text-xs focus:outline-none w-full font-semibold"
                      title="Peso en score"
                    />
                  </div>
                  
                  <!-- Remove Rule Button -->
                  <button
                    v-if="scoreRules.length > 1"
                    @click="removeRow('scoreRule', idx)"
                    class="bg-white dark:bg-slate-800 hover:bg-red-50 dark:hover:bg-red-955/20 border border-slate-200 hover:border-red-200 text-slate-400 hover:text-red-500 p-2.5 rounded-lg transition-all duration-200 ease-in-out cursor-pointer self-end sm:self-center hover:scale-105 active:scale-95"
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
        <div v-else-if="operation === 'CONCAT'" class="flex flex-col gap-6 animate-fade-in text-slate-800 dark:text-slate-205">
          
          <!-- Highlighted Top Row: Excel/Log Meta-data with subtle green/teal background accent -->
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 bg-emerald-50/80 dark:bg-emerald-955/15 border border-emerald-200 dark:border-emerald-900/40 p-5 rounded-2xl shadow-inner">
            <!-- Acción dropdown -->
            <div class="flex flex-col gap-1.5">
              <label class="text-[10px] text-emerald-800 dark:text-emerald-450 font-bold uppercase tracking-wider font-mono select-none">Tipo de Acción (Acción)</label>
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
              <label class="text-[10px] text-emerald-800 dark:text-emerald-450 font-bold uppercase tracking-wider font-mono select-none">Identificador de Hoja (Excel)</label>
              <input
                v-model="concatSheetId"
                type="text"
                placeholder="DATOS1"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-emerald-200 dark:border-emerald-800 focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20 rounded-xl px-4 py-2 text-sm font-mono focus:outline-none font-semibold transition-all duration-200 ease-in-out"
              />
            </div>
          </div>

          <!-- Standard database structural inputs in a grid below -->
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 bg-slate-50 dark:bg-slate-800/40 p-5 rounded-2xl border border-slate-200 dark:border-slate-800">
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
                placeholder="table_name"
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

          <!-- Scope filter WHERE (IDs) with Operator Select -->
          <div class="flex flex-col gap-2 bg-slate-50 dark:bg-slate-800/40 p-5 rounded-2xl border border-slate-200 dark:border-slate-800">
            <label class="text-[10px] text-slate-500 font-bold uppercase tracking-wider font-mono select-none">Filtro de IDs (Scope WHERE)</label>
            <div class="flex gap-2">
              <select
                v-model="concatOperator"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-205 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-xl px-3 py-2.5 text-xs font-mono font-bold w-32 appearance-none text-center cursor-pointer focus:ring-2 focus:ring-indigo-500/20"
              >
                <option value="IN">IN</option>
                <option value="NOT IN">NOT IN</option>
              </select>
              <input
                v-model="concatFilteredIds"
                type="text"
                placeholder="1, 2"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20 rounded-xl px-4 py-2.5 text-sm font-mono focus:outline-none flex-grow transition-all duration-200 ease-in-out"
              />
            </div>
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
          <div class="mt-4 p-3 bg-slate-900/50 dark:bg-slate-950/80 border border-slate-950 dark:border-slate-950/50 rounded-xl">
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
              <li v-if="operation !== 'SCORE' && operation !== 'CONCAT'">Para el operador <code class="text-[10px] text-indigo-300 font-mono">IN</code> o <code class="text-[10px] text-indigo-300 font-mono">NOT IN</code>, separa los valores por coma para auto-formatear a lista, o usa subqueries.</li>
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

  <!-- Nested layout if depth > 0 -->
  <div v-else :class="bgClass" class="rounded-2xl p-4 sm:p-5 flex flex-col gap-5 transition-colors duration-200 shadow-inner">
    
    <!-- SELECT Row inside Nested -->
    <div class="flex flex-col gap-2.5">
      <span class="text-indigo-650 dark:text-indigo-400 font-mono font-extrabold text-xs sm:text-sm tracking-wider uppercase select-none">SELECT</span>
      
      <div class="flex flex-wrap items-center gap-2 bg-white/40 dark:bg-slate-900/40 p-3 rounded-xl border border-slate-200 dark:border-slate-800">
        <div v-for="(field, idx) in selectFields" :key="idx" class="flex items-center gap-1 bg-white dark:bg-slate-900 p-1 rounded-md border border-slate-200 dark:border-slate-800 shadow-sm focus-within:ring-2 focus-within:ring-indigo-500/20 focus-within:border-indigo-500 transition-all duration-200 ease-in-out">
          <input
            v-model="selectFields[idx]"
            type="text"
            placeholder="campo"
            class="bg-transparent text-slate-800 dark:text-slate-200 border-none font-mono text-xs focus:outline-none w-16 sm:w-20 placeholder:text-slate-400 dark:placeholder:text-slate-600 font-medium"
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
          class="bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-indigo-600 dark:text-indigo-400 p-1.5 rounded-md text-xs font-bold flex items-center justify-center cursor-pointer transition-all duration-200 ease-in-out hover:scale-105 active:scale-95 shadow-sm"
        >
          <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5"><path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4" /></svg>
        </button>
      </div>
    </div>

    <!-- FROM Row inside Nested -->
    <div class="flex flex-col gap-3">
      <div class="flex flex-wrap items-center gap-3">
        <span class="text-indigo-650 dark:text-indigo-400 font-mono font-extrabold text-xs sm:text-sm tracking-wider uppercase select-none w-10">FROM</span>
        
        <!-- Table / Subquery Source Toggle inside Nested -->
        <div class="flex items-center bg-white/60 dark:bg-slate-900/60 p-0.5 rounded-lg border border-slate-200 dark:border-slate-800 shadow-inner">
          <button
            type="button"
            @click="state.fromType = 'table'"
            :class="[
              'px-2.5 py-1 rounded-md text-[10px] font-bold font-mono uppercase transition-all duration-200 cursor-pointer',
              state.fromType !== 'subquery'
                ? 'bg-white dark:bg-slate-800 text-indigo-600 dark:text-indigo-400 shadow-sm'
                : 'text-slate-500 hover:text-slate-700 dark:text-slate-400'
            ]"
          >
            Tabla
          </button>
          <button
            type="button"
            @click="state.fromType = 'subquery'; handleFromTypeChange()"
            :class="[
              'px-2.5 py-1 rounded-md text-[10px] font-bold font-mono uppercase transition-all duration-200 cursor-pointer',
              state.fromType === 'subquery'
                ? 'bg-white dark:bg-slate-800 text-indigo-600 dark:text-indigo-400 shadow-sm'
                : 'text-slate-500 hover:text-slate-700 dark:text-slate-400'
            ]"
          >
            Subquery
          </button>
        </div>

        <input
          v-if="state.fromType !== 'subquery'"
          v-model="tableName"
          type="text"
          placeholder="nombre_tabla"
          class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-lg px-3 py-1.5 text-xs font-mono focus:outline-none focus:ring-2 focus:ring-indigo-500/20 transition-all duration-200 w-48 font-semibold"
        />
      </div>

      <!-- FROM Subquery inside Nested -->
      <div v-if="state.fromType === 'subquery'" class="border-l-4 border-indigo-500/50 pl-3 bg-white/30 dark:bg-slate-950/20 rounded-r-xl p-3 flex flex-col gap-3 shadow-inner">
        <div class="flex items-center gap-2">
          <span class="text-[10px] font-bold text-slate-500 dark:text-slate-400 font-mono uppercase">Alias:</span>
          <input
            v-model="state.subqueryFromAlias"
            type="text"
            placeholder="sub_tabla"
            class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-lg px-2 py-1 text-xs font-mono focus:outline-none w-32 font-semibold"
          />
        </div>
        <div v-if="depth < 2">
          <SqlQueryBuilder v-model="state.subqueryFromData" :depth="depth + 1" />
        </div>
        <div v-else class="text-xs text-red-500 font-semibold p-2 bg-red-50/50 dark:bg-red-900/20 border border-red-200 dark:border-red-900/40 rounded-lg">
          Límite de anidación alcanzado.
        </div>
      </div>
    </div>

    <!-- WHERE Row inside Nested -->
    <div class="flex flex-col gap-2.5">
      <span class="text-indigo-650 dark:text-indigo-400 font-mono font-extrabold text-xs sm:text-sm tracking-wider uppercase select-none">WHERE</span>
      
      <div class="flex flex-col gap-3">
        <div v-for="(cond, idx) in whereConditions" :key="idx" class="flex flex-col gap-3">
          
          <div v-if="idx > 0" class="self-center flex items-center">
            <div class="relative flex items-center">
              <select
                v-model="cond.logicalOperator"
                class="bg-indigo-50/70 dark:bg-indigo-950/40 hover:bg-indigo-100 dark:hover:bg-indigo-900/50 text-indigo-700 dark:text-indigo-400 border border-indigo-200 dark:border-indigo-800 rounded-lg px-4 py-0.5 text-[10px] font-mono font-bold tracking-widest cursor-pointer focus:outline-none appearance-none pr-6 text-center select-none"
              >
                <option value="AND">AND</option>
                <option value="OR">OR</option>
              </select>
              <div class="absolute inset-y-0 right-1.5 flex items-center pointer-events-none text-indigo-600">
                <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M19 9l-7 7-7-7" /></svg>
              </div>
            </div>
          </div>
          
          <div class="flex flex-col gap-2 p-2.5 bg-white/40 dark:bg-slate-900/40 rounded-xl border border-slate-200 dark:border-slate-800 shadow-sm">
            <div class="flex flex-wrap sm:flex-nowrap items-center gap-1.5">
              <input
                v-model="cond.column"
                type="text"
                placeholder="columna"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-md px-2 py-1.5 text-xs font-mono focus:outline-none flex-1 min-w-[80px] font-semibold"
              />
              
              <select
                v-model="cond.operator"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-md px-2 py-1.5 text-xs font-mono focus:outline-none cursor-pointer w-20 appearance-none text-center font-semibold"
              >
                <option value="=">=</option>
                <option value="!=">!=</option>
                <option value="LIKE">LIKE</option>
                <option value="IN">IN</option>
                <option value="NOT IN">NOT IN</option>
              </select>
              
              <select
                v-model="cond.valueType"
                @change="handleValueTypeChange(cond)"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-md px-2 py-1.5 text-xs font-mono focus:outline-none cursor-pointer w-24 appearance-none text-center font-semibold"
              >
                <option value="literal">Literal</option>
                <option value="subquery">Subquery</option>
              </select>
              
              <input
                v-if="cond.valueType === 'literal'"
                v-model="cond.value"
                type="text"
                :placeholder="getPlaceholderForValue(cond.operator, cond.valueType)"
                class="bg-white dark:bg-slate-900 text-slate-800 dark:text-slate-200 border border-slate-200 dark:border-slate-700 focus:border-indigo-500 rounded-md px-2 py-1.5 text-xs font-mono focus:outline-none flex-grow min-w-[90px] font-semibold"
              />
              
              <button
                @click="removeRow('where', idx)"
                class="bg-white dark:bg-slate-800 hover:bg-red-50 hover:border-red-200 text-slate-400 hover:text-red-500 p-1.5 rounded-md transition-all duration-200 cursor-pointer active:scale-95"
              >
                <svg class="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
              </button>
            </div>
            
            <!-- Nested WHERE Subquery inside Nested -->
            <div v-if="cond.valueType === 'subquery'" class="mt-1 border-l-4 border-indigo-500/50 pl-3 bg-white/30 dark:bg-slate-950/20 rounded-r-xl p-2.5 shadow-inner">
              <div v-if="depth < 2">
                <SqlQueryBuilder v-model="cond.subqueryData" :depth="depth + 1" />
              </div>
              <div v-else class="text-xs text-red-500 font-semibold p-2 bg-red-50/50 dark:bg-red-950/20 border border-red-200 dark:border-red-900/40 rounded-lg">
                Límite de anidación alcanzado.
              </div>
            </div>
          </div>
        </div>

        <button
          @click="addRow('where')"
          class="bg-white dark:bg-slate-800 hover:bg-slate-50 dark:hover:bg-slate-700 border border-slate-200 dark:border-slate-700 text-indigo-600 dark:text-indigo-400 px-2.5 py-1.5 rounded-lg text-[10px] font-bold self-start cursor-pointer flex items-center gap-1 shadow-sm"
        >
          <svg class="w-3 h-3" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5"><path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4"/></svg>
          Añadir Condición
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
