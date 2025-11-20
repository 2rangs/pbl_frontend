<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  LogarithmicScale,
  PointElement,
  LineElement,
  BarElement,
  ArcElement,
  Tooltip,
  Legend,
} from 'chart.js'
import type { ChartOptions } from 'chart.js'
import { Line, Bar, Pie } from 'vue-chartjs'

// Chart.js 전역 등록
ChartJS.register(
    CategoryScale,
    LinearScale,
    LogarithmicScale,
    PointElement,
    LineElement,
    BarElement,
    ArcElement,
    Tooltip,
    Legend,
)

// === API 기본 설정 ===
const API_BASE =
    import.meta.env.VITE_API_BASE_URL || 'http://54.180.31.33:8000'

// === 타입 정의 ===
interface OverviewCards {
  total: number
  anomalies: number
  anomaly_rate: number
  threshold: number
}

interface TimelineSeries {
  name: string
  data: number[]
}
interface TimelineResponse {
  labels: string[]
  series: TimelineSeries[]
}

interface SeverityResponse {
  labels: string[]
  data: number[]
}

interface DistributionResponse {
  bins: number[]
  counts: number[]
}

interface TrendResponse {
  labels: string[]
  data: number[]
}

interface PieBarResponse {
  labels: string[]
  data: number[]
}

interface SampleRow {
  Destination?: string
  Dst_Port?: number
  Flow_ID?: string
  Length?: number
  Length_Scaled?: number
  Protocol?: string
  Protocol_Enc?: number
  Source?: string
  Src_Port?: number
  Time?: string
  anomaly_score?: number
  __source_file__?: string
}

interface PingResponse {
  status: string
}

// === 전역 상태 ===
const globalLoading = ref(true)
const apiHealthy = ref<boolean | null>(null)
const lastUpdated = ref<Date | null>(null)

// KPI
const cards = ref<OverviewCards>({
  total: 0,
  anomalies: 0,
  anomaly_rate: 0,
  threshold: 0,
})

// 차트 원본 데이터
const timelineRaw = ref<TimelineResponse>({ labels: [], series: [] })
const severityRaw = ref<SeverityResponse>({ labels: [], data: [] })
const distributionRaw = ref<DistributionResponse>({ bins: [], counts: [] })
const trendRaw = ref<TrendResponse>({ labels: [], data: [] })
const topDstRaw = ref<PieBarResponse>({ labels: [], data: [] })

// 테이블
const tableRows = ref<SampleRow[]>([])
const tableLoading = ref(false)

// === 유틸 ===
function formatDateLabel(label: string) {
  const d = new Date(label)
  if (isNaN(d.getTime())) return label
  const y = d.getFullYear()
  const m = String(d.getMonth() + 1).padStart(2, '0')
  const day = String(d.getDate()).padStart(2, '0')
  return `${y}-${m}-${day}`
}

// processed_packets_20251111_111134.csv → 2025-11-11
function filenameToDateLabel(label: string) {
  const m = label.match(/(\d{4})(\d{2})(\d{2})/)
  if (!m) return label
  const [, y, mo, d] = m
  return `${y}-${mo}-${d}`
}

// 타임라인용 다운샘플 인덱스 생성 (프론트 하드코딩)
function buildDownsampleIndices(length: number, target = 300): number[] {
  if (length <= target) {
    return Array.from({ length }, (_, i) => i)
  }
  const step = Math.ceil(length / target)
  const indices: number[] = []
  for (let i = 0; i < length; i += step) {
    indices.push(i)
  }
  if (indices[indices.length - 1] !== length - 1) {
    indices.push(length - 1)
  }
  return indices
}

async function safeFetchJson<T>(url: string): Promise<T> {
  const res = await fetch(url)
  if (!res.ok) throw new Error(`${url} 요청 실패 (${res.status})`)
  return res.json()
}

