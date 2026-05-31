<script setup lang="ts">
import {
  ArrowDown,
  ArrowUp,
  CircleAlert,
  Cpu,
  Languages,
  LayoutGrid,
  List,
  MemoryStick,
  MonitorCog,
  Moon,
  RefreshCcw,
  Server,
  Sun,
} from '@lucide/vue'
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

type Appearance = 'light' | 'dark' | 'system'
type ViewMode = 'grid' | 'table'
type ConnectionState = 'connecting' | 'online' | 'offline'
type Density = 'comfortable' | 'compact'
type Language = 'en-US' | 'zh-CN'

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

interface PingRecordsResponse {
  count: number
  records: Array<{
    task_id: number
    time: string
    value: number
  }>
  tasks: Array<{
    id: number
    interval: number
    name: string
    loss: number
  }>
}

interface PingTaskSummary {
  id: number
  loss: number
  name: string
}

interface MetricDefinition {
  key: string
  label: string
  value: number
  text: string
  tone: 'normal' | 'warning' | 'danger' | 'muted'
}

const MESSAGES = {
  'en-US': {
    admin: 'Admin',
    adminAria: 'Open Komari admin',
    allGroups: 'ALL GROUPS',
    avgCpu: 'AVG CPU',
    avgMem: 'AVG MEM',
    defaultGroup: 'default',
    density: 'density',
    disk: 'Disk',
    down: 'DOWN',
    gridView: 'Grid view',
    language: 'Language',
    load: 'LOAD',
    loss: 'LOSS',
    memory: 'Memory',
    netIn: 'NET-IN',
    netOut: 'NET-OUT',
    network: 'Network',
    node: 'Node',
    nodes: 'nodes',
    online: 'ONLINE',
    refresh: 'Refresh node data',
    status: 'Status',
    switchLanguage: '切换中文',
    switchTheme: 'Switch color mode',
    tableView: 'Table view',
    unknown: 'unknown',
    unknownOs: 'UNKNOWN OS',
    unknownRegion: 'UNKNOWN REGION',
    up: 'UP',
    uptime: 'Uptime',
    usingDemo: 'Using local demo telemetry:',
    footer: 'Nexus / minimal telemetry surface',
  },
  'zh-CN': {
    admin: '后台',
    adminAria: '打开 Komari 后台',
    allGroups: '全部分组',
    avgCpu: '平均 CPU',
    avgMem: '平均内存',
    defaultGroup: '默认',
    density: '密度',
    disk: '磁盘',
    down: '下行',
    gridView: '网格视图',
    language: '语言',
    load: '负载',
    loss: '丢包',
    memory: '内存',
    netIn: '下行',
    netOut: '上行',
    network: '网络',
    node: '节点',
    nodes: '个节点',
    online: '在线',
    refresh: '刷新节点数据',
    status: '状态',
    switchLanguage: 'Switch English',
    switchTheme: '切换明暗模式',
    tableView: '列表视图',
    unknown: '未知',
    unknownOs: '未知系统',
    unknownRegion: '未知区域',
    up: '上行',
    uptime: '运行时间',
    usingDemo: '正在使用本地演示数据：',
    footer: 'Nexus / 极简遥测界面',
  },
} satisfies Record<Language, Record<string, string>>

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

const FALLBACK_PING_TASKS: Record<string, PingTaskSummary[]> = {
  'demo-sg-01': [
    { id: 1, name: '广东电信', loss: 0 },
    { id: 2, name: '上海联通', loss: 1.2 },
  ],
  'demo-hk-02': [{ id: 1, name: '广东电信', loss: 3.6 }],
  'demo-la-03': [
    { id: 1, name: '广东电信', loss: 12.5 },
    { id: 2, name: '北京移动', loss: 7.8 },
  ],
}

const appearance = ref<Appearance>('system')
const connectionState = ref<ConnectionState>('connecting')
const density = ref<Density>('comfortable')
const errorMessage = ref('')
const isRefreshing = ref(false)
const isNarrowViewport = ref(false)
const language = ref<Language>('zh-CN')
const lastUpdated = ref<Date | null>(null)
const nodes = ref<KomariNode[]>([])
const onlineUuids = ref<Set<string>>(new Set())
const pingTasksByUuid = ref<Record<string, PingTaskSummary[]>>({})
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

