# Gearbox Designer - 变速箱设计工具

基于 Vue 3 的变速箱齿轮级数设计可视化工具，用于计算公比、等比序列、传动方案分解及转速图生成。

## 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Vue 3 | ^3.5.32 | 前端框架（Composition API） |
| TypeScript | ~6.0.0 | 类型安全 |
| Vuetify | ^4.0.8 | UI 组件库 |
| ECharts | ^6.1.0 | 图表渲染 |
| Vite | ^8.0.8 | 构建工具 |

## 项目结构

```
src/
├── main.ts                    # 应用入口，初始化 Vue 和 Vuetify
├── App.vue                    # 顶层状态管理
├── assets/
│   └── recommended.svg        # 推荐图标
└── components/
    ├── InputPanel.vue         # 输入面板（驱动参数）
    ├── CalculationPanel.vue   # 计算面板（核心逻辑）
    └── OutputPanel.vue        # 输出面板（转速图）
```

## 组件架构

### 1. App.vue - 顶层状态管理

负责管理全局状态，通过 `v-model` 或 `props` 传递给子组件。

**核心变量：**

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `inputSpeed` | `number` | 输入转速 (rpm)，默认 1500 |
| `outputSpeed` | `number` | 输出转速（单级模式），默认 0 |
| `outputSpeedMin` | `number` | 输出转速下限，默认 1000 |
| `outputSpeedMax` | `number` | 输出转速上限，默认 2000 |
| `gearStages` | `number` | 变速级数，默认 5 |
| `transmissionMode` | `'speed-down' \| 'speed-up'` | 传动模式 |
| `selectedSchema` | `number[] \| null` | 选定的传动方案（质因数数组） |
| `referenceSpeedAxis` | `ReferenceSpeedAxisModel \| null` | 参考转速轴数据 |
| `selectedExpansionSteps` | `number[] \| null` | 选定的扩大顺序步距 |
| `transmissionDirection` | `'down' \| 'up'` | 传动方向 |
| `diagramData` | `DiagramModel \| null` | 转速图数据 |

**类型定义：**

```typescript
type ReferenceSpeedAxisModel = {
  values: number[]           // 所有轴上的转速值
  baseCount: number          // 基础序列数量
  leftExtensionCount: number // 左侧延伸数量
  rightExtensionCount: number // 右侧延伸数量
}

type DiagramModel = {
  shafts: string[]                                      // 轴名称（罗马数字）
  yAxisLabels: string[]                                 // Y轴标签（转速值）
  lines: Array<{ coords: [[string, string], [string, string]] }>  // 连线数据
  scatterPoints: Array<[string, string]>                // 散点数据
}
```

**监听器：**
- 当 `gearStages === 1` 时，同步 `outputSpeed`、`outputSpeedMin`、`outputSpeedMax` 三个值

---

### 2. InputPanel.vue - 输入面板

通过 `defineModel` 暴露双向绑定参数，收集用户输入的设计驱动参数。

**双向绑定变量：**

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `inputSpeed` | `number` | 输入转速 |
| `outputSpeed` | `number` | 输出转速（单级） |
| `outputSpeedMin` | `number` | 输出转速下限 |
| `outputSpeedMax` | `number` | 输出转速上限 |
| `gearStages` | `number` | 变速级数（1-20） |

**约束逻辑：**
- 所有转速输入值强制非负（通过 watch 实现）

**UI 组件：**
- 输入转速：`v-text-field` 数字输入框
- 变速级数：`v-slider` 滑块（1-20）
- 输出转速：单级显示单个输入框，多级显示上下限两个输入框

---

### 3. CalculationPanel.vue - 计算面板（核心）

核心计算逻辑所在，包含公比计算、序列生成、方案分解等功能。

#### 3.1 Props（父组件传入）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| `gearStages` | `number` | 变速级数 |
| `transmissionMode` | `'speed-down' \| 'speed-up'` | 传动模式 |
| `inputSpeed` | `number \| null` | 输入转速 |
| `outputSpeedMin` | `number \| null` | 输出转速下限 |
| `outputSpeedMax` | `number \| null` | 输出转速上限 |

