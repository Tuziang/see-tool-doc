# 文本行过滤/筛选 在线工具核心JS实现

这个工具的核心目标很直接：给定一段多行文本和多条条件，快速得到“保留匹配行”或“删除匹配行”的结果。实现上我把逻辑拆成两层：**过滤引擎**负责纯文本计算，**页面逻辑**负责参数整理、错误提示、复制与下载。

> 在线工具网址：[https://see-tool.com/text-line-filter](https://see-tool.com/text-line-filter)  
> 工具截图：  
> ![工具截图](工具截图.png)

## 过滤引擎的输入约定

过滤函数接收三部分数据：原始文本、条件数组、选项对象。选项统一收口后，后续分支会非常清晰。

```javascript
function filterLines(inputText, filterConditions, options) {
  const {
    filterMode = 'contains', // contains | exact
    useRegex = false,        // 是否启用正则
    ignoreCase = true,       // 是否忽略大小写
    matchAll = false,        // false=任一命中，true=全部命中
    action = 'keep'          // keep | remove
  } = options

  if (!inputText || !filterConditions || filterConditions.length === 0) {
    return []
  }

  const lines = inputText.split('\n')
  const matchedLines = []
  // 后续按行处理
}
```

这里有两个关键点：

- 条件以数组传入，避免在核心函数里再做字符串切分。
- 空输入直接返回空数组，调用方只需要处理“有结果/无结果”两种状态。

## 单行匹配：普通文本与正则双通道

每一行都会遍历全部条件，得到一个布尔数组 `matchResults`。这个数组是后面 AND/OR 逻辑的基础。

```javascript
for (const line of lines) {
  const matchResults = []

  for (const condition of filterConditions) {
    let isMatch = false

    if (useRegex) {
      const flags = ignoreCase ? 'i' : ''
      const regex = new RegExp(condition, flags)
      isMatch = regex.test(line)
    } else {
      const searchText = ignoreCase ? condition.toLowerCase() : condition
      const lineText = ignoreCase ? line.toLowerCase() : line

      if (filterMode === 'contains') {
        isMatch = lineText.includes(searchText)
      } else {
        isMatch = lineText === searchText
      }
    }

    matchResults.push(isMatch)
  }
}
```

这段实现解决了三个常见需求：

- **模糊匹配**：`includes` 处理“行内包含关键词”。
- **整行匹配**：`===` 处理“整行完全一致”。
- **正则匹配**：由用户条件直接构造 `RegExp`，支持复杂表达式。

## 多条件组合：OR 与 AND

工具支持两种条件关系：

- `matchAll = false`：任意一个条件命中即视为命中（OR）
- `matchAll = true`：所有条件都命中才算命中（AND）

实现非常直接：

```javascript
const finalMatch = matchAll
  ? matchResults.every(result => result)
  : matchResults.some(result => result)
```

有了 `finalMatch` 后，再叠加动作类型：

```javascript
if ((action === 'keep' && finalMatch) || (action === 'remove' && !finalMatch)) {
  matchedLines.push(line)
}
```

这一步把“保留匹配”和“删除匹配”统一到了同一套流程里，避免写两份几乎重复的过滤逻辑。

## 正则错误处理

正则场景最容易出现语法错误（比如括号未闭合）。引擎层统一用 `try/catch` 包裹，在异常时抛出可读错误：

```javascript
try {
  // 过滤主流程
} catch (error) {
  throw new Error('正则表达式语法错误：' + error.message)
}
```

这样页面层只需要捕获一次并提示用户，不需要关心底层失败细节。

## 页面逻辑：参数整理与结果回填

页面层主要做四件事：

1. 校验输入文本和条件是否为空。
2. 把条件文本按换行拆成数组并过滤空行。
3. 调用过滤引擎并拿到结果数组。
4. 用 `join('\n')` 回填到输出框。

核心调用方式：

```javascript
const conditions = filterConditionsText
  .split('\n')
  .filter(condition => condition.trim())

const result = TextLineFilter.filterLines(inputText, conditions, {
  filterMode,
  useRegex,
  ignoreCase,
  matchAll,
  action
})

outputText = result.length > 0 ? result.join('\n') : ''
```

这里刻意让页面层只承担“数据进出”，把匹配规则全部留在引擎层，后续扩展新选项时改动面会更小。

## 复制与下载的JS闭环

过滤完成后，工具支持直接复制和导出文本。复制优先使用现代剪贴板 API，失败时降级到 `textarea + execCommand('copy')`，保证更多浏览器可用。

下载则通过 `Blob` 生成文本文件，并使用临时 `<a>` 标签触发保存：

```javascript
const blob = new Blob([outputText], { type: 'text/plain;charset=utf-8' })
const url = URL.createObjectURL(blob)

const a = document.createElement('a')
a.href = url
a.download = `过滤结果-${dateStr}.txt`
a.click()

URL.revokeObjectURL(url)
```

到这里，文本输入、条件匹配、结果输出、复制下载就形成了完整的功能链路。
