<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, computed, nextTick } from 'vue'
import * as echarts from 'echarts'

type ReferenceSpeedAxisModel = {
  values: number[]
  baseCount: number
  leftExtensionCount: number
  rightExtensionCount: number
}

const props = defineProps<{
  schema: number[] | null
  gearStages: number
  referenceSpeedAxis: ReferenceSpeedAxisModel | null
  inputSpeed: number | null
  expansionSteps: number[] | null
  expansionDirection: 'down' | 'up'
}>()

const chartRef = ref<HTMLElement | null>(null)
let chartInstance: echarts.ECharts | null = null

const getRoman = (num: number) => {
  const romans = ['Ⅰ', 'Ⅱ', 'Ⅲ', 'Ⅳ', 'Ⅴ', 'Ⅵ', 'Ⅶ', 'Ⅷ', 'Ⅸ', 'Ⅹ', 'Ⅺ', 'Ⅻ']
  return romans[num - 1] || String(num)
}

const formatSpeed = (value: number, index: number = 0) => String(Math.round(value)) + "\u200B".repeat(index)

const getYAxisData = () => {
  const axis = props.referenceSpeedAxis
  if (axis && axis.values.length > 0) {
    return axis.values.map((v, i) => formatSpeed(v, i))
  }

  const stages = props.gearStages || 1
  return Array.from({ length: stages }, (_, index) => `n${index + 1}`)
}

const getYAxisLabel = (level: number) => {
  const axis = props.referenceSpeedAxis
  if (!axis || axis.values.length === 0) {
    return `n${level}`
  }

  const position = axis.leftExtensionCount + level - 1
  const value = axis.values[position]
  return value == null ? null : formatSpeed(value, position)
}

// 计算离输入转速最近的转速级（在基础序列范围内查找，用于分支算法起始级）
const getInputSpeedLevel = () => {
  const axis = props.referenceSpeedAxis
  const inputSpeed = props.inputSpeed
  const stages = props.gearStages || 1

  if (!axis || axis.values.length === 0 || inputSpeed == null) {
    return stages // 无数据时默认取最高级
  }

  let bestLevel = stages
  let bestDiff = Infinity

  for (let level = 1; level <= stages; level++) {
    const position = axis.leftExtensionCount + level - 1
    const speed = axis.values[position]
    if (speed == null) continue
    const diff = Math.abs(speed - inputSpeed)
    if (diff < bestDiff) {
      bestLevel = level
      bestDiff = diff
    }
  }

  return bestLevel
}

// 在整个转速轴（含扩展序列）中查找离输入转速最近的Y轴标签
// 用于轴I上输入转速点的精确定位
const getClosestAxisValue = () => {
  const axis = props.referenceSpeedAxis
  const inputSpeed = props.inputSpeed

  if (!axis || axis.values.length === 0 || inputSpeed == null) return null

  let bestIndex = 0
  let bestDiff = Infinity

  for (let i = 0; i < axis.values.length; i++) {
    const speed = axis.values[i]!
    const diff = Math.abs(speed - inputSpeed)
    if (diff < bestDiff) {
      bestIndex = i
      bestDiff = diff
    }
  }

  return formatSpeed(axis.values[bestIndex]!, bestIndex)
}

const dynamicHeight = ref('350px')

const updateChart = async () => {
  if (!chartInstance) return

  const yData = getYAxisData()
  
  // 动态计算高度，每级约 40px，加上上下留白
  const calculatedHeight = Math.max(350, yData.length * 40 + 60)
  dynamicHeight.value = `${calculatedHeight}px`
  
  // 等待DOM更新后再调整组件大小
  await nextTick()
  chartInstance.resize()

  const stages = props.gearStages || 1
  const schema = props.schema

  const xData: string[] = []

  // 没有选中时，清空数据呈现空白网格
  if (!schema || schema.length === 0) {
    const defaultShafts = schema ? schema.length + 1 : 2
    for (let i = 1; i <= defaultShafts; i++) {
      xData.push(getRoman(i))
    }
    chartInstance.setOption({
      xAxis: { data: xData },
      yAxis: { data: yData },
      series: []
    }, { replaceMerge: ['xAxis', 'yAxis', 'series'] })
    return
  }

  // 竖轴数量 = 传动组数 + 1
  const shafts = schema.length + 1
  for (let i = 1; i <= shafts; i++) {
    xData.push(getRoman(i))
  }

  // 使用用户选择的扩大顺序步距
  const steps = props.expansionSteps ?? (() => {
    // 默认步距：标准扩大顺序
    const defaultSteps = [1]
    let current = 1
    for (let i = 0; i < schema.length; i++) {
      current *= (schema[i] as number)
      defaultSteps.push(current)
    }
    return defaultSteps.slice(0, schema.length)
  })()

  const lines: any[] = []
  const points = new Set<string>() // 用 Set 避免重复节点，格式为 轴序号(0开始)-转速n

  // 轴I上的输入转速点：在整个转速轴（含扩展）中找离输入转速最近的横轴位置
  const inputYLabel = getClosestAxisValue()
  if (inputYLabel != null) {
    points.add(`0-${inputYLabel}`)
  }

  // 根据传动方向确定锚点和扩展方向
  // 降速传动：锚点在最高级（stages），向下扩展
  // 升速传动：锚点在最低级（1），向上扩展
  const isDown = props.expansionDirection === 'down'
  const anchorLevel = isDown ? stages : 1
  let currentLayer = [anchorLevel]

  for (let i = 0; i < schema.length; i++) {
    const nextLayer: number[] = []
    const p = schema[i] as number
    const xStep = steps[i] as number

    for (const u of currentLayer) {
      for (let j = 0; j < p; j++) {
        // 根据传动方向决定扩展方向：降速向下，升速向上
        const v = isDown ? (u - j * xStep) : (u + j * xStep)
        if (v >= 1 && v <= stages) {
          nextLayer.push(v)
          
          const startAxis = getRoman(i + 1)
          const endAxis = getRoman(i + 2)
          // 轴I（第一组传动）的连线起点用输入转速的精确Y位置
          const startY = (i === 0) ? inputYLabel : getYAxisLabel(u)
          const endY = getYAxisLabel(v)

          if (startY != null && endY != null) {
            lines.push({ coords: [[startAxis, startY], [endAxis, endY]] })
            points.add(`${i + 1}-${endY}`)
          }
        }
      }
    }
    // 过滤掉当前轴重复到达的点
    currentLayer = Array.from(new Set(nextLayer))
  }

  // 确保最后一个轴（输出轴）上的点对应输出转速等比数列（级1~stages，不含扩展序列）
  for (let level = 1; level <= stages; level++) {
    const yLabel = getYAxisLabel(level)
    if (yLabel != null) {
      points.add(`${schema.length}-${yLabel}`)
    }
  }

  // 整理散点数据
  const scatterData = Array.from(points).map(pt => {
    const [axisIdx, yVal] = pt.split('-')
    return [getRoman(Number(axisIdx) + 1), yVal]
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
      axisLabel: { interval: 0, fontSize: 14, fontWeight: 'bold', color: '#000', margin: 10 }
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
watch(() => [props.schema, props.gearStages, props.inputSpeed, props.expansionSteps, props.expansionDirection], () => {
  updateChart()
}, { deep: true })

watch(() => props.referenceSpeedAxis, () => {
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
        <div ref="chartRef" :style="{ width: '100%', maxWidth: '600px', height: dynamicHeight, margin: '0 auto' }"></div>
      </div>
    </v-card-text>
  </v-card>
</template>