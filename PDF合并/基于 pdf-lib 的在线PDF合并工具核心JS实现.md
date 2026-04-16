# 基于 pdf-lib 的在线PDF合并工具核心JS实现

这篇只讲本项目里“PDF合并”工具的功能层 JS，不展开页面结构，只看文件进入列表、加密识别、页面复制、结果导出这条主链路。

整个流程可以概括成：

`选择文件 -> 过滤 PDF -> 检查是否加密 -> 调整顺序 -> 逐个读取并复制页面 -> 导出合并结果`

> 在线工具网址：[https://see-tool.com/pdf-merge](https://see-tool.com/pdf-merge)  
> 工具截图：  
> ![工具截图](工具截图.png)

## 1）先把运行状态收口

这个工具的核心不是“点一下就合并”，而是先把整个过程涉及的状态集中管理：

```js
const state = {
  items: [],
  merging: false,
  mergedBlob: null,
  mergedUrl: ""
}
```

这里的设计很直接：

- `items` 存待合并文件列表，同时保存文件名、大小、是否加密、密码等信息
- `merging` 用来锁住重复点击，避免并发合并
- `mergedBlob` 和 `mergedUrl` 用来承接最终导出的 PDF 结果

这种状态拆分的价值在于，文件列表、合并过程、下载结果彼此独立，逻辑不会互相污染。

## 2）文件进入列表前，先做类型过滤和加密识别

工具不会把用户选择的所有文件直接塞进队列，而是先过滤真正可用的 PDF：

```js
function isPdfFile(file) {
  const byType = file.type === "application/pdf"
  const byName = /\.pdf$/i.test(file.name)
  return byType || byName
}
```

之所以同时检查 MIME 和文件名后缀，是因为有些浏览器或系统环境下，`file.type` 并不稳定，只靠一种判断容易漏掉合法文件。

通过过滤后，工具会继续做一件关键的事：提前检测文件是否加密。

```js
async function detectEncrypted(file) {
  try {
    const bytes = await file.arrayBuffer()
    await PDFDocument.load(bytes)
    return false
  } catch (error) {
    const msg = String(error && error.message ? error.message : error).toLowerCase()
    return msg.includes("encrypted") || msg.includes("password")
  }
}
```

这里没有自己解析 PDF 结构，而是直接借助 `pdf-lib` 的加载结果做判断。能正常加载就说明未加密；如果报错信息里出现 `encrypted` 或 `password`，就把它标记为加密文件。这样后面真正合并时，就能提前走密码分支，而不是等到主流程里再统一报错。

## 3）加入列表时，把文件元信息一次性补齐

添加文件时，工具会边遍历边补齐每个条目的完整数据：

```js
for (const file of list) {
  const encrypted = await detectEncrypted(file)
  state.items.push({
    id: createId(),
    file,
    name: file.name,
    size: file.size,
    encrypted,
    password: null
  })
}
```

这里的重点不是 `push`，而是每个列表项本身就是一个“可合并任务对象”。后面的排序、删除、密码缓存、逐个读取，都是围绕这个对象展开的。

尤其是 `password` 字段，它允许工具在用户第一次输入密码后缓存下来，避免同一个文件在流程内反复询问。

## 4）加密 PDF 不直接失败，而是进入可重试加载

真正读取源 PDF 时，逻辑被单独收口成 `loadSourcePdf`。未加密文件直接加载；加密文件则通过弹窗要求输入密码，并允许输错后重试：

```js
async function loadSourcePdf(item, bytes) {
  if (!item.encrypted) {
    return PDFDocument.load(bytes)
  }

  if (!item.password) {
    const pwd = window.prompt('文件“' + item.name + '”已加密，请输入密码：')
    if (pwd === null) {
      return null
    }
    item.password = pwd
  }

  while (true) {
    try {
      return await PDFDocument.load(bytes, { password: item.password })
    } catch (error) {
      const msg = String(error && error.message ? error.message : error).toLowerCase()
      if (msg.includes("password") || msg.includes("encrypted")) {
        const next = window.prompt('文件“' + item.name + '”密码错误，请重新输入：')
        if (next === null) {
          return null
        }
        item.password = next
        continue
      }
      throw error
    }
  }
}
```

这个函数有两个很实用的特点：

- 用户取消输入时返回 `null`，主流程可以把这个文件当作“跳过”处理
- 真正的异常继续向外抛，避免把所有错误都误判成密码问题

也就是说，这里不是简单地“有密码就加载”，而是把密码交互、重试和异常分类放进了一个独立入口。

## 5）合并主流程的核心，是“复制页面”而不是“拼接字节”

PDF 合并不能直接把多个文件的二进制流硬拼起来，真正可靠的做法是：先加载源文档，再把页面复制到新的目标文档里。

```js
const merged = await PDFDocument.create()

for (let i = 0; i < state.items.length; i += 1) {
  const item = state.items[i]
  const bytes = await item.file.arrayBuffer()
  const src = await loadSourcePdf(item, bytes)
  if (!src) continue

  const pages = await merged.copyPages(src, src.getPageIndices())
  pages.forEach(function (page) {
    merged.addPage(page)
  })
}
```

这里的关键点有两个：

- `PDFDocument.create()` 先创建一个全新的目标文档
- `copyPages` 根据源文档页码列表复制页面对象，再逐页 `addPage`

这就是整个工具最核心的功能逻辑。它不是在文件层做合并，而是在 PDF 页面模型层做重组，所以可以稳定处理多个来源文件。

## 6）主流程里同时处理进度、跳过文件和最终结果

完整的 `mergePdf` 并不只是页面复制，它还把进度更新和失败兜底一起处理了：

```js
const skipped = []

for (let i = 0; i < state.items.length; i += 1) {
  const item = state.items[i]
  const bytes = await item.file.arrayBuffer()

  let src
  try {
    src = await loadSourcePdf(item, bytes)
  } catch (error) {
    skipped.push(item.name)
    setProgress(Math.round(((i + 1) / state.items.length) * 100))
    continue
  }

  if (!src) {
    skipped.push(item.name)
    setProgress(Math.round(((i + 1) / state.items.length) * 100))
    continue
  }

  const pages = await merged.copyPages(src, src.getPageIndices())
  pages.forEach(function (page) {
    merged.addPage(page)
  })

  setProgress(Math.round(((i + 1) / state.items.length) * 100))
}
```

这里的思路很清楚：单个文件出问题，不拖垮整个批次，而是记录到 `skipped` 后继续处理下一个。这样用户最终拿到的不是“全量失败”，而是“成功多少、跳过多少”的结果。

如果全部文件都不可用，还会额外拦一次：

```js
if (merged.getPageCount() === 0) {
  throw new Error("没有可合并的页面")
}
```

这一步避免了生成一个空 PDF，让错误语义更准确。

## 7）导出结果时，用 Blob 和对象 URL 收口

合并完成后，新的 PDF 会先被保存为字节数组，再转成浏览器可下载对象：

```js
const bytes = await merged.save()
state.mergedBlob = new Blob([bytes], { type: "application/pdf" })
state.mergedUrl = URL.createObjectURL(state.mergedBlob)
```

点击下载时，并不依赖后端接口，而是临时创建一个 `a` 标签直接触发保存：

```js
const a = document.createElement("a")
a.href = state.mergedUrl
a.download = "merged.pdf"
document.body.appendChild(a)
a.click()
document.body.removeChild(a)
```

这意味着整个 PDF 合并链路都在浏览器端完成：文件读取在本地，文档重组在本地，结果导出也在本地。

## 8）列表排序、删除和清空，本质上都在维护同一个任务队列

这个工具支持上移、下移、删除，本质上是在操作 `state.items` 这份队列。顺序之所以重要，是因为最终合并页序完全取决于这个数组的先后顺序。

删除或调整顺序后，工具会先清空之前的合并结果，再重新渲染列表。这样可以避免一种常见问题：用户已经改了文件顺序，但界面上还保留着旧的下载结果。

清空时还会多做一步资源释放：

```js
if (state.mergedUrl) {
  URL.revokeObjectURL(state.mergedUrl)
}
```

这一步的意义是销毁旧的对象 URL，避免结果反复生成后留下无效引用。

## 9）这个工具的核心，其实是异步任务编排

“PDF合并”表面上只是几个文件拼成一个文件，真正的功能层重点其实有三件事：

- 把每个文件抽象成可管理的任务对象
- 把加密识别、密码输入、页面复制和失败跳过串成一条异步链
- 把结果导出和旧结果清理收口到统一状态里

所以这个工具最核心的 JS 价值，不只是调用了 `pdf-lib`，而是把“文件队列 + 密码交互 + 页面复制 + 本地下载”这几步组织成了一条稳定可控的流程。
