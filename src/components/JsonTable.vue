<template>
  <table class="table w-full">
    <thead>
      <tr>
        <th>Key</th>
        <th v-for="file in files" :key="file.name">{{ file.name }}</th>
      </tr>
    </thead>
    <tbody>
      <template v-for="key in allKeys" :key="key">
        <tr :class="{ 'bg-gray-100': level % 2 === 1, 'array-group': isArray(getValue(0, key)) }">
          <td :style="{ paddingLeft: `${level * 20}px` }" class="flex items-center gap-2">
            <template v-if="isPrimitive(getValue(0, key))">
              <button class="btn btn-xs btn-outline mr-1" @click="pasteToField(key)">Paste</button>
            </template>
            <button
              v-if="isObject(getValue(0, key)) || isArray(getValue(0, key))"
              class="btn btn-xs btn-ghost"
              @click="toggleCollapse(key)"
              :aria-label="collapsed[key] ? 'Expand' : 'Collapse'"
            >
              <span v-if="collapsed[key]">▶</span>
              <span v-else>▼</span>
            </button>
            <span>{{ key }}</span>
            <span v-if="isArray(getValue(0, key))" class="ml-2 badge badge-info"
              >{{ getMaxArrayLength(key) }} items</span
            >
          </td>
          <td v-for="(file, idx) in files" :key="file.name">
            <template v-if="isPrimitive(getValue(idx, key))">
              <input
                v-model="jsons[idx][key]"
                class="input input-bordered w-full"
                :type="typeof getValue(idx, key) === 'number' ? 'number' : 'text'"
              />
            </template>
            <template v-else-if="isObject(getValue(idx, key))">
              <span v-if="collapsed[key]" class="text-gray-400 italic">[object]</span>
            </template>
            <template v-else-if="isArray(getValue(idx, key))">
              <span v-if="collapsed[key]" class="text-gray-400 italic">[array]</span>
            </template>
            <template v-else>
              <span class="text-gray-400 italic">[unknown]</span>
            </template>
          </td>
        </tr>
        <tr v-if="!collapsed[key] && isObject(getValue(0, key))">
          <td :colspan="files.length + 1" class="p-0">
            <json-table
              :jsons="jsons.map((j) => j[key] || (j[key] = {}))"
              :files="files"
              :level="level + 1"
            />
          </td>
        </tr>
        <tr v-if="!collapsed[key] && isArray(getValue(0, key))">
          <td :colspan="files.length + 1" class="p-0 array-bg">
            <div class="border rounded p-2 bg-blue-50">
              <div
                v-for="arrIdx of Array(getMaxArrayLength(key)).keys()"
                :key="arrIdx"
                class="mb-2 flex items-start gap-2 array-row"
              >
                <button
                  v-if="jsons.some((j) => isObject(j[key]?.[arrIdx]))"
                  class="btn btn-xs btn-ghost mr-1"
                  @click="
                    collapsed[getArrayItemCollapseKey(key, arrIdx)] =
                      !collapsed[getArrayItemCollapseKey(key, arrIdx)]
                  "
                  :aria-label="
                    collapsed[getArrayItemCollapseKey(key, arrIdx)] ? 'Expand' : 'Collapse'
                  "
                >
                  <span v-if="collapsed[getArrayItemCollapseKey(key, arrIdx)]">▶</span>
                  <span v-else>▼</span>
                </button>
                <span class="text-xs text-gray-500 minw40">[{{ arrIdx }}]</span>
                <!-- Paste button for array of string/number/boolean -->
                <button
                  v-if="
                    jsons.every(
                      (j) => isPrimitive(j[key]?.[arrIdx]) || j[key]?.[arrIdx] === undefined,
                    )
                  "
                  class="btn btn-xs btn-outline mr-1"
                  @click="pasteToArrayField(key, arrIdx)"
                >
                  Paste
                </button>
                <template v-for="(file, idx) in files" :key="file.name">
                  <template v-if="isPrimitive(jsons[idx][key]?.[arrIdx])">
                    <input
                      v-model="jsons[idx][key][arrIdx]"
                      class="input input-bordered w-full"
                      :type="typeof jsons[idx][key][arrIdx] === 'number' ? 'number' : 'text'"
                    />
                  </template>
                  <template v-else-if="isObject(jsons[idx][key]?.[arrIdx])">
                    <span
                      v-if="collapsed[getArrayItemCollapseKey(key, arrIdx)]"
                      class="text-gray-400 italic"
                      >[object]</span
                    >
                  </template>
                  <template v-else-if="isArray(jsons[idx][key]?.[arrIdx])">
                    <span class="text-gray-400 italic">[nested array]</span>
                  </template>
                  <template v-else>
                    <span class="text-gray-400 italic">[empty]</span>
                  </template>
                </template>
                <!-- Nested object table for this array item -->
                <div
                  v-if="
                    jsons.some((j) => isObject(j[key]?.[arrIdx])) &&
                    !collapsed[getArrayItemCollapseKey(key, arrIdx)]
                  "
                  style="width: 100%"
                >
                  <json-table
                    :jsons="jsons.map((j) => j[key][arrIdx] || (j[key][arrIdx] = {}))"
                    :files="files"
                    :level="level + 2"
                  />
                </div>
              </div>
            </div>
          </td>
        </tr>
      </template>
    </tbody>
  </table>
