<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, type CSSProperties } from 'vue'
import * as XLSX from 'xlsx'
import UiButton from '@/components/ui-button.vue'
import UiCard from '@/components/ui-card.vue'
import UiInput from '@/components/ui-input.vue'
import fallbackNamesCsvUrl from '../names.csv?url'

const GITHUB_NAMES_CSV_URL = 'https://raw.githubusercontent.com/JoelHer/wheel-of-doom/main/names.csv'
const WHEEL_SIZE = 440
const WHEEL_CENTER = WHEEL_SIZE / 2
const LABEL_RADIUS = 136
const INVALID_PROTOCOL_COUNT = -1

type SheetCell = string | number | boolean | null | undefined
type Participant = {
  name: string
  protocolCount: number
}
type WheelEntry = Participant & {
  weight: number
  chance: number
  startAngle: number
  endAngle: number
  midAngle: number
}
type CelebrationParticle = {
  id: number
  originX: string
  originY: string
  travelX: string
  travelY: string
  rotationStart: string
  rotationEnd: string
  size: string
  height: string
  color: string
  duration: string
  delay: string
  opacity: string
  borderRadius: string
  blur: string
  shape: 'dot' | 'shard'
}

const participants = ref<Participant[]>([])
const namesText = ref('')
const baseSpeed = ref(7)
const spinDuration = ref(6)
const spinning = ref(false)
const winner = ref('')
const winnerDetails = ref<WheelEntry | null>(null)
const showWinnerPopup = ref(false)
const editingIndex = ref<number | null>(null)
const editName = ref('')
const editProtocolCount = ref('0')
const editError = ref('')
const rotation = ref(0)
const importError = ref('')
const loadingNames = ref(false)
const namesSource = ref('')
const celebrationParticles = ref<CelebrationParticle[]>([])

const palette = ['#2a0505', '#511010', '#7c1212', '#a01807', '#8d3b08', '#4d0707', '#2f1406', '#bf4d14']
const celebrationPalette = [...palette, '#f6d7b0', '#ffb347', '#f97316', '#ffd166']
let nextParticleId = 0
let celebrationRunId = 0
const celebrationTimers: number[] = []

const wheelEntries = computed<WheelEntry[]>(() => {
  const parsedEntries = participants.value.map((participant) => {
    const protocolCount = participant.protocolCount
    const weight = 1 / (protocolCount + 1)

    return {
      name: participant.name,
      protocolCount,
      weight,
    }
  })

  const totalWeight = parsedEntries.reduce((sum, entry) => sum + entry.weight, 0)
  let currentAngle = 0

  return parsedEntries.map((entry, index) => {
    const sliceAngle = totalWeight > 0 ? (entry.weight / totalWeight) * 360 : 0
    const startAngle = currentAngle
    const endAngle = index === parsedEntries.length - 1 ? 360 : currentAngle + sliceAngle
    currentAngle = endAngle

    return {
      ...entry,
      chance: totalWeight > 0 ? entry.weight / totalWeight : 0,
      startAngle,
      endAngle,
      midAngle: startAngle + (endAngle - startAngle) / 2,
    }
  })
})

const winnerChanceLabel = computed(() =>
  winnerDetails.value ? `Odds of doom: ${(winnerDetails.value.chance * 100).toFixed(1)}% this spin` : '',
)

const wheelStyle = computed(() => {
  if (!wheelEntries.value.length) return {}

  const slices = wheelEntries.value
    .map((entry, idx) => {
      const start = entry.startAngle
      const end = entry.endAngle
      return `${palette[idx % palette.length]} ${start}deg ${end}deg`
    })
    .join(', ')

  return {
    background: `conic-gradient(${slices})`,
    transform: `rotate(${rotation.value}deg)`,
  }
})

function randomBetween(min: number, max: number) {
  return min + Math.random() * (max - min)
}

function syncNamesTextFromParticipants() {
  namesText.value = participants.value.map((participant) => participant.name).join('\n')
}

function setParticipants(nextParticipants: Participant[], options?: { syncText?: boolean }) {
  const cleanedParticipants = nextParticipants
    .map((participant) => ({
      name: participant.name.trim(),
      protocolCount: Math.max(0, participant.protocolCount),
    }))
    .filter((participant) => participant.name)

  participants.value = cleanedParticipants

  if (options?.syncText !== false) {
    syncNamesTextFromParticipants()
  }
}

