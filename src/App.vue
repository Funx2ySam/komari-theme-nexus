<script setup lang="ts">
import {
  Activity,
  ArrowDown,
  ArrowUp,
  CircleAlert,
  Cpu,
  LayoutGrid,
  List,
  MemoryStick,
  Moon,
  RefreshCcw,
  Server,
  Sun,
  Terminal,
  Wifi,
  WifiOff,
} from '@lucide/vue'
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

type Appearance = 'light' | 'dark' | 'system'
type ViewMode = 'grid' | 'table'
type ConnectionState = 'connecting' | 'online' | 'offline'
type Density = 'comfortable' | 'compact'

interface ApiResponse<T> {
  status: string
  message: string
  data: T
}

interface PublicSettings {
  description?: string
  record_enabled?: boolean
  record_preserve_time?: number
  sitename?: string
  theme_settings?: Record<string, unknown> | null
}

interface KomariNode {
  uuid: string
  name: string
  cpu_name?: string
  virtualization?: string
  arch?: string
  cpu_cores?: number
  os?: string
  kernel_version?: string
  region?: string
  mem_total?: number
  disk_total?: number
  group?: string
  tags?: string
  public_remark?: string
  hidden?: boolean
  traffic_limit?: number
  traffic_limit_type?: string
  weight?: number
}

interface ClientRealtime {
  cpu?: { usage?: number }
  ram?: { total?: number; used?: number }
  swap?: { total?: number; used?: number }
  load?: { load1?: number; load5?: number; load15?: number }
  disk?: { total?: number; used?: number }
  network?: { up?: number; down?: number; totalUp?: number; totalDown?: number }
  connections?: { tcp?: number; udp?: number }
  uptime?: number
  process?: number
  message?: string
  updated_at?: string
}

interface RealtimeEnvelope {
  online?: string[]
  data?: Record<string, ClientRealtime>
}

interface ClientMessage {
  status: string
  data?: RealtimeEnvelope
}

interface MetricDefinition {
  key: string
  label: string
  value: number
  text: string
  tone: 'normal' | 'warning' | 'danger' | 'muted'
}

const FALLBACK_NODES: KomariNode[] = [
  {
    uuid: 'demo-sg-01',
    name: 'DEBIAN-SG-01',
    cpu_name: 'EPYC Virtual Core',
    cpu_cores: 4,
    os: 'Debian 12',
    region: 'Singapore',
    group: 'Edge',
    public_remark: 'Demo node shown when Komari API is unavailable.',
    tags: 'demo,edge',
    mem_total: 8 * 1024 ** 3,
    disk_total: 120 * 1024 ** 3,
  },
  {
    uuid: 'demo-hk-02',
    name: 'ALPINE-HK-02',
    cpu_name: 'Xeon Shared',
    cpu_cores: 2,
    os: 'Alpine 3',
    region: 'Hong Kong',
    group: 'Core',
    tags: 'demo,core',
    mem_total: 4 * 1024 ** 3,
    disk_total: 80 * 1024 ** 3,
  },
  {
    uuid: 'demo-la-03',
    name: 'UBUNTU-LA-03',
    cpu_name: 'Ryzen VPS',
    cpu_cores: 6,
    os: 'Ubuntu 24.04',
    region: 'Los Angeles',
    group: 'Lab',
    tags: 'demo,lab',
    mem_total: 16 * 1024 ** 3,
    disk_total: 240 * 1024 ** 3,
  },
]

