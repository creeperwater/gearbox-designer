<script setup lang="ts">
import { computed, ref, watch } from 'vue'

// App.vue: 顶层状态与组件绑定
// - 管理全局的输入/输出转速、级数与传动模式
// - 将状态通过 v-model 或 props 传递到各子组件，子组件可通过 defineModel 与父组件联动
import InputPanel from './components/InputPanel.vue'
import CalculationPanel from './components/CalculationPanel.vue'
import OutputPanel from './components/OutputPanel.vue'

type ReferenceSpeedAxisModel = {
  values: number[]
  baseCount: number
  leftExtensionCount: number
  rightExtensionCount: number
}

type DiagramModel = {
  shafts: string[]
  yAxisLabels: string[]
  lines: Array<{ coords: [[string, string], [string, string]] }>
  scatterPoints: Array<[string, string]>
  middleShaftIndices: number[]
}

const inputSpeed = ref<number>(1500)
const outputSpeed = ref<number>(0)
const outputSpeedMin = ref<number>(1000)
const outputSpeedMax = ref<number>(2000)
const gearStages = ref<number>(5)
const transmissionMode = ref<'speed-down' | 'speed-up'>('speed-down')
const selectedSchema = ref<number[] | null>(null)
const referenceSpeedAxis = ref<ReferenceSpeedAxisModel | null>(null)
const transmissionDirection = ref<'down' | 'up'>('down')
const diagramData = ref<DiagramModel | null>(null)

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
          v-model:referenceSpeedAxis="referenceSpeedAxis"
          v-model:transmissionDirection="transmissionDirection"
          v-model:diagramData="diagramData"
        />

        <OutputPanel 
          :diagramData="diagramData"
        />

      </v-container>
    </v-main>
  </v-app>
</template>

<style scoped>
/* App 级别的样式放到这里，其他的移到了组件里 */
</style>