function syncParticipantsFromText(value: string) {
  const nextNames = value
    .split(/[\n,;]/)
    .map((name) => name.trim())
    .filter(Boolean)

  const preservedIndices = new Set<number>()
  const nextParticipants = nextNames.map((name, index) => {
    const currentParticipant = participants.value[index]

    if (currentParticipant && getNameKey(currentParticipant.name) === getNameKey(name)) {
      preservedIndices.add(index)
      return {
        name,
        protocolCount: currentParticipant.protocolCount,
      }
    }

    return {
      name,
      protocolCount: INVALID_PROTOCOL_COUNT,
    }
  })

  const remainingMatches = new Map<string, number[]>()

  for (const [index, participant] of participants.value.entries()) {
    if (preservedIndices.has(index)) continue

    const key = getNameKey(participant.name)
    const queue = remainingMatches.get(key) ?? []
    queue.push(index)
    remainingMatches.set(key, queue)
  }

  for (const participant of nextParticipants) {
    if (participant.protocolCount !== INVALID_PROTOCOL_COUNT) continue

    const key = getNameKey(participant.name)
    const matchIndex = remainingMatches.get(key)?.shift()
    participant.protocolCount = matchIndex !== undefined ? participants.value[matchIndex].protocolCount : 0
  }

  setParticipants(nextParticipants, { syncText: false })
}

function handleNamesTextInput(event: Event) {
  const input = event.target as HTMLTextAreaElement
  namesText.value = input.value
  syncParticipantsFromText(input.value)
}