const FALLBACK_REALTIME: Record<string, ClientRealtime> = {
  'demo-sg-01': {
    cpu: { usage: 42.5 },
    ram: { total: 8 * 1024 ** 3, used: 4.8 * 1024 ** 3 },
    disk: { total: 120 * 1024 ** 3, used: 54 * 1024 ** 3 },
    network: { up: 190_000, down: 2_400_000, totalUp: 820 * 1024 ** 3, totalDown: 1.6 * 1024 ** 4 },
    connections: { tcp: 126, udp: 12 },
    load: { load1: 1.7, load5: 1.1, load15: 0.8 },
    uptime: 186_420,
    process: 96,
    updated_at: new Date().toISOString(),
  },
  'demo-hk-02': {
    cpu: { usage: 12.8 },
    ram: { total: 4 * 1024 ** 3, used: 1.4 * 1024 ** 3 },
    disk: { total: 80 * 1024 ** 3, used: 24 * 1024 ** 3 },
    network: { up: 80_000, down: 640_000, totalUp: 240 * 1024 ** 3, totalDown: 520 * 1024 ** 3 },
    connections: { tcp: 51, udp: 3 },
    load: { load1: 0.4, load5: 0.3, load15: 0.2 },
    uptime: 81_220,
    process: 42,
    updated_at: new Date(Date.now() - 4_000).toISOString(),
  },
  'demo-la-03': {
    cpu: { usage: 76.3 },
    ram: { total: 16 * 1024 ** 3, used: 11.8 * 1024 ** 3 },
    disk: { total: 240 * 1024 ** 3, used: 190 * 1024 ** 3 },
    network: { up: 1_200_000, down: 9_600_000, totalUp: 2.8 * 1024 ** 4, totalDown: 4.2 * 1024 ** 4 },
    connections: { tcp: 409, udp: 48 },
    load: { load1: 3.9, load5: 2.7, load15: 2.1 },
    uptime: 962_700,
    process: 231,
    message: 'High memory pressure detected',
    updated_at: new Date(Date.now() - 11_000).toISOString(),
  },
}

const appearance = ref<Appearance>('system')
const connectionState = ref<ConnectionState>('connecting')
const density = ref<Density>('comfortable')
const errorMessage = ref('')
const isRefreshing = ref(false)
const isNarrowViewport = ref(false)
const lastUpdated = ref<Date | null>(null)
const nodes = ref<KomariNode[]>([])
const onlineUuids = ref<Set<string>>(new Set())
const publicSettings = ref<PublicSettings | null>(null)
const realtimeByUuid = ref<Record<string, ClientRealtime>>({})
const selectedGroup = ref('all')
const viewMode = ref<ViewMode>('grid')

let isUnmounted = false
let pollTimer: number | undefined
let reconnectTimer: number | undefined
let socket: WebSocket | undefined
let systemThemeQuery: MediaQueryList | undefined
let viewportQuery: MediaQueryList | undefined

const themeSettings = computed(() => publicSettings.value?.theme_settings ?? {})

const siteTitle = computed(() => {
  const configuredTitle = asString(themeSettings.value.nexus_title)

  return configuredTitle || publicSettings.value?.sitename || 'Komari Nexus'
})

const showConsole = computed(() => themeSettings.value.nexus_show_console !== false)

const allGroups = computed(() => {
  const groups = new Set<string>()

  for (const node of nodes.value) {
    if (node.group) {
      groups.add(node.group)
    }
  }

  return ['all', ...Array.from(groups).sort((a, b) => a.localeCompare(b))]
})

const visibleNodes = computed(() =>
  nodes.value
    .filter((node) => !node.hidden)
    .filter((node) => selectedGroup.value === 'all' || node.group === selectedGroup.value)
    .sort((a, b) => (b.weight ?? 0) - (a.weight ?? 0) || a.name.localeCompare(b.name)),
)

const onlineSet = computed(() => {
  const online = new Set<string>()

  for (const node of visibleNodes.value) {
    if (onlineUuids.value.has(node.uuid)) {
      online.add(node.uuid)
    }
  }

  return online
})

const onlineCount = computed(() => onlineSet.value.size)

const totalCount = computed(() => visibleNodes.value.length)

const offlineCount = computed(() => Math.max(totalCount.value - onlineCount.value, 0))

const averageCpu = computed(() => averageMetric((realtime) => realtime.cpu?.usage))
const averageMemory = computed(() => averageMetric((realtime) => ratioPercent(realtime.ram?.used, realtime.ram?.total)))
const effectiveViewMode = computed<ViewMode>(() => (isNarrowViewport.value ? 'grid' : viewMode.value))
const totalDownload = computed(() => sumMetric((realtime) => realtime.network?.down))
const totalUpload = computed(() => sumMetric((realtime) => realtime.network?.up))

const consoleEvents = computed(() => {
  const events = visibleNodes.value.slice(0, 6).map((node) => {
    const realtime = realtimeByUuid.value[node.uuid]
    const status = realtime ? 'ONLINE' : 'SILENT'
    const message = realtime?.message || `${formatPercent(realtime?.cpu?.usage)} CPU / ${formatBytes(realtime?.network?.down ?? 0)}/s DOWN`

    return {
      id: node.uuid,
      line: `${new Date().toLocaleTimeString()} ${status.padEnd(7)} ${node.name.padEnd(16)} ${message}`,
      tone: realtime?.message ? 'danger' : realtime ? 'normal' : 'muted',
    }
  })

  if (events.length > 0) {
    return events
  }

  return [
    {
      id: 'boot',
      line: `${new Date().toLocaleTimeString()} WAITING KOMARI PUBLIC ENDPOINTS`,
      tone: 'muted',
    },
  ]
})

