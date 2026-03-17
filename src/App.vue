<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import * as XLSX from 'xlsx'
import UiButton from '@/components/ui-button.vue'
import UiCard from '@/components/ui-card.vue'
import UiInput from '@/components/ui-input.vue'
import fallbackNamesCsvUrl from '../names.csv?url'

const GITHUB_NAMES_CSV_URL = 'https://raw.githubusercontent.com/JoelHer/wheel-of-doom/refs/heads/main/names.csv'
const WHEEL_SIZE = 440
const WHEEL_CENTER = WHEEL_SIZE / 2
const LABEL_RADIUS = 136

type SheetCell = string | number | boolean | null | undefined

const namesText = ref('')
const baseSpeed = ref(7)
const spinDuration = ref(6)
const spinning = ref(false)
const winner = ref('')
const rotation = ref(0)
const importError = ref('')
const loadingNames = ref(false)
const namesSource = ref('')

const palette = ['#8b5cf6', '#06b6d4', '#22c55e', '#f97316', '#f43f5e', '#a855f7', '#14b8a6', '#84cc16']

const names = computed(() =>
  namesText.value
    .split(/[\n,;]/)
    .map((name) => name.trim())
    .filter(Boolean),
)

const wheelStyle = computed(() => {
  if (!names.value.length) return {}
  const angleStep = 360 / names.value.length
  const slices = names.value
    .map((_, idx) => {
      const start = idx * angleStep
      const end = start + angleStep
      return `${palette[idx % palette.length]} ${start}deg ${end}deg`
    })
    .join(', ')

  return {
    background: `conic-gradient(${slices})`,
    transform: `rotate(${rotation.value}deg)`,
  }
})

function setNames(values: string[]) {
  namesText.value = values.join('\n')
}

function normalizeCell(cell: SheetCell) {
  return String(cell ?? '').trim()
}

function rowsFromSheet(sheet: XLSX.WorkSheet | undefined) {
  if (!sheet) return []

  return XLSX.utils
    .sheet_to_json<SheetCell[]>(sheet, { header: 1, blankrows: false, defval: '' })
    .map((row) => row.map(normalizeCell))
    .filter((row) => row.some(Boolean))
}

function extractNames(rows: string[][]) {
  if (!rows.length) return []

  const [firstRow, ...restRows] = rows
  const normalizedHeaders = firstRow.map((cell) => cell.toLowerCase())
  const nameColumnIndex = normalizedHeaders.findIndex((cell) => ['name', 'names', 'participant', 'player'].includes(cell))

  if (nameColumnIndex >= 0 && restRows.length) {
    return restRows.map((row) => row[nameColumnIndex] ?? '').filter(Boolean)
  }

  const firstColumnValues = rows
    .map((row) => row.find(Boolean) ?? '')
    .filter(Boolean)

  if (normalizedHeaders[0] === 'name' && firstColumnValues.length > 1) {
    return firstColumnValues.slice(1)
  }

  return firstColumnValues
}

function parseCSVContent(content: string) {
  const workbook = XLSX.read(content, { type: 'string' })
  const sheet = workbook.Sheets[workbook.SheetNames[0]]
  return extractNames(rowsFromSheet(sheet))
}

function parseExcelBuffer(data: ArrayBuffer) {
  const workbook = XLSX.read(data, { type: 'array' })
  const sheet = workbook.Sheets[workbook.SheetNames[0]]
  return extractNames(rowsFromSheet(sheet))
}

function chooseWinner(finalRotation: number) {
  const normalized = ((finalRotation % 360) + 360) % 360
  const pointerAngle = (360 - normalized + 270) % 360
  const step = 360 / names.value.length
  const index = Math.floor(pointerAngle / step) % names.value.length
  winner.value = names.value[index]
}

function spin() {
  if (spinning.value || names.value.length < 2) return
  spinning.value = true
  winner.value = ''

  const minTurns = baseSpeed.value * spinDuration.value
  const randomTurns = Math.random() * 3
  const finalRotation = rotation.value + (minTurns + randomTurns) * 360

  const start = performance.now()
  const durationMs = spinDuration.value * 1000
  const initial = rotation.value

  const animate = (now: number) => {
    const elapsed = now - start
    const progress = Math.min(elapsed / durationMs, 1)
    const eased = 1 - Math.pow(1 - progress, 3)
    rotation.value = initial + (finalRotation - initial) * eased

    if (progress < 1) {
      requestAnimationFrame(animate)
      return
    }

    chooseWinner(finalRotation)
    spinning.value = false
  }

  requestAnimationFrame(animate)
}

function getLabelStyle(index: number) {
  const angleStep = 360 / names.value.length
  const angle = (index + 0.5) * angleStep - 90
  const radians = (angle * Math.PI) / 180
  const x = WHEEL_CENTER + Math.cos(radians) * LABEL_RADIUS
  const y = WHEEL_CENTER + Math.sin(radians) * LABEL_RADIUS
  const arcWidth = (Math.PI * LABEL_RADIUS * angleStep) / 180
  const maxWidth = Math.max(76, Math.min(176, arcWidth - 18))

  return {
    left: `${x}px`,
    top: `${y}px`,
    maxWidth: `${maxWidth}px`,
    transform: 'translate(-50%, -50%)',
  }
}

