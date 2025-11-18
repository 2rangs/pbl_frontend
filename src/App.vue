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
    LogarithmicScale, // ✅ 로그 스케일 등록
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

interface FileTimelineResponse {
  labels: string[]
  series: Record<string, number[]>
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
const distributionRaw = ref<DistributionResponse>({
  bins: [],
  counts: [],
})
const fileTimelineRaw = ref<FileTimelineResponse>({
  labels: [],
  series: {},
})

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
    // 1) ping 먼저 → 실패하면 나머지 호출 안 함
    const ping = await safeFetchJson<PingResponse>(`${API_BASE}/ping`)
    apiHealthy.value = ping.status === 'ok' || ping.status === 'alive'

    // 2) 나머지 병렬 호출
    const [
      overview,
      timeline,
      severity,
      distribution,
      fileTimeline,
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
      safeFetchJson<FileTimelineResponse>(
          `${API_BASE}/charts/file_timeline?bucket=day&tz=Asia/Seoul`,
      ),
      safeFetchJson<{ rows: SampleRow[] }>(
          `${API_BASE}/table/samples?n=10`,
      ),
    ])

    cards.value = overview
    timelineRaw.value = timeline
    severityRaw.value = severity
    distributionRaw.value = distribution
    fileTimelineRaw.value = fileTimeline
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

// === Chart 공통 ===
const axisTextColor = '#475569'
const gridColor = '#e2e8f0'
const lineColors = ['#16a34a', '#ef4444', '#64748b', '#2563eb', '#f97316']
const noAnim = { animation: false as const }

// 1) 시간대별 트래픽 변화 (곡선형 라인)
const timelineChartData = computed(() => ({
  labels: timelineRaw.value.labels.map(formatDateLabel),
  datasets: timelineRaw.value.series.map((s, idx) => ({
    label: s.name,
    data: s.data,
    borderWidth: 2,
    borderColor: lineColors[idx % lineColors.length],
    backgroundColor: lineColors[idx % lineColors.length] + '33',
    tension: 0.45,
    cubicInterpolationMode: 'monotone' as const,
    borderCapStyle: 'round' as const,
    borderJoinStyle: 'round' as const,
    pointRadius: 2,
    pointHoverRadius: 4,
  })),
}))

const timelineChartOptions: ChartOptions<'line'> = {
  responsive: true,
  maintainAspectRatio: false,
  ...noAnim,
  plugins: {
    legend: {
      labels: { color: axisTextColor, font: { size: 11 } },
    },
  },
  scales: {
    x: {
      ticks: { color: axisTextColor, maxRotation: 0 },
      grid: { color: gridColor },
    },
    y: {
      beginAtZero: true,
      ticks: { color: axisTextColor },
      grid: { color: gridColor },
    },
  },
}

