<script setup lang="ts">
import { computed, watch, ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import recommendedIcon from '../assets/recommended.svg'
import * as echarts from 'echarts'

// CalculationPanel.vue 概要：
// - 计算并展示基于“公比”的等比序列（用于齿轮级数设计可视化）
// - 支持将理论公比（由上下限和级数计算）标准化为若干预定义值或可选的非标值
// - 根据所选公比生成基础序列，并在输入超出范围时向左右延伸序列
// - 使用 ECharts 自定义渲染（custom series）绘制“箱型”表示序列范围，垂直刻度表示每个节点，绿点表示原始输入/上下限
// 注意：本文件内的计算和渲染顺序不可随意重排；改动前请确保不改变已有计算逻辑

// 父组件传入的关键参数：级数、传动模式、输入与输出上下限
const props = defineProps<{
  gearStages: number
  transmissionMode: 'speed-down' | 'speed-up'
  inputSpeed: number | null
  outputSpeedMin: number | null
  outputSpeedMax: number | null
}>()

const selectedSchema = defineModel<number[] | null>('selectedSchema')
const referenceSpeedAxis = defineModel<{
  values: number[]
  baseCount: number
  leftExtensionCount: number
  rightExtensionCount: number
} | null>('referenceSpeedAxis')
const transmissionDirection = defineModel<'down' | 'up'>('transmissionDirection', { default: 'down' })

// ------- 公比计算部分 -------
// theoreticalRatioRaw: 根据输出上下限与级数计算得到的“原始公比”（数值）
// originalRatio / nonStandardRatio: 用于界面展示的格式化字符串
// selectedStandardRatio / selectedRatioMode: 管理用户在标准/非标公比之间的选择
// selectedActualRatio: 最终用于序列生成的数值（优先使用用户选择）
// 原始公比（原始数值与展示字符串）
const theoreticalRatioRaw = computed<number | null>(() => {
  const minSpeed = Math.min(props.outputSpeedMin || 0, props.outputSpeedMax || 0)
  const maxSpeed = Math.max(props.outputSpeedMin || 0, props.outputSpeedMax || 0)
  const stages = props.gearStages

  if (stages <= 1 || minSpeed <= 0 || maxSpeed <= 0) return null

  const ratio = Math.pow(maxSpeed / minSpeed, 1 / (stages - 1))
  if (!Number.isFinite(ratio) || ratio <= 0) return null
  return ratio
})

const originalRatio = computed(() => {
  const r = theoreticalRatioRaw.value
  return r == null ? '--' : r.toFixed(4)
})

const nonStandardRatio = computed(() => {
  const r = theoreticalRatioRaw.value
  return r == null ? '--' : r.toFixed(2)
})

// 预定义的 7 个标准公比
const standardRatios = [1.06, 1.12, 1.26, 1.41, 1.58, 1.78, 2.0]
// 当前用户选择的标准公比（优先于理论公比）
const selectedStandardRatio = ref<number | null>(null)
const selectedRatioMode = ref<'standard' | 'non-standard'>('standard')

const ratioOptions = computed(() => {
  const options = standardRatios.map(value => ({
    key: `std-${value}`,
    value,
    label: value.toFixed(2),
    isNonStandard: false
  }))

  if (theoreticalRatioRaw.value == null) return options

  return [
    ...options,
    {
      key: 'non-standard',
      value: Number(nonStandardRatio.value),
      label: nonStandardRatio.value,
      isNonStandard: true
    }
  ]
})

// 推荐的标准公比（7个标准值里距离理论公比最近的一个）
const recommendedStandardRatio = computed<number | null>(() => {
  const r = theoreticalRatioRaw.value
  if (r == null) return null
  const firstStandard = standardRatios[0] ?? 1
  let best: number = firstStandard
  let bestDiff = Math.abs(best - r)
  for (const s of standardRatios) {
    const d = Math.abs(s - r)
    if (d < bestDiff) {
      best = s
      bestDiff = d
    }
  }
  return best
})

// 当理论公比变化时，默认选择最接近的标准值
watch(theoreticalRatioRaw, (r) => {
  if (r == null) return
  const firstStandard = standardRatios[0] ?? 1
  let best = firstStandard
  let bestDiff = Math.abs(firstStandard - r)
  for (const s of standardRatios) {
    const d = Math.abs(s - r)
    if (d < bestDiff) {
      best = s
      bestDiff = d
    }
  }
  selectedStandardRatio.value = best
  selectedRatioMode.value = 'standard'
}, { immediate: true })

const selectedActualRatio = computed<number | null>(() => {
  if (selectedRatioMode.value === 'non-standard') {
    const r = Number(nonStandardRatio.value)
    return Number.isFinite(r) ? r : null
  }
  return selectedStandardRatio.value
})

function selectRatioOption(value: number, isNonStandard = false) {
  if (isNonStandard) {
    selectedRatioMode.value = 'non-standard'
    return
  }
  selectedRatioMode.value = 'standard'
  selectedStandardRatio.value = value
}

// ------- 等比序列生成 -------
// geometricSequence: 根据所选公比和级数生成基础等比数列。
// 规则：如果选定公比小于等于原始公比，则从输出下限开始向上乘；否则从输出上限开始向下除
const geometricSequence = computed(() => {
  const stages = props.gearStages
  const ratio = selectedActualRatio.value ?? theoreticalRatioRaw.value
  const originalRatio = theoreticalRatioRaw.value

  if (!ratio || !originalRatio || stages <= 0) return [] as number[]

  const useLowerBound = ratio <= originalRatio
  const start = useLowerBound ? (props.outputSpeedMin ?? null) : (props.outputSpeedMax ?? null)

  if (!start || stages === 1) return start ? [start] : []

  const sequence: number[] = [start]
  for (let i = 1; i < stages; i++) {
    const previous = sequence[i - 1]!
    sequence.push(useLowerBound ? previous * ratio : previous / ratio)
  }
  return sequence
})

const sortedBaseSequence = computed(() => [...geometricSequence.value].sort((a, b) => a - b))

// 生成延伸序列（当原始输入在基础序列之外时使用，以保持可视化的连续性）
function buildLeftExtensionSequence(minValue: number, ratio: number, steps: number) {
  return Array.from({ length: steps }, (_, index) => minValue / Math.pow(ratio, steps - index))
}

function buildRightExtensionSequence(maxValue: number, ratio: number, steps: number) {
  return Array.from({ length: steps }, (_, index) => maxValue * Math.pow(ratio, index + 1))
}

function getExtensionSteps(ratio: number, outerValue: number, rawInput: number) {
  if (!Number.isFinite(ratio) || ratio <= 1 || outerValue <= 0 || rawInput <= 0) return 0
  const rawSteps = Math.log(rawInput / outerValue) / Math.log(ratio)
  return Math.max(1, Math.round(Math.abs(rawSteps)))
}

function getStandardInputSpeed(rawInput: number | null, sequence: number[], ratio: number | null) {
  if (rawInput == null || !Number.isFinite(rawInput) || !ratio || ratio <= 1 || sequence.length === 0) {
    return null
  }

  const minValue = sequence[0]!
  const maxValue = sequence[sequence.length - 1]!

  if (rawInput <= minValue) {
    let lower = minValue
    let upper = minValue

    while (lower > rawInput) {
      upper = lower
      lower = lower / ratio
    }

    return Math.abs(rawInput - lower) <= Math.abs(upper - rawInput) ? lower : upper
  }

  if (rawInput >= maxValue) {
    let lower = maxValue
    let upper = maxValue

    while (upper < rawInput) {
      lower = upper
      upper = upper * ratio
    }

    return Math.abs(rawInput - lower) <= Math.abs(upper - rawInput) ? lower : upper
  }

  let nearest = sequence[0]!
  let smallestDiff = Math.abs(sequence[0]! - rawInput)
  for (const value of sequence) {
    const diff = Math.abs(value - rawInput)
    if (diff < smallestDiff) {
      nearest = value
      smallestDiff = diff
    }
  }
  return nearest
}

const referenceSpeedAxisValue = computed(() => {
  const sequence = sortedBaseSequence.value
  const inputSpeed = props.inputSpeed ?? null
  const ratio = selectedActualRatio.value ?? theoreticalRatioRaw.value
  if (sequence.length === 0) return null

  const minValue = sequence[0]!
  const maxValue = sequence[sequence.length - 1]!
  const isBelow = inputSpeed != null && inputSpeed < minValue
  const isAbove = inputSpeed != null && inputSpeed > maxValue

  const leftExtensionSteps = isBelow && ratio ? getExtensionSteps(ratio, minValue, inputSpeed as number) : 0
  const rightExtensionSteps = isAbove && ratio ? getExtensionSteps(ratio, maxValue, inputSpeed as number) : 0
  const leftExtensionValues = leftExtensionSteps > 0 && ratio ? buildLeftExtensionSequence(minValue, ratio, leftExtensionSteps) : []
  const rightExtensionValues = rightExtensionSteps > 0 && ratio ? buildRightExtensionSequence(maxValue, ratio, rightExtensionSteps) : []

  return {
    values: [...leftExtensionValues, ...sequence, ...rightExtensionValues],
    baseCount: sequence.length,
    leftExtensionCount: leftExtensionValues.length,
    rightExtensionCount: rightExtensionValues.length
  }
})

const boxChartModel = computed(() => {
  const sequence = sortedBaseSequence.value
  const inputSpeed = props.inputSpeed ?? null
  const rawMinSpeed = props.outputSpeedMin ?? null
  const rawMaxSpeed = props.outputSpeedMax ?? null
  const ratio = selectedActualRatio.value ?? theoreticalRatioRaw.value
  const standardInputSpeed = getStandardInputSpeed(inputSpeed, sequence, ratio)

  if (sequence.length === 0) return null

  const minValue = sequence[0]!
  const maxValue = sequence[sequence.length - 1]!

  const isBelow = inputSpeed != null && inputSpeed < minValue
  const isAbove = inputSpeed != null && inputSpeed > maxValue

  const leftExtensionSteps = isBelow && ratio ? getExtensionSteps(ratio, minValue, inputSpeed as number) : 0
  const rightExtensionSteps = isAbove && ratio ? getExtensionSteps(ratio, maxValue, inputSpeed as number) : 0
  const leftExtensionValues = leftExtensionSteps > 0 && ratio ? buildLeftExtensionSequence(minValue, ratio, leftExtensionSteps) : []
  const rightExtensionValues = rightExtensionSteps > 0 && ratio ? buildRightExtensionSequence(maxValue, ratio, rightExtensionSteps) : []

  const axisValues = [
    minValue,
    maxValue,
    inputSpeed,
    standardInputSpeed,
    rawMinSpeed,
    rawMaxSpeed,
    ...leftExtensionValues,
    ...rightExtensionValues
  ].filter((value): value is number => value != null && Number.isFinite(value))

  const axisMinCandidate = Math.min(...axisValues)
  const axisMaxCandidate = Math.max(...axisValues)
  const axisSpan = axisMaxCandidate - axisMinCandidate
  const axisPadding = axisSpan > 0 ? axisSpan * 0.12 : Math.max(maxValue * 0.12, 100)

  const axisMin = Math.max(0, axisMinCandidate - axisPadding)
  const axisMax = axisMaxCandidate + axisPadding

  return {
    axisMin,
    axisMax,
    isBelow,
    isAbove,
    inputSpeed,
    rawMinSpeed,
    rawMaxSpeed,
    standardInputSpeed,
    inputLabel: standardInputSpeed != null ? standardInputSpeed.toFixed(2) : null,
    sequence,
    leftExtensionValues,
    rightExtensionValues,
    minValue,
    maxValue
  }
})

watch(referenceSpeedAxisValue, (value) => {
  referenceSpeedAxis.value = value
}, { immediate: true })

// ------- ECharts 渲染与生命周期管理 -------
// 使用 ECharts 的 custom series 在同一图层内精确绘制箱体、刻度、延伸线与标记。
// 注意：自定义渲染中使用的 z2 值控制图形叠放顺序，绿点应比箱体更上层以保证可见性。
// initChart / disposeChart / updateChart: 分别负责初始化、清理以及根据模型刷新图表
// resizeHandler: 在窗口大小变化时触发图表重绘
// renderItem 内部的 addTick / addRawMarker 用于分别绘制蓝色刻度与绿色原始值标记
// 注意不要随意修改 renderItem 中的坐标转换 api.coord 的调用顺序
// ECharts 绑定
const chartRef = ref<HTMLElement | null>(null)
let chartInstance: echarts.ECharts | null = null
const resizeHandler = () => chartInstance?.resize()

function initChart() {
  if (!chartRef.value) return
  chartInstance = echarts.init(chartRef.value)
  updateChart()
}

function disposeChart() {
  if (chartInstance) {
    chartInstance.dispose()
    chartInstance = null
  }
}

function updateChart() {
  if (!chartInstance) return
  const view = boxChartModel.value
  if (!view) {
    chartInstance.clear()
    return
  }

  const option = {
    grid: { left: 10, right: 10, top: 22, bottom: 34, containLabel: true },
    xAxis: {
      type: 'value',
      min: view.axisMin,
      max: view.axisMax,
      interval: undefined,
      axisLine: { lineStyle: { color: '#999' } },
      axisTick: { show: true },
      axisLabel: {
        formatter: (value: number) => `${Math.round(value)}`,
        color: '#555',
        fontSize: 12,
        margin: 12,
        hideOverlap: true
      }
    },
    yAxis: {
      type: 'category',
      data: [''],
      axisLabel: { show: false },
      axisTick: { show: false },
      axisLine: { show: false },
      splitLine: { show: false }
    },
    series: [{
      type: 'custom',
      name: '箱型图',
      renderItem: (params: any, api: any) => {
        const coordY = api.coord([0, 0])[1]
        const boxHeight = 28
        const halfHeight = boxHeight / 2
        const boxStartX = api.coord([view.minValue, 0])[0]
        const boxEndX = api.coord([view.maxValue, 0])[0]
        const children: any[] = []

        const addTick = (value: number, color = '#4c6ef5') => {
          const x = api.coord([value, 0])[0]
          children.push({
            type: 'line',
            z2: 2,
            shape: {
              x1: x,
              y1: coordY - halfHeight,
              x2: x,
              y2: coordY + halfHeight
            },
            style: {
              stroke: color,
              lineWidth: 2
            }
          })
          children.push({
            type: 'text',
            z2: 3,
            style: {
              text: String(Math.round(value)),
              x,
              y: coordY - halfHeight - 8,
              fill: color,
              fontSize: 11,
              fontWeight: 600,
              textAlign: 'center',
              textVerticalAlign: 'bottom'
            }
          })
        }

        const addRawMarker = (value: number | null, label: string, color = '#2f9e44') => {
          if (value == null || !Number.isFinite(value)) return
          const x = api.coord([value, 0])[0]
          children.push({
            type: 'circle',
            z2: 30,
            shape: {
              cx: x,
              cy: coordY,
              r: 4.5
            },
            style: {
              fill: color,
              stroke: '#ffffff',
              lineWidth: 1.5
            }
          })
          children.push({
            type: 'text',
            z2: 31,
            style: {
              text: label,
              x,
              y: coordY + halfHeight + 14,
              fill: color,
              fontSize: 11,
              fontWeight: 600,
              textAlign: 'center',
              textVerticalAlign: 'top'
            }
          })
        }

        addRawMarker(view.rawMinSpeed, `${Math.round(view.rawMinSpeed ?? 0)}`)
        addRawMarker(view.rawMaxSpeed, `${Math.round(view.rawMaxSpeed ?? 0)}`)
        // input speed should be highlighted in red
        addRawMarker(view.inputSpeed, `${Math.round(view.inputSpeed ?? 0)}`, '#d73a2a')

        children.push({
          type: 'rect',
          z2: 1,
          shape: {
            x: boxStartX,
            y: coordY - halfHeight,
            width: Math.max(boxEndX - boxStartX, 1),
            height: boxHeight
          },
          style: {
            fill: 'rgba(76,110,245,0.28)',
            stroke: '#4c6ef5',
            lineWidth: 2
          }
        })

        for (const value of view.sequence) {
          addTick(value)
        }

        if (view.leftExtensionValues.length > 0) {
          const leftStartX = api.coord([view.leftExtensionValues[0], 0])[0]
          children.push({
            type: 'line',
            z2: 2,
            shape: {
              x1: leftStartX,
              y1: coordY,
              x2: boxStartX,
              y2: coordY
            },
            style: {
              stroke: '#4c6ef5',
              lineWidth: 2
            }
          })
          for (const value of view.leftExtensionValues) {
            addTick(value)
          }
        }

        if (view.rightExtensionValues.length > 0) {
          const rightEndX = api.coord([view.rightExtensionValues[view.rightExtensionValues.length - 1], 0])[0]
          children.push({
            type: 'line',
            z2: 2,
            shape: {
              x1: boxEndX,
              y1: coordY,
              x2: rightEndX,
              y2: coordY
            },
            style: {
              stroke: '#4c6ef5',
              lineWidth: 2
            }
          })
          for (const value of view.rightExtensionValues) {
            addTick(value)
          }
        }

        if (view.isBelow || view.isAbove) {
          const whiskerX = api.coord([view.standardInputSpeed ?? 0, 0])[0]
          const boxEdgeX = view.isBelow ? boxStartX : boxEndX
          const capWidth = 12

          children.push({
            type: 'line',
            z2: 2,
            shape: {
              x1: whiskerX,
              y1: coordY,
              x2: boxEdgeX,
              y2: coordY
            },
            style: {
              stroke: '#4c6ef5',
              lineWidth: 2
            }
          })

          children.push({
            type: 'line',
            z2: 2,
            shape: {
              x1: whiskerX,
              y1: coordY - capWidth / 2,
              x2: whiskerX,
              y2: coordY + capWidth / 2
            },
            style: {
              stroke: '#4c6ef5',
              lineWidth: 2
            }
          })

        }

        return {
          type: 'group',
          children
        }
      },
      data: [0]
    }],
    tooltip: {
      trigger: 'item',
      formatter: () => {
        const seqText = view.sequence.map(v => `${v.toFixed(2)} rpm`).join('，')
        const leftText = view.leftExtensionValues.length > 0 ? view.leftExtensionValues.map(v => `${v.toFixed(2)} rpm`).join('，') : '--'
        const rightText = view.rightExtensionValues.length > 0 ? view.rightExtensionValues.map(v => `${v.toFixed(2)} rpm`).join('，') : '--'
        return [
          `等比数列范围：${view.minValue.toFixed(2)} rpm - ${view.maxValue.toFixed(2)} rpm`,
          `标准输入速度：${view.standardInputSpeed?.toFixed(2) ?? '--'} rpm（原始输入 ${view.inputSpeed?.toFixed(2) ?? '--'} rpm，已按标准值量化）`,
          `原始输出下限：${view.rawMinSpeed?.toFixed(2)} rpm`,
          `原始输出上限：${view.rawMaxSpeed?.toFixed(2)} rpm`,
          `基础序列：${seqText}`,
          `左延伸：${leftText}`,
          `右延伸：${rightText}`
        ].join('<br/>')
      }
    }
  }
  chartInstance.setOption(option)
}

onMounted(() => {
  nextTick(() => initChart())
  window.addEventListener('resize', resizeHandler)
})

onBeforeUnmount(() => {
  disposeChart()
  window.removeEventListener('resize', resizeHandler)
})

watch(boxChartModel, () => updateChart(), { immediate: true, deep: true })

// 质因数分解（纯函数）
function getPrimeFactors(n: number): number[] {
  const factors: number[] = []
  let divisor = 2
  let current = n
  while (current >= 2) {
    if (current % divisor === 0) {
      factors.push(divisor)
      current = current / divisor
    } else {
      divisor++
    }
  }
  return factors
}

// 获取所有不重复的排列组合（纯函数）
function getUniquePermutations(elements: number[]): number[][] {
  if (elements.length === 0) return [[]]
  if (elements.length === 1) return [[elements[0] as number]]
  
  const result: number[][] = []
  const uniqueElements = Array.from(new Set(elements))
  
  for (const el of uniqueElements) {
    const remaining = [...elements]
    remaining.splice(remaining.indexOf(el), 1)
    const subPerms = getUniquePermutations(remaining)
    for (const sub of subPerms) {
      result.push([el, ...sub])
    }
  }
  return result
}

// ------- 传动类型判定 -------
// 根据输入转速与输出转速上下限的几何平均数比较，判定是升速还是降速传动
const transmissionType = computed(() => {
  const inputSpeed = props.inputSpeed
  const outputMin = props.outputSpeedMin
  const outputMax = props.outputSpeedMax

  if (inputSpeed == null || outputMin == null || outputMax == null) {
    return null
  }

  if (outputMin <= 0 || outputMax <= 0) {
    return null
  }

  // 计算几何平均数：sqrt(outputMin * outputMax)
  const geometricMean = Math.sqrt(outputMin * outputMax)

  if (inputSpeed < geometricMean) {
    return {
      type: 'speed-up',
      label: '升速传动',
      geometricMean: geometricMean.toFixed(2),
      description: `输入转速 (${inputSpeed} rpm) < 几何平均数 (${geometricMean.toFixed(2)} rpm)`
    }
  } else {
    return {
      type: 'speed-down',
      label: '降速传动',
      geometricMean: geometricMean.toFixed(2),
      description: `输入转速 (${inputSpeed} rpm) ≥ 几何平均数 (${geometricMean.toFixed(2)} rpm)`
    }
  }
})

// 计算当前级数的全部分解结果，并根据传动类型排序
const processedPermutations = computed(() => {
  const n = props.gearStages
  if (!n || n < 1) return []
  if (n === 1) return [{ structure: [1], isRecommended: true, text: '1 = 1' }]
  
  const factors = getPrimeFactors(n)
  const perms = getUniquePermutations(factors)

  // 根据传动类型判定推荐方案
  const type = transmissionType.value?.type ?? props.transmissionMode

  return perms.map(p => {
    let isRecommended = false
    if (type === 'speed-down') {
      // 降速传动：推荐质因数逐渐减小的方案（前大后小）
      isRecommended = p.every((val, i, arr) => !i || (val <= (arr[i - 1] as number)))
    } else {
      // 升速传动：推荐质因数逐渐增加的方案（前小后大）
      isRecommended = p.every((val, i, arr) => !i || (val >= (arr[i - 1] as number)))
    }

    return {
      structure: p,
      isRecommended,
      text: `${p.join(' × ')} = ${n}`
    }
  }).sort((a, b) => Number(b.isRecommended) - Number(a.isRecommended))
})

// 当全部分解结果更新时，默认选中推荐的方案
watch(
  () => processedPermutations.value,
  (newPerms) => {
    if (newPerms && newPerms.length > 0) {
      const recommended = newPerms.find(p => p.isRecommended)
      selectedSchema.value = recommended ? recommended.structure : (newPerms[0]?.structure ?? null)
    } else {
      selectedSchema.value = null
    }
  },
  { immediate: true }
)

// ------- 固定步距计算 -------
// 步距恒定：基本组步距=1，第i扩大组步距=前序所有组级数之积
// 即 steps[0]=1, steps[k]=schema[0]×schema[1]×...×schema[k-1]
const fixedExpansionSteps = computed(() => {
  const schema = selectedSchema.value
  if (!schema || schema.length === 0) return null
  if (schema.length === 1) return [1]
  const steps: number[] = [1]
  let product = 1
  for (let i = 0; i < schema.length - 1; i++) {
    product *= schema[i]!
    steps.push(product)
  }
  return steps
})

// 当传动类型变化时，更新传动方向
watch(
  () => transmissionType.value,
  (t) => {
    transmissionDirection.value = (t?.type === 'speed-up') ? 'up' : 'down'
  },
  { immediate: true }
)

// ------- 参考方案图表数据 -------
// 将所有计算结果打包为结构化的图表数据，供 OutputPanel 直接渲染

type DiagramModel = {
  shafts: string[]
  yAxisLabels: string[]
  lines: Array<{ coords: [[string, string], [string, string]] }>
  scatterPoints: Array<[string, string]>
  middleShaftIndices: number[]
}

const getRoman = (num: number) => {
  const romans = ['Ⅰ', 'Ⅱ', 'Ⅲ', 'Ⅳ', 'Ⅴ', 'Ⅵ', 'Ⅶ', 'Ⅷ', 'Ⅸ', 'Ⅹ', 'Ⅺ', 'Ⅻ']
  return romans[num - 1] || String(num)
}

const formatSpeed = (value: number, index: number = 0) => String(Math.round(value)) + "\u200B".repeat(index)

const getYAxisData = () => {
  const axis = referenceSpeedAxisValue.value
  if (axis && axis.values.length > 0) {
    return axis.values.map((v, i) => formatSpeed(v, i))
  }
  const stages = props.gearStages || 1
  return Array.from({ length: stages }, (_, index) => `n${index + 1}`)
}

const getYAxisLabel = (level: number) => {
  const axis = referenceSpeedAxisValue.value
  if (!axis || axis.values.length === 0) {
    return `n${level}`
  }
  const position = axis.leftExtensionCount + level - 1
  const value = axis.values[position]
  return value == null ? null : formatSpeed(value, position)
}

const getClosestAxisValue = () => {
  const axis = referenceSpeedAxisValue.value
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

const diagramDataValue = computed<DiagramModel | null>(() => {
  const schema = selectedSchema.value
  if (!schema || schema.length === 0) return null

  const stages = props.gearStages || 1
  const shafts = schema.length + 1
  const shaftNames = Array.from({ length: shafts }, (_, i) => getRoman(i + 1))
  const yAxisLabels = getYAxisData()

  // 使用固定步距（标准扩大顺序）
  const steps = fixedExpansionSteps.value ?? (() => {
    const defaultSteps = [1]
    let current = 1
    for (let i = 0; i < schema.length; i++) {
      current *= schema[i]!
      defaultSteps.push(current)
    }
    return defaultSteps.slice(0, schema.length)
  })()

  const lines: Array<{ coords: [[string, string], [string, string]] }> = []
  const points = new Set<string>()

  // 轴I上的输入转速点
  const inputYLabel = getClosestAxisValue()
  if (inputYLabel != null) {
    points.add(`0-${inputYLabel}`)
  }

  // 根据传动方向逐级扩散
  const isDown = transmissionDirection.value === 'down'
  const anchorLevel = isDown ? stages : 1
  let currentLayer = [anchorLevel]

  for (let i = 0; i < schema.length; i++) {
    const nextLayer: number[] = []
    const p = schema[i]!
    const xStep = steps[i]!

    for (const u of currentLayer) {
      for (let j = 0; j < p; j++) {
        const v = isDown ? (u - j * xStep) : (u + j * xStep)
        if (v >= 1 && v <= stages) {
          nextLayer.push(v)
          const startAxis = getRoman(i + 1)
          const endAxis = getRoman(i + 2)
          const startY = (i === 0) ? inputYLabel : getYAxisLabel(u)
          const endY = getYAxisLabel(v)
          if (startY != null && endY != null) {
            lines.push({ coords: [[startAxis, startY], [endAxis, endY]] })
          }
        }
      }
    }
    currentLayer = Array.from(new Set(nextLayer))

    // 按轴层级添加散点：当前轴(i+1)上所有实际存在的转速级
    for (const level of currentLayer) {
      const yLabel = getYAxisLabel(level)
      if (yLabel != null) {
        points.add(`${i + 1}-${yLabel}`)
      }
    }
  }

  const scatterPoints: Array<[string, string]> = Array.from(points).map(pt => {
    const parts = pt.split('-')
    return [getRoman(Number(parts[0]!) + 1), parts[1]!] as [string, string]
  })

  const middleShaftIndices = shaftNames.length > 2
    ? Array.from({ length: shaftNames.length - 2 }, (_, i) => i + 1)
    : []

  return { shafts: shaftNames, yAxisLabels, lines, scatterPoints, middleShaftIndices }
})

const diagramData = defineModel<DiagramModel | null>('diagramData')

watch(diagramDataValue, (value) => {
  diagramData.value = value
}, { immediate: true })

</script>

<template>
  <v-card class="mb-6" elevation="2">
    <v-card-title class="text-h5 font-weight-bold">计算过程</v-card-title>
    <v-card-text>
      <div class="py-4">
        <div class="text-body-1 mb-4 text-green-darken-2 font-weight-bold">
          原始公比：{{ originalRatio }}
        </div>

        <!-- 标准公比选择（参考级数分解表格样式） -->
              <div class="mb-3">
                <div class="text-body-2 mb-2">公比标准化</div>
                <div class="sequence-grid">
                  <div v-for="option in ratioOptions" :key="option.key" class="sequence-grid-item">
                    <v-card
                      variant="outlined"
                      :class="[
                        option.isNonStandard
                          ? (selectedRatioMode === 'non-standard' ? 'selected-card non-standard-card' : 'non-standard-card')
                          : (selectedRatioMode === 'standard' && selectedStandardRatio === option.value ? 'selected-card' : 'normal-card')
                      ]"
                      style="cursor: pointer; transition: all 0.08s;"
                      @click="selectRatioOption(option.value, option.isNonStandard)"
                      density="compact"
                    >
                      <img v-if="!option.isNonStandard && option.value === recommendedStandardRatio" :src="recommendedIcon" class="recommended-badge" alt="recommended" />
                      <v-card-text class="px-2 py-1 compact-card-text">
                        <div class="small-value text-center">{{ option.isNonStandard ? `非标公比 ${option.label}` : option.label }}</div>
                      </v-card-text>
                    </v-card>
                  </div>
                </div>
              </div>

        <!-- 等比数列图表（基于输出转速下限与选择的标准公比或理论公比生成） -->
        <div class="mb-4">
          <div ref="chartRef" style="width:100%;height:220px;border:1px solid #e6e6e6;border-radius:4px;"></div>
          <div v-if="geometricSequence.length > 0" class="text-body-2 mt-2">
            使用公比：<strong class="mx-1">{{ selectedActualRatio != null ? selectedActualRatio.toFixed(2) : '--' }}</strong>
            等比序列：
            <span v-for="(v, i) in geometricSequence" :key="i">{{ v.toFixed(2) }} rpm<span v-if="i < geometricSequence.length - 1">, </span></span>
          </div>
          <div v-else class="text-grey mt-2">无法生成等比序列（请检查输出转速上下限和级数）</div>
        </div>

        <!-- 传动类型判定 -->
        <div v-if="transmissionType" class="mb-4 p-3" :style="{ backgroundColor: transmissionType.type === 'speed-down' ? '#e8f5e9' : '#fff3e0', borderRadius: '4px', borderLeft: '4px solid ' + (transmissionType.type === 'speed-down' ? '#4caf50' : '#ff9800') }">
          <div class="text-h6 mb-2" :style="{ color: transmissionType.type === 'speed-down' ? '#2e7d32' : '#ef6c00' }">
            {{ transmissionType.label }}
          </div>
          <div class="text-body-2 text-grey-darken-1">
            {{ transmissionType.description }}
          </div>
          <div class="text-caption text-grey mt-1">
            判定依据：输出转速下限 ({{ outputSpeedMin }} rpm) 与上限 ({{ outputSpeedMax }} rpm) 的几何平均数为 {{ transmissionType.geometricMean }} rpm
          </div>
        </div>
        <div v-else class="mb-4 p-3" style="backgroundColor: #fafafa; borderRadius: 4px; borderLeft: 4px solid #bdbdbd;">
          <div class="text-body-2 text-grey">请输入完整的转速参数以判定传动类型</div>
        </div>

        <div class="text-h6 mb-2">第一步：级数分解（传动方案）</div>
        <div class="text-body-2 mb-3 text-grey-darken-1">
          将变速级数 <strong>{{ gearStages }}</strong> 分解为质因数，并列出所有可能的传动组排列方案：
        </div>
        
        <div v-if="processedPermutations.length === 0" class="text-grey">无法分解</div>
        <div v-else class="sequence-grid">
          <div
            v-for="(item, index) in processedPermutations"
            :key="index"
            class="sequence-grid-item"
          >
            <v-card
              variant="outlined"
              :class="[
                JSON.stringify(selectedSchema) === JSON.stringify(item.structure) ? 'selected-card' : (item.isRecommended ? 'recommended-card' : 'normal-card')
              ]"
              style="cursor: pointer; transition: all 0.08s;"
              @click="selectedSchema = item.structure"
              density="compact"
            >
              <img v-if="item.isRecommended" :src="recommendedIcon" class="recommended-badge" alt="recommended" />
              <v-card-text class="px-2 py-1 compact-card-text">
                <div class="small-value text-center">{{ item.text }}</div>
              </v-card-text>
            </v-card>
          </div>
        </div>

        <!-- 步距信息（固定值，不可选择） -->
        <div v-if="fixedExpansionSteps && selectedSchema" class="mt-4">
          <div class="text-body-2 text-grey-darken-1">
            传动方案 <strong>{{ selectedSchema.join(' × ') }} = {{ gearStages }}</strong> 的步距：<strong>{{ fixedExpansionSteps.join(', ') }}</strong>
            （基本组步距=1，各扩大组步距=前序组级数之积）
          </div>
        </div>

        <v-divider class="my-6"></v-divider>
        <div class="text-body-1 text-grey-darken-1">
          第二步已完成。后续将等待完善其他设计驱动参数，从而进一步展开转速图的计算和齿轮齿数的分配...
        </div>
      </div>
    </v-card-text>
  </v-card>
</template>

<style scoped>
/* Compact grid styles for方案列表（与 InputPanel.vue 保持一致） */
.sequence-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  gap: 0px;
}
.sequence-grid-item { padding: 0; }
.compact-card-text { padding: 6px 4px !important; }
.small-value { font-size: 0.95rem; font-weight: 600; }
.selected-card { background-color: rgba(147,197,253,0.45); box-shadow: none; border: 1px solid rgba(59,130,246,0.9); border-radius: 2px; }
.recommended-card { border: 1px solid rgba(59,130,246,0.9); background-color: rgba(219,234,254,0.35); border-radius: 2px; }
.normal-card { border: 1px solid rgba(229,231,235,0.9); border-radius: 2px; }
.non-standard-card { border: 1px solid rgba(220,38,38,0.95); background-color: rgba(254,226,226,0.95); border-radius: 2px; }
/* recommended badge positioning */
.sequence-grid-item .v-card { position: relative; }
.recommended-badge { position: absolute; top: 0px; right: 2px; width: 22px; height: 22px; pointer-events: none; }
@media (min-width: 1024px) { .sequence-grid { grid-template-columns: repeat(3, 1fr); } }
</style>