<script setup lang="ts">
import { computed, watch, ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import * as echarts from 'echarts'

const props = defineProps<{
  gearStages: number
  transmissionMode: 'speed-down' | 'speed-up'
  inputSpeed: number | null
  outputSpeedMin: number | null
  outputSpeedMax: number | null
}>()

const selectedSchema = defineModel<number[] | null>('selectedSchema')
const minPairs = defineModel<number>('minPairs', { default: 0 })

// 计算输入转速与输出边界的传动比，用于最小传动副数量
const maxTransmissionRatioRaw = computed(() => {
  const inSpeed = props.inputSpeed || 0
  const minSpeed = props.outputSpeedMin || 0
  if (inSpeed <= 0 || minSpeed <= 0) return 0
  return inSpeed / minSpeed
})

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

// 生成基础等比数列：根据“标准化之后的公比”与原始公比大小，选择从下限或上限开始
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

// 计算最小传动副数量 (以 4 为底的对数，然后向上取整)
const minTransmissionPairs = computed(() => {
  const r = maxTransmissionRatioRaw.value
  // 如果没有合理的变速比(<=1)，传动副数记为0
  if (r <= 1) return { raw: '0.00', rounded: 0 }
  const rawVal = Math.log(r) / Math.log(4) // 换底公式：log4(r) = ln(r) / ln(4)
  return {
    raw: rawVal.toFixed(2),
    rounded: Math.ceil(rawVal)
  }
})

// 把最小传动副数量暴露给父组件联动给图表
watch(() => minTransmissionPairs.value.rounded, (newVal) => {
  minPairs.value = newVal
}, { immediate: true })

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

        const addRawMarker = (value: number | null, label: string) => {
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
              fill: '#2f9e44',
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
              fill: '#2f9e44',
              fontSize: 11,
              fontWeight: 600,
              textAlign: 'center',
              textVerticalAlign: 'top'
            }
          })
        }

        addRawMarker(view.rawMinSpeed, `${Math.round(view.rawMinSpeed ?? 0)}`)
        addRawMarker(view.rawMaxSpeed, `${Math.round(view.rawMaxSpeed ?? 0)}`)
        addRawMarker(view.inputSpeed, `${Math.round(view.inputSpeed ?? 0)}`)

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

// 计算当前级数的全部分解结果，并根据传动模式排序
const processedPermutations = computed(() => {
  const n = props.gearStages
  if (!n || n < 1) return []
  if (n === 1) return [{ structure: [1], isRecommended: true, text: '1 = 1' }]
  
  const factors = getPrimeFactors(n)
  const perms = getUniquePermutations(factors)

  return perms.map(p => {
    // 检查是否完美符合升/降序要求
    let isRecommended = false
    if (props.transmissionMode === 'speed-down') {
      // 降速传动：前大后小更为平稳（升序）
      isRecommended = p.every((val, i, arr) => !i || (val >= (arr[i - 1] as number)))
    } else {
      // 升速传动：降序置顶
      isRecommended = p.every((val, i, arr) => !i || (val <= (arr[i - 1] as number)))
    }

    return {
      structure: p,
      isRecommended,
      text: `${p.join(' × ')} = ${n}`
    }
  }).sort((a, b) => Number(b.isRecommended) - Number(a.isRecommended))
})

// 当全部分解结果更新时，默认选中第一项（即最推荐的方案）
watch(
  () => processedPermutations.value,
  (newPerms) => {
    if (newPerms && newPerms.length > 0 && newPerms[0]) {
      selectedSchema.value = newPerms[0].structure
    } else {
      selectedSchema.value = null
    }
  },
  { immediate: true }
)

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
              <v-card-text class="px-2 py-1 compact-card-text">
                <div class="small-value text-center">{{ item.text }}</div>
              </v-card-text>
            </v-card>
          </div>
        </div>

        <v-divider class="my-6"></v-divider>
        <div class="text-body-1 text-grey-darken-1">
          第一步已完成。后续将等待完善其他设计驱动参数，从而进一步展开转速图的计算和齿轮齿数的分配...
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
@media (min-width: 1024px) { .sequence-grid { grid-template-columns: repeat(3, 1fr); } }
</style>