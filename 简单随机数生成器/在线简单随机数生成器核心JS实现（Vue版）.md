# 在线简单随机数生成器核心JS实现（Vue版）

这篇只讲这个工具的功能 JS。整体基于 Vue 组合式 API，核心流程是：

`参数输入 -> 参数校验 -> 生成随机数 -> 排序与拼接 -> 统计/复制/下载`

> 在线工具网址：[https://see-tool.com/random-number-generator](https://see-tool.com/random-number-generator)  
> 工具截图：  
> ![工具截图](工具截图.png)

## 1）状态建模：先把可变参数拆清楚

随机数工具看起来简单，但可变项不少：数字类型、是否去重、小数位、排序、分隔符、数量、范围、结果文本、结果数组。实现里把这些都放在响应式状态里，保证按钮点击、统计卡片、结果区同步更新。

```js
const numberType = ref("integer");
const uniqueNumbers = ref("false");
const decimalPlaces = ref(2);
const sortOrder = ref("none");
const separatorSelect = ref("\\n");
const customSeparator = ref("");
const count = ref(10);
const minValue = ref(0);
const maxValue = ref(100);
const result = ref("");
const generatedNumbers = ref([]);
```

## 2）参数校验：先拦截，再计算

生成动作入口先做三层校验：

1. `min < max`，否则区间无效。
2. `count` 限制在 `1 ~ 10000`。
3. 整数去重模式下，生成数量不能超过区间容量。

第三条是关键，区间容量通过 `Math.floor(max) - Math.ceil(min)` 计算，避免出现“要求不重复但可选值不够”的情况。

```js
if (minValue.value >= maxValue.value) return;
if (count.value <= 0 || count.value > 10000) return;

if (isUnique && numberType.value === "integer") {
  const range = Math.floor(maxValue.value) - Math.ceil(minValue.value);
  if (count.value > range) return;
}
```

## 3）随机生成核心：整数与小数两套逻辑

整数模式采用左闭右开区间 `[min, max)`：

```js
randomNum =
  Math.floor(Math.random() * (maxValue.value - minValue.value)) +
  minValue.value;
```

小数模式先算随机值，再按用户设定小数位做舍入：

```js
randomNum = Math.random() * (maxValue.value - minValue.value) + minValue.value;
randomNum = parseFloat(randomNum.toFixed(decimalPlaces.value));
```

这样一来，小数展示和复制结果保持一致，不会出现界面显示 2 位、复制出去是长小数的问题。

## 4）去重实现：Set + 重试上限

去重模式使用 `Set` 记录已出现数字。每次生成如果命中重复就继续抽，直到拿到新值或达到重试上限。

```js
const usedNumbers = new Set();
let attempts = 0;
const maxAttempts = 10000;

do {
  // 生成 randomNum
  attempts++;
} while (isUnique && usedNumbers.has(randomNum) && attempts < maxAttempts);
```

这个上限是防护措施，避免极端参数下死循环，保证函数一定能结束。

## 5）结果整理：排序、分隔符、文本输出一次完成

生成完数组后按选项排序，再用分隔符拼接成文本：

```js
if (sortOrder.value === "asc") numbers.sort((a, b) => a - b);
else if (sortOrder.value === "desc") numbers.sort((a, b) => b - a);

generatedNumbers.value = numbers;
result.value = numbers.join(separator);
```

分隔符支持预设和自定义，并把 `\n`、`\t` 转为真实换行和制表符，方便直接导出到文档或表格。

## 6）统计计算：基于结果数组实时派生

统计信息不单独存储，而是由 `generatedNumbers` 计算得到，包含数量、最小值、最大值、平均值。平均值通过 `reduce` 计算，展示前统一走格式化函数，避免浮点尾数影响阅读。

```js
const avg = numbers.reduce((sum, num) => sum + num, 0) / numbers.length;

const formatNumber = (num) => {
  if (Number.isInteger(num)) return num.toString();
  return parseFloat(num.toFixed(6)).toString();
};
```

## 7）交互闭环：复制、下载、清空

这部分是工具可用性的最后一环：

- 复制：调用 `navigator.clipboard.writeText(result)`。
- 下载：把结果写入 `Blob`，生成临时链接触发下载。
- 清空：重置结果文本和结果数组。

```js
const blob = new Blob([result.value], { type: "text/plain;charset=utf-8" });
const url = URL.createObjectURL(blob);
```

到这里，这个随机数工具的核心 JS 就完整了：先校验参数，再稳定生成，最后把结果以“可读、可复制、可下载”的形式交付给用户。