// === API 호출 ===
async function fetchAll() {
  globalLoading.value = true
  tableLoading.value = true

  try {
    const ping = await safeFetchJson<PingResponse>(`${API_BASE}/ping`)
    apiHealthy.value = ping.status === 'ok' || ping.status === 'alive'

    const [
      overview,
      timeline,
      severity,
      distribution,
      trend,
      topDst,
      table,
    ] = await Promise.all([
      safeFetchJson<OverviewCards>(`${API_BASE}/cards/overview`),
      safeFetchJson<TimelineResponse>(
          `${API_BASE}/charts/timeline?bucket=min&tz=Asia/Seoul`,
      ),
      safeFetchJson<SeverityResponse>(`${API_BASE}/charts/severity`),
      safeFetchJson<DistributionResponse>(
          `${API_BASE}/charts/distribution`,
      ),
      safeFetchJson<TrendResponse>(`${API_BASE}/charts/trend`),
      safeFetchJson<PieBarResponse>(
          `${API_BASE}/charts/top/dst_ip?n=5`,
      ),
      safeFetchJson<{ rows: SampleRow[] }>(
          `${API_BASE}/table/samples?n=10`,
      ),
    ])

    cards.value = overview
    timelineRaw.value = timeline
    severityRaw.value = severity
    distributionRaw.value = distribution
    trendRaw.value = trend
    topDstRaw.value = topDst
    tableRows.value = table.rows ?? []
    lastUpdated.value = new Date()
  } catch (e) {
    console.error(e)
    apiHealthy.value = false
    tableRows.value = []
  } finally {
    globalLoading.value = false
    tableLoading.value = false
  }
}

onMounted(fetchAll)

// === KPI ===
const anomalyRatePercent = computed(
    () => ((cards.value.anomaly_rate ?? 0) * 100).toFixed(2),
)

// === 한글 라벨 매핑 ===
const timelineLabelMap: Record<string, string> = {
  total: '전체 트래픽',
  anomalies: '이상 트래픽',
  normal: '정상 트래픽',
  rate: '이상 비율',
}

const severityLabelMap: Record<string, string> = {
  low: '낮음',
  medium: '보통',
  high: '높음',
  critical: '심각',
}

// === Chart 공통 ===
const axisTextColor = '#64748b'
const gridColor = '#e2e8f0'
const lineColors = ['#16a34a', '#ef4444', '#64748b', '#2563eb', '#f97316']
const noAnim = { animation: false as const }

// 1) 시간대별 트래픽 변화 (프론트 다운샘플링)
const timelineChartData = computed(() => {
  const rawLabels = timelineRaw.value.labels
  const series = timelineRaw.value.series

  if (!rawLabels.length || !series.length) {
    return { labels: [] as string[], datasets: [] as any[] }
  }

  // 프론트 하드코딩: 최대 300 포인트 정도만 사용
  const indices = buildDownsampleIndices(rawLabels.length, 300)
  const labels = indices.map((i) => formatDateLabel(rawLabels[i]))

  const datasets = series.map((s, idx) => ({
    label: timelineLabelMap[s.name] ?? s.name,
    data: indices.map((i) => s.data[i] ?? null),
    borderWidth: 2,
    borderColor: lineColors[idx % lineColors.length],
    backgroundColor: lineColors[idx % lineColors.length] + '33',
    tension: 0.4,
    cubicInterpolationMode: 'monotone' as const,
    borderCapStyle: 'round' as const,
    borderJoinStyle: 'round' as const,
    pointRadius: 0,
    pointHoverRadius: 3,
    pointHitRadius: 6,
  }))

  return { labels, datasets }
})

const timelineChartOptions: ChartOptions<'line'> = {
  responsive: true,
  maintainAspectRatio: false,
  ...noAnim,
  interaction: {
    mode: 'index',
    intersect: false,
  },
  plugins: {
    legend: {
      labels: { color: axisTextColor, font: { size: 11 } },
    },
  },
  scales: {
    x: {
      ticks: {
        color: axisTextColor,
        maxRotation: 0,
        autoSkip: true,
        maxTicksLimit: 8,
      },
      grid: { color: gridColor },
    },
    y: {
      beginAtZero: true,
      ticks: { color: axisTextColor },
      grid: { color: gridColor },
    },
  },
}

