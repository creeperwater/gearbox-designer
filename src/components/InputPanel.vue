<script setup lang="ts">
import { watch } from 'vue'

// InputPanel 说明：
// - 通过 `defineModel` 暴露与父组件双向绑定的参数：`inputSpeed`, `outputSpeed`, `outputSpeedMin`, `outputSpeedMax`, `gearStages`
// - 约束：所有输入值均强制非负
const inputSpeed = defineModel<number>('inputSpeed', { default: 0 })
const outputSpeed = defineModel<number>('outputSpeed', { default: 0 })
const outputSpeedMin = defineModel<number>('outputSpeedMin', { default: 0 })
const outputSpeedMax = defineModel<number>('outputSpeedMax', { default: 0 })
const gearStages = defineModel<number>('gearStages', { default: 1 })

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

      <v-divider class="my-3"></v-divider>
    </v-card-text>
  </v-card>
</template>