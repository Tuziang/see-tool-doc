# BMI计算器工具开发技术实现

本文主要分享一下我最近开发的 BMI 计算器工具的技术实现细节。这个工具基于 Vue 3 和 Nuxt.js 构建，包含核心计算逻辑和交互式的用户界面。我们将重点关注其功能实现部分。

> 在线工具网址：[https://see-tool.com/bmi-calculator](https://see-tool.com/bmi-calculator)

> 工具截图：
> ![工具截图](工具截图.png)

## 项目结构

这个工具的实现主要分为两个部分：

1.  **逻辑层**：`utils/bmi-calculator.js` —— 负责核心的 BMI 数值计算和状态判定。
2.  **视图层**：`pages/bmi-calculator.vue` —— 负责用户交互、输入验证和结果展示。

## 1. 核心计算逻辑

计算逻辑封装在 `calculateBmi` 函数中。它接收用户的身高（cm）和体重（kg）作为输入，返回计算后的 BMI 值以及对应的身体状态类别和健康风险等级。

### 1.1 输入验证

在进行计算之前，我们需要确保输入的数据是有效的数值且大于 0。如果输入无效，函数会抛出一个错误，以便前端捕获处理。

```javascript
  const height = Number(heightCm)
  const weight = Number(weightKg)

  if (!Number.isFinite(height) || !Number.isFinite(weight) || height <= 0 || weight <= 0) {
    throw new Error('INVALID_INPUT')
  }
```

### 1.2 BMI 计算公式

BMI 的计算公式是：体重（公斤）除以身高（米）的平方。

```javascript
  const heightInMeters = height / 100
  // 体重 / (身高^2)
  const bmiRaw = weight / (heightInMeters * heightInMeters)
  // 保留一位小数
  const bmi = Number(bmiRaw.toFixed(1))
```

### 1.3 状态判定

根据计算出的 BMI 值，我们可以判定用户的身体状态。这里我们参照了常见的 BMI 标准进行分类：

*   **BMI < 18.5**: 偏瘦（Underweight），存在营养不良风险。
*   **18.5 ≤ BMI < 24**: 正常（Normal），健康风险低。
*   **24 ≤ BMI < 28**: 超重（Overweight），通过轻度风险。
*   **BMI ≥ 28**: 肥胖（Obese），存在较高健康风险。

```javascript
  if (bmi < 18.5) {
    return { bmi, categoryKey: 'underweight', riskKey: 'malnutrition' }
  }
  if (bmi < 24) {
    return { bmi, categoryKey: 'normal', riskKey: 'low' }
  }
  if (bmi < 28) {
    return { bmi, categoryKey: 'overweight', riskKey: 'mild' }
  }
  return { bmi, categoryKey: 'obese', riskKey: 'high' }
```

## 2. Vue 页面实现

页面组件主要由输入表单和结果展示两大部分组成。使用 Vue 3 的 Composition API (`<script setup>`) 来管理状态和逻辑。

### 2.1 状态管理

我们使用 `ref` 来定义响应式变量，用于存储用户的输入和计算结果。

```javascript
const heightCm = ref('')  // 用户输入的身高
const weightKg = ref('')  // 用户输入的体重
const result = ref(null)  // 用于存储计算结果对象，初始为 null
```

### 2.2 用户交互处理

#### 计算操作
当用户点击“计算”按钮或在体重输入框按下回车时，会触发 `handleCalculate` 方法。

该方法首先调用核心计算函数 `calculateBmi`。如果计算成功，将结果赋值给 `result`，页面会自动渲染结果区域；如果捕获到错误（如输入无效），则会提示用户。

```javascript
const handleCalculate = () => {
  try {
    // 调用工具函数进行计算
    const r = calculateBmi(Number(heightCm.value), Number(weightKg.value))
    result.value = r
  } catch (e) {
    // 计算失败，清空结果并提示错误
    result.value = null
    safeMessage('error', '请输入有效的身高和体重')
  }
}
```

#### 加载示例
为了方便用户快速体验，我们提供了一个 `loadExample` 方法，一键填入预设的示例数据并触发计算。

```javascript
const loadExample = () => {
  heightCm.value = '170'
  weightKg.value = '65'
  handleCalculate()
}
```

#### 清空重置
`clearForm` 方法用于重置所有输入和结果，让用户可以重新开始。

```javascript
const clearForm = () => {
  heightCm.value = ''
  weightKg.value = ''
  result.value = null
}
```

### 2.3 结果动态展示

在模板中，我们使用 `v-if="result"` 来控制结果卡片的显示。只有当 `result` 有值时，结果区域才会渲染。这种设计保证了页面初始状态的整洁。

结果卡片通过 grid 布局展示了三个关键信息：BMI 数值、身体状态和健康风险。这些信息都直接来自于 `result` 对象。

```html
<div v-if="result" class="...">
  <!-- BMI 数值 -->
  <p>{{ result.bmi }}</p>
  
  <!-- 身体状态分类 -->
  <p>{{ t(`bmiCalculator.result.categoryMap.${result.categoryKey}`) }}</p>
  
  <!-- 健康风险评估 -->
  <p>{{ t(`bmiCalculator.result.riskMap.${result.riskKey}`) }}</p>
</div>
```

通过将计算逻辑与界面展示分离，我们保持了代码的清晰和可维护性。Vue 强大的响应式系统让我们能够轻松地通过改变数据状态来驱动界面的更新。
