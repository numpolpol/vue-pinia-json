<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Multi-language JSON Editor</h1>
    <div v-if="step === 1">
      <input type="file" multiple accept=".json" @change="onFilesSelected" />
      <div class="mt-4">
        <button class="btn btn-primary" :disabled="files.length === 0" @click="step = 2">
          Ready
        </button>
      </div>
    </div>
    <div v-else-if="step === 2">
      <div class="flex items-center gap-2 mb-4">
        <input
          v-model="search"
          type="text"
          placeholder="Search key..."
          class="input input-bordered w-64"
        />
        <button class="btn btn-outline" @click="search = ''">Clear</button>
        <button class="btn btn-outline" @click="expandAll">Expand All</button>
        <button class="btn btn-outline" @click="collapseAll">Collapse All</button>
        <button class="btn btn-info" @click="pasteToFields">Paste</button>
      </div>
      <div class="overflow-x-auto">
        <json-table
          :jsons="jsons"
          :files="files"
          :search="search"
          :expand-signal="expandSignal"
          :collapse-signal="collapseSignal"
        />
      </div>
      <div class="mt-4 flex gap-2">
        <button class="btn btn-success" @click="exportAll">Export All</button>
        <button class="btn btn-secondary" @click="reset">Back</button>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref } from 'vue'
import JsonTable from '@/components/JsonTable.vue'

const files = ref<File[]>([])
const jsons = ref<Record<string, any>[]>([])
const step = ref(1)
const search = ref('')

const expandSignal = ref(0)
const collapseSignal = ref(0)

function onFilesSelected(e: Event) {
  const input = e.target as HTMLInputElement
  if (!input.files) return
  files.value = Array.from(input.files)
  jsons.value = []
  const readers = files.value.map(
    (file) =>
      new Promise<void>((resolve) => {
        const reader = new FileReader()
        reader.onload = () => {
          try {
            jsons.value.push(JSON.parse(reader.result as string))
          } catch {
            jsons.value.push({})
          }
          resolve()
        }
        reader.readAsText(file)
      }),
  )
  Promise.all(readers).then(() => {})
}

function exportAll() {
  files.value.forEach((file, idx) => {
    const blob = new Blob([JSON.stringify(jsons.value[idx], null, 2)], { type: 'application/json' })
    const a = document.createElement('a')
    a.href = URL.createObjectURL(blob)
    a.download = file.name
    a.click()
    URL.revokeObjectURL(a.href)
  })
}

function reset() {
  files.value = []
  jsons.value = []
  step.value = 1
}

function expandAll() {
  expandSignal.value++
}
function collapseAll() {
  collapseSignal.value++
}

const pasteBuffer = ref('')

async function pasteToFields() {
  // Try to read clipboard
  let text = ''
  try {
    text = await navigator.clipboard.readText()
  } catch {
    // fallback prompt
    text = window.prompt('Paste your text here:') || ''
  }
  if (!text) return
  // Split by tab or whitespace, fallback to space
  let values = text.split(/\t|\s{2,}|\n/).filter(Boolean)
  if (values.length < files.value.length) {
    // fallback: split by any whitespace
    values = text.split(/\s+/).filter(Boolean)
  }
  // Find all visible input fields (by key order in table)
  const flatKeys: { key: string; prefix: string }[] = []
  function collectKeys(obj: any, prefix = '') {
    if (typeof obj !== 'object' || obj === null) return
    Object.keys(obj).forEach((key: string) => {
      if (typeof obj[key] === 'string' || typeof obj[key] === 'number') {
        flatKeys.push({ key, prefix })
      } else if (typeof obj[key] === 'object') {
        collectKeys(obj[key], prefix ? `${prefix}.${key}` : key)
      }
    })
  }
  if (jsons.value.length) collectKeys(jsons.value[0])
  // Paste only to visible fields in order
  for (let i = 0; i < Math.min(flatKeys.length, values.length); i++) {
    const { key, prefix } = flatKeys[i]
    for (let f = 0; f < files.value.length; f++) {
      let target = jsons.value[f]
      if (prefix) {
        prefix.split('.').forEach((k: string) => {
          if (!target[k]) target[k] = {}
          target = target[k]
        })
      }
      target[key] = values[i]
    }
  }
}
</script>

<style scoped>
.container {
  max-width: 900px;
}
</style>
