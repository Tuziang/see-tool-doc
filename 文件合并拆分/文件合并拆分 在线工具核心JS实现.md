# 文件合并拆分 在线工具核心JS实现

这个工具最早是为了处理我自己的日志文件。一个文件几万行，直接发给别人不方便；反过来，多个小文件又很难一次看完。所以我把它做成了两个动作：**拆分**和**合并**。下面只聊核心 JavaScript。

> 在线工具网址：[https://see-tool.com/file-merge-split](https://see-tool.com/file-merge-split)  
> 工具截图：  
> ![工具截图](工具截图.png)

## 先把数据结构定死

我一开始踩过坑：拆分结果和合并结果字段不一致，后面下载和展示写起来很别扭。后来统一成一个最小结构，事情就顺了。

```javascript
{
  name: 'orders_part_001.csv',
  content: '...文本...',
  lineCount: 1000,
  byteSize: 32768
}
```

只要是“输出文件”，都长这个样子。后续逻辑基本就能复用。

## 读文件之前，先做标准化

文本处理最烦的是换行。Windows 常见 `\r\n`，旧系统可能是 `\r`，如果不统一，行数和大小计算会不准。

```javascript
const normalizeLineBreaks = (text) => String(text || '').replace(/\r\n?/g, '\n')
const getByteSize = (text) => new Blob([String(text || '')]).size

const countLines = (content) => {
  const normalized = normalizeLineBreaks(content)
  if (!normalized) return 0
  return normalized.split('\n').length
}
```

文件读取这里我固定 UTF-8：

```javascript
const readFileAsUtf8Text = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()

    reader.onload = (event) => {
      resolve(normalizeLineBreaks(event.target?.result || ''))
    }

    reader.onerror = () => reject(new Error('read-failed'))
    reader.readAsText(file, 'utf-8')
  })
}
```

## 分片命名别偷懒

命名看起来是小事，但用户下载后通常会按文件名排序看结果。序号不补零的话，`part_10` 会排到 `part_2` 前面。

```javascript
const getFileBaseName = (fileName) => {
  const name = String(fileName || '').trim() || 'file'
  const dot = name.lastIndexOf('.')

  if (dot <= 0 || dot === name.length - 1) {
    return { baseName: name, extension: 'txt' }
  }

  return {
    baseName: name.slice(0, dot),
    extension: name.slice(dot + 1)
  }
}

const createPartName = (fileName, index) => {
  const { baseName, extension } = getFileBaseName(fileName)
  const seq = String(index).padStart(3, '0')
  return `${baseName}_part_${seq}.${extension}`
}
```

## 按行拆分：最稳的一条路

按行拆分逻辑很直白：分行后按步长切片。

```javascript
const createChunk = (fileName, index, content) => {
  const text = String(content || '')
  return {
    name: createPartName(fileName, index),
    content: text,
    lineCount: countLines(text),
    byteSize: getByteSize(text)
  }
}

const splitByLines = (content, linesPerFile, fileName = 'file.txt') => {
  const normalized = normalizeLineBreaks(content)
  if (!normalized) return []

  const limit = Math.max(1, Number(linesPerFile) || 1)
  const lines = normalized.split('\n')
  const chunks = []

  for (let i = 0; i < lines.length; i += limit) {
    const chunkContent = lines.slice(i, i + limit).join('\n')
    chunks.push(createChunk(fileName, chunks.length + 1, chunkContent))
  }

  return chunks
}
```

## 按大小拆分：关键在“逐行累加”

这个版本我改过一次。第一次写得太粗暴，直接按字符长度估算，中文场景会偏差。后来改成按字节算，并且只在行边界切。

```javascript
const splitBySize = (content, bytesPerFile, fileName = 'file.txt') => {
  const normalized = normalizeLineBreaks(content)
  if (!normalized) return []

  const targetBytes = Math.max(1, Number(bytesPerFile) || 1)
  const lines = normalized.split('\n')
  const chunks = []

  let currentLines = []
  let currentSize = 0

  for (const line of lines) {
    const lineSize = getByteSize(line)

    // 已有内容时，加上一个换行符的字节占位
    const nextSize = currentLines.length > 0
      ? currentSize + 1 + lineSize
      : lineSize

    // 超过阈值：先提交当前块，再开启新块
    if (currentLines.length > 0 && nextSize > targetBytes) {
      chunks.push(createChunk(fileName, chunks.length + 1, currentLines.join('\n')))
      currentLines = [line]
      currentSize = lineSize
      continue
    }

    currentLines.push(line)
    currentSize = nextSize
  }

  if (currentLines.length > 0) {
    chunks.push(createChunk(fileName, chunks.length + 1, currentLines.join('\n')))
  }

  return chunks
}
```

这个写法的好处是：不会把一行硬切成两段，导出的文件可读性更好。

## 合并逻辑：分隔符是核心开关

合并本身就是 `join`，真正要处理的是“分隔符输入”。

```javascript
const normalizedCustomSeparator = computed(() => {
  return String(customSeparator.value || '')
    .replace(/\\r/g, '\r')
    .replace(/\\n/g, '\n')
    .replace(/\\t/g, '\t')
})

const effectiveSeparator = computed(() => {
  if (separatorMode.value === 'doubleNewline') return '\n\n'
  if (separatorMode.value === 'space') return ' '
  if (separatorMode.value === 'none') return ''
  if (separatorMode.value === 'custom') return normalizedCustomSeparator.value
  return '\n'
})

const mergeFiles = (files, separator = '\n') => {
  const list = Array.isArray(files) ? files : []
  const mergedContent = list.map(item => String(item.content || '')).join(String(separator ?? ''))

  return {
    name: 'merged.txt',
    content: mergedContent,
    lineCount: countLines(mergedContent),
    byteSize: getByteSize(mergedContent)
  }
}
```

## 执行入口：只做分发，不做复杂判断

```javascript
const runSplit = () => {
  if (!splitFile.value) return

  splitting.value = true
  try {
    splitResults.value = splitMode.value === 'lines'
      ? splitByLines(splitFile.value.content, splitLinesLimit.value, splitFile.value.name)
      : splitBySize(splitFile.value.content, toBytes(splitSizeLimit.value, splitSizeUnit.value), splitFile.value.name)
  } finally {
    splitting.value = false
  }
}

const runMerge = () => {
  if (!mergeFileList.value.length) return

  merging.value = true
  try {
    mergedResult.value = mergeFiles(mergeFileList.value, effectiveSeparator.value)
  } finally {
    merging.value = false
  }
}
```

这里我尽量让入口函数“短小且可读”，复杂逻辑都放到工具函数里。

## 下载：文本直下 + 多文件打包

单文件下载：

```javascript
const downloadTextFile = (content, fileName, mimeType = 'text/plain;charset=utf-8') => {
  const blob = new Blob([String(content || '')], { type: mimeType })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')

  link.href = url
  link.download = fileName
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
  URL.revokeObjectURL(url)
}
```

批量下载时，用 ZIP 一次打包：

```javascript
const loadScript = (src) => {
  return new Promise((resolve, reject) => {
    const existed = document.querySelector(`script[src="${src}"]`)
    if (existed) return resolve()

    const script = document.createElement('script')
    script.src = src
    script.onload = () => resolve()
    script.onerror = () => reject(new Error(`load-failed:${src}`))
    document.head.appendChild(script)
  })
}

const downloadFilesAsZip = async (files, zipName = 'split-files.zip') => {
  await loadScript('/js/jszip.min.js')
  await loadScript('/js/FileSaver.min.js')

  const zip = new window.JSZip()
  ;(Array.isArray(files) ? files : []).forEach((item, idx) => {
    const name = String(item?.name || `part_${idx + 1}.txt`)
    const content = String(item?.content || '')
    zip.file(name, content)
  })

  const blob = await zip.generateAsync({ type: 'blob' })
  window.saveAs(blob, zipName)
}
```

## 最后一句

这个工具的核心 JS 不算花哨，重点是把几个容易出错的细节处理扎实：换行统一、字节计算、分片命名、下载闭环。只要这四个点稳住，拆分和合并就会非常顺手。
