<script setup lang="ts">
import { computed, ref, watch } from 'vue'

import InputPanel from './components/InputPanel.vue'
import CalculationPanel from './components/CalculationPanel.vue'
import OutputPanel from './components/OutputPanel.vue'

const inputSpeed = ref<number>(1500)
const outputSpeed = ref<number>(0)
const outputSpeedMin = ref<number>(1000)
const outputSpeedMax = ref<number>(2000)
const gearStages = ref<number>(5) // 默认为 5 级，方便调试序列图
const transmissionMode = ref<'speed-down' | 'speed-up'>('speed-down') // 升降速传动选择
const selectedSchema = ref<number[] | null>(null) // 选中的参考方案
const minPairs = ref<number>(0) // 最小传动副数量

watch(
  [gearStages, outputSpeed],
  ([stages, speed]) => {
    if (stages === 1) {
      outputSpeedMin.value = speed
      outputSpeedMax.value = speed
    }
  },
  { immediate: true }
)

watch(gearStages, (stages) => {
  if (stages === 1) {
    const syncedValue = outputSpeedMin.value || outputSpeedMax.value || outputSpeed.value
    outputSpeed.value = syncedValue
    outputSpeedMin.value = syncedValue
    outputSpeedMax.value = syncedValue
  }
})
</script>

<template>
  <v-app>
    <v-main>
      <v-container class="py-8" max-width="960">
        
        <InputPanel 
          v-model:inputSpeed="inputSpeed"
          v-model:outputSpeed="outputSpeed"
          v-model:outputSpeedMin="outputSpeedMin"
          v-model:outputSpeedMax="outputSpeedMax"
          v-model:gearStages="gearStages"
        />

        <CalculationPanel 
          :gearStages="gearStages"
          :transmissionMode="transmissionMode"
          :inputSpeed="inputSpeed"
          :outputSpeedMin="outputSpeedMin"
          :outputSpeedMax="outputSpeedMax"
          v-model:selectedSchema="selectedSchema"
          v-model:minPairs="minPairs"
        />

        <OutputPanel 
          :gearStages="gearStages"
          :schema="selectedSchema"
          :minPairs="minPairs"
        />

      </v-container>
    </v-main>
  </v-app>
</template>

<style scoped>
/* App 级别的样式放到这里，其他的移到了组件里 */
</style>