#### 3.2 双向绑定变量 (defineModel)

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `selectedSchema` | `number[] \| null` | 选定的传动方案 |
| `referenceSpeedAxis` | `ReferenceSpeedAxisModel \| null` | 参考转速轴 |
| `selectedExpansionSteps` | `number[] \| null` | 扩大顺序步距 |
| `transmissionDirection` | `'down' \| 'up'` | 传动方向 |
| `diagramData` | `DiagramModel \| null` | 转速图数据 |

#### 3.3 公比计算

**变量：**

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `theoreticalRatioRaw` | `computed<number \| null>` | 理论公比原始值 |
| `originalRatio` | `computed<string>` | 理论公比（4位小数） |
| `nonStandardRatio` | `computed<string>` | 非标公比（2位小数） |
| `standardRatios` | `number[]` | 预定义的7个标准公比：`[1.06, 1.12, 1.26, 1.41, 1.58, 1.78, 2.0]` |
| `selectedStandardRatio` | `ref<number \| null>` | 用户选择的标准公比 |
| `selectedRatioMode` | `ref<'standard' \| 'non-standard'>` | 公比模式 |
| `recommendedStandardRatio` | `computed<number \| null>` | 推荐的标准公比（最接近理论值） |
| `selectedActualRatio` | `computed<number \| null>` | 最终使用的公比值 |
| `ratioOptions` | `computed` | 公比选项列表（标准 + 非标） |

**公式：**
```
理论公比 = (输出转速上限 / 输出转速下限) ^ (1 / (级数 - 1))
```

**函数：**

| 函数名 | 参数 | 返回值 | 说明 |
|--------|------|--------|------|
| `selectRatioOption` | `value: number, isNonStandard: boolean` | `void` | 选择公比选项 |

#### 3.4 等比序列生成

**变量：**

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `geometricSequence` | `computed<number[]>` | 基础等比序列 |
| `sortedBaseSequence` | `computed<number[]>` | 排序后的基础序列 |

**生成规则：**
- 若选定公比 ≤ 理论公比：从输出下限开始向上乘
- 若选定公比 > 理论公比：从输出上限开始向下除

**函数：**

| 函数名 | 参数 | 返回值 | 说明 |
|--------|------|--------|------|
| `buildLeftExtensionSequence` | `minValue: number, ratio: number, steps: number` | `number[]` | 生成左侧延伸序列 |
| `buildRightExtensionSequence` | `maxValue: number, ratio: number, steps: number` | `number[]` | 生成右侧延伸序列 |
| `getExtensionSteps` | `ratio: number, outerValue: number, rawInput: number` | `number` | 计算延伸步数 |
| `getStandardInputSpeed` | `rawInput: number, sequence: number[], ratio: number` | `number \| null` | 获取标准化的输入转速 |

#### 3.5 箱型图模型

**变量：**

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `referenceSpeedAxisValue` | `computed` | 参考转速轴完整数据 |
| `boxChartModel` | `computed` | 箱型图渲染模型 |

**ECharts 相关：**

| 变量/函数 | 类型 | 说明 |
|-----------|------|------|
| `chartRef` | `ref<HTMLElement>` | 图表 DOM 引用 |
| `chartInstance` | `ECharts \| null` | ECharts 实例 |
| `initChart` | `() => void` | 初始化图表 |
| `disposeChart` | `() => void` | 销毁图表 |
| `updateChart` | `() => void` | 更新图表 |
| `resizeHandler` | `() => void` | 窗口大小变化处理 |

#### 3.6 传动类型判定

**变量：**

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `transmissionType` | `computed` | 传动类型信息 |

**判定规则：**
```
几何平均数 = √(输出下限 × 输出上限)
若 输入转速 < 几何平均数 → 升速传动
若 输入转速 ≥ 几何平均数 → 降速传动
```