onMounted(() => {
  restorePreferences()
  setupAppearanceListener()
  setupViewportListener()
  void refreshAll()
  connectRealtime()

  pollTimer = window.setInterval(() => {
    requestRealtimeSnapshot()
  }, 1_000)
})

onBeforeUnmount(() => {
  isUnmounted = true

  if (pollTimer) {
    window.clearInterval(pollTimer)
  }

  if (reconnectTimer) {
    window.clearTimeout(reconnectTimer)
  }

  socket?.close()
  systemThemeQuery?.removeEventListener('change', applyAppearance)
  viewportQuery?.removeEventListener('change', applyViewportMode)
})

function asString(value: unknown): string {
  return typeof value === 'string' ? value.trim() : ''
}

function averageMetric(read: (realtime: ClientRealtime) => number | undefined): number {
  const values = visibleNodes.value
    .map((node) => realtimeByUuid.value[node.uuid])
    .map((realtime) => (realtime ? read(realtime) : undefined))
    .filter((value): value is number => typeof value === 'number' && Number.isFinite(value))

  if (values.length === 0) {
    return 0
  }

  return values.reduce((total, value) => total + value, 0) / values.length
}

async function fetchApi<T>(path: string): Promise<T> {
  const response = await fetch(path, {
    headers: { Accept: 'application/json' },
  })

  if (!response.ok) {
    throw new Error(`${path} returned ${response.status}`)
  }

  const payload = (await response.json()) as ApiResponse<T>

  if (payload.status !== 'success') {
    throw new Error(payload.message || `${path} request failed`)
  }

  return payload.data
}

async function refreshAll(): Promise<void> {
  isRefreshing.value = true
  errorMessage.value = ''

  try {
    const [settings, nodeList] = await Promise.all([fetchApi<PublicSettings>('/api/public'), fetchApi<KomariNode[]>('/api/nodes')])
    publicSettings.value = settings
    nodes.value = nodeList
    applyManagedSettings()
    validateSelectedGroup()
    lastUpdated.value = new Date()
  } catch (error) {
    publicSettings.value ??= {
      description: 'A simple server monitor tool.',
      record_enabled: true,
      sitename: 'Komari Nexus',
      theme_settings: {},
    }
    nodes.value = FALLBACK_NODES
    onlineUuids.value = new Set(Object.keys(FALLBACK_REALTIME))
    realtimeByUuid.value = FALLBACK_REALTIME
    validateSelectedGroup()
    connectionState.value = 'offline'
    errorMessage.value = error instanceof Error ? error.message : 'Unable to load Komari public API.'
  } finally {
    isRefreshing.value = false
  }
}

function applyManagedSettings(): void {
  const managedDensity = themeSettings.value.nexus_density

  if (managedDensity === 'compact' || managedDensity === 'comfortable') {
    density.value = managedDensity
  }
}

function validateSelectedGroup(): void {
  if (!allGroups.value.includes(selectedGroup.value)) {
    setGroup('all')
  }
}

function connectRealtime(): void {
  if (!('WebSocket' in window)) {
    connectionState.value = 'offline'
    return
  }

  const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:'
  const wsUrl = `${protocol}//${window.location.host}/api/clients`

  try {
    socket = new WebSocket(wsUrl)
    connectionState.value = 'connecting'

    socket.addEventListener('open', () => {
      connectionState.value = 'online'
      requestRealtimeSnapshot()
    })

    socket.addEventListener('message', (event: MessageEvent<string>) => {
      ingestRealtimeMessage(event.data)
    })

    socket.addEventListener('close', () => {
      connectionState.value = 'offline'
      onlineUuids.value = new Set()
      scheduleReconnect()
    })

    socket.addEventListener('error', () => {
      connectionState.value = 'offline'
      onlineUuids.value = new Set()
    })
  } catch {
    connectionState.value = 'offline'
    scheduleReconnect()
  }
}

function ingestRealtimeMessage(rawData: string): void {
  try {
    const payload = JSON.parse(rawData) as ClientMessage
    const data = payload.data?.data ?? {}

    onlineUuids.value = new Set(payload.data?.online ?? Object.keys(data))
    realtimeByUuid.value = data
    lastUpdated.value = new Date()
  } catch {
    // Ignore malformed websocket frames to keep the monitor resilient.
  }
}