const t = computed(() => MESSAGES[language.value])

const showPingLoss = computed(() => themeSettings.value.nexus_show_ping_loss !== false)

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

const averageCpu = computed(() => averageMetric((realtime) => realtime.cpu?.usage))
const averageMemory = computed(() => averageMetric((realtime) => ratioPercent(realtime.ram?.used, realtime.ram?.total)))
const effectiveViewMode = computed<ViewMode>(() => (isNarrowViewport.value ? 'grid' : viewMode.value))
const totalDownload = computed(() => sumMetric((realtime) => realtime.network?.down))
const totalUpload = computed(() => sumMetric((realtime) => realtime.network?.up))

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
    .map((node) => nodeRealtime(node))
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
    await loadPingSummaries(nodeList)
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
    pingTasksByUuid.value = FALLBACK_PING_TASKS
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

async function loadPingSummaries(nodeList: KomariNode[]): Promise<void> {
  if (!showPingLoss.value) {
    pingTasksByUuid.value = {}
    return
  }

  const visibleNodeList = nodeList.filter((node) => !node.hidden)
  const results = await Promise.allSettled(
    visibleNodeList.map(async (node) => [node.uuid, await fetchPingTasks(node.uuid)] as const),
  )
  const nextTasks: Record<string, PingTaskSummary[]> = {}

  for (const result of results) {
    if (result.status === 'fulfilled' && result.value[1].length > 0) {
      nextTasks[result.value[0]] = result.value[1]
    }
  }

  pingTasksByUuid.value = nextTasks
}