function getNameKey(name: string) {
  return name.trim().toLowerCase()
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

function parseProtocolCount(value: string) {
  const normalizedValue = value.trim().toLowerCase()

  if (!normalizedValue) return 0

  const numericValue = Number(normalizedValue)
  if (Number.isFinite(numericValue) && numericValue >= 0) {
    return numericValue
  }

  if (['yes', 'true', 'y', 'ja'].includes(normalizedValue)) {
    return 1
  }

  if (['no', 'false', 'n', 'nein'].includes(normalizedValue)) {
    return 0
  }

  return 0
}

function extractParticipants(rows: string[][]) {
  if (!rows.length) return [] as Participant[]

  const [firstRow, ...restRows] = rows
  const normalizedHeaders = firstRow.map((cell) => cell.toLowerCase())
  const nameColumnIndex = normalizedHeaders.findIndex((cell) => ['name', 'names', 'participant', 'player'].includes(cell))
  const protocolColumnIndex = normalizedHeaders.findIndex((cell) =>
    ['hasdoneprotocol', 'protocolcount', 'protocols', 'count'].includes(cell.replace(/\s+/g, '')),
  )

  if (nameColumnIndex >= 0 && restRows.length) {
    return restRows
      .map((row) => ({
        name: row[nameColumnIndex] ?? '',
        protocolCount: protocolColumnIndex >= 0 ? parseProtocolCount(row[protocolColumnIndex] ?? '') : 0,
      }))
      .filter((participant) => participant.name)
  }

  return rows
    .map((row) => {
      const firstValue = row.find(Boolean) ?? ''
      return {
        name: firstValue,
        protocolCount: 0,
      }
    })
    .filter((participant) => participant.name)
}

function parseCSVContent(content: string) {
  const workbook = XLSX.read(content, { type: 'string' })
  const sheet = workbook.Sheets[workbook.SheetNames[0]]
  return extractParticipants(rowsFromSheet(sheet))
}

function parseExcelBuffer(data: ArrayBuffer) {
  const workbook = XLSX.read(data, { type: 'array' })
  const sheet = workbook.Sheets[workbook.SheetNames[0]]
  return extractParticipants(rowsFromSheet(sheet))
}

function clearCelebrationTimers() {
  while (celebrationTimers.length) {
    const timerId = celebrationTimers.pop()

    if (typeof timerId === 'number') {
      window.clearTimeout(timerId)
    }
  }
}

function clearCelebration() {
  clearCelebrationTimers()
  celebrationParticles.value = []
}

function createCelebrationParticles(count: number, originX: number, originY: number) {
  return Array.from({ length: count }, () => {
    const angle = randomBetween(0, Math.PI * 2)
    const distance = randomBetween(180, 760)
    const particleWidth = randomBetween(5, 14)
    const isShard = Math.random() > 0.42
    const travelX = Math.cos(angle) * distance + randomBetween(-80, 80)
    const travelY = Math.sin(angle) * distance + randomBetween(-120, 220)

    return {
      id: nextParticleId++,
      originX: `${originX}%`,
      originY: `${originY}%`,
      travelX: `${travelX.toFixed(0)}px`,
      travelY: `${travelY.toFixed(0)}px`,
      rotationStart: `${randomBetween(0, 180).toFixed(0)}deg`,
      rotationEnd: `${randomBetween(-1080, 1080).toFixed(0)}deg`,
      size: `${particleWidth.toFixed(0)}px`,
      height: `${(isShard ? particleWidth * randomBetween(1.8, 3.4) : particleWidth).toFixed(0)}px`,
      color: celebrationPalette[Math.floor(Math.random() * celebrationPalette.length)],
      duration: `${randomBetween(1800, 3600).toFixed(0)}ms`,
      delay: `${randomBetween(0, 220).toFixed(0)}ms`,
      opacity: randomBetween(0.68, 1).toFixed(2),
      borderRadius: isShard ? `${randomBetween(2, 6).toFixed(0)}px` : '999px',
      blur: `${randomBetween(0, 0.8).toFixed(2)}px`,
      shape: isShard ? 'shard' : 'dot',
    } satisfies CelebrationParticle
  })
}

function triggerCelebration() {
  clearCelebration()
  const runId = ++celebrationRunId
  const waves = [
    { count: 220, originX: 50, originY: 50, delay: 0 },
    { count: 120, originX: 28, originY: 56, delay: 90 },
    { count: 120, originX: 72, originY: 56, delay: 150 },
    { count: 100, originX: 50, originY: 28, delay: 220 },
  ]

  for (const wave of waves) {
    celebrationTimers.push(
      window.setTimeout(() => {
        if (runId !== celebrationRunId) return
        celebrationParticles.value = [
          ...celebrationParticles.value,
          ...createCelebrationParticles(wave.count, wave.originX, wave.originY),
        ]
      }, wave.delay),
    )
  }

  celebrationTimers.push(
    window.setTimeout(() => {
      if (runId !== celebrationRunId) return
      celebrationParticles.value = []
    }, 4600),
  )
}

function closeWinnerPopup() {
  showWinnerPopup.value = false
}

function closeEntryEditor() {
  editingIndex.value = null
  editName.value = ''
  editProtocolCount.value = '0'
  editError.value = ''
}

function clearWinnerState() {
  winner.value = ''
  winnerDetails.value = null
  showWinnerPopup.value = false
  clearCelebration()
}

function openEntryEditor(index: number) {
  if (spinning.value) return

  const participant = participants.value[index]
  if (!participant) return

  editingIndex.value = index
  editName.value = participant.name
  editProtocolCount.value = String(participant.protocolCount)
  editError.value = ''
}

function saveEntryEdits() {
  if (editingIndex.value === null) return

  const nextName = editName.value.trim()
  if (!nextName) {
    editError.value = 'The condemned must have a name.'
    return
  }

  const numericCount = Number(editProtocolCount.value)
  if (!Number.isFinite(numericCount) || numericCount < 0) {
    editError.value = 'Times protocol written must be a non-negative number.'
    return
  }

  const nextParticipants = [...participants.value]
  const currentParticipant = nextParticipants[editingIndex.value]

  if (!currentParticipant) {
    closeEntryEditor()
    return
  }

  nextParticipants[editingIndex.value] = {
    name: nextName,
    protocolCount: Math.round(numericCount),
  }

  setParticipants(nextParticipants)
  clearWinnerState()
  closeEntryEditor()
}

function deleteEntry() {
  if (editingIndex.value === null) return

  setParticipants(participants.value.filter((_, index) => index !== editingIndex.value))
  clearWinnerState()
  closeEntryEditor()
}

function spinAgain() {
  closeWinnerPopup()
  spin()
}

function getParticleStyle(particle: CelebrationParticle) {
  return {
    left: particle.originX,
    top: particle.originY,
    width: particle.size,
    height: particle.height,
    background: particle.color,
    borderRadius: particle.borderRadius,
    filter: `blur(${particle.blur})`,
    boxShadow: `0 0 18px ${particle.color}`,
    '--particle-x': particle.travelX,
    '--particle-y': particle.travelY,
    '--particle-rotation-start': particle.rotationStart,
    '--particle-rotation-end': particle.rotationEnd,
    '--particle-duration': particle.duration,
    '--particle-delay': particle.delay,
    '--particle-opacity': particle.opacity,
  } as CSSProperties
}

function chooseWinner(finalRotation: number) {
  const normalized = ((finalRotation % 360) + 360) % 360
  const pointerAngle = (360 - normalized) % 360
  const winningEntry =
    wheelEntries.value.find((entry) => pointerAngle >= entry.startAngle && pointerAngle < entry.endAngle) ??
    wheelEntries.value[wheelEntries.value.length - 1]

  if (!winningEntry) {
    winner.value = ''
    winnerDetails.value = null
    showWinnerPopup.value = false
    return
  }

  winner.value = winningEntry.name
  winnerDetails.value = { ...winningEntry }
  showWinnerPopup.value = true
  triggerCelebration()
}

function spin() {
  if (spinning.value || wheelEntries.value.length < 2) return
  spinning.value = true
  closeEntryEditor()
  clearWinnerState()

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
  const entry = wheelEntries.value[index]
  if (!entry) return {}

  const angle = entry.midAngle - 90
  const radians = (angle * Math.PI) / 180
  const x = WHEEL_CENTER + Math.cos(radians) * LABEL_RADIUS
  const y = WHEEL_CENTER + Math.sin(radians) * LABEL_RADIUS
  const arcWidth = (Math.PI * LABEL_RADIUS * (entry.endAngle - entry.startAngle)) / 180
  const maxWidth = Math.max(56, Math.min(176, arcWidth - 10))
  const fontScale = Math.max(0.56, Math.min(0.88, 0.56 + (entry.endAngle - entry.startAngle) / 180))

  return {
    left: `${(x / WHEEL_SIZE) * 100}%`,
    top: `${(y / WHEEL_SIZE) * 100}%`,
    maxWidth: `${(maxWidth / WHEEL_SIZE) * 100}%`,
    '--entry-font-scale': `${fontScale}rem`,
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
      setParticipants(values)
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
      setParticipants(parseCSVContent(await file.text()))
      namesSource.value = file.name
    } else if (file.name.endsWith('.xls') || file.name.endsWith('.xlsx')) {
      setParticipants(parseExcelBuffer(await file.arrayBuffer()))
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

onUnmounted(() => {
  clearCelebration()
})
</script>

<template>
  <main class="doom-layout mx-auto grid max-w-7xl gap-6 px-4 py-6 sm:px-6 lg:grid-cols-[390px_1fr] lg:gap-8 lg:px-8 lg:py-10">
    <UiCard class="doom-panel doom-panel--controls space-y-6">
      <div class="space-y-4">
        <p class="doom-eyebrow">Cursed Selection Engine</p>
        <div class="space-y-3">
          <h1 class="doom-title">Wheel of Doom</h1>
          <p class="doom-copy">
            Feed the ritual with names and protocol counts. The more often someone has written the protocol,
            the smaller their slice when the wheel hungers for its next victim.
          </p>
        </div>
      </div>

      <div class="doom-stat-grid">
        <div class="doom-stat">
          <span class="doom-stat-label">Souls Loaded</span>
          <strong class="doom-stat-value">{{ participants.length }}</strong>
        </div>
        <div class="doom-stat">
          <span class="doom-stat-label">Summoned From</span>
          <strong class="doom-stat-value doom-stat-value--small">
            {{ loadingNames ? 'The void…' : namesSource || 'Manual ritual' }}
          </strong>
        </div>
      </div>

      <div class="space-y-2">
        <label class="doom-label">Names of the condemned</label>
        <textarea
          :value="namesText"
          rows="10"
          class="doom-textarea"
          @input="handleNamesTextInput"
        />
      </div>

      <div class="space-y-2">
        <label class="doom-label">Offerings from CSV / Excel</label>
        <UiInput class="doom-input" type="file" @change="importFile" />
        <p class="doom-hint">Expected format: `Name,HasDoneProtocol`.</p>
        <p v-if="importError" class="text-sm text-rose-300">{{ importError }}</p>
      </div>

      <div class="grid gap-4 sm:grid-cols-2">
        <div class="space-y-2">
          <label class="doom-label">Ritual speed</label>
          <UiInput class="doom-input" :model-value="baseSpeed" type="number" :min="1" :max="30" :step="0.5" @update:model-value="(v) => (baseSpeed = Number(v) || 1)" />
        </div>
        <div class="space-y-2">
          <label class="doom-label">Summoning duration</label>
          <UiInput class="doom-input" :model-value="spinDuration" type="number" :min="1" :max="20" :step="0.5" @update:model-value="(v) => (spinDuration = Number(v) || 1)" />
        </div>
      </div>

      <UiButton class="doom-spin-button w-full" :disabled="spinning || wheelEntries.length < 2" @click="spin">
        {{ spinning ? 'Summoning Doom…' : 'Unleash the Wheel' }}
      </UiButton>
      <p v-if="winner" class="doom-winner-banner">
        Doom falls upon: {{ winner }}
      </p>
    </UiCard>

    <UiCard class="doom-panel doom-panel--wheel flex flex-col items-center justify-center gap-6">
      <div class="doom-wheel-copy">
        <p class="doom-eyebrow doom-eyebrow--center">Fate Chamber</p>
        <h2 class="doom-section-title">One spin. One victim.</h2>
      </div>

      <div class="wheel-stage">
        <div class="wheel-sigil wheel-sigil--left" aria-hidden="true" />
        <div class="wheel-sigil wheel-sigil--right" aria-hidden="true" />
        <div class="wheel-shadow" aria-hidden="true" />
        <div class="wheel-flare" aria-hidden="true" />
        <div class="wheel-pointer" aria-hidden="true" />
        <div
          class="wheel-surface relative rounded-full"
          :style="wheelStyle"
        >
          <div
            v-for="(entry, index) in wheelEntries"
            :key="entry.name + index"
            class="wheel-entry absolute"
            :style="getLabelStyle(index)"
          >
            <button
              type="button"
              class="wheel-entry-action"
              :disabled="spinning"
              :aria-label="`Edit ${entry.name}`"
              @click="openEntryEditor(index)"
            >
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                <path d="M12 20h9" />
                <path d="M16.5 3.5a2.1 2.1 0 1 1 3 3L7 19l-4 1 1-4z" />
              </svg>
            </button>
            <span class="wheel-entry-name">{{ entry.name }}</span>
          </div>
          <div class="wheel-center absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 rounded-full" />
        </div>
      </div>

      <p class="doom-footer">
        {{ loadingNames ? 'Summoning names from the void…' : `${participants.length} condemned souls loaded${namesSource ? ` from ${namesSource}` : ''}.` }}
      </p>
    </UiCard>
  </main>

  <transition name="editor-popup">
    <div v-if="editingIndex !== null" class="editor-overlay" @click.self="closeEntryEditor">
      <div class="editor-modal" role="dialog" aria-modal="true" aria-labelledby="editor-title">
        <p class="editor-kicker">Mark the absent</p>
        <h2 id="editor-title" class="editor-title">Edit doomed entry</h2>
        <p class="editor-copy">Change the name, adjust how many times they wrote the protocol, or remove them from the wheel entirely.</p>

        <div class="editor-grid">
          <div class="space-y-2">
            <label class="doom-label" for="edit-name">Name</label>
            <UiInput
              id="edit-name"
              class="doom-input"
              :model-value="editName"
              placeholder="Enter a name"
              @update:model-value="(value) => (editName = value)"
            />
          </div>

          <div class="space-y-2">
            <label class="doom-label" for="edit-protocol-count">Times Protocol Written</label>
            <UiInput
              id="edit-protocol-count"
              class="doom-input"
              :model-value="editProtocolCount"
              type="number"
              :min="0"
              :step="1"
              @update:model-value="(value) => (editProtocolCount = value)"
            />
          </div>
        </div>

        <p v-if="editError" class="editor-error">{{ editError }}</p>

        <div class="editor-actions">
          <UiButton class="doom-spin-button min-w-[10rem]" @click="saveEntryEdits">Save changes</UiButton>
          <UiButton variant="outline" class="doom-secondary-button min-w-[10rem]" @click="closeEntryEditor">Cancel</UiButton>
          <UiButton variant="outline" class="editor-delete-button min-w-[10rem]" @click="deleteEntry">Delete</UiButton>
        </div>
      </div>
    </div>
  </transition>

  <div v-if="celebrationParticles.length" class="winner-particle-layer" aria-hidden="true">
    <div
      v-for="particle in celebrationParticles"
      :key="particle.id"
      class="winner-particle"
      :class="particle.shape === 'shard' ? 'winner-particle--shard' : 'winner-particle--dot'"
      :style="getParticleStyle(particle)"
    />
  </div>

  <transition name="winner-popup">
      <div v-if="showWinnerPopup && winnerDetails" class="winner-overlay" @click.self="closeWinnerPopup">
      <div class="winner-modal" role="dialog" aria-modal="true" aria-labelledby="winner-title">
        <p class="winner-kicker">DOOM HAS CHOSEN</p>
        <h2 id="winner-title" class="winner-name">{{ winnerDetails.name }}</h2>
        <p class="winner-meta">
          Protocol count: {{ winnerDetails.protocolCount }}
        </p>
        <p class="winner-meta">{{ winnerChanceLabel }}</p>
        <div class="winner-actions">
          <UiButton class="doom-spin-button min-w-[10rem]" @click="spinAgain">Spin again</UiButton>
          <UiButton variant="outline" class="doom-secondary-button min-w-[10rem]" @click="closeWinnerPopup">Seal the chamber</UiButton>
        </div>
      </div>
    </div>
  </transition>
</template>