// 2) 심각도 도넛
const severityChartData = computed(() => {
  const colors = ['#16a34a', '#60a5fa', '#f97316', '#ef4444']
  const hoverColors = ['#15803d', '#3b82f6', '#ea580c', '#dc2626']

  const rawLabels = severityRaw.value.labels
  const displayLabels = rawLabels.map(
      (l) => severityLabelMap[l] ?? l,
  )

  return {
    labels: displayLabels,
    datasets: [
      {
        data: severityRaw.value.data,
        backgroundColor: rawLabels.map(
            (_l, i) => colors[i % colors.length],
        ),
        hoverBackgroundColor: rawLabels.map(
            (_l, i) => hoverColors[i % hoverColors.length],
        ),
        borderWidth: 2,
        borderColor: '#ffffff',
        hoverOffset: 8,
      },
    ],
  }
})

const pieOptions: ChartOptions<'pie'> = {
  responsive: true,
  maintainAspectRatio: false,
  ...noAnim,
  cutout: '70%',
  plugins: {
    legend: {
      position: 'bottom',
      labels: {
        color: axisTextColor,
        font: { size: 11 },
        usePointStyle: true,
        padding: 8,
      },
    },
    tooltip: {
      backgroundColor: '#0f172a',
      borderColor: '#1e293b',
      borderWidth: 1,
      titleColor: '#e5e7eb',
      bodyColor: '#cbd5f5',
      padding: 10,
      callbacks: {
        label: (ctx: any) => {
          const label = ctx.label ?? ''
          const value = ctx.parsed ?? 0
          const dataArr = ctx.chart.data.datasets[0].data as number[]
          const total =
              dataArr.reduce(
                  (sum, v) => sum + (typeof v === 'number' ? v : 0),
                  0,
              ) || 1
          const ratio = ((value / total) * 100).toFixed(1)
          return `${label}: ${value.toLocaleString()}건 (${ratio}%)`
        },
      },
    },
  },
  layout: { padding: 8 },
}

// 3) 이상 점수 히스토그램
const distributionChartData = computed(() => {
  const bins = distributionRaw.value.bins
  const counts = distributionRaw.value.counts

  if (!bins.length || !counts.length) {
    return {
      labels: [] as string[],
      datasets: [
        {
          label: '샘플 수',
          data: [] as number[],
          backgroundColor: 'rgba(37, 99, 235, 0.7)',
          borderColor: '#2563eb',
          borderWidth: 1,
        },
      ],
    }
  }

  const labels: string[] = []
  for (let i = 0; i < bins.length - 1; i++) {
    const start = bins[i]
    const end = bins[i + 1]
    if (start == null || end == null) continue
    labels.push(`${start.toFixed(1)}–${end.toFixed(1)}`)
  }

  return {
    labels,
    datasets: [
      {
        label: '샘플 수',
        data: counts,
        backgroundColor: 'rgba(37, 99, 235, 0.7)',
        borderColor: '#2563eb',
        borderWidth: 1,
      },
    ],
  }
})

const barOptions: ChartOptions<'bar'> = {
  responsive: true,
  maintainAspectRatio: false,
  ...noAnim,
  plugins: { legend: { display: false } },
  scales: {
    x: {
      ticks: { color: axisTextColor },
      grid: { color: gridColor },
    },
    y: {
      type: 'logarithmic',
      min: 1,
      ticks: { color: axisTextColor },
      grid: { color: gridColor },
    } as any,
  },
}

// 4) 날짜별 평균 이상 점수 (임계값 기준 색 분리 + 축 확대)
const trendChartData = computed(() => {
  const labels = trendRaw.value.labels.map(filenameToDateLabel)
  const data = trendRaw.value.data
  const threshold = cards.value.threshold ?? 0

  const backgroundColors = data.map((v) =>
      v >= threshold ? 'rgba(239,68,68,0.85)' : 'rgba(59,130,246,0.85)',
  )
  const borderColors = data.map((v) =>
      v >= threshold ? '#ef4444' : '#3b82f6',
  )

  return {
    labels,
    datasets: [
      {
        label: `평균 이상 점수 (임계값 ${threshold.toFixed(2)})`,
        data,
        backgroundColor: backgroundColors,
        borderColor: borderColors,
        borderWidth: 1,
        borderRadius: 6,
        barThickness: 32,
        maxBarThickness: 40,
      },
    ],
  }
})

