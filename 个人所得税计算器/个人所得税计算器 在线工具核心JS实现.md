# 个人所得税计算器 在线工具核心JS实现

这篇只讲功能层 JavaScript 实现。这个工具的核心思路是：把税率规则、五险一金规则、累计预扣法放进一个计算类里，输入 12 个月收入和扣除配置后，一次产出整年的月度明细。

> 在线工具网址：[https://see-tool.com/text-character-count](https://see-tool.com/text-character-count)  
> 工具截图：  
> ![工具截图](工具截图.png)

## 1. 核心数据结构

计算器初始化时，先准备三类基础数据：

- 累计预扣法税率表（含速算扣除数）
- 城市社保/公积金缴费基数上下限
- 每月减除费用（5000）

```javascript
class IncomeTaxCalculator {
  constructor() {
    // 累计预扣税率区间
    this.taxBrackets = [
      { min: -1, max: 36000, rate: 0.03, deduction: 0 },
      { min: 36000, max: 144000, rate: 0.10, deduction: 2520 },
      { min: 144000, max: 300000, rate: 0.20, deduction: 16920 },
      { min: 300000, max: 420000, rate: 0.25, deduction: 31920 },
      { min: 420000, max: 660000, rate: 0.30, deduction: 52920 },
      { min: 660000, max: 960000, rate: 0.35, deduction: 85920 },
      { min: 960000, max: Infinity, rate: 0.45, deduction: 181920 }
    ]

    // 每月减除费用
    this.monthlyDeduction = 5000
  }
}
```

这一步把规则常量和计算逻辑解耦，后续计算函数不需要硬编码税率数字。

## 2. 月度五险一金计算

月度扣除支持三种基数来源：

- 按月工资
- 单一自定义基数
- 养老/医疗/失业/公积金分别设置基数

关键点在“实际基数”处理：要同时受用户输入、封顶线、保底线、当月工资四个条件约束。

```javascript
// 计算实际缴费基数
const getActualBase = (baseValue, capValue, floorValue, monthlyIncome) => {
  // 统一兜底，避免 NaN 和负数
  const safeBase = Math.max(0, baseValue || 0)
  const safeCap = Math.max(0, capValue || 0)
  const safeFloor = Math.max(0, floorValue || 0)

  // 先做上限约束：不能超过上限，也不能超过月薪
  let result = safeCap > 0
    ? Math.min(safeBase, safeCap, monthlyIncome)
    : Math.min(safeBase, monthlyIncome)

  // 再做下限约束
  if (safeFloor > 0) result = Math.max(result, safeFloor)

  return result
}
```

得到各项实际基数后，再乘对应费率得到养老、医疗、失业、公积金金额，并汇总 `total`。这一层输出会直接参与个税应纳税所得额计算。

## 3. 累计预扣税额计算

个税函数按“累计应纳税所得额”查税率区间并套公式：

```javascript
calculateTaxForIncome(cumulativeIncome, cumulativeDeduction, cumulativeSpecialDeductions = 0) {
  // 累计应纳税所得额
  const taxableIncome = cumulativeIncome - cumulativeDeduction - cumulativeSpecialDeductions
  if (taxableIncome <= 0) return 0

  for (const bracket of this.taxBrackets) {
    if (taxableIncome > bracket.min && taxableIncome <= bracket.max) {
      return Math.max(0, taxableIncome * bracket.rate - bracket.deduction)
    }
  }

  // 兜底走最高档
  const highestBracket = this.taxBrackets[this.taxBrackets.length - 1]
  return Math.max(0, taxableIncome * highestBracket.rate - highestBracket.deduction)
}
```

这里返回的是“截至当前月的累计应纳税额”，不是当月税额。

## 4. 年度主流程：一次产出12个月明细

主流程会循环 12 次，每月做四件事：

1. 计算当月五险一金  
2. 更新累计收入、累计专项扣除、累计附加扣除  
3. 计算累计应纳税额  
4. 反推当月税额并得到税后收入

核心公式是：

```javascript
// 当月应纳税额 = 累计应纳税额 - 上月累计已纳税额
const monthlyTax = Math.max(0, cumulativeTaxAmount - cumulativeTax)

// 税后收入 = 税前收入 - 五险一金 - 当月个税
const afterTaxIncome = monthlyIncome - insurance.total - monthlyTax
```

每月结果会记录为结构化对象（收入、扣除、税额、税后、累计值等），最终返回一个 12 项数组，界面层可直接用于表格展示和汇总统计。

## 5. 工具方法

核心逻辑外还有两个实用方法：

- 金额格式化：把数字转成带千分位、保留两位小数的货币字符串
- 城市基数读取：按城市键返回对应的社保/公积金上下限配置

这两个方法让计算层对外输出更稳定，页面调用时不需要重复写格式化和城市映射逻辑。

整套实现的重点是“规则集中、计算分层、月度与累计并行维护”。这样既能保证个税计算口径一致，也方便后续扩展更多收入场景。