</template>

<script lang="ts" setup>
import { computed, defineProps, ref, watch } from 'vue'

const props = defineProps<{
  jsons: Record<string, any>[]
  files: File[]
  level?: number
  expandSignal?: number
  collapseSignal?: number
}>()

const level = props.level ?? 0

function deepEqual(a: any, b: any): boolean {
  if (a === b) return true
  if (typeof a !== typeof b) return false
  if (typeof a !== 'object' || a === null || b === null) return false
  if (Array.isArray(a) && Array.isArray(b)) {
    if (a.length !== b.length) return false
    for (let i = 0; i < a.length; i++) {
      if (!deepEqual(a[i], b[i])) return false
    }
    return true
  }
  if (!Array.isArray(a) && !Array.isArray(b)) {
    const aKeys = Object.keys(a)
    const bKeys = Object.keys(b)
    if (aKeys.length !== bKeys.length) return false
    for (const k of aKeys) {
      if (!deepEqual(a[k], b[k])) return false
    }
    return true
  }
  return false
}

function isAllValuesDeepEqual(key: string) {
  const first = props.jsons[0]?.[key]
  return props.jsons.every((j) => deepEqual(j?.[key], first))
}

const allKeys = computed(() => {
  // Collect all keys at this level from all jsons, but only include each key once
  const keySet = new Set<string>()
  props.jsons.forEach((obj) => {
    if (obj && typeof obj === 'object') {
      Object.keys(obj).forEach((key) => keySet.add(key))
    }
  })
  let keys = Array.from(keySet)
  // Ensure only unique keys at this level
  keys = [...new Set(keys)]
  return keys
})

const collapsed = ref<Record<string, boolean>>({})

function getArrayItemCollapseKey(key: string, arrIdx: number) {
  // Unique key for each array item collapse state
  return `${key}__${arrIdx}`
}

watch(
  () => props.expandSignal,
  () => {
    allKeys.value.forEach((key) => (collapsed.value[key] = false))
  },
)
watch(
  () => props.collapseSignal,
  () => {
    allKeys.value.forEach((key) => (collapsed.value[key] = true))
  },
)

function toggleCollapse(key: string) {
  collapsed.value[key] = !collapsed.value[key]
}
function getMaxArrayLength(key: string) {
  // Find the max length of this array key across all files
  return Math.max(...props.jsons.map((j) => (Array.isArray(j[key]) ? j[key].length : 0)))
}

function isPrimitive(val: any) {
  return typeof val === 'string' || typeof val === 'number' || typeof val === 'boolean'
}
function isObject(val: any) {
  return val && typeof val === 'object' && !Array.isArray(val)
}
function isArray(val: any) {
  return Array.isArray(val)
}
function getValue(idx: number, key: string) {
  return props.jsons[idx]?.[key]
}
async function pasteToField(key: string) {
  let text = ''
  try {
    text = await navigator.clipboard.readText()
  } catch {
    text = window.prompt('Paste your text here:') || ''
  }
  if (!text) return
  // Split by tab, double space, or newline, fallback to single space
  let values = text.split(/\t|\s{2,}|\n/).filter(Boolean)
  if (values.length < props.files.length) {
    values = text.split(/\s+/).filter(Boolean)
  }
  for (let f = 0; f < props.files.length; f++) {
    if (props.jsons[f] && typeof props.jsons[f][key] !== 'object') {
      props.jsons[f][key] = values[f] ?? ''
    }
  }
}
async function pasteToArrayField(key: string, arrIdx: number) {
  let text = ''
  try {
    text = await navigator.clipboard.readText()
  } catch {
    text = window.prompt('Paste your text here:') || ''
  }
  if (!text) return
  let values = text.split(/\t|\s{2,}|\n/).filter(Boolean)
  if (values.length < props.files.length) {
    values = text.split(/\s+/).filter(Boolean)
  }
  for (let f = 0; f < props.files.length; f++) {
    if (
      props.jsons[f] &&
      (isPrimitive(props.jsons[f][key]?.[arrIdx]) || props.jsons[f][key]?.[arrIdx] === undefined)
    ) {
      if (!Array.isArray(props.jsons[f][key])) props.jsons[f][key] = []
      props.jsons[f][key][arrIdx] = values[f] ?? ''
    }
  }
}
</script>

<style scoped>
td {
  vertical-align: top;
}
.btn-ghost {
  min-width: 1.5rem;
  height: 1.5rem;
  padding: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 1rem;
  background: none;
  border: none;
  cursor: pointer;
}
.bg-gray-100 {
  background-color: #222;
  color: #fff;
}
.array-bg {
  background-color: #e5e7eb;
  color: #222;
}
.array-group > td {
  border-top: 2px solid #bbb;
}
.array-row {
  background: #f3f4f6;
  color: #222;
  border-radius: 0.25rem;
  padding: 0.25rem 0.5rem;
}
.minw40 {
  min-width: 40px;
  display: inline-block;
}
.badge-info {
  background: #fff;
  color: #6b7280;
  border-radius: 0.5rem;
  padding: 0.1rem 0.5rem;
  font-size: 0.75rem;
  margin-left: 0.5rem;
  border: 1px solid #bbb;
}
.input-bordered {
  background: #fff;
  color: #222;
  border: 1px solid #bbb;
}
</style>
