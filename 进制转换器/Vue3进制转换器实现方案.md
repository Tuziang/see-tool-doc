# 聊聊那个“十进制中转”的转换器逻辑

最近实在受不了网上那些满屏广告的转换工具了，干脆自己撸了一个。
本来以为也就是写几个 `parseInt` 的事，结果写起来发现为了让手感顺滑，还是有不少细节得抠。记一下几个比较有意思的点，当做备忘。


在线工具网址：[https://see-tool.com/base-converter](https://see-tool.com/base-converter)

工具截图：
![在这里插入图片描述](Snipaste_2026-01-23_19-03-32.png.png)


## 1. 核心思路：找个“老实人”做中转

最开始构思的时候，差点掉进“排列组合”的坑里。
要是写具体的“二转八”、“八转十六”、“十六转二”... 这代码量得按阶乘增长，维护起来绝对火葬场。
最聪明的办法还是**找个中转站——十进制**。
不管用户往哪个框里输，先把它强转成十进制（Number），然后再把这个十进制数分发给其他进制的框。
复杂度直接降维打击，瞬间清爽。

核心函数 `onSyncInput` 就干了这么点事：

```javascript
const onSyncInput = (source) => {
  // 重点一：防死锁。
  // 因为A框变了改B框，B框变了又想改A框，不加锁就是死循环。
  if (isUpdating) return
  isUpdating = true

  let decimalNum = 0
  
  // 第一步：清洗 + 归一化
  // 比如 hex 里的 0x，或者用户复制粘贴带进来的空格，都得干掉
  if (source === 'hex') {
     // 用正则把非 hex 字符全洗掉，保证 parseInt 不报错
     const clean = cleanValue.replace(/[^0-9A-Fa-f]/g, '')
     decimalNum = parseInt(clean, 16)
  } 
  // ...其他进制同理

  // 第二步：广播更新
  // 如果是当前输入的框（source），就别回填了，不然光标会乱跳
  if (source !== 'binary') binaryValue.value = formatValue(decimalNum.toString(2), 2)
  if (source !== 'octal')  octalValue.value = formatValue(decimalNum.toString(8), 8)
  
  // 完事解锁
  isUpdating = false
}
```

## 2. 也是才知道：原生支持到 36 进制

以前一直以为 `parseInt` 和 `toString` 也就是处理下常见的 2/8/10/16 进制。
查了文档才发现，这俩货最大支持到 **36 进制**（0-9 + a-z，正好 36 个符号）。
所以那个看起来很高大上的“自定义进制”功能，其实就是捡了个现成的便宜：

```javascript
// 比如要把 7 进制转成 32 进制
// 1. 任意进制 -> 十进制
const decimal = parseInt(input, fromBase)
// 2. 十进制 -> 任意进制
const result = decimal.toString(toBase)
```
连算法都不用自己写，JS 引擎底层优化得比我好多了。

## 3. 那些让体验变好的“小细节”

### 脏数据清洗
用户经常直接从别的地方复制一串代码进来，比如 `0x1A2B` 或者 `1 0010`。
如果在解析前不把这些空格、前缀洗干净，`parseInt` 分分钟解析出 `NaN` 或者截断的错误的数。
所以我在输入处理里加了比较严格的正则清洗，只留该进制允许的字符，其他的全当没看见。

### 视觉降噪（Formatting）
一长串 `110101010111...` 没人能看数清楚。
在此我做了一个微小的格式化工作：
*   **二进制/十六进制**：每 4 位切一刀，加个空格。
*   **十进制/八进制**：习惯上是 3 位一组（千分位）。
逻辑其实就是简单的字符串切割，但对可读性的提升是巨大的。不过要注意，**复制**的时候得把这些空格去掉，不然用户粘到代码里报错了还得骂我。

### 大数的坑
目前是用 `parseInt` (Number 类型) 扛着的。这有个隐患：超过 `Number.MAX_SAFE_INTEGER` (2^53 - 1) 精度就会丢。
日常用用足够了，真要处理超大整数，以后可能得换成 `BigInt`。不过那就是另一个故事了，`BigInt` 和 `Number` 混算得小心翼翼，暂且先不折腾。

---

总的来说，这个工具虽然简单，但把**防抖（Lock）、数据清洗、格式化、原生API利用**这几点串起来，也是个挺完整的练手小玩意儿。
