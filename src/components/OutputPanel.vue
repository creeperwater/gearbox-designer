<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch, nextTick } from 'vue'
import * as echarts from 'echarts'

type DiagramModel = {
  shafts: string[]
  yAxisLabels: string[]
  lines: Array<{ coords: [[string, string], [string, string]] }>
  scatterPoints: Array<[string, string]>
  middleShaftIndices: number[]
}

const props = defineProps<{
  diagramData: DiagramModel | null
}>()

const chartRef = ref<HTMLElement | null>(null)
let chartInstance: echarts.ECharts | null = null

const dynamicHeight = ref('350px')

// ------- 拖动偏移状态 -------
const shaftOffsets = ref<Record<number, number>>({})
const isDragging = ref(false)
const dragShaftIndex = ref<number | null>(null)
const dragStartPixelY = ref(0)
const dragOriginOffset = ref(0)
const cachedSpacing = ref(40) // 拖动开始时缓存 Y 类别像素间距，避免每次 mousemove 都调用 convertToPixel

// ------- 工具函数 -------
function findYLabelIndex(yLabel: string, labels: string[]): number {
  return labels.indexOf(yLabel)
}

function shiftYLabel(yLabel: string, labels: string[], offset: number): string | null {
  const idx = findYLabelIndex(yLabel, labels)
  if (idx === -1) return null
  const newIdx = idx + offset
  if (newIdx < 0 || newIdx >= labels.length) return null
  return labels[newIdx]!
}

// 裁剪偏移量：确保该轴所有散点偏移后仍在 yAxisLabels 范围内
function clampOffsetForShaft(shaftIdx: number, offset: number, data: DiagramModel): number {
  const labels = data.yAxisLabels
  const shaftName = data.shafts[shaftIdx]!

  const pointIndices: number[] = []
  for (const pt of data.scatterPoints) {
    if (pt[0] === shaftName) {
      const idx = findYLabelIndex(pt[1], labels)
      if (idx !== -1) pointIndices.push(idx)
    }
  }

  if (pointIndices.length === 0) return offset

  const minIdx = Math.min(...pointIndices)
  const maxIdx = Math.max(...pointIndices)

  const minOffset = -minIdx
  const maxOffset = labels.length - 1 - maxIdx

  return Math.max(minOffset, Math.min(maxOffset, offset))
}

// ------- 计算默认居中偏移 -------
// 将中间轴的散点放到 Y 轴靠近中间的位置
function computeDefaultOffsets(data: DiagramModel): Record<number, number> {
  const offsets: Record<number, number> = {}
  const labels = data.yAxisLabels
  // Y 轴中心索引
  const targetCenter = (labels.length - 1) / 2

  for (const shaftIdx of data.middleShaftIndices) {
    const shaftName = data.shafts[shaftIdx]!

    // 找到该轴散点的 Y 索引范围
    const pointIndices: number[] = []
    for (const pt of data.scatterPoints) {
      if (pt[0] === shaftName) {
        const idx = findYLabelIndex(pt[1], labels)
        if (idx !== -1) pointIndices.push(idx)
      }
    }

    if (pointIndices.length === 0) continue

    // 当前散点群中心 → 目标中心
    const currentCenter = (Math.min(...pointIndices) + Math.max(...pointIndices)) / 2
    const offset = Math.round(targetCenter - currentCenter)

    offsets[shaftIdx] = clampOffsetForShaft(shaftIdx, offset, data)
  }

  return offsets
}

// 方案切换时：计算居中偏移 + 全量重绘
watch(
  () => props.diagramData?.shafts,
  () => {
    if (props.diagramData) {
      shaftOffsets.value = computeDefaultOffsets(props.diagramData)
    } else {
      shaftOffsets.value = {}
    }
    fullRedraw()
  }
)