async function loadNamesFromUrl(url: string) {
  const response = await fetch(url, { cache: 'no-store' })

  if (!response.ok) {
    throw new Error(`Request failed with status ${response.status}.`)
  }

  const values = parseCSVContent(await response.text())

  if (!values.length) {
    throw new Error('No names were found in the CSV file.')
  }

  return values
}

async function loadInitialNames() {
  loadingNames.value = true
  importError.value = ''

  const sources = [
    { label: 'GitHub', url: GITHUB_NAMES_CSV_URL },
    { label: 'the bundled fallback file', url: fallbackNamesCsvUrl },
  ]

  let lastError = 'Unable to load names.'

  for (const source of sources) {
    try {
      const values = await loadNamesFromUrl(source.url)
      setNames(values)
      namesSource.value = source.label
      loadingNames.value = false
      return
    } catch (error) {
      lastError = error instanceof Error ? error.message : 'Unable to load names.'
    }
  }

  namesSource.value = ''
  loadingNames.value = false
  importError.value = `Failed to load names automatically. ${lastError}`
}

async function importFile(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return

  importError.value = ''

  try {
    if (file.name.endsWith('.csv')) {
      setNames(parseCSVContent(await file.text()))
      namesSource.value = file.name
    } else if (file.name.endsWith('.xls') || file.name.endsWith('.xlsx')) {
      setNames(parseExcelBuffer(await file.arrayBuffer()))
      namesSource.value = file.name
    } else {
      importError.value = 'Unsupported file type. Use CSV or Excel (.xls/.xlsx).'
    }
  } catch {
    importError.value = 'Failed to import file. Please check the format and try again.'
  }

  input.value = ''
}

onMounted(() => {
  void loadInitialNames()
})
</script>

<template>
  <main class="mx-auto grid max-w-6xl gap-6 p-6 lg:grid-cols-[360px_1fr]">
    <UiCard class="space-y-4">
      <div>
        <h1 class="text-2xl font-bold">Wheel of Names</h1>
        <p class="text-sm text-gray-300">Configure names, speed, duration, then spin.</p>
      </div>

      <div class="space-y-2">
        <label class="text-sm font-medium">Names (one per line)</label>
        <textarea
          v-model="namesText"
          rows="10"
          class="w-full rounded-md border border-border bg-background p-3 text-sm focus:outline-none focus:ring-2 focus:ring-primary"
        />
      </div>

      <div class="space-y-2">
        <label class="text-sm font-medium">Import CSV / Excel</label>
        <UiInput type="file" @change="importFile" />
        <p v-if="importError" class="text-sm text-rose-300">{{ importError }}</p>
      </div>

      <div class="grid gap-4 sm:grid-cols-2">
        <div class="space-y-2">
          <label class="text-sm font-medium">Rotation speed (turns/sec)</label>
          <UiInput :model-value="baseSpeed" type="number" :min="1" :max="30" :step="0.5" @update:model-value="(v) => (baseSpeed = Number(v) || 1)" />
        </div>
        <div class="space-y-2">
          <label class="text-sm font-medium">Spin duration (seconds)</label>
          <UiInput :model-value="spinDuration" type="number" :min="1" :max="20" :step="0.5" @update:model-value="(v) => (spinDuration = Number(v) || 1)" />
        </div>
      </div>

      <UiButton class="w-full" :disabled="spinning || names.length < 2" @click="spin">
        {{ spinning ? 'Spinning…' : 'Spin the wheel' }}
      </UiButton>
      <p v-if="winner" class="rounded-md bg-primary/20 p-3 text-center text-lg font-semibold">
        Winner: {{ winner }}
      </p>
    </UiCard>

    <UiCard class="flex flex-col items-center justify-center gap-6">
      <div class="relative">
        <div class="absolute left-1/2 top-0 z-10 h-0 w-0 -translate-x-1/2 -translate-y-2 border-x-[16px] border-b-[24px] border-x-transparent border-b-primary" />
        <div
          class="relative h-[440px] w-[440px] rounded-full border-4 border-border shadow-2xl"
          :style="wheelStyle"
        >
          <div
            v-for="(name, index) in names"
            :key="name + index"
            class="pointer-events-none absolute text-center text-sm font-medium leading-tight text-white [text-shadow:0_1px_2px_rgba(0,0,0,0.55)]"
            :style="getLabelStyle(index)"
          >
            {{ name }}
          </div>
          <div class="absolute left-1/2 top-1/2 h-8 w-8 -translate-x-1/2 -translate-y-1/2 rounded-full border border-border bg-background" />
        </div>
      </div>
      <p class="text-sm text-gray-400">
        {{ loadingNames ? 'Loading names…' : `${names.length} entries loaded${namesSource ? ` from ${namesSource}` : ''}.` }}
      </p>
    </UiCard>
  </main>
</template>