// 2) 심각도 도넛 (예쁘게)
const severityChartData = computed(() => {
  const colors = ['#16a34a', '#60a5fa', '#f97316', '#ef4444']
  const hoverColors = ['#15803d', '#3b82f6', '#ea580c', '#dc2626']

  return {
    labels: severityRaw.value.labels,
    datasets: [
      {
        data: severityRaw.value.data,
        backgroundColor: severityRaw.value.labels.map(
            (_l, i) => colors[i % colors.length],
        ),
        hoverBackgroundColor: severityRaw.value.labels.map(
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
      position: 'bottom', // ✅ right → bottom으로 변경
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
  layout: {
    padding: 8, // ✅ 캔버스 주변 여유 공간
  },
}

// 3) 이상 점수 히스토그램 (로그 스케일)
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

    // TS2532 방어: undefined/null 값은 스킵
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
  plugins: {
    legend: { display: false },
  },
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

// 4) 날짜별 발생 추이 (곡선형 라인)
const fileTimelineChartData = computed(() => {
  const labels = fileTimelineRaw.value.labels.map(formatDateLabel)
  const series = fileTimelineRaw.value.series

  const datasets = Object.entries(series).map(([name, data], idx) => ({
    label: name,
    data,
    borderWidth: 2,
    borderColor: lineColors[(idx + 1) % lineColors.length],
    backgroundColor: lineColors[(idx + 1) % lineColors.length] + '33',
    tension: 0.45,
    cubicInterpolationMode: 'monotone' as const,
    borderCapStyle: 'round' as const,
    borderJoinStyle: 'round' as const,
    pointRadius: 2,
    pointHoverRadius: 4,
  }))

  return { labels, datasets }
})

const fileTimelineOptions: ChartOptions<'line'> = {
  responsive: true,
  maintainAspectRatio: false,
  ...noAnim,
  plugins: {
    legend: {
      labels: { color: axisTextColor, font: { size: 11 } },
    },
  },
  scales: {
    x: {
      ticks: { color: axisTextColor, maxRotation: 0 },
      grid: { color: gridColor },
    },
    y: {
      beginAtZero: true,
      ticks: { color: axisTextColor },
      grid: { color: gridColor },
    },
  },
}

// 5) 패킷 전송 순위 (Top5)
const rankRows = computed(() => tableRows.value.slice(0, 5))

const maxAnomalyScore = computed(() => {
  if (!rankRows.value.length) return 1
  const max = Math.max(
      ...rankRows.value.map((r) => r.anomaly_score ?? 0),
      0.001,
  )
  return max || 1
})

function barWidth(row: SampleRow) {
  const score = row.anomaly_score ?? 0
  return `${(score / maxAnomalyScore.value) * 100}%`
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
            로그 기반 LSTM 이상 탐지 결과를 시간·날짜·점수별로 시각화한 관제 화면
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
            <!-- ✅ 래퍼로 감싸서 영역 안에 고정 -->
            <Pie :data="severityChartData" :options="pieOptions" />
          </div>
        </article>

        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4 h-72">
          <h2 class="text-sm font-semibold mb-2">이상 점수 별 히스토그램</h2>
          <Bar :data="distributionChartData" :options="barOptions" />
        </article>
      </section>

      <!-- 3행: 날짜별 발생 추이 / 패킷 전송 순위 -->
      <section class="grid grid-cols-1 md:grid-cols-2 gap-4">
        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4 h-72">
          <h2 class="text-sm font-semibold mb-2">날짜별 발생 추이</h2>
          <Line :data="fileTimelineChartData" :options="fileTimelineOptions" />
        </article>

        <article class="bg-white border border-slate-200 rounded-xl px-5 py-4">
          <h2 class="text-sm font-semibold mb-3">패킷 전송 순위</h2>

          <div
              v-if="tableLoading"
              class="text-xs text-slate-500 py-4"
          >
            랭킹 데이터 로딩 중…
          </div>

          <div v-else-if="!rankRows.length" class="text-xs text-slate-400 py-4">
            표시할 랭킹 데이터가 없습니다.
          </div>

          <ol v-else class="space-y-2 text-xs">
            <li
                v-for="(row, idx) in rankRows"
                :key="idx"
                class="flex items-center gap-3"
            >
              <span class="w-4 text-right font-semibold text-slate-500">
                {{ idx + 1 }}.
              </span>
              <div class="flex-1">
                <div class="flex justify-between items-center mb-1">
                  <div class="font-medium text-slate-800 truncate">
                    {{ row.Source ?? row.Destination ?? '-' }}
                  </div>
                  <div class="text-[11px] text-slate-500 ml-2">
                    {{ row.anomaly_score?.toFixed(3) ?? '-' }}
                  </div>
                </div>
                <div class="h-2 w-full bg-slate-100 rounded-full overflow-hidden">
                  <div
                      class="h-full rounded-full bg-blue-500"
                      :style="{ width: barWidth(row) }"
                  ></div>
                </div>
              </div>
            </li>
          </ol>
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
              <th class="py-2 px-3 text-left">파일</th>
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
                {{ row.__source_file__ ?? '-' }}
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
