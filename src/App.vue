<script setup lang="ts">
import { ref, computed } from 'vue'

const inputSpeed = ref<number | null>(null)
const outputSpeedMin = ref<number | null>(null)
const outputSpeedMax = ref<number | null>(null)
const gearStages = ref<number>(3) // 默认为 3 级
const transmissionMode = ref<'speed-down' | 'speed-up'>('speed-down') // 升降速传动选择

// 质因数分解
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

// 获取所有不重复的排列组合（对应变速箱的不同传动方案结构）
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
  const n = gearStages.value
  if (!n || n < 1) return []
  if (n === 1) return [{ structure: [1], isRecommended: true, text: '1 = 1' }]
  
  const factors = getPrimeFactors(n)
  const perms = getUniquePermutations(factors)

  return perms.map(p => {
    // 检查是否完美符合升/降序要求
    let isRecommended = false
    if (transmissionMode.value === 'speed-down') {
      // 降速传动：前大后小更为平稳（或遵循特定行业习惯：这里按照要求“依次升高”即[小,中,大]放在最前面推荐，即升序）
      // 用户要求：“降速传动，几个质因数大小依次升高的方案置顶”
      isRecommended = p.every((val, i, arr) => !i || (val >= (arr[i - 1] as number)))
    } else {
      // 升速传动：用户要求“几个质因数大小依次降低的方案置顶”
      isRecommended = p.every((val, i, arr) => !i || (val <= (arr[i - 1] as number)))
    }

    return {
      structure: p,
      isRecommended,
      text: `${p.join(' × ')} = ${n}`
    }
  }).sort((a, b) => Number(b.isRecommended) - Number(a.isRecommended))
})
</script>

<template>
  <v-app>
    <v-main>
      <v-container class="py-8" max-width="960">
        
        <!-- 输入区域 -->
        <v-card class="mb-6" elevation="2">
          <v-card-title class="text-h5 font-weight-bold pb-2">设计驱动参数</v-card-title>
          <v-card-text>
            <v-row class="mt-2">
              <v-col cols="12" md="4">
                <v-text-field
                  v-model.number="inputSpeed"
                  label="输入转速 (rpm)"
                  type="number"
                  placeholder="例如: 1500"
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
                  placeholder="下限"
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
                  placeholder="上限"
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

        <!-- 运算区域 -->
        <v-card class="mb-6" elevation="2">
          <v-card-title class="text-h5 font-weight-bold">计算过程</v-card-title>
          <v-card-text>
            <div class="py-4">
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
                    :class="item.isRecommended ? 'border-primary bg-blue-lighten-5' : 'border-grey-lighten-2'"
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

        <!-- 输出区域 -->
        <v-card elevation="2">
          <v-card-title class="text-h5 font-weight-bold">参考方案</v-card-title>
          <v-card-text>
            <div class="text-body-1 text-grey-darken-1 py-4">
              <!-- 最终方案占位 -->
              传动方案生成结果将展示在这里...
            </div>
          </v-card-text>
        </v-card>

      </v-container>
    </v-main>
  </v-app>
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
  width: 90px; /* 固定宽度保证滑块比例 */
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

/* 使用 Vuetify 全局样式及栅格系统，移除旧有自定义样式 */
</style>