function requestRealtimeSnapshot(): void {
  if (socket?.readyState === WebSocket.OPEN) {
    socket.send('get')
  }
}

function scheduleReconnect(): void {
  if (isUnmounted) {
    return
  }

  if (reconnectTimer) {
    return
  }

  reconnectTimer = window.setTimeout(() => {
    reconnectTimer = undefined
    connectRealtime()
  }, 3_000)
}

function restorePreferences(): void {
  const storedAppearance = localStorage.getItem('appearance')
  const storedGroup = localStorage.getItem('nodeSelectedGroup')
  const storedViewMode = localStorage.getItem('nodeViewMode')

  if (storedAppearance === 'light' || storedAppearance === 'dark' || storedAppearance === 'system') {
    appearance.value = storedAppearance
  }

  if (storedGroup) {
    selectedGroup.value = storedGroup
  }

  if (storedViewMode === 'grid' || storedViewMode === 'table') {
    viewMode.value = storedViewMode
  }
}

function setupAppearanceListener(): void {
  systemThemeQuery = window.matchMedia('(prefers-color-scheme: dark)')
  systemThemeQuery.addEventListener('change', applyAppearance)
  applyAppearance()
}

function setupViewportListener(): void {
  viewportQuery = window.matchMedia('(max-width: 639px)')
  viewportQuery.addEventListener('change', applyViewportMode)
  applyViewportMode()
}

function applyAppearance(): void {
  const isDark = appearance.value === 'dark' || (appearance.value === 'system' && systemThemeQuery?.matches)
  document.documentElement.classList.toggle('dark', Boolean(isDark))
}

function applyViewportMode(): void {
  isNarrowViewport.value = Boolean(viewportQuery?.matches)
}

function setAppearance(nextAppearance: Appearance): void {
  appearance.value = nextAppearance
  localStorage.setItem('appearance', nextAppearance)
  applyAppearance()
}

function setGroup(group: string): void {
  selectedGroup.value = group
  localStorage.setItem('nodeSelectedGroup', group)
}

function setViewMode(mode: ViewMode): void {
  viewMode.value = mode
  localStorage.setItem('nodeViewMode', mode)
}

function nodeStatus(node: KomariNode): 'online' | 'offline' | 'warning' {
  const realtime = realtimeByUuid.value[node.uuid]

  if (!realtime) {
    return 'offline'
  }

  if (realtime.message || (realtime.cpu?.usage ?? 0) >= 85 || ratioPercent(realtime.ram?.used, realtime.ram?.total) >= 85) {
    return 'warning'
  }

  return 'online'
}

function nodeMetrics(node: KomariNode): MetricDefinition[] {
  const realtime = realtimeByUuid.value[node.uuid]

  return [
    metric('cpu', 'CPU', realtime?.cpu?.usage ?? 0, formatPercent(realtime?.cpu?.usage), 70, 85),
    metric('mem', 'MEM', ratioPercent(realtime?.ram?.used, realtime?.ram?.total), `${formatBytes(realtime?.ram?.used ?? 0)} / ${formatBytes(realtime?.ram?.total ?? node.mem_total ?? 0)}`, 70, 85),
    metric('disk', 'DSK', ratioPercent(realtime?.disk?.used, realtime?.disk?.total), `${formatBytes(realtime?.disk?.used ?? 0)} / ${formatBytes(realtime?.disk?.total ?? node.disk_total ?? 0)}`, 75, 90),
  ]
}

function metric(key: string, label: string, value: number, text: string, warningAt: number, dangerAt: number): MetricDefinition {
  return {
    key,
    label,
    value: clamp(value),
    text,
    tone: value >= dangerAt ? 'danger' : value >= warningAt ? 'warning' : value <= 0 ? 'muted' : 'normal',
  }
}

function sparklinePoints(node: KomariNode): string {
  const realtime = realtimeByUuid.value[node.uuid]
  const seed = hashString(node.uuid)
  const cpu = realtime?.cpu?.usage ?? 24
  const network = Math.min((realtime?.network?.down ?? 0) / 1024 / 1024, 20)
  const values = Array.from({ length: 18 }, (_, index) => {
    const wave = Math.sin((index + seed) * 0.72) * 9
    const pulse = Math.cos((index * seed) % 9) * 4

    return clamp(cpu * 0.55 + network * 1.4 + wave + pulse, 8, 88)
  })

  return values.map((value, index) => `${(index / (values.length - 1)) * 100},${100 - value}`).join(' ')
}

