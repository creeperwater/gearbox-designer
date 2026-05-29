<script setup lang="ts">
import { computed, watch } from 'vue'

const props = defineProps<{
  gearStages: number
  transmissionMode: 'speed-down' | 'speed-up'
  inputSpeed: number | null
  outputSpeedMin: number | null
  outputSpeedMax: number | null
}>()

const selectedSchema = defineModel<number[] | null>('selectedSchema')

// 计算最大变速比原始数值
const maxSpeedRatioRaw = computed(() => {
  const inSpeed = props.inputSpeed || 0
  const minSpeed = props.outputSpeedMin || 0
  const maxSpeed = props.outputSpeedMax || 0
  
  const numbers: number[] = [0] // 默认给一个0作为保底，未输入也能显示 0.00
  
  if (inSpeed > 0) {
    if (minSpeed > 0) {
      numbers.push(inSpeed / minSpeed)
      numbers.push(minSpeed / inSpeed)
    }
    
    if (maxSpeed > 0) {
      numbers.push(inSpeed / maxSpeed)
      numbers.push(maxSpeed / inSpeed)
    }
  }
  
  return Math.max(...numbers)
})

const maxSpeedRatio = computed(() => maxSpeedRatioRaw.value.toFixed(2))

// 计算最小传动副数量 (以 4 为底的对数，然后向上取整)
const minTransmissionPairs = computed(() => {
  const r = maxSpeedRatioRaw.value
  // 如果没有合理的变速比(<=1)，传动副数记为0
  if (r <= 1) return { raw: '0.00', rounded: 0 }
  const rawVal = Math.log(r) / Math.log(4) // 换底公式：log4(r) = ln(r) / ln(4)
  return {
    raw: rawVal.toFixed(2),
    rounded: Math.ceil(rawVal)
  }
})

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
        <!-- 最大变速比显示，保底会显示 0.00 -->
        <div class="text-body-1 mb-1 text-primary font-weight-bold">
          最大变速比：{{ maxSpeedRatio }}
        </div>
        
        <!-- 最小传动副数量显示 -->
        <div class="text-body-1 mb-4 text-green-darken-2 font-weight-bold">
          最小传动副数量：{{ minTransmissionPairs.rounded }} <span class="text-grey-darken-1 text-body-2 font-weight-regular">(取整前: {{ minTransmissionPairs.raw }})</span>
        </div>

        <div class="text-h6 mb-2">第一步：级数分解（传动方案）</div>
        <div class="text-body-2 mb-3 text-grey-darken-1">
          将变速级数 <strong>{{ gearStages }}</strong> 分解为质因数，并列出所有可能的传动组排列方案：
        </div>
        
        <div v-if="processedPermutations.length === 0" class="text-grey">无法分解</div>
        <v-row dense v-else>
          <v-col
            v-for="(item, index) in processedPermutations"
            :key="index"
            cols="12"
            sm="6"
            md="4"
          >
            <v-card
              variant="outlined"
              :class="[
                JSON.stringify(selectedSchema) === JSON.stringify(item.structure) 
                  ? 'bg-blue-lighten-4 elevation-3 border-opacity-100 border-primary' 
                  : (item.isRecommended ? 'border-primary bg-blue-lighten-5' : 'border-grey-lighten-2')
              ]"
              style="cursor: pointer; transition: all 0.2s;"
              @click="selectedSchema = item.structure"
            >
              <v-card-title class="d-flex justify-space-between align-center px-4 pt-4 pb-2">
                <span class="text-subtitle-1 font-weight-bold">方案 {{ index + 1 }}</span>
                <v-chip
                  v-if="item.isRecommended"
                  color="primary"
                  size="small"
                  variant="flat"
                >
                  <v-icon icon="mdi-thumb-up-outline" start size="small"></v-icon>
                  推荐
                </v-chip>
              </v-card-title>
              <v-card-text class="px-4 pb-4">
                <div class="text-h6 text-center py-3 bg-white rounded elevation-1">
                  {{ item.text }}
                </div>
              </v-card-text>
            </v-card>
          </v-col>
        </v-row>

        <v-divider class="my-6"></v-divider>
        <div class="text-body-1 text-grey-darken-1">
          第一步已完成。后续将等待完善其他设计驱动参数，从而进一步展开转速图的计算和齿轮齿数的分配...
        </div>
      </div>
    </v-card-text>
  </v-card>
</template>