// ------- 偏移后的展示数据 -------
const adjustedData = computed(() => {
  const data = props.diagramData
  if (!data) return null

  const offsets = shaftOffsets.value
  const labels = data.yAxisLabels
  const middleIndices = data.middleShaftIndices
  const shaftNames = data.shafts

  function getOffsetForShaft(shaftName: string): number {
    const idx = shaftNames.indexOf(shaftName)
    if (idx === -1 || !middleIndices.includes(idx)) return 0
    return offsets[idx] ?? 0
  }

  // 偏移散点
  const adjustedScatter: Array<[string, string]> = []
  for (const pt of data.scatterPoints) {
    const [shaftName, yLabel] = pt
    const offset = getOffsetForShaft(shaftName)
    if (offset === 0) {
      adjustedScatter.push(pt)
    } else {
      const newLabel = shiftYLabel(yLabel, labels, offset)
      if (newLabel != null) {
        adjustedScatter.push([shaftName, newLabel])
      }
    }
  }

  // 偏移连线
  const adjustedLines: Array<{ coords: [[string, string], [string, string]] }> = []
  for (const line of data.lines) {
    const [[startShaft, startY], [endShaft, endY]] = line.coords
    const startOffset = getOffsetForShaft(startShaft)
    const endOffset = getOffsetForShaft(endShaft)

    const newStartY = startOffset !== 0 ? shiftYLabel(startY, labels, startOffset) : startY
    const newEndY = endOffset !== 0 ? shiftYLabel(endY, labels, endOffset) : endY

    if (newStartY != null && newEndY != null) {
      adjustedLines.push({ coords: [[startShaft, newStartY], [endShaft, newEndY]] })
    }
  }

  return {
    shafts: shaftNames,
    yAxisLabels: labels,
    lines: adjustedLines,
    scatterPoints: adjustedScatter,
    middleShaftIndices: middleIndices
  }
})

// ------- 区分可拖动/锁定点的样式 -------
const scatterDataWithStyle = computed(() => {
  const data = adjustedData.value
  if (!data) return []

  return data.scatterPoints.map(pt => {
    const [shaftName] = pt
    const shaftIdx = data.shafts.indexOf(shaftName)
    const isDraggable = data.middleShaftIndices.includes(shaftIdx)

    return {
      value: pt,
      symbolSize: isDraggable ? 14 : 10,
      itemStyle: isDraggable
        ? { color: '#1867c0', borderColor: '#1867c0', borderWidth: 2 }
        : { color: '#fff', borderColor: '#999', borderWidth: 2 }
    }
  })
})

// ------- 渲染图表 -------
// 全量重绘：用于方案切换、初始化
async function fullRedraw() {
  if (!chartInstance) return

  const data = adjustedData.value

  if (!data || data.shafts.length === 0) {
    chartInstance.setOption({
      xAxis: { data: ['Ⅰ', 'Ⅱ'] },
      yAxis: { data: [] },
      series: []
    }, { replaceMerge: ['xAxis', 'yAxis', 'series'] })
    return
  }

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
        data: scatterDataWithStyle.value,
        symbol: 'circle',
        z: 10
      }
    ]
  }, { replaceMerge: ['xAxis', 'yAxis', 'series'] })
}

// 增量更新：拖动过程中只更新 series 数据，关闭动画实现闪现跟手
function dragUpdate() {
  if (!chartInstance || !adjustedData.value) return

  chartInstance.setOption({
    animation: false,
    series: [
      {
        type: 'lines',
        coordinateSystem: 'cartesian2d',
        data: adjustedData.value.lines,
        lineStyle: { color: '#1867c0', width: 2, type: 'solid', opacity: 1 },
        silent: true
      },
      {
        type: 'scatter',
        data: scatterDataWithStyle.value,
        symbol: 'circle',
        z: 10
      }
    ]
  })
  // 不用 replaceMerge，ECharts 会增量合并 series 数据
}

// ------- 拖动交互 -------
function computeCategorySpacing(): number {
  if (!chartInstance || !adjustedData.value) return 40
  const labels = adjustedData.value.yAxisLabels
  if (labels.length < 2) return 40

  try {
    const y0 = chartInstance.convertToPixel({ yAxisIndex: 0 }, labels[0]!)
    const y1 = chartInstance.convertToPixel({ yAxisIndex: 0 }, labels[1]!)
    if (y0 != null && y1 != null) {
      return Math.abs(y1 - y0)
    }
  } catch {
    // 回退
  }
  return 40
}

function onChartMouseDown(params: any) {
  if (params.seriesType !== 'scatter') return
  if (!adjustedData.value) return
  const data = adjustedData.value

  const shaftName = params.data?.value?.[0] ?? params.data?.[0]
  if (!shaftName) return

  const shaftIdx = data.shafts.indexOf(shaftName)
  if (!data.middleShaftIndices.includes(shaftIdx)) return

  isDragging.value = true
  dragShaftIndex.value = shaftIdx
  dragStartPixelY.value = params.event?.event?.clientY ?? 0
  dragOriginOffset.value = shaftOffsets.value[shaftIdx] ?? 0
  // 缓存间距，拖动期间不再重复计算
  cachedSpacing.value = computeCategorySpacing()

  if (chartRef.value) {
    chartRef.value.style.cursor = 'grabbing'
  }
}

function onDomMouseMove(e: MouseEvent) {
  if (!isDragging.value || dragShaftIndex.value == null) return

  const pixelDelta = e.clientY - dragStartPixelY.value
  // Y 轴从上到下，clientY 增大 = 向下拖 = 级别减小
  // 向上拖（clientY 减小）= 偏移量增加（往高级别移）
  const levelDelta = -Math.round(pixelDelta / cachedSpacing.value)

  const newOffset = dragOriginOffset.value + levelDelta

  const data = props.diagramData
  if (!data) return

  const clampedOffset = clampOffsetForShaft(dragShaftIndex.value, newOffset, data)

  shaftOffsets.value = {
    ...shaftOffsets.value,
    [dragShaftIndex.value]: clampedOffset
  }

  // 跟手：增量更新，只刷新散点和连线
  dragUpdate()
}

function onDomMouseUp() {
  if (!isDragging.value) return

  isDragging.value = false
  dragShaftIndex.value = null

  if (chartRef.value) {
    chartRef.value.style.cursor = ''
  }
}

// ------- 生命周期 -------
onMounted(() => {
  if (chartRef.value) {
    chartInstance = echarts.init(chartRef.value)

    // 初始化时计算居中偏移 + 全量重绘
    if (props.diagramData) {
      shaftOffsets.value = computeDefaultOffsets(props.diagramData)
    }
    fullRedraw()

    // ECharts mousedown
    chartInstance.on('mousedown', onChartMouseDown)

    // DOM 鼠标事件
    chartRef.value.addEventListener('mousemove', onDomMouseMove)
    window.addEventListener('mouseup', onDomMouseUp)
    window.addEventListener('resize', handleResize)
  }
})

onUnmounted(() => {
  if (chartInstance) {
    chartInstance.off('mousedown', onChartMouseDown)
    chartInstance.dispose()
    chartInstance = null
  }
  if (chartRef.value) {
    chartRef.value.removeEventListener('mousemove', onDomMouseMove)
  }
  window.removeEventListener('mouseup', onDomMouseUp)
  window.removeEventListener('resize', handleResize)
})

// 基础数据变化时的全量重绘（偏移已在 watch(shafts) 中处理）
watch(() => props.diagramData, () => {
  fullRedraw()
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
        <div v-if="diagramData && diagramData.middleShaftIndices.length > 0" class="text-caption text-grey mt-2 text-center">
          拖动中间轴上的蓝色圆点可整轴上下平移
        </div>
      </div>
    </v-card-text>
  </v-card>
</template>
