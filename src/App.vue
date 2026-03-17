<script setup lang="ts">
import { computed, ref } from 'vue'
import * as XLSX from 'xlsx'
import UiButton from '@/components/ui-button.vue'
import UiCard from '@/components/ui-card.vue'
import UiInput from '@/components/ui-input.vue'

const namesText = ref('Alice\nBob\nCharlie\nDiana\nEve\nFrank')
const baseSpeed = ref(7)
const spinDuration = ref(6)
const spinning = ref(false)
const winner = ref('')
const rotation = ref(0)
const importError = ref('')

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

function parseCSV(content: string) {
  const rows = content
    .split(/\r?\n/)
    .map((line) => line.trim())
    .filter(Boolean)

  const values = rows
    .flatMap((line) => line.split(','))
    .map((entry) => entry.replace(/^"|"$/g, '').trim())
    .filter(Boolean)

  namesText.value = values.join('\n')
}

function parseExcel(data: ArrayBuffer) {
  const workbook = XLSX.read(data, { type: 'array' })
  const sheet = workbook.Sheets[workbook.SheetNames[0]]
  const rows = XLSX.utils.sheet_to_json<Array<string | number>>(sheet, { header: 1 })
  const values = rows.flat().map((entry) => String(entry ?? '').trim())
  namesText.value = values.filter(Boolean).join('\n')
}

async function importFile(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return

  importError.value = ''

  try {
    if (file.name.endsWith('.csv')) {
      parseCSV(await file.text())
    } else if (file.name.endsWith('.xls') || file.name.endsWith('.xlsx')) {
      parseExcel(await file.arrayBuffer())
    } else {
      importError.value = 'Unsupported file type. Use CSV or Excel (.xls/.xlsx).'
    }
  } catch {
    importError.value = 'Failed to import file. Please check the format and try again.'
  }

  input.value = ''
}
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
          <UiInput :model-value="baseSpeed" type="number" min="1" max="30" step="0.5" @update:model-value="(v) => (baseSpeed = Number(v) || 1)" />
        </div>
        <div class="space-y-2">
          <label class="text-sm font-medium">Spin duration (seconds)</label>
          <UiInput :model-value="spinDuration" type="number" min="1" max="20" step="0.5" @update:model-value="(v) => (spinDuration = Number(v) || 1)" />
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
            class="pointer-events-none absolute left-1/2 top-1/2 origin-left text-sm font-medium text-white"
            :style="{
              transform: `rotate(${(index + 0.5) * (360 / names.length)}deg) translateX(70px)`,
            }"
          >
            {{ name }}
          </div>
          <div class="absolute left-1/2 top-1/2 h-8 w-8 -translate-x-1/2 -translate-y-1/2 rounded-full border border-border bg-background" />
        </div>
      </div>
      <p class="text-sm text-gray-400">{{ names.length }} entries loaded.</p>
    </UiCard>
  </main>
</template>
