# Base64编码/解码 核心JS实现

这篇只讲核心 JavaScript：输入如何进入转换管线、三种 Base64 格式如何统一处理、文本与文件模式如何共用同一套规则。

> 在线工具网址：[https://see-tool.com/base64-converter](https://see-tool.com/base64-converter)  
> 工具截图：  
> ![工具截图](工具截图.png)

## 1）状态拆分：文本模式与文件模式并行

工具把两种使用场景完全拆开管理，避免互相覆盖结果。

```js
const activeTab = ref('text')

// 文本模式
const mode = ref('encode')
const inputText = ref('')
const outputText = ref('')
const textBase64Format = ref('standard')

// 文件模式
const fileMode = ref('encode')
const selectedFile = ref(null)
const processedFileData = ref(null)
const fileBase64Format = ref('standard')
const fileResult = ref(false)
const fileResultInfo = ref('')
```

这种拆分让逻辑有两个好处：

- 文本转换失败不会污染文件状态。
- 文件二进制缓存不会进入文本输入输出链路。

格式模型只保留三个值，所有编码/解码都围绕这三个值展开：

```js
const base64FormatOptions = [
  { value: 'standard' },
  { value: 'mime' },
  { value: 'urlSafe' }
]
```

## 2）文本转换入口：`performConvert`

文本输入最终都进入一个入口函数，先判断空输入，再按模式分支。

```js
const performConvert = () => {
  if (!inputText.value) {
    outputText.value = ''
    return
  }

  try {
    if (mode.value === 'encode') {
      const encoder = new TextEncoder()
      const uint8Array = encoder.encode(inputText.value)
      const binaryStr = Array.from(uint8Array).map(b => String.fromCharCode(b)).join('')
      const encoded = btoa(binaryStr)
      outputText.value = formatBase64ByType(encoded, textBase64Format.value)
    } else {
      try {
        const normalized = normalizeBase64Input(inputText.value, textBase64Format.value)
        const decoded = atob(normalized)
        const bytes = new Uint8Array(decoded.length)
        for (let i = 0; i < decoded.length; i++) {
          bytes[i] = decoded.charCodeAt(i)
        }
        const decoder = new TextDecoder('utf-8')
        outputText.value = decoder.decode(bytes)
      } catch (e) {
        outputText.value = '解码失败，请检查输入内容和格式'
      }
    }
  } catch (e) {
    outputText.value = `转换失败：${e.message}`
  }
}
```

这个入口的关键设计点：

- 编码链路与解码链路都用 UTF-8 字节作为中间态，避免中文乱码。
- 解码分支单独再包一层 `try/catch`，把格式错误与系统错误区分开。

## 3）为什么不能直接 `btoa(input)`

`btoa/atob` 本质处理的是“字节串（0-255）”，不是任意 Unicode 字符串。

如果直接对中文文本 `btoa('你好')`，会抛错。正确做法是：

1. `TextEncoder` 把文本编码为 UTF-8 字节。
2. 把每个字节映射为单字符二进制串。
3. `btoa` 对二进制串编码。

对应实现：

```js
const encoder = new TextEncoder()
const uint8Array = encoder.encode(inputText.value)
const binaryStr = Array.from(uint8Array).map(b => String.fromCharCode(b)).join('')
const encoded = btoa(binaryStr)
```

解码时反向执行：

```js
const decoded = atob(normalized)
const bytes = new Uint8Array(decoded.length)
for (let i = 0; i < decoded.length; i++) {
  bytes[i] = decoded.charCodeAt(i)
}
const decoder = new TextDecoder('utf-8')
const text = decoder.decode(bytes)
```

这样才能保证多字节字符和 Emoji 都能正确往返。

## 4）三种格式的统一输出：`formatBase64ByType`

编码后的 Base64 字符串统一走一个格式化函数。

```js
const formatBase64ByType = (value, formatType) => {
  if (!value) return ''

  if (formatType === 'mime') {
    const chunks = value.match(/.{1,76}/g)
    return chunks ? chunks.join('\n') : value
  }

  if (formatType === 'urlSafe') {
    return value
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/g, '')
  }

  return value
}
```

规则对应关系：

- `standard`：保持原样。
- `mime`：每 76 字符换行，符合邮件场景常见格式。
- `urlSafe`：字符集替换并移除尾部 `=`。

## 5）三种格式的统一输入：`normalizeBase64Input`

解码前一定先归一化，不然不同来源的数据很容易失败。

```js
const normalizeBase64Input = (value, formatType) => {
  let text = String(value || '').trim().replace(/\s+/g, '')

  if (formatType === 'urlSafe') {
    text = text.replace(/-/g, '+').replace(/_/g, '/')
  }

  const remainder = text.length % 4
  if (remainder === 1) {
    throw new Error('invalid base64 length')
  }
  if (remainder > 0) {
    text += '='.repeat(4 - remainder)
  }

  return text
}
```

这段逻辑做了三层防护：

- 去掉所有空白字符，兼容多行 MIME 输入。
- URL Safe 字符回转为标准字符集。
- 自动补齐 padding；长度余数为 1 时直接判定非法。

“余数为 1 非法”是 Base64 的结构性约束：有效输入长度不可能出现 `4n + 1`。

## 6）文本输入防抖：避免频繁触发

输入框每次变动不立即转换，而是延迟 300ms，减少连续键入时的重复计算。

```js
let debounceTimer = null

const handleInput = () => {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => {
    performConvert()
  }, 300)
}
```

销毁阶段清理定时器，避免组件卸载后残留回调：

```js
onUnmounted(() => {
  if (debounceTimer) {
    clearTimeout(debounceTimer)
  }
})
```

## 7）文件输入：选择与拖拽统一到同一处理器

文件来源有两个入口：文件选择和拖拽，最终都只做三件事：

1. 记录文件对象。
2. 计算可读文件大小文本。
3. 调用 `processFile()`。

```js
const handleFileSelect = (event) => {
  const file = event.target.files[0]
  if (file) {
    selectedFile.value = file
    fileSize.value = formatFileSize(file.size)
    processFile()
  }
}

const handleFileDrop = (event) => {
  const file = event.dataTransfer.files[0]
  if (file) {
    selectedFile.value = file
    fileSize.value = formatFileSize(file.size)
    processFile()
  }
}
```

## 8）文件编码：`ArrayBuffer -> Base64`

文件编码时按原始字节读取，不经文本解码，避免二进制文件损坏。

```js
if (fileMode.value === 'encode') {
  reader.onload = (e) => {
    const arrayBuffer = e.target.result
    const bytes = new Uint8Array(arrayBuffer)
    const binaryStr = Array.from(bytes).map(b => String.fromCharCode(b)).join('')
    const encoded = btoa(binaryStr)
    processedFileData.value = formatBase64ByType(encoded, fileBase64Format.value)
    const resultSize = formatFileSize(processedFileData.value.length)
    fileResultInfo.value = `编码完成，结果大小：${resultSize}`
    fileResult.value = true
  }
  reader.readAsArrayBuffer(selectedFile.value)
}
```

这里 `processedFileData` 是字符串，后续下载按文本文件输出。

## 9）文件解码：`Base64 文本 -> Uint8Array`

文件解码先按文本读取，再还原字节数组。

```js
reader.onload = (e) => {
  try {
    const base64Str = e.target.result
    const normalized = normalizeBase64Input(base64Str, fileBase64Format.value)
    const decoded = atob(normalized)
    const bytes = new Uint8Array(decoded.length)
    for (let i = 0; i < decoded.length; i++) {
      bytes[i] = decoded.charCodeAt(i)
    }
    processedFileData.value = bytes
    const resultSize = formatFileSize(bytes.length)
    fileResultInfo.value = `解码完成，结果大小：${resultSize}`
    fileResult.value = true
  } catch (err) {
    // 提示“解码失败”
  }
}
reader.readAsText(selectedFile.value)
```

解码完成后 `processedFileData` 是 `Uint8Array`，下载时按二进制 Blob 输出。

## 10）下载逻辑：根据当前模式切换 Blob 类型

文本结果下载：

```js
const blob = new Blob([outputText.value], { type: 'text/plain' })
const url = URL.createObjectURL(blob)
const a = document.createElement('a')
a.href = url
a.download = mode.value === 'encode' ? 'base64-encoded.txt' : 'base64-decoded.txt'
a.click()
URL.revokeObjectURL(url)
```

文件结果下载：

```js
if (fileMode.value === 'encode') {
  blob = new Blob([processedFileData.value], { type: 'text/plain' })
  filename = 'encoded_' + (selectedFile.value?.name || 'file')
} else {
  blob = new Blob([processedFileData.value], { type: 'application/octet-stream' })
  const originalName = selectedFile.value?.name || 'file'
  filename = 'decoded_' + originalName.replace(/^encoded_/, '')
}
```

这部分的核心是“模式驱动输出类型”，避免把二进制误当文本写出。

## 11）辅助功能的实现思路

除了转换主流程，工具还补齐了常用操作：

- 清空输入：同步清空输入与输出。
- 复制输入/输出：调用 `navigator.clipboard.writeText`。
- 粘贴输入：`navigator.clipboard.readText` 后直接触发转换。
- 文件大小格式化：按 `B / KB / MB` 输出可读文本。

文件大小格式化函数：

```js
const formatFileSize = (bytes) => {
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(2) + ' KB'
  return (bytes / (1024 * 1024)).toFixed(2) + ' MB'
}
```

## 12）这套实现的核心抽象

从 JS 结构上看，整个工具可以拆成四层：

1. 输入层：文本输入、文件输入、剪贴板输入。
2. 归一化层：`normalizeBase64Input` 与格式转换前处理。
3. 转换层：`TextEncoder/TextDecoder + btoa/atob`、`FileReader`。
4. 输出层：格式化显示、下载、结果状态提示。

因为每层职责单一，所以文本与文件两条链路虽然输入不同，但能稳定共用同一套 Base64 格式规则。
