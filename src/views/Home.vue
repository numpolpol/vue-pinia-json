<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Multi-language JSON Editor</h1>
    <div v-if="step === 1">
      <div class="mb-4 p-4 bg-blue-50 border border-blue-200 rounded text-blue-900">
        <b>วิธีใช้งาน:</b><br />
        1. คลิก <b>Choose Files</b> เพื่อเลือกไฟล์ JSON หลายภาษา (เช่น th.json, en.json)<br />
        2. ตรวจสอบว่าไฟล์ที่เลือกเป็น JSON format ที่ถูกต้อง<br />
        3. คลิก <b>Ready</b> เพื่อเข้าสู่หน้าตารางแก้ไข<br />
        <span class="text-red-600"
          >* หากไฟล์ JSON format ผิด จะมีข้อความแจ้งเตือนและไม่นำเข้าไฟล์นั้น</span
        >
      </div>
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
        <button class="btn btn-outline" @click="showInfo = true" title="Info">ℹ️</button>
      </div>
      <div
        v-if="showInfo"
        class="mb-4 p-4 bg-blue-50 border border-blue-200 rounded text-blue-900 relative"
      >
        <button class="absolute top-2 right-2 text-lg" @click="showInfo = false">✕</button>
        <b>คำแนะนำการใช้งาน Paste:</b><br />
        <b>Paste ข้อมูลจาก Google Sheet หรือ Excel:</b> สามารถคัดลอกข้อมูลจากตาราง (เลือก cell
        หลายภาษา → กด Ctrl+C หรือ Cmd+C) แล้วคลิกปุ่ม
        <b>Paste</b> ในแต่ละฟิลด์ของตารางเพื่อวางค่าหลายภาษาได้ทันที (รองรับ tab, space, หรือ
        newline)
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
const showInfo = ref(false)

function onFilesSelected(e: Event) {
  const input = e.target as HTMLInputElement
  if (!input.files) return
  files.value = Array.from(input.files)
  jsons.value = []
  let errorFiles: string[] = []
  const readers = files.value.map(
    (file) =>
      new Promise<void>((resolve) => {
        const reader = new FileReader()
        reader.onload = () => {
          try {
            jsons.value.push(JSON.parse(reader.result as string))
          } catch {
            jsons.value.push({})
            errorFiles.push(file.name)
          }
          resolve()
        }
        reader.readAsText(file)
      }),
  )
  Promise.all(readers).then(() => {
    if (errorFiles.length > 0) {
      window.alert(
        `ไฟล์ต่อไปนี้ไม่ใช่ JSON format ที่ถูกต้อง และจะไม่ถูกนำเข้า:\n` + errorFiles.join('\n'),
      )
      // ลบไฟล์ที่ผิดออกจาก files และ jsons
      const validFiles: File[] = []
      const validJsons: any[] = []
      files.value.forEach((file, idx) => {
        if (!errorFiles.includes(file.name)) {
          validFiles.push(file)
          validJsons.push(jsons.value[idx])
        }
      })
      files.value = validFiles
      jsons.value = validJsons
    }
  })
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
