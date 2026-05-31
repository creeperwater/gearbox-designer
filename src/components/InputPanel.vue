<script setup lang="ts">
import { computed, ref, watch } from 'vue'

// InputPanel 说明：
// - 通过 `defineModel` 暴露与父组件双向绑定的参数：`inputSpeed`, `outputSpeed`, `outputSpeedMin`, `outputSpeedMax`, `gearStages`
// - 提供基于上下限与级数的等差序列计算（用于向用户展示候选输出转速），并在输入超出序列范围时适当扩展序列
// - 约束：所有输入值均强制非负
const inputSpeed = defineModel<number>('inputSpeed', { default: 0 })
const outputSpeed = defineModel<number>('outputSpeed', { default: 0 })
const outputSpeedMin = defineModel<number>('outputSpeedMin', { default: 0 })
const outputSpeedMax = defineModel<number>('outputSpeedMax', { default: 0 })
const gearStages = defineModel<number>('gearStages', { default: 1 })

// 输出转速等差数列，数量等于变速级数
const outputSpeedSequence = computed(() => {
  const stages = gearStages.value
  const input = inputSpeed.value
  const start = outputSpeedMin.value
  const end = outputSpeedMax.value

  if (stages <= 1) {
    return [start.toFixed(2)]
  }

  const low = Math.min(start, end)
  const high = Math.max(start, end)
  let step = (high - low) / (stages - 1)

  if (step === 0) {
    const fallbackSpan = Math.abs(input - low)
    step = fallbackSpan > 0 ? fallbackSpan / (stages - 1) : 1
  }

  const sequence = Array.from({ length: stages }, (_, index) => low + step * index)
  const sequenceMin = sequence[0]
  const sequenceMax = sequence[sequence.length - 1]

  if (input < sequenceMin) {
    const extraCount = Math.ceil((sequenceMin - input) / step)
    for (let index = 0; index < extraCount; index++) {
      sequence.unshift(sequence[0] - step)
    }
  } else if (input > sequenceMax) {
    const extraCount = Math.ceil((input - sequenceMax) / step)
    for (let index = 0; index < extraCount; index++) {
      sequence.push(sequence[sequence.length - 1] + step)
    }
  }

  return sequence.map(value => value.toFixed(2))
})

// 过滤掉等于 0 或者小于 0 的项（用户要求：等差数列里去掉零，不要有0；另外三个转速都不能小于零）
const filteredSequenceNumbers = computed(() => {
  return outputSpeedSequence.value
    .map(s => parseFloat(s))
    .filter(v => !isNaN(v) && v !== 0 && v >= 0)
})

const filteredSequenceStrings = computed(() => filteredSequenceNumbers.value.map(v => v.toFixed(2)))

// 推荐：序列里与输入转速最接近的那个
const recommendedIndex = computed(() => {
  const seq = filteredSequenceNumbers.value
  if (!seq || seq.length === 0) return -1
  let best = 0
  let bestDiff = Math.abs(seq[0] - inputSpeed.value)
  for (let i = 1; i < seq.length; i++) {
    const d = Math.abs(seq[i] - inputSpeed.value)
    if (d < bestDiff) {
      best = i
      bestDiff = d
    }
  }
  return best
})

const selectedIndex = ref<number>(-1)

// 当序列或输入速度变化时，默认选中推荐项
watch([filteredSequenceNumbers, () => inputSpeed.value], () => {
  const r = recommendedIndex.value
  if (r >= 0) {
    selectedIndex.value = r
  } else {
    selectedIndex.value = -1
  }
}, { immediate: true })

// 当用户手动点击选择时更新 selectedSequenceValue
function selectSequence(index: number) {
  selectedIndex.value = index
}

// 对输入进行非负约束，防止用户输入小于 0 的值
watch(inputSpeed, (v) => { if (v < 0) inputSpeed.value = 0 })
watch(outputSpeed, (v) => { if (v < 0) outputSpeed.value = 0 })
watch(outputSpeedMin, (v) => { if (v < 0) outputSpeedMin.value = 0 })
watch(outputSpeedMax, (v) => { if (v < 0) outputSpeedMax.value = 0 })
</script>

<template>
  <v-card class="mb-6" elevation="2">
    <v-card-title class="text-h5 font-weight-bold pb-2">设计驱动参数</v-card-title>
    <v-card-text>
      <v-row class="mt-2" align="center">
        <v-col cols="12" md="4">
          <v-text-field
            v-model.number="inputSpeed"
            label="输入转速 (rpm)"
            type="number"
            step="10"
            :min="0"
            placeholder="0"
            variant="outlined"
            density="comfortable"
            hide-details
          ></v-text-field>
        </v-col>
        <v-col cols="12" md="8">
          <v-slider
            v-model="gearStages"
            label="变速级数"
            min="1"
            max="20"
            step="1"
            thumb-label="always"
            color="primary"
            hide-details
          ></v-slider>
        </v-col>
      </v-row>

      <v-row class="mt-2">
        <v-col cols="12" v-if="gearStages === 1">
          <v-text-field
            v-model.number="outputSpeed"
            label="输出转速 (rpm)"
            type="number"
            step="10"
            :min="0"
            placeholder="0"
            variant="outlined"
            density="comfortable"
            hide-details
          ></v-text-field>
        </v-col>

        <template v-else>
          <v-col cols="12" sm="6" md="6">
            <v-text-field
              v-model.number="outputSpeedMin"
              label="输出转速下限 (rpm)"
              type="number"
              step="10"
              :min="0"
              placeholder="0"
              variant="outlined"
              density="comfortable"
              hide-details
            ></v-text-field>
          </v-col>
          <v-col cols="12" sm="6" md="6">
            <v-text-field
              v-model.number="outputSpeedMax"
              label="输出转速上限 (rpm)"
              type="number"
              step="10"
              :min="0"
              placeholder="0"
              variant="outlined"
              density="comfortable"
              hide-details
            ></v-text-field>
          </v-col>
        </template>
      </v-row>

      <!-- 已移除“输入转速锚定”序列网格 -->

      <v-divider class="my-3"></v-divider>
    </v-card-text>
  </v-card>
</template>

<style scoped>
.summary-sheet {
  background: #f8fafc;
}

.sequence-sheet {
  background: #f8fafc;
}

.summary-item {
  padding: 8px 0;
}

.summary-label {
  color: #64748b;
  font-size: 0.85rem;
  margin-bottom: 4px;
}

.summary-value {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
}

/* 新的紧凑网格样式 */
.sequence-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  gap: 0px;
}
.sequence-grid-item {
  display: block;
  padding: 0;
}
.compact-card-text {
  padding: 6px 4px !important;
}
.small-value {
  font-size: 0.95rem;
  font-weight: 600;
}
.selected-card {
  background-color: rgba(147,197,253,0.45);
  box-shadow: none;
  border: 1px solid rgba(59,130,246,0.9);
  border-radius: 2px;
}
.recommended-card {
  border: 1px solid rgba(59,130,246,0.9);
  background-color: rgba(219,234,254,0.35);
  border-radius: 2px;
}
.normal-card {
  border: 1px solid rgba(229,231,235,0.9);
  border-radius: 2px;
}

@media (min-width: 1024px) {
  .sequence-grid { grid-template-columns: repeat(3, 1fr); }
}

@media (min-width: 1400px) {
  .sequence-grid { grid-template-columns: repeat(4, 1fr); }
}
</style>