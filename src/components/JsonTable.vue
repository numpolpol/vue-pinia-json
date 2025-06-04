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
                v-for="arrIdx in getMaxArrayLength(key)"
                :key="arrIdx"
                class="mb-2 flex items-start gap-2 array-row"
              >
                <span class="text-xs text-gray-500 minw40">[{{ arrIdx }}]</span>
                <template v-for="(file, idx) in files" :key="file.name">
                  <template v-if="isPrimitive(jsons[idx][key]?.[arrIdx])">
                    <input
                      v-model="jsons[idx][key][arrIdx]"
                      class="input input-bordered w-full"
                      :type="typeof jsons[idx][key][arrIdx] === 'number' ? 'number' : 'text'"
                    />
                  </template>
                  <template v-else-if="isObject(jsons[idx][key]?.[arrIdx])">
                    <json-table
                      :jsons="jsons.map((j) => j[key][arrIdx] || (j[key][arrIdx] = {}))"
                      :files="files"
                      :level="level + 2"
                    />
                  </template>
                  <template v-else-if="isArray(jsons[idx][key]?.[arrIdx])">
                    <span class="text-gray-400 italic">[nested array]</span>
                  </template>
                  <template v-else>
                    <span class="text-gray-400 italic">[empty]</span>
                  </template>
                </template>
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
  search?: string
  expandSignal?: number
  collapseSignal?: number
}>()

const level = props.level ?? 0

function filterKeys(keys: string[], parentPath = ''): string[] {
  if (!props.search) return keys
  const searchLower = props.search.toLowerCase()
  return keys.filter((key) => {
    const fullPath = parentPath ? `${parentPath}.${key}` : key
    if (fullPath.toLowerCase().includes(searchLower)) return true
    // Check nested object/array for match
    for (const obj of props.jsons) {
      const val = obj[key]
      if (isObject(val)) {
        const nestedKeys = Object.keys(val)
        if (filterKeys(nestedKeys, fullPath).length > 0) return true
      } else if (isArray(val)) {
        for (let i = 0; i < val.length; i++) {
          const arrVal = val[i]
          if (isObject(arrVal)) {
            const nestedKeys = Object.keys(arrVal)
            if (filterKeys(nestedKeys, `${fullPath}[${i}]`).length > 0) return true
          }
        }
      }
    }
    return false
  })
}

const allKeys = computed(() => {
  const keySet = new Set<string>()
  props.jsons.forEach((obj) => {
    if (obj && typeof obj === 'object') {
      Object.keys(obj).forEach((key) => keySet.add(key))
    }
  })
  let keys = Array.from(keySet)
  keys = filterKeys(keys)
  return keys
})

const collapsed = ref<Record<string, boolean>>({})

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