const trendOptions = computed<ChartOptions<'bar'>>(() => {
  const data = trendRaw.value.data

  if (!data.length) {
    return {
      responsive: true,
      maintainAspectRatio: false,
      ...noAnim,
      plugins: { legend: { display: false } },
      scales: {
        x: {
          ticks: { color: axisTextColor },
          grid: { color: gridColor },
        },
        y: {
          beginAtZero: true,
          ticks: { color: axisTextColor },
          grid: { color: gridColor },
        },
      },
    }
  }

  const min = Math.min(...data)
  const max = Math.max(...data)
  const range = max - min || 0.01
  const padding = range * 0.3
  const suggestedMin = Math.max(0, min - padding)
  const suggestedMax = max + padding

  return {
    responsive: true,
    maintainAspectRatio: false,
    ...noAnim,
    plugins: { legend: { display: false } },
    scales: {
      x: {
        ticks: {
          color: axisTextColor,
          autoSkip: true,
          maxTicksLimit: 8,
          maxRotation: 0,
        },
        grid: { color: gridColor },
      },
      y: {
        beginAtZero: false,
        min: suggestedMin,
        max: suggestedMax,
        ticks: { color: axisTextColor },
        grid: { color: gridColor },
      },
    },
  }
})

// 5) 상위 목적지 IP
const topDstChartData = computed(() => ({
  labels: topDstRaw.value.labels,
  datasets: [
    {
      label: '이상 건수',
      data: topDstRaw.value.data,
      backgroundColor: 'rgba(239,68,68,0.75)',
      borderColor: '#ef4444',
      borderWidth: 1,
      borderRadius: 10,
      barThickness: 20,
      maxBarThickness: 28,
    },
  ],
}))

const topDstOptions: ChartOptions<'bar'> = {
  responsive: true,
  maintainAspectRatio: false,
  ...noAnim,
  indexAxis: 'y',
  plugins: { legend: { display: false } },
  scales: {
    x: {
      beginAtZero: true,
      ticks: {
        color: axisTextColor,
        callback: (value) =>
            typeof value === 'number' ? value.toLocaleString() : value,
      },
      grid: { color: gridColor },
    },
    y: {
      ticks: { color: axisTextColor },
      grid: { color: gridColor },
    },
  },
}
</script>

