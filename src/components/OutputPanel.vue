<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, nextTick } from 'vue'
import * as echarts from 'echarts'

type DiagramModel = {
  shafts: string[]
  yAxisLabels: string[]
  lines: Array<{ coords: [[string, string], [string, string]] }>
  scatterPoints: Array<[string, string]>
}

const props = defineProps<{
  diagramData: DiagramModel | null
}>()

const chartRef = ref<HTMLElement | null>(null)
let chartInstance: echarts.ECharts | null = null

const dynamicHeight = ref('350px')

const updateChart = async () => {
  if (!chartInstance) return

  const data = props.diagramData

  if (!data || data.shafts.length === 0) {
    chartInstance.setOption({
      xAxis: { data: ['Ⅰ', 'Ⅱ'] },
      yAxis: { data: [] },
      series: []
    }, { replaceMerge: ['xAxis', 'yAxis', 'series'] })
    return
  }

  // 动态计算高度，每级约 40px，加上上下留白
  const calculatedHeight = Math.max(350, data.yAxisLabels.length * 40 + 60)
  dynamicHeight.value = `${calculatedHeight}px`

  await nextTick()
  chartInstance.resize()

  chartInstance.setOption({
    grid: {
      top: 40, bottom: 20, left: 40, right: 40,
      show: true, borderColor: '#000', borderWidth: 1
    },
    xAxis: {
      type: 'category',
      data: data.shafts,
      boundaryGap: false, position: 'top',
      axisLine: { show: false }, axisTick: { show: false },
      splitLine: { show: true, lineStyle: { color: '#000' } },
      axisLabel: { fontSize: 16, fontWeight: 'bold', color: '#000', margin: 10 }
    },
    yAxis: {
      type: 'category',
      data: data.yAxisLabels,
      boundaryGap: false, position: 'right',
      axisLine: { show: false }, axisTick: { show: false },
      splitLine: { show: true, lineStyle: { color: '#000' } },
      axisLabel: { interval: 0, fontSize: 14, fontWeight: 'bold', color: '#000', margin: 10 }
    },
    series: [
      {
        type: 'lines',
        coordinateSystem: 'cartesian2d',
        data: data.lines,
        lineStyle: { color: '#1867c0', width: 2, type: 'solid', opacity: 1 },
        silent: true
      },
      {
        type: 'scatter',
        data: data.scatterPoints,
        symbol: 'circle', symbolSize: 10,
        itemStyle: { color: '#fff', borderColor: '#1867c0', borderWidth: 2, opacity: 1 },
        z: 10
      }
    ]
  }, { replaceMerge: ['xAxis', 'yAxis', 'series'] })
}

onMounted(() => {
  if (chartRef.value) {
    chartInstance = echarts.init(chartRef.value)
    updateChart()
    window.addEventListener('resize', handleResize)
  }
})

onUnmounted(() => {
  if (chartInstance) {
    chartInstance.dispose()
    window.removeEventListener('resize', handleResize)
  }
})

watch(() => props.diagramData, () => {
  updateChart()
}, { deep: true })

const handleResize = () => {
  chartInstance?.resize()
}
</script>

<template>
  <v-card elevation="2">
    <v-card-title class="text-h5 font-weight-bold">参考方案</v-card-title>
    <v-card-text>
      <div class="py-4">
        <div ref="chartRef" :style="{ width: '100%', maxWidth: '600px', height: dynamicHeight, margin: '0 auto' }"></div>
      </div>
    </v-card-text>
  </v-card>
</template>