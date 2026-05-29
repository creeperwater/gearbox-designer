<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue'
import * as echarts from 'echarts'

const props = defineProps<{
  schema: number[] | null
  gearStages: number
}>()

const chartRef = ref<HTMLElement | null>(null)
let chartInstance: echarts.ECharts | null = null

const getRoman = (num: number) => {
  const romans = ['Ⅰ', 'Ⅱ', 'Ⅲ', 'Ⅳ', 'Ⅴ', 'Ⅵ', 'Ⅶ', 'Ⅷ', 'Ⅸ', 'Ⅹ', 'Ⅺ', 'Ⅻ']
  return romans[num - 1] || String(num)
}

const updateChart = () => {
  if (!chartInstance) return

  const stages = props.gearStages || 1
  const schema = props.schema

  const xData: string[] = []
  const yData: string[] = []

  // Y轴有变速级数那么多个坐标
  for (let i = 1; i <= stages; i++) {
    yData.push(`n${i}`)
  }

  // 没有选中时，清空数据呈现空白网格
  if (!schema || schema.length === 0) {
    chartInstance.setOption({
      xAxis: { data: ['Ⅰ'] },
      yAxis: { data: yData },
      series: [{ data: [] }, { data: [] }]
    }, { replaceMerge: ['xAxis', 'yAxis', 'series'] })
    return
  }

  // X轴竖线数量：几组传动就有多少+1个轴
  const shafts = schema.length + 1
  for (let i = 1; i <= shafts; i++) {
    xData.push(getRoman(i))
  }

  // 计算每一组传动的扩大步距 x 
  // 第一扩大组 x[0] = 1, 第二扩大组 x[1] = p[0]*x[0] ...
  const xMultipliers = [1]
  let currentX = 1
  for (let i = 0; i < schema.length; i++) {
    currentX *= (schema[i] as number)
    xMultipliers.push(currentX) 
  }

  const lines: any[] = []
  const points = new Set<string>() // 用 Set 避免重复节点，格式为 轴序号(0开始)-转速n

  // 从第一根轴的最高转速（1轴上 n_stages 的点）开始画结构网
  let currentLayer = [stages] 
  points.add(`0-${stages}`)

  for (let i = 0; i < schema.length; i++) {
    const nextLayer: number[] = []
    const p = schema[i] as number
    const xStep = xMultipliers[i] as number

    for (const u of currentLayer) {
      for (let j = 0; j < p; j++) {
        // 向下一根轴按步距分散
        const v = u - j * xStep 
        if (v >= 1 && v <= stages) {
          nextLayer.push(v)
          
          const startAxis = getRoman(i + 1)
          const endAxis = getRoman(i + 2)
          const startY = `n${u}`
          const endY = `n${v}`
          
          lines.push({ coords: [[startAxis, startY], [endAxis, endY]] })
          points.add(`${i+1}-${v}`)
        }
      }
    }
    // 过滤掉当前轴重复到达的点
    currentLayer = Array.from(new Set(nextLayer))
  }

  // 整理散点数据
  const scatterData = Array.from(points).map(pt => {
    const [axisIdx, yVal] = pt.split('-')
    return [getRoman(Number(axisIdx) + 1), `n${yVal}`]
  })

  // 使用 replaceMerge 更新使得即使列数变化依然顺滑过渡
  chartInstance.setOption({
    grid: {
      top: 40, bottom: 20, left: 40, right: 40,
      show: true, borderColor: '#000', borderWidth: 1
    },
    xAxis: {
      type: 'category',
      data: xData,
      boundaryGap: false, position: 'top',
      axisLine: { show: false }, axisTick: { show: false },
      splitLine: { show: true, lineStyle: { color: '#000' } },
      axisLabel: { fontSize: 16, fontWeight: 'bold', color: '#000', margin: 10 }
    },
    yAxis: {
      type: 'category',
      data: yData,
      boundaryGap: false, position: 'right', // 转速级数
      axisLine: { show: false }, axisTick: { show: false },
      splitLine: { show: true, lineStyle: { color: '#000' } },
      axisLabel: {
        // n1 -> n, 1设为sub下标
        formatter: (value: string) => value.replace('n', 'n{sub|') + '}',
        rich: { sub: { verticalAlign: 'bottom', fontSize: 10 } },
        fontSize: 14, fontWeight: 'bold', color: '#000', margin: 10
      }
    },
    series: [
      {
        type: 'lines',
        coordinateSystem: 'cartesian2d',
        data: lines,
        lineStyle: { color: '#1867c0', width: 2, type: 'solid', opacity: 1 },
        silent: true
      },
      {
        type: 'scatter',
        data: scatterData,
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

// 监听外界数据变化动态更新图表
watch(() => [props.schema, props.gearStages], () => {
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
        <!-- 这里插入 ECharts 容器，直接映射原图样式 -->
        <div ref="chartRef" style="width: 100%; max-width: 600px; height: 350px; margin: 0 auto;"></div>
      </div>
    </v-card-text>
  </v-card>
</template>