async function fetchPingTasks(uuid: string): Promise<PingTaskSummary[]> {
  const params = new URLSearchParams({ uuid, hours: '1' })
  const data = await fetchApi<PingRecordsResponse>(`/api/records/ping?${params.toString()}`)

  return data.tasks.map((task) => ({
    id: task.id,
    loss: clamp(task.loss ?? 0),
    name: task.name || `PING ${task.id}`,
  }))
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
  const storedLanguage = localStorage.getItem('i18nextLng')
  const storedViewMode = localStorage.getItem('nodeViewMode')

  if (storedAppearance === 'light' || storedAppearance === 'dark' || storedAppearance === 'system') {
    appearance.value = storedAppearance
  }

  if (storedGroup) {
    selectedGroup.value = storedGroup
  }

  if (storedLanguage === 'en-US' || storedLanguage === 'zh-CN') {
    language.value = storedLanguage
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

function toggleLanguage(): void {
  language.value = language.value === 'zh-CN' ? 'en-US' : 'zh-CN'
  localStorage.setItem('i18nextLng', language.value)
}

function setViewMode(mode: ViewMode): void {
  viewMode.value = mode
  localStorage.setItem('nodeViewMode', mode)
}

function nodeStatus(node: KomariNode): 'online' | 'offline' | 'warning' {
  const realtime = nodeRealtime(node)

  if (!realtime) {
    return 'offline'
  }

  if (realtime.message || (realtime.cpu?.usage ?? 0) >= 85 || ratioPercent(realtime.ram?.used, realtime.ram?.total) >= 85) {
    return 'warning'
  }

  return 'online'
}

function nodeMetrics(node: KomariNode): MetricDefinition[] {
  const realtime = nodeRealtime(node)

  return [
    metric('cpu', 'CPU', realtime?.cpu?.usage ?? 0, formatPercent(realtime?.cpu?.usage), 70, 85),
    metric('mem', 'MEM', ratioPercent(realtime?.ram?.used, realtime?.ram?.total), `${formatBytes(realtime?.ram?.used ?? 0)} / ${formatBytes(realtime?.ram?.total ?? node.mem_total ?? 0)}`, 70, 85),
    metric('disk', 'DSK', ratioPercent(realtime?.disk?.used, realtime?.disk?.total), `${formatBytes(realtime?.disk?.used ?? 0)} / ${formatBytes(realtime?.disk?.total ?? node.disk_total ?? 0)}`, 75, 90),
  ]
}

function nodeRealtime(node: KomariNode): ClientRealtime | undefined {
  return onlineUuids.value.has(node.uuid) ? realtimeByUuid.value[node.uuid] : undefined
}

function nodePingTasks(node: KomariNode): PingTaskSummary[] {
  return showPingLoss.value ? pingTasksByUuid.value[node.uuid] ?? [] : []
}

function lossSegments(loss: number): boolean[] {
  const segmentCount = 16
  const activeCount = Math.round((clamp(100 - loss) / 100) * segmentCount)

  return Array.from({ length: segmentCount }, (_, index) => index < activeCount)
}

function lossToneClass(loss: number): string {
  if (loss >= 20) return 'bg-destructive'
  if (loss >= 5) return 'bg-warning'
  return 'bg-online'
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

function statusLabel(status: ReturnType<typeof nodeStatus>): string {
  if (status === 'online') {
    return t.value.online
  }

  if (status === 'warning') {
    return language.value === 'zh-CN' ? '注意' : 'ATTN'
  }

  return language.value === 'zh-CN' ? '静默' : 'SILENT'
}

function osIconId(node: KomariNode): string {
  const value = `${node.os ?? ''} ${node.name}`.toLowerCase()

  if (value.includes('debian')) return 'os-debian'
  if (value.includes('ubuntu')) return 'os-ubuntu'
  if (value.includes('alpine')) return 'os-alpine'
  if (value.includes('centos')) return 'os-centos'
  if (value.includes('fedora')) return 'os-fedora'
  if (value.includes('arch')) return 'os-arch'
  if (value.includes('windows')) return 'os-windows'
  if (value.includes('darwin') || value.includes('macos')) return 'os-apple'
  if (value.includes('freebsd')) return 'os-freebsd'
  if (value.includes('linux')) return 'os-linux'

  return 'os-generic'
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
    .map((node) => nodeRealtime(node))
    .reduce((total, realtime) => total + (realtime ? read(realtime) ?? 0 : 0), 0)
}

function formatLoss(loss: number): string {
  return `${clamp(loss).toFixed(loss >= 10 ? 0 : 1)}%`
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

</script>

<template>
  <div class="min-h-dvh bg-background text-foreground selection:bg-primary selection:text-primary-foreground">
    <svg class="hidden" aria-hidden="true" xmlns="http://www.w3.org/2000/svg">
      <symbol id="os-linux" viewBox="0 0 24 24">
        <path fill="currentColor" d="M12.2 2.2c-2.3 0-3.7 1.7-3.7 4.3 0 1-.2 1.9-.6 2.7l-3.1 5.7c-.5 1-.7 2.1-.3 3 .3.9 1.1 1.5 2.1 1.5.5 0 1-.1 1.5-.3.9 1.5 2.3 2.4 4 2.4 1.6 0 3-.9 3.9-2.4.5.2 1 .3 1.5.3 1 0 1.8-.6 2.1-1.5.4-.9.2-2-.3-3l-3.1-5.7c-.4-.8-.6-1.7-.6-2.7 0-2.6-1.4-4.3-3.4-4.3Zm-1.6 4.1c.4 0 .7.4.7.8s-.3.8-.7.8-.7-.4-.7-.8.3-.8.7-.8Zm3 0c.4 0 .7.4.7.8s-.3.8-.7.8-.7-.4-.7-.8.3-.8.7-.8Zm-2.9 3.5h2.8l-1.4 1.2-1.4-1.2Zm1.4 9.8c-.9 0-1.7-.5-2.2-1.4h4.4c-.5.9-1.3 1.4-2.2 1.4Z" />
      </symbol>
      <symbol id="os-debian" viewBox="0 0 24 24">
        <path fill="currentColor" d="M12.7 3.1c-4.5-.5-8.4 2.2-8.7 6.1-.3 3.3 2.3 6.1 5.7 6.4 2.4.2 4.7-.8 5.8-2.5.5-.8.5-1.6 0-2.1-.5-.4-1.2-.3-1.7.3-.7 1-2.1 1.6-3.5 1.5-2-.2-3.5-1.7-3.3-3.4.2-2.2 2.7-3.8 5.4-3.5 3.6.4 6.1 3.1 5.8 6.4-.4 4.3-5.2 7.2-10.5 6.4-.5-.1-.9.2-1 .7-.1.5.2.9.7 1 6.4 1 12.1-2.7 12.6-8 .4-4.4-2.8-8.7-7.3-9.3Zm-.4 4.7c-1.9-.2-3.6.8-3.7 2.2-.1 1.2.9 2.2 2.3 2.3 1 .1 2-.3 2.5-1 .3-.4.2-.9-.1-1.1-.4-.2-.8-.1-1.1.2-.2.3-.6.4-1 .4-.6-.1-1-.4-1-.8.1-.5.9-.9 1.8-.8 1.4.1 2.4 1 2.3 2.1-.1.6.3 1 .8 1.1.5.1.9-.3 1-.8.2-1.9-1.5-3.5-3.8-3.8Z" />
      </symbol>
      <symbol id="os-ubuntu" viewBox="0 0 24 24">
        <path fill="currentColor" d="M12 5.8a6.2 6.2 0 0 1 5.1 2.7l-2.2 1.3a3.7 3.7 0 0 0-6.5 2.5H5.8A6.2 6.2 0 0 1 12 5.8Zm0 12.4a6.2 6.2 0 0 1-5.1-2.7l2.2-1.3a3.7 3.7 0 0 0 6.5-2.5h2.6a6.2 6.2 0 0 1-6.2 6.5ZM4.5 9.7a2 2 0 1 1 0-4 2 2 0 0 1 0 4Zm15 0a2 2 0 1 1 0-4 2 2 0 0 1 0 4Zm0 8.6a2 2 0 1 1 0-4 2 2 0 0 1 0 4ZM12 9.8a2.2 2.2 0 1 1 0 4.4 2.2 2.2 0 0 1 0-4.4Z" />
      </symbol>
      <symbol id="os-alpine" viewBox="0 0 24 24">
        <path fill="currentColor" d="m2.5 18.5 7.1-13 3.1 5.5 1.7-2.8 7.1 10.3h-4.2l-3-4.5-1.7 2.7-3-5.4-4 7.2H2.5Z" />
      </symbol>
      <symbol id="os-centos" viewBox="0 0 24 24">
        <path fill="currentColor" d="M5 5h6v6H5V5Zm8 0h6v6h-6V5ZM5 13h6v6H5v-6Zm8 0h6v6h-6v-6Zm-2-8 2 2-2 2-2-2 2-2Zm2 10 2-2 2 2-2 2-2-2ZM7 13l2 2-2 2-2-2 2-2Zm10-8 2 2-2 2-2-2 2-2Z" />
      </symbol>
      <symbol id="os-fedora" viewBox="0 0 24 24">
        <path fill="currentColor" d="M12 3a9 9 0 1 0 0 18h2.1a4.9 4.9 0 0 0 0-9.8h-1.8V9.6c0-1.3 1-2.4 2.4-2.4h1.4a6.8 6.8 0 0 0-4.1-1.4Zm2.1 11a2.1 2.1 0 1 1 0 4.2H12V14h2.1ZM9.7 7.2A4.7 4.7 0 0 0 5 11.9v4.9h2.8v-4.9c0-1 .8-1.9 1.9-1.9h.9V7.2h-.9Z" />
      </symbol>
      <symbol id="os-arch" viewBox="0 0 24 24">
        <path fill="currentColor" d="M12 2.7 3.6 21.3c2.1-1.3 4.1-2.1 6.1-2.4l2.3-5.6 2.3 5.6c2 .3 4 .1 6.1-.7L12 2.7Zm0 4.6 1.6 4.1c-1 .6-2 .6-3.2 0L12 7.3Z" />
      </symbol>
      <symbol id="os-windows" viewBox="0 0 24 24">
        <path fill="currentColor" d="M3 5.3 10.7 4v7.3H3V5.3Zm9-1.5L21 2.3v9h-9V3.8ZM3 12.7h7.7V20L3 18.7v-6Zm9 0h9v9l-9-1.5v-7.5Z" />
      </symbol>
      <symbol id="os-apple" viewBox="0 0 24 24">
        <path fill="currentColor" d="M16.5 12.7c0-2 1.6-3 1.7-3.1-1-1.4-2.5-1.6-3-1.6-1.3-.1-2.5.8-3.1.8-.7 0-1.7-.8-2.8-.7-1.4 0-2.7.8-3.4 2.1-1.5 2.6-.4 6.5 1.1 8.6.7 1 1.6 2.2 2.7 2.1 1.1 0 1.5-.7 2.8-.7s1.7.7 2.8.7c1.2 0 1.9-1.1 2.6-2.1.8-1.2 1.1-2.3 1.1-2.4 0 0-2.5-1-2.5-3.7ZM14.4 6.6c.6-.8 1.1-1.8 1-2.9-.9.1-1.9.6-2.5 1.3-.6.7-1.1 1.7-.9 2.8.9.1 1.8-.5 2.4-1.2Z" />
      </symbol>
      <symbol id="os-freebsd" viewBox="0 0 24 24">
        <path fill="currentColor" d="M7.3 4.6 5.6 2.4c-.3-.4.1-.9.6-.7l2.6 1.2c.9-.4 2-.6 3.2-.6s2.3.2 3.2.6l2.6-1.2c.5-.2.9.3.6.7l-1.7 2.2c1.2 1.1 1.9 2.7 1.9 4.8 0 5.1-3 10.2-6.6 10.2S5.4 14.5 5.4 9.4c0-2.1.7-3.7 1.9-4.8Zm2 6.1c.6 0 1-.5 1-1.1s-.4-1.1-1-1.1-1 .5-1 1.1.4 1.1 1 1.1Zm5.4 0c.6 0 1-.5 1-1.1s-.4-1.1-1-1.1-1 .5-1 1.1.4 1.1 1 1.1ZM9.8 15.3c1.4.8 3 .8 4.4 0 .4-.2.8.3.5.7-.6.9-1.6 1.4-2.7 1.4s-2.1-.5-2.7-1.4c-.3-.4.1-.9.5-.7Z" />
      </symbol>
      <symbol id="os-generic" viewBox="0 0 24 24">
        <path fill="currentColor" d="M4 5.5A2.5 2.5 0 0 1 6.5 3h11A2.5 2.5 0 0 1 20 5.5v8a2.5 2.5 0 0 1-2.5 2.5h-11A2.5 2.5 0 0 1 4 13.5v-8ZM8 19h8v2H8v-2Zm3-3h2v3h-2v-3Z" />
      </symbol>
    </svg>
    <header class="sticky top-0 z-40 border-b border-border bg-background/78 backdrop-blur-md">
      <div class="mx-auto flex h-14 w-full max-w-7xl items-center justify-between px-4 sm:px-6 lg:px-8">
        <div class="flex min-w-0 items-center gap-3">
          <div class="grid size-8 place-items-center rounded-sm border border-border bg-card">
            <Server :size="15" :stroke-width="1.7" aria-hidden="true" />
          </div>
          <div class="min-w-0">
            <p class="truncate font-mono text-[11px] font-semibold uppercase tracking-[0.28em]">{{ siteTitle }} // NEXUS</p>
            <p class="hidden truncate font-mono text-[10px] uppercase tracking-[0.18em] text-muted-foreground sm:block">
              {{ onlineCount }}/{{ totalCount }} {{ t.online }} · {{ formatTime(lastUpdated) }}
            </p>
          </div>
        </div>

        <nav class="flex items-center gap-1" aria-label="Nexus controls">
          <button
            type="button"
            class="nexus-icon-button hidden sm:inline-flex"
            :aria-pressed="viewMode === 'grid'"
            :aria-label="t.gridView"
            @click="setViewMode('grid')"
          >
            <LayoutGrid :size="15" :stroke-width="1.7" aria-hidden="true" />
          </button>
          <button
            type="button"
            class="nexus-icon-button hidden sm:inline-flex"
            :aria-pressed="viewMode === 'table'"
            :aria-label="t.tableView"
            @click="setViewMode('table')"
          >
            <List :size="15" :stroke-width="1.7" aria-hidden="true" />
          </button>
          <a class="nexus-icon-button" href="/admin" :aria-label="t.adminAria" :title="t.admin">
            <MonitorCog :size="15" :stroke-width="1.7" aria-hidden="true" />
          </a>
          <button type="button" class="nexus-icon-button" :aria-label="t.switchLanguage" :title="t.language" @click="toggleLanguage">
            <Languages :size="15" :stroke-width="1.7" aria-hidden="true" />
            <span class="sr-only">{{ t.switchLanguage }}</span>
          </button>
          <button type="button" class="nexus-icon-button" :aria-label="t.refresh" :disabled="isRefreshing" @click="refreshAll">
            <RefreshCcw :size="15" :stroke-width="1.7" :class="isRefreshing ? 'animate-spin' : ''" aria-hidden="true" />
          </button>
          <button
            type="button"
            class="nexus-icon-button"
            :aria-label="t.switchTheme"
            @click="setAppearance(appearance === 'dark' ? 'light' : 'dark')"
          >
            <Sun v-if="appearance === 'dark'" :size="15" :stroke-width="1.7" aria-hidden="true" />
            <Moon v-else :size="15" :stroke-width="1.7" aria-hidden="true" />
          </button>
        </nav>
      </div>
    </header>

    <main class="mx-auto w-full max-w-7xl px-4 py-6 sm:px-6 lg:px-8 lg:py-8">
      <section class="grid gap-4 sm:grid-cols-2 lg:grid-cols-4">
          <article class="nexus-panel p-4 min-w-0">
            <div class="flex items-center justify-between text-muted-foreground">
              <Cpu :size="15" :stroke-width="1.7" aria-hidden="true" />
              <span class="nexus-kicker">{{ t.avgCpu }}</span>
            </div>
            <p class="mt-5 font-mono text-3xl font-light tracking-[-0.07em]">{{ formatPercent(averageCpu) }}</p>
          </article>
          <article class="nexus-panel p-4 min-w-0">
            <div class="flex items-center justify-between text-muted-foreground">
              <MemoryStick :size="15" :stroke-width="1.7" aria-hidden="true" />
              <span class="nexus-kicker">{{ t.avgMem }}</span>
            </div>
            <p class="mt-5 font-mono text-3xl font-light tracking-[-0.07em]">{{ formatPercent(averageMemory) }}</p>
          </article>
          <article class="nexus-panel p-4 min-w-0">
            <div class="flex items-center justify-between text-muted-foreground">
              <ArrowDown :size="15" :stroke-width="1.7" aria-hidden="true" />
              <span class="nexus-kicker">{{ t.down }}</span>
            </div>
            <p class="mt-5 font-mono text-2xl font-light tracking-[-0.06em]">{{ formatBytes(totalDownload) }}/s</p>
          </article>
          <article class="nexus-panel p-4 min-w-0">
            <div class="flex items-center justify-between text-muted-foreground">
              <ArrowUp :size="15" :stroke-width="1.7" aria-hidden="true" />
              <span class="nexus-kicker">{{ t.up }}</span>
            </div>
            <p class="mt-5 font-mono text-2xl font-light tracking-[-0.06em]">{{ formatBytes(totalUpload) }}/s</p>
          </article>
      </section>

      <div v-if="errorMessage" class="mt-5 flex items-start gap-3 rounded-sm border border-warning/40 bg-warning/10 p-3 text-xs text-muted-foreground" role="status">
        <CircleAlert :size="15" :stroke-width="1.7" class="mt-0.5 shrink-0 text-warning" aria-hidden="true" />
        <p>
          {{ t.usingDemo }} <span class="font-mono text-foreground">{{ errorMessage }}</span>
        </p>
      </div>

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
            {{ group === 'all' ? t.allGroups : group }}
          </button>
        </div>
        <p class="font-mono text-[10px] uppercase tracking-[0.18em] text-muted-foreground">
          {{ visibleNodes.length }} {{ t.nodes }} · {{ density }} {{ t.density }}
        </p>
      </section>

      <section v-if="effectiveViewMode === 'grid'" class="mt-5 grid gap-4 sm:grid-cols-2 xl:grid-cols-3" :class="density === 'compact' ? '2xl:grid-cols-4' : ''">
        <article
          v-for="node in visibleNodes"
          :key="node.uuid"
          class="nexus-card"
          :class="[density === 'compact' ? 'p-4' : 'p-5', nodeStatus(node) === 'offline' ? 'nexus-card-offline' : '']"
        >
          <div class="flex items-start justify-between gap-4">
            <div class="min-w-0">
              <div class="flex items-center gap-2">
                <span class="nexus-os-icon" :class="nodeStatus(node) === 'offline' ? 'opacity-45 grayscale' : ''" :aria-label="`${node.os || t.unknownOs} OS`">
                  <svg class="size-4.5" aria-hidden="true">
                    <use :href="`#${osIconId(node)}`" />
                  </svg>
                </span>
                <h2 class="truncate font-mono text-sm font-medium uppercase tracking-[0.08em]">{{ node.name }}</h2>
              </div>
              <p class="mt-1 truncate font-mono text-[11px] uppercase tracking-[0.12em] text-muted-foreground">
                {{ node.region || t.unknownRegion }} · {{ node.os || t.unknownOs }}
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

          <div v-if="nodePingTasks(node).length > 0" class="mb-4 space-y-2">
            <div v-for="task in nodePingTasks(node)" :key="task.id" class="nexus-loss-row">
              <span class="truncate">{{ task.name }}</span>
              <div class="flex min-w-24 flex-1 items-center gap-0.5" :aria-label="`${task.name} ${t.loss} ${formatLoss(task.loss)}`">
                <span
                  v-for="(active, index) in lossSegments(task.loss)"
                  :key="index"
                  class="h-3 flex-1 rounded-[1px]"
                  :class="active ? lossToneClass(task.loss) : 'bg-secondary'"
                />
              </div>
              <span class="tabular-nums">{{ formatLoss(task.loss) }}</span>
            </div>
          </div>

          <div class="grid grid-cols-2 gap-3 font-mono text-[11px]">
            <div>
              <p class="text-muted-foreground">{{ t.netIn }}</p>
              <p class="mt-1 tabular-nums">{{ formatBytes(realtimeByUuid[node.uuid]?.network?.down ?? 0) }}/s</p>
            </div>
            <div>
              <p class="text-muted-foreground">{{ t.netOut }}</p>
              <p class="mt-1 tabular-nums">{{ formatBytes(realtimeByUuid[node.uuid]?.network?.up ?? 0) }}/s</p>
            </div>
            <div>
              <p class="text-muted-foreground">{{ t.load }}</p>
              <p class="mt-1 tabular-nums">{{ realtimeByUuid[node.uuid]?.load?.load1?.toFixed(2) ?? '—' }}</p>
            </div>
            <div>
              <p class="text-muted-foreground">{{ t.uptime }}</p>
              <p class="mt-1 tabular-nums">{{ formatDuration(realtimeByUuid[node.uuid]?.uptime) }}</p>
            </div>
          </div>

        </article>
      </section>

      <section v-else class="nexus-panel mt-5 overflow-hidden">
        <div class="overflow-x-auto">
          <table class="w-full min-w-[760px] text-left font-mono text-xs">
            <thead class="border-b border-border text-[10px] uppercase tracking-[0.18em] text-muted-foreground">
              <tr>
                <th class="px-4 py-3 font-medium">{{ t.node }}</th>
                <th class="px-4 py-3 font-medium">{{ t.status }}</th>
                <th class="px-4 py-3 font-medium">CPU</th>
                <th class="px-4 py-3 font-medium">{{ t.memory }}</th>
                <th class="px-4 py-3 font-medium">{{ t.disk }}</th>
                <th class="px-4 py-3 font-medium">{{ t.network }}</th>
                <th class="px-4 py-3 font-medium">{{ t.uptime }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="node in visibleNodes" :key="node.uuid" class="border-b border-border/80 last:border-0">
                <td class="px-4 py-3">
                  <p class="font-medium uppercase tracking-[0.08em]">{{ node.name }}</p>
                  <p class="mt-1 text-[10px] uppercase tracking-[0.14em] text-muted-foreground">{{ node.group || t.defaultGroup }} · {{ node.region || t.unknown }}</p>
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

    </main>

    <footer class="mx-auto flex w-full max-w-7xl flex-col gap-2 px-4 pb-8 pt-2 font-mono text-[10px] uppercase tracking-[0.18em] text-muted-foreground sm:flex-row sm:items-center sm:justify-between sm:px-6 lg:px-8">
      <p>Powered by Komari Monitor.</p>
      <p>{{ t.footer }}</p>
    </footer>
  </div>
</template>