#### 3.7 级数分解

**变量：**

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `processedPermutations` | `computed` | 所有分解方案（含推荐标记） |

**函数：**

| 函数名 | 参数 | 返回值 | 说明 |
|--------|------|--------|------|
| `getPrimeFactors` | `n: number` | `number[]` | 质因数分解 |
| `getUniquePermutations` | `elements: number[]` | `number[][]` | 获取不重复排列 |

**推荐规则：**
- 降速传动：推荐质因数逐渐减小的方案（前大后小）
- 升速传动：推荐质因数逐渐增加的方案（前小后大）

#### 3.8 扩大顺序方案

**变量：**

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `expansionSchemes` | `computed` | 所有扩大顺序方案 |

**函数：**

| 函数名 | 参数 | 返回值 | 说明 |
|--------|------|--------|------|
| `getAllLevelPermutations` | `n: number` | `number[][]` | 获取扩大级全排列 |
| `computeStepsFromLevels` | `schema: number[], expansionLevels: number[]` | `number[]` | 计算步距 |

**步距计算规则：**
```
基本组步距 = 1
第N扩大组步距 = 基本组级数 × 第1扩大组级数 × ... × 第(N-1)扩大组级数
```

#### 3.9 转速图数据生成

**变量：**

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `diagramDataValue` | `computed<DiagramModel \| null>` | 转速图数据 |

**辅助函数：**

| 函数名 | 参数 | 返回值 | 说明 |
|--------|------|--------|------|
| `getRoman` | `num: number` | `string` | 数字转罗马数字 |
| `formatSpeed` | `value: number, index: number` | `string` | 格式化转速值 |
| `getYAxisData` | - | `string[]` | 获取 Y 轴数据 |
| `getYAxisLabel` | `level: number` | `string \| null` | 获取指定层级的 Y 轴标签 |
| `getClosestAxisValue` | - | `string \| null` | 获取最接近输入转速的轴值 |

---

### 4. OutputPanel.vue - 输出面板

渲染转速图，展示传动方案的可视化结果。

**Props：**

| 属性名 | 类型 | 说明 |
|--------|------|------|
| `diagramData` | `DiagramModel \| null` | 转速图数据 |

**变量：**

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `chartRef` | `ref<HTMLElement>` | 图表 DOM 引用 |
| `chartInstance` | `ECharts \| null` | ECharts 实例 |
| `dynamicHeight` | `ref<string>` | 动态图表高度 |

**函数：**

| 函数名 | 说明 |
|--------|------|
| `updateChart` | 更新图表配置并渲染 |
| `handleResize` | 处理窗口大小变化 |

**图表配置：**
- X 轴：分类轴，显示轴名称（罗马数字 Ⅰ、Ⅱ、Ⅲ...）
- Y 轴：分类轴，显示转速值
- 系列 1：`lines` 类型，绘制传动连线
- 系列 2：`scatter` 类型，绘制转速节点

---

## 数据流

```
用户输入 (InputPanel)
    ↓ v-model
App.vue (状态管理)
    ↓ props
CalculationPanel (计算处理)
    ├→ 公比计算 → 等比序列
    ├→ 级数分解 → 传动方案
    ├→ 扩大顺序 → 步距方案
    └→ 转速图数据生成
         ↓ v-model
    App.vue
         ↓ props
    OutputPanel (图表渲染)
```

## 核心业务流程

1. **输入阶段**：用户输入输入转速、输出转速范围、变速级数
2. **公比计算**：根据输出转速范围和级数计算理论公比，标准化为标准公比或保留非标值
3. **序列生成**：基于选定公比生成等比序列，必要时延伸
4. **方案分解**：将级数分解为质因数排列，推荐最优方案
5. **扩大顺序**：为传动方案分配扩大级，计算步距
6. **图表渲染**：生成转速图，展示传动链路

## 启动开发

```bash
npm install
npm run dev
```

## 构建

```bash
npm run build
```