<template>
  <div class="min-h-screen bg-slate-100">
    <div class="max-w-7xl mx-auto px-4 py-6 space-y-6">
      <!-- 헤더 -->
      <section
          class="bg-white border border-slate-200 rounded-xl shadow-sm px-6 py-4 flex flex-col sm:flex-row sm:items-center sm:justify-between gap-3"
      >
        <div>
          <h1 class="text-xl font-semibold tracking-tight">
            네트워크 트래픽 분석 대시보드
          </h1>
          <p class="mt-1 text-xs text-slate-500">
            제작자 : 김이랑 & 최현석
          </p>
          <p v-if="lastUpdated" class="mt-1 text-[11px] text-slate-400">
            마지막 갱신: {{ lastUpdated.toLocaleString() }}
          </p>
        </div>

        <div class="flex flex-col items-end gap-2">
          <button
              class="text-xs px-3 py-1 rounded-full border border-slate-300 bg-white hover:bg-slate-50"
              @click="fetchAll"
          >
            {{ globalLoading ? '데이터 갱신 중…' : '데이터 새로고침' }}
          </button>
        </div>
      </section>

      <!-- KPI -->
      <section class="grid grid-cols-2 lg:grid-cols-4 gap-4">
        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4">
          <div class="text-[11px] text-slate-500">전체 트래픽</div>
          <div class="mt-1 text-xl font-semibold">
            {{ cards.total.toLocaleString() }}
          </div>
        </article>

        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4">
          <div class="text-[11px] text-slate-500">이상 건수</div>
          <div class="mt-1 text-xl font-semibold">
            {{ cards.anomalies.toLocaleString() }}
          </div>
        </article>

        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4">
          <div class="text-[11px] text-slate-500">고위험 임계값</div>
          <div class="mt-1 text-xl font-semibold">
            {{ cards.threshold.toFixed(2) }}
          </div>
        </article>

        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4">
          <div class="text-[11px] text-slate-500">이상 비율</div>
          <div class="mt-1 text-xl font-semibold">
            {{ anomalyRatePercent }}%
          </div>
        </article>
      </section>

      <!-- 1행: 시간대별 트래픽 변화 -->
      <section class="bg-white border border-slate-200 rounded-xl px-5 py-4 h-72">
        <h2 class="text-sm font-semibold mb-2">시간대별 트래픽 변화</h2>
        <Line :data="timelineChartData" :options="timelineChartOptions" />
      </section>

      <!-- 2행: 심각도 / 히스토그램 -->
      <section class="grid grid-cols-1 md:grid-cols-2 gap-4">
        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4 h-72">
          <h2 class="text-sm font-semibold mb-2">심각도 분포</h2>
          <div
              class="h-[calc(100%-1.5rem)] flex items-center justify-center overflow-hidden"
          >
            <Pie :data="severityChartData" :options="pieOptions" />
          </div>
        </article>

        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4 h-72">
          <h2 class="text-sm font-semibold mb-2">이상 점수 별 히스토그램</h2>
          <Bar :data="distributionChartData" :options="barOptions" />
        </article>
      </section>

      <!-- 3행: 날짜별 평균 점수 / 상위 목적지 IP -->
      <section class="grid grid-cols-1 md:grid-cols-2 gap-4">
        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4 h-72">
          <h2 class="text-sm font-semibold mb-2">날짜별 평균 이상 점수</h2>
          <Bar :data="trendChartData" :options="trendOptions" />
        </article>

        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4 h-72">
          <h2 class="text-sm font-semibold mb-2">상위 목적지 IP</h2>
          <Bar :data="topDstChartData" :options="topDstOptions" />
        </article>
      </section>

      <!-- 4행: 전체 테이블 -->
      <section class="bg-white border border-slate-200 rounded-xl px-5 py-4">
        <div class="flex justify-between items-center mb-3">
          <h2 class="text-sm font-semibold">이상 트래픽 상위 샘플</h2>
          <button
              class="text-xs px-3 py-1 rounded-full border border-slate-300 bg-white hover:bg-slate-50"
              @click="fetchAll"
          >
            {{ tableLoading ? '로딩 중…' : '테이블 새로고침' }}
          </button>
        </div>

        <div v-if="tableLoading" class="text-xs text-slate-500 py-3">
          테이블 로딩 중…
        </div>

        <div v-else class="overflow-x-auto">
          <table class="w-full text-xs border-collapse">
            <thead
                class="bg-slate-50 text-slate-600 border-b border-slate-200"
            >
            <tr>
              <th class="py-2 px-3 text-left">#</th>
              <th class="py-2 px-3 text-left">Time</th>
              <th class="py-2 px-3 text-left">Source</th>
              <th class="py-2 px-3 text-left">Src_Port</th>
              <th class="py-2 px-3 text-left">Destination</th>
              <th class="py-2 px-3 text-left">Dst_Port</th>
              <th class="py-2 px-3 text-left">Protocol</th>
              <th class="py-2 px-3 text-left">anomaly_score</th>
            </tr>
            </thead>
            <tbody>
            <tr
                v-for="(row, idx) in tableRows"
                :key="idx"
                class="border-b border-slate-100 hover:bg-slate-50"
            >
              <td class="py-2 px-3 text-slate-500">
                {{ idx + 1 }}
              </td>
              <td class="py-2 px-3">
                {{ row.Time ?? '-' }}
              </td>
              <td class="py-2 px-3 font-mono">
                {{ row.Source ?? '-' }}
              </td>
              <td class="py-2 px-3">
                {{ row.Src_Port ?? '-' }}
              </td>
              <td class="py-2 px-3 font-mono">
                {{ row.Destination ?? '-' }}
              </td>
              <td class="py-2 px-3">
                {{ row.Dst_Port ?? '-' }}
              </td>
              <td class="py-2 px-3">
                {{ row.Protocol ?? '-' }}
              </td>
              <td class="py-2 px-3 font-mono text-blue-600">
                {{
                  row.anomaly_score != null
                      ? row.anomaly_score.toFixed(3)
                      : '-'
                }}
              </td>
            </tr>

            <tr v-if="!tableRows.length">
              <td
                  colspan="9"
                  class="py-4 text-center text-[11px] text-slate-400"
              >
                표시할 데이터가 없습니다.
              </td>
            </tr>
            </tbody>
          </table>
        </div>
      </section>
    </div>
  </div>
</template>