function statusLabel(status: ReturnType<typeof nodeStatus>): string {
  return status === 'online' ? 'ONLINE' : status === 'warning' ? 'ATTN' : 'SILENT'
}

function statusDotClass(status: ReturnType<typeof nodeStatus>): string {
  return {
    online: 'bg-online shadow-[0_0_10px_var(--status-online)] status-dot-pulse',
    warning: 'bg-warning shadow-[0_0_10px_var(--status-warning)] status-dot-pulse',
    offline: 'bg-offline',
  }[status]
}

function ratioPercent(used?: number, total?: number): number {
  if (!used || !total || total <= 0) {
    return 0
  }

  return clamp((used / total) * 100)
}

function clamp(value: number, min = 0, max = 100): number {
  if (!Number.isFinite(value)) {
    return min
  }

  return Math.min(Math.max(value, min), max)
}

function sumMetric(read: (realtime: ClientRealtime) => number | undefined): number {
  return visibleNodes.value
    .map((node) => realtimeByUuid.value[node.uuid])
    .reduce((total, realtime) => total + (realtime ? read(realtime) ?? 0 : 0), 0)
}

function formatPercent(value?: number): string {
  if (typeof value !== 'number' || !Number.isFinite(value)) {
    return '0.0%'
  }

  return `${value.toFixed(1)}%`
}

function formatBytes(bytes: number): string {
  if (!Number.isFinite(bytes) || bytes <= 0) {
    return '0 B'
  }

  const units = ['B', 'KB', 'MB', 'GB', 'TB', 'PB']
  const index = Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), units.length - 1)
  const value = bytes / 1024 ** index

  return `${value >= 10 || index === 0 ? value.toFixed(0) : value.toFixed(1)} ${units[index]}`
}

function formatDuration(seconds?: number): string {
  if (typeof seconds !== 'number' || seconds < 0) {
    return '—'
  }

  const days = Math.floor(seconds / 86_400)
  const hours = Math.floor((seconds % 86_400) / 3_600)

  return days > 0 ? `${days}d ${hours}h` : `${hours}h ${Math.floor((seconds % 3_600) / 60)}m`
}

function formatTime(date: Date | null): string {
  if (!date) {
    return 'WAITING'
  }

  return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' })
}

function hashString(value: string): number {
  return value.split('').reduce((hash, char) => hash + char.charCodeAt(0), 0) || 1
}
</script>

