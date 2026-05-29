<script setup lang="ts">
const inputSpeed = defineModel<number | null>('inputSpeed')
const outputSpeedMin = defineModel<number | null>('outputSpeedMin')
const outputSpeedMax = defineModel<number | null>('outputSpeedMax')
const gearStages = defineModel<number>('gearStages', { default: 1 })
const transmissionMode = defineModel<'speed-down' | 'speed-up'>('transmissionMode', { default: 'speed-down' })
</script>

<template>
  <v-card class="mb-6" elevation="2">
    <v-card-title class="text-h5 font-weight-bold pb-2">设计驱动参数</v-card-title>
    <v-card-text>
      <v-row class="mt-2">
        <v-col cols="12" md="4">
          <v-text-field
            v-model.number="inputSpeed"
            label="输入转速 (rpm)"
            type="number"
            variant="outlined"
            density="comfortable"
            hide-details
          ></v-text-field>
        </v-col>
        <v-col cols="12" sm="6" md="4">
          <v-text-field
            v-model.number="outputSpeedMin"
            label="输出转速下限 (rpm)"
            type="number"
            variant="outlined"
            density="comfortable"
            hide-details
          ></v-text-field>
        </v-col>
        <v-col cols="12" sm="6" md="4">
          <v-text-field
            v-model.number="outputSpeedMax"
            label="输出转速上限 (rpm)"
            type="number"
            variant="outlined"
            density="comfortable"
            hide-details
          ></v-text-field>
        </v-col>
      </v-row>
      <v-row align="center">
        <v-col cols="12" md="4">
          <div class="text-subtitle-2 mb-2 text-grey-darken-1">传动类型</div>
          <div class="sliding-toggle">
            <div 
              class="sliding-bg" 
              :class="transmissionMode === 'speed-down' ? 'pos-left' : 'pos-right'"
            ></div>
            <button 
              type="button" 
              class="toggle-btn" 
              :class="{ active: transmissionMode === 'speed-down' }" 
              @click="transmissionMode = 'speed-down'"
            >
              降速
            </button>
            <button 
              type="button" 
              class="toggle-btn" 
              :class="{ active: transmissionMode === 'speed-up' }" 
              @click="transmissionMode = 'speed-up'"
            >
              升速
            </button>
          </div>
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
    </v-card-text>
  </v-card>
</template>

<style scoped>
/* 自定义滑动切换控件（类似 iOS Segmented Control） */
.sliding-toggle {
  position: relative;
  display: inline-flex;
  background-color: #f1f5f9;
  border-radius: 9999px;
  padding: 4px;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.1);
}

.sliding-bg {
  position: absolute;
  top: 4px;
  bottom: 4px;
  width: calc(50% - 4px);
  background-color: #1867c0; /* Vuetify Primary Color */
  border-radius: 9999px;
  transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.15);
}

.sliding-bg.pos-left {
  transform: translateX(0);
}

.sliding-bg.pos-right {
  transform: translateX(100%);
}

.toggle-btn {
  position: relative;
  z-index: 1;
  width: 90px;
  border: none;
  background: transparent;
  padding: 8px 16px;
  border-radius: 9999px;
  cursor: pointer;
  font-weight: 700;
  font-size: 0.95rem;
  transition: color 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.toggle-btn.active {
  color: white;
}

.toggle-btn:not(.active) {
  color: #64748b;
}

.toggle-btn:not(.active):hover {
  color: #334155;
}
</style>