<script setup lang="ts">
import { ref } from 'vue'

import InputPanel from './components/InputPanel.vue'
import CalculationPanel from './components/CalculationPanel.vue'
import OutputPanel from './components/OutputPanel.vue'

const inputSpeed = ref<number | null>(null)
const outputSpeedMin = ref<number | null>(null)
const outputSpeedMax = ref<number | null>(null)
const gearStages = ref<number>(1) // 默认为 1 级
const transmissionMode = ref<'speed-down' | 'speed-up'>('speed-down') // 升降速传动选择
const selectedSchema = ref<number[] | null>(null) // 选中的参考方案
</script>

<template>
  <v-app>
    <v-main>
      <v-container class="py-8" max-width="960">
        
        <InputPanel 
          v-model:inputSpeed="inputSpeed"
          v-model:outputSpeedMin="outputSpeedMin"
          v-model:outputSpeedMax="outputSpeedMax"
          v-model:gearStages="gearStages"
          v-model:transmissionMode="transmissionMode"
        />

        <CalculationPanel 
          :gearStages="gearStages"
          :transmissionMode="transmissionMode"
          :inputSpeed="inputSpeed"
          :outputSpeedMin="outputSpeedMin"
          :outputSpeedMax="outputSpeedMax"
          v-model:selectedSchema="selectedSchema"
        />

        <OutputPanel 
          :gearStages="gearStages"
          :schema="selectedSchema"
        />

      </v-container>
    </v-main>
  </v-app>
</template>

<style scoped>
/* App 级别的样式放到这里，其他的移到了组件里 */
</style>