<template>
  <div class="min-h-dvh bg-background text-foreground selection:bg-primary selection:text-primary-foreground">
    <header class="sticky top-0 z-40 border-b border-border bg-background/78 backdrop-blur-md">
      <div class="mx-auto flex h-14 w-full max-w-7xl items-center justify-between px-4 sm:px-6 lg:px-8">
        <div class="flex min-w-0 items-center gap-3">
          <div class="grid size-8 place-items-center rounded-sm border border-border bg-card">
            <Server :size="15" :stroke-width="1.7" aria-hidden="true" />
          </div>
          <div class="min-w-0">
            <p class="truncate font-mono text-[11px] font-semibold uppercase tracking-[0.28em]">{{ siteTitle }} // NEXUS</p>
            <p class="hidden truncate font-mono text-[10px] uppercase tracking-[0.18em] text-muted-foreground sm:block">
              {{ onlineCount }}/{{ totalCount }} online · {{ formatTime(lastUpdated) }}
            </p>
          </div>
        </div>

        <nav class="flex items-center gap-1" aria-label="Nexus controls">
          <button
            type="button"
            class="nexus-icon-button hidden sm:inline-flex"
            :aria-pressed="viewMode === 'grid'"
            aria-label="Grid view"
            @click="setViewMode('grid')"
          >
            <LayoutGrid :size="15" :stroke-width="1.7" aria-hidden="true" />
          </button>
          <button
            type="button"
            class="nexus-icon-button hidden sm:inline-flex"
            :aria-pressed="viewMode === 'table'"
            aria-label="Table view"
            @click="setViewMode('table')"
          >
            <List :size="15" :stroke-width="1.7" aria-hidden="true" />
          </button>
          <button type="button" class="nexus-icon-button" aria-label="Refresh node data" :disabled="isRefreshing" @click="refreshAll">
            <RefreshCcw :size="15" :stroke-width="1.7" :class="isRefreshing ? 'animate-spin' : ''" aria-hidden="true" />
          </button>
          <button
            type="button"
            class="nexus-icon-button"
            :aria-label="appearance === 'dark' ? 'Switch to light mode' : 'Switch to dark mode'"
            @click="setAppearance(appearance === 'dark' ? 'light' : 'dark')"
          >
            <Sun v-if="appearance === 'dark'" :size="15" :stroke-width="1.7" aria-hidden="true" />
            <Moon v-else :size="15" :stroke-width="1.7" aria-hidden="true" />
          </button>
        </nav>
      </div>
    </header>

    <main class="mx-auto w-full max-w-7xl px-4 py-6 sm:px-6 lg:px-8 lg:py-8">
      <section class="grid gap-4 lg:grid-cols-[1.2fr_0.8fr] lg:items-stretch">
        <div class="nexus-panel overflow-hidden p-5 sm:p-6">
          <div class="flex flex-col gap-8 sm:flex-row sm:items-end sm:justify-between">
            <div class="max-w-2xl">
              <p class="nexus-kicker">Komari Monitor Theme</p>
              <h1 class="mt-3 text-balance text-3xl font-medium tracking-[-0.05em] text-foreground sm:text-5xl">
                Quiet infrastructure telemetry.
              </h1>
              <p class="mt-4 max-w-xl text-sm leading-6 text-muted-foreground">
                A minimal Komari surface for node health, resource pressure and network flow. Silent in normal conditions, precise when attention is needed.
              </p>
            </div>

            <div class="grid min-w-52 grid-cols-2 gap-2 font-mono text-xs sm:text-right">
              <div class="rounded-sm border border-border bg-secondary/40 p-3">
                <p class="text-[10px] uppercase tracking-[0.18em] text-muted-foreground">Connection</p>
                <p class="mt-2 flex items-center gap-2 sm:justify-end">
                  <span
                    class="size-2 rounded-full"
                    :class="connectionState === 'online' ? 'bg-online status-dot-pulse' : connectionState === 'connecting' ? 'bg-warning status-dot-pulse' : 'bg-offline'"
                  />
                  {{ connectionState.toUpperCase() }}
                </p>
              </div>
              <div class="rounded-sm border border-border bg-secondary/40 p-3">
                <p class="text-[10px] uppercase tracking-[0.18em] text-muted-foreground">Offline</p>
                <p class="mt-2 text-lg font-medium tracking-[-0.04em]">{{ offlineCount }}</p>
              </div>
            </div>
          </div>

          <div v-if="errorMessage" class="mt-6 flex items-start gap-3 rounded-sm border border-warning/40 bg-warning/10 p-3 text-xs text-muted-foreground" role="status">
            <CircleAlert :size="15" :stroke-width="1.7" class="mt-0.5 shrink-0 text-warning" aria-hidden="true" />
            <p>
              Using local demo telemetry: <span class="font-mono text-foreground">{{ errorMessage }}</span>
            </p>
          </div>
        </div>

        <div class="grid grid-cols-2 gap-4">
          <article class="nexus-panel p-4">
            <div class="flex items-center justify-between text-muted-foreground">
              <Cpu :size="15" :stroke-width="1.7" aria-hidden="true" />
              <span class="nexus-kicker">AVG CPU</span>
            </div>
            <p class="mt-5 font-mono text-3xl font-light tracking-[-0.07em]">{{ formatPercent(averageCpu) }}</p>
          </article>
          <article class="nexus-panel p-4">
            <div class="flex items-center justify-between text-muted-foreground">
              <MemoryStick :size="15" :stroke-width="1.7" aria-hidden="true" />
              <span class="nexus-kicker">AVG MEM</span>
            </div>
            <p class="mt-5 font-mono text-3xl font-light tracking-[-0.07em]">{{ formatPercent(averageMemory) }}</p>
          </article>
          <article class="nexus-panel p-4">
            <div class="flex items-center justify-between text-muted-foreground">
              <ArrowDown :size="15" :stroke-width="1.7" aria-hidden="true" />
              <span class="nexus-kicker">DOWN</span>
            </div>
            <p class="mt-5 font-mono text-2xl font-light tracking-[-0.06em]">{{ formatBytes(totalDownload) }}/s</p>
          </article>
          <article class="nexus-panel p-4">
            <div class="flex items-center justify-between text-muted-foreground">
              <ArrowUp :size="15" :stroke-width="1.7" aria-hidden="true" />
              <span class="nexus-kicker">UP</span>
            </div>
            <p class="mt-5 font-mono text-2xl font-light tracking-[-0.06em]">{{ formatBytes(totalUpload) }}/s</p>
          </article>
        </div>
      </section>

      <section class="mt-5 flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
        <div class="flex flex-wrap gap-2">
          <button
            v-for="group in allGroups"
            :key="group"
            type="button"
            class="nexus-chip"
            :aria-pressed="selectedGroup === group"
            @click="setGroup(group)"
          >
            {{ group === 'all' ? 'ALL GROUPS' : group }}
          </button>
        </div>
        <p class="font-mono text-[10px] uppercase tracking-[0.18em] text-muted-foreground">
          {{ visibleNodes.length }} nodes · {{ density }} density
        </p>
      </section>

      <section v-if="effectiveViewMode === 'grid'" class="mt-5 grid gap-4 sm:grid-cols-2 xl:grid-cols-3" :class="density === 'compact' ? '2xl:grid-cols-4' : ''">
        <article
          v-for="node in visibleNodes"
          :key="node.uuid"
          class="nexus-card"
          :class="density === 'compact' ? 'p-4' : 'p-5'"
        >
          <div class="flex items-start justify-between gap-4">
            <div class="min-w-0">
              <div class="flex items-center gap-2">
                <component :is="nodeStatus(node) === 'offline' ? WifiOff : Wifi" :size="15" :stroke-width="1.7" class="text-muted-foreground" aria-hidden="true" />
                <h2 class="truncate font-mono text-sm font-medium uppercase tracking-[0.08em]">{{ node.name }}</h2>
              </div>
              <p class="mt-1 truncate font-mono text-[11px] uppercase tracking-[0.12em] text-muted-foreground">
                {{ node.region || 'UNKNOWN REGION' }} · {{ node.os || 'UNKNOWN OS' }}
              </p>
            </div>
            <div class="flex shrink-0 items-center gap-2 font-mono text-[10px] uppercase tracking-[0.16em] text-muted-foreground">
              <span class="size-2 rounded-full" :class="statusDotClass(nodeStatus(node))" />
              {{ statusLabel(nodeStatus(node)) }}
            </div>
          </div>

          <div class="my-4 h-px bg-border" />

          <div class="space-y-3">
            <div v-for="item in nodeMetrics(node)" :key="item.key" class="space-y-1.5">
              <div class="flex items-center justify-between gap-4 font-mono text-[11px]">
                <span class="text-muted-foreground">{{ item.label }}</span>
                <span
                  class="tabular-nums"
                  :class="item.tone === 'danger' ? 'text-destructive' : item.tone === 'warning' ? 'text-warning' : item.tone === 'muted' ? 'text-muted-foreground' : 'text-foreground'"
                >
                  {{ item.text }}
                </span>
              </div>
              <div
                class="h-1 overflow-hidden rounded-none bg-secondary"
                role="progressbar"
                :aria-label="`${node.name} ${item.label} usage`"
                :aria-valuenow="Math.round(item.value)"
                aria-valuemin="0"
                aria-valuemax="100"
              >
                <div
                  class="h-full rounded-none transition-[width,background-color] duration-300"
                  :class="item.tone === 'danger' ? 'bg-destructive' : item.tone === 'warning' ? 'bg-warning' : 'bg-primary'"
                  :style="{ width: `${item.value}%` }"
                />
              </div>
            </div>
          </div>

          <div class="my-4 h-px bg-border" />

          <div class="grid grid-cols-2 gap-3 font-mono text-[11px]">
            <div>
              <p class="text-muted-foreground">NET-IN</p>
              <p class="mt-1 tabular-nums">{{ formatBytes(realtimeByUuid[node.uuid]?.network?.down ?? 0) }}/s</p>
            </div>
            <div>
              <p class="text-muted-foreground">NET-OUT</p>
              <p class="mt-1 tabular-nums">{{ formatBytes(realtimeByUuid[node.uuid]?.network?.up ?? 0) }}/s</p>
            </div>
            <div>
              <p class="text-muted-foreground">LOAD</p>
              <p class="mt-1 tabular-nums">{{ realtimeByUuid[node.uuid]?.load?.load1?.toFixed(2) ?? '—' }}</p>
            </div>
            <div>
              <p class="text-muted-foreground">UPTIME</p>
              <p class="mt-1 tabular-nums">{{ formatDuration(realtimeByUuid[node.uuid]?.uptime) }}</p>
            </div>
          </div>

          <svg class="mt-5 hidden h-10 w-full text-primary sm:block" viewBox="0 0 100 100" preserveAspectRatio="none" aria-hidden="true">
            <polyline
              :points="sparklinePoints(node)"
              fill="none"
              stroke="currentColor"
              stroke-width="1.4"
              vector-effect="non-scaling-stroke"
              stroke-linecap="round"
              stroke-linejoin="round"
              opacity="0.82"
            />
          </svg>
        </article>
      </section>

      <section v-else class="nexus-panel mt-5 overflow-hidden">
        <div class="overflow-x-auto">
          <table class="w-full min-w-[760px] text-left font-mono text-xs">
            <thead class="border-b border-border text-[10px] uppercase tracking-[0.18em] text-muted-foreground">
              <tr>
                <th class="px-4 py-3 font-medium">Node</th>
                <th class="px-4 py-3 font-medium">Status</th>
                <th class="px-4 py-3 font-medium">CPU</th>
                <th class="px-4 py-3 font-medium">Memory</th>
                <th class="px-4 py-3 font-medium">Disk</th>
                <th class="px-4 py-3 font-medium">Network</th>
                <th class="px-4 py-3 font-medium">Uptime</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="node in visibleNodes" :key="node.uuid" class="border-b border-border/80 last:border-0">
                <td class="px-4 py-3">
                  <p class="font-medium uppercase tracking-[0.08em]">{{ node.name }}</p>
                  <p class="mt-1 text-[10px] uppercase tracking-[0.14em] text-muted-foreground">{{ node.group || 'default' }} · {{ node.region || 'unknown' }}</p>
                </td>
                <td class="px-4 py-3">
                  <span class="inline-flex items-center gap-2 text-[10px] uppercase tracking-[0.16em] text-muted-foreground">
                    <span class="size-2 rounded-full" :class="statusDotClass(nodeStatus(node))" />
                    {{ statusLabel(nodeStatus(node)) }}
                  </span>
                </td>
                <td class="px-4 py-3 tabular-nums">{{ formatPercent(realtimeByUuid[node.uuid]?.cpu?.usage) }}</td>
                <td class="px-4 py-3 tabular-nums">{{ formatPercent(ratioPercent(realtimeByUuid[node.uuid]?.ram?.used, realtimeByUuid[node.uuid]?.ram?.total)) }}</td>
                <td class="px-4 py-3 tabular-nums">{{ formatPercent(ratioPercent(realtimeByUuid[node.uuid]?.disk?.used, realtimeByUuid[node.uuid]?.disk?.total)) }}</td>
                <td class="px-4 py-3 tabular-nums">
                  ↓ {{ formatBytes(realtimeByUuid[node.uuid]?.network?.down ?? 0) }}/s · ↑ {{ formatBytes(realtimeByUuid[node.uuid]?.network?.up ?? 0) }}/s
                </td>
                <td class="px-4 py-3 tabular-nums">{{ formatDuration(realtimeByUuid[node.uuid]?.uptime) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>

      <section v-if="showConsole" class="nexus-console mt-5 p-4" aria-label="System stream">
        <div class="mb-3 flex items-center justify-between">
          <div class="flex items-center gap-2 text-muted-foreground">
            <Terminal :size="15" :stroke-width="1.7" aria-hidden="true" />
            <p class="nexus-kicker">System Stream</p>
          </div>
          <Activity :size="15" :stroke-width="1.7" class="text-online" aria-hidden="true" />
        </div>
        <div class="space-y-1 font-mono text-[11px] leading-relaxed">
          <p
            v-for="event in consoleEvents"
            :key="event.id"
            :class="event.tone === 'danger' ? 'text-destructive' : event.tone === 'muted' ? 'text-muted-foreground' : 'text-online'"
          >
            {{ event.line }}
          </p>
        </div>
      </section>
    </main>

    <footer class="mx-auto flex w-full max-w-7xl flex-col gap-2 px-4 pb-8 pt-2 font-mono text-[10px] uppercase tracking-[0.18em] text-muted-foreground sm:flex-row sm:items-center sm:justify-between sm:px-6 lg:px-8">
      <p>Powered by Komari Monitor.</p>
      <p>Nexus / minimal telemetry surface</p>
    </footer>
  </div>
</template>
