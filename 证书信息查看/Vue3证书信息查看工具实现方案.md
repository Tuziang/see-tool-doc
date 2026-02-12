
本文记录「证书信息查看」这个工具在本项目中的实现方案，主要围绕 Vue 端页面结构和功能 JS 逻辑展开，方便后续维护和扩展同类工具。

> 在线工具网址：[https://see-tool.com/certificate-info-viewer](https://see-tool.com/certificate-info-viewer)  
> 工具截图：  
> ![工具截图](工具截图.png)

### 页面结构与状态设计

证书信息查看页面基于 Vue 3 `<script setup>` 编写，核心分为三块：

- **输入区**：通过 `inputType` 区分「粘贴证书文本」「上传证书文件」「输入域名查询」三种模式，对应不同的输入控件。  
- **结果展示区**：依赖 `resultData` 渲染证书主体信息、签发者信息和证书详情三大表格，字段配置用 `subjectFields`、`issuerFields`、`certFields` 这些 `computed` 从配置/文案中统一读取。  
- **操作区**：提供「查询」「清空」按钮，以及在有结果时的「下载证书 PEM」「下载公钥 PEM」按钮。  

页面内部通过一组 `ref` 保存当前状态：`inputType`、`certContent`、`domainInput`、`selectedFile`、`selectedFileName`、`isDragOver`、`isLoading`、`errorMessage`、`resultData` 等，这些状态与模板上的 `v-model`、条件渲染和按钮禁用态形成一套比较清晰的单向数据流。

```ts
const inputType = ref<'paste' | 'file' | 'domain'>('paste')
const certContent = ref('')
const domainInput = ref('')
const selectedFile = ref<File | null>(null)
const selectedFileName = ref('')
const isDragOver = ref(false)
const isLoading = ref(false)
const errorMessage = ref('')
const resultData = ref<any | null>(null)

const subjectFields = computed(() => tm('certificateInfoViewer.result.subjectFields') || [])
const issuerFields = computed(() => tm('certificateInfoViewer.result.issuerFields') || [])
const certFields = computed(() => tm('certificateInfoViewer.result.certFields') || [])
```

### 输入校验与错误提示

在触发查询前，所有输入都会先经过 `validateInput()`：  

- 当模式是「粘贴文本」时，要求 `certContent` 非空；  
- 当模式是「上传文件」时，要求已选择 `selectedFile`；  
- 当模式是「域名查询」时，要求 `domainInput` 非空。  

错误信息使用统一的 `setError(code, fallback)` 函数设置，内部会通过 `resolveErrorMessage` 先按错误码查映射表，找不到时再退回到兜底文案，同时调用 `notify` 在页面右上角弹出提示，这样前端各处只需要抛出语义化的错误码即可。

```ts
const resolveErrorMessage = (code: string, fallback?: string) => {
  const mapped = errorMap.value?.[code]
  if (mapped) return mapped
  if (fallback) return fallback
  return t('certificateInfoViewer.messages.queryFailed')
}

const setError = (code: string, fallback?: string) => {
  errorMessage.value = resolveErrorMessage(code, fallback)
  notify(errorMessage.value, 'error')
}

const validateInput = () => {
  if (inputType.value === 'paste' && !certContent.value.trim()) {
    setError('CONTENT_REQUIRED')
    return false
  }
  if (inputType.value === 'file' && !selectedFile.value) {
    setError('FILE_EMPTY')
    return false
  }
  if (inputType.value === 'domain' && !domainInput.value.trim()) {
    setError('DOMAIN_REQUIRED')
    return false
  }
  return true
}
```

### 文件上传与拖拽处理

针对证书文件上传，页面实现了一套独立的小模块：

- 使用 `setSelectedFile(file)` 做统一入口，校验文件大小（`maxFileSize`）和扩展名（`allowedExtensions`，只允许 `.pem/.crt/.cer`），并更新 `selectedFile` 与 `selectedFileName`。  
- `handleFileChange` 负责响应 `<input type="file">` 的变更事件，`handleDragOver` / `handleDragLeave` / `handleDrop` 则实现拖拽高亮和文件接收逻辑。  
- 当输入方式切换为非文件模式时，通过监听 `inputType` 的 `watch` 自动清空已选择的文件，避免状态残留。  

这部分逻辑和 UI 解耦较好，未来如果要复用到其他上传类工具，只需要复用这几个函数即可。

```ts
const maxFileSize = 10 * 1024 * 1024
const allowedExtensions = ['.pem', '.crt', '.cer']

const getFileExtension = (filename: string) => {
  const lastDot = filename.lastIndexOf('.')
  if (lastDot === -1) return ''
  return filename.slice(lastDot).toLowerCase()
}

const setSelectedFile = (file: File | null) => {
  if (!file) {
    selectedFile.value = null
    selectedFileName.value = ''
    return
  }
  if (file.size > maxFileSize) {
    setError('FILE_TOO_LARGE')
    return
  }
  const ext = getFileExtension(file.name)
  if (!allowedExtensions.includes(ext)) {
    setError('FILE_TYPE_NOT_ALLOWED')
    return
  }
  selectedFile.value = file
  selectedFileName.value = file.name
}

const handleDrop = (event: DragEvent) => {
  isDragOver.value = false
  const file = event.dataTransfer?.files?.[0] || null
  setSelectedFile(file)
}
```

### 证书查询与上传请求

两种请求方式的区别在于请求体构造和上传方式：  

- **文本 / 域名模式**：`queryCertificateInfo()` 先构造包含 `mode`、`content`、`domain` 的 `payload`，然后以 JSON 形式 `fetch('/api/certificate-info')`。  
- **文件上传模式**：`uploadCertificate()` 构造包含 `mode`、`filename` 的 `payload`，再创建 `FormData`，将文件二进制和相关字段一起塞入，以 multipart 方式 `fetch('/api/certificate-info-upload')`。  

对应的后端 API 在接收到请求后，大致会做三件事：首先根据 `mode` 决定是从文本、上传文件还是目标域名中取出证书原始数据；然后对证书做解析与字段提取，生成主体信息、签发者信息、有效期、用途、SAN 等结构化结果，并计算 SHA 指纹等摘要；最后把格式化好的 `subjectInfo`、`issuerInfo`、`certInfo` 以及可下载的证书、公钥 PEM 文本一并封装成 `data` 返回给前端。  
其中「域名查询」模式下，服务端会先把用户输入的 URL 解析成 `host` 和 `port`，在 Node.js 环境中发起一次 TLS 连接，仅完成握手就从连接中拿到对端证书的原始二进制数据，随后走与文本/文件模式相同的解析与字段提取流程。

两种请求都会对返回的 `result.status` 做统一判断，不为 `ok` 则通过 `setError(result.message, result.message)` 报错，为 `ok` 时把 `result.data` 写入 `resultData`，驱动页面结果区渲染。

```ts
const queryCertificateInfo = async () => {
  const payload = {
    mode: pendingType.value,
    content: certContent.value.trim(),
    domain: domainInput.value.trim()
  }

  try {
    const response = await fetch('/api/certificate-info', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    })
    const result = await response.json()
    if (result.status !== 'ok') {
      setError(result.message, result.message)
      return
    }
    resultData.value = result.data
  } catch (error: any) {
    setError('REQUEST_FAILED', error?.message)
  }
}

const uploadCertificate = async () => {
  if (!selectedFile.value) {
    setError('FILE_EMPTY')
    return
  }

  const payload = {
    mode: 'upload',
    filename: selectedFile.value.name
  }

  try {
    const formData = new FormData()
    formData.append('cert', selectedFile.value)
    Object.entries(payload).forEach(([key, value]) => {
      formData.append(key, String(value))
    })
    const response = await fetch('/api/certificate-info-upload', {
      method: 'POST',
      body: formData
    })
    const result = await response.json()
    if (result.status !== 'ok') {
      setError(result.message, result.message)
      return
    }
    resultData.value = result.data
  } catch (error: any) {
    setError('REQUEST_FAILED', error?.message)
  }
}
```

### 结果数据格式化与下载能力

为了让证书信息更易读，页面做了多层格式化封装：  

- `formatValue` 负责通用的字符串和数组转展示文本，空值统一显示为 `-`；  
- `formatKeyUsage`、`formatExtKeyUsage` 会把枚举值数组映射成带中文说明的字符串；  
- `formatBasicConstraints` 把 CA 标志和路径长度组合成人类可读的描述。  

下载能力则由 `downloadCertificate()` 和 `downloadPublicKey()` 两个函数提供：  

- 首先检查 `resultData.downloads` 中是否存在对应的 PEM 内容，不存在则弹出下载不可用提示；  
- 然后使用 `sanitizeFileName()` 基于证书主题中的 `commonName` 生成安全的文件名；  
- 最后通过 `downloadText(content, filename)` 封装 Blob、创建临时链接并触发浏览器下载。  

整体来看，这个 Vue3 实现把「输入 → 校验 → 请求 → 渲染结果 → 下载」整条链路拆分成若干职责清晰的小函数，既方便前端页面维护，也为后续扩展更多证书相关工具打好了基础。  

```ts
const formatValue = (value: unknown) => {
  if (Array.isArray(value)) {
    return value.length ? value.join('\n') : '-'
  }
  if (value === null || value === undefined || value === '') {
    return '-'
  }
  return String(value)
}

const formatCertValue = (key: string) => {
  const value = resultData.value?.certInfo?.[key]
  if (key === 'keyUsage') return formatKeyUsage(value)
  if (key === 'extendedKeyUsage') return formatExtKeyUsage(value)
  if (key === 'basicConstraints') return formatBasicConstraints(value)
  if (key === 'publicKeySize' && value) {
    return t('certificateInfoViewer.result.bits', { value })
  }
  return formatValue(value)
}

const downloadText = (content: string, filename: string) => {
  const blob = new Blob([content], { type: 'application/x-pem-file' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = filename
  link.click()
  URL.revokeObjectURL(url)
}

const downloadCertificate = () => {
  if (!resultData.value?.downloads?.certPem) {
    notify(t('certificateInfoViewer.messages.downloadUnavailable'), 'warning')
    return
  }
  const cn = sanitizeFileName(resultData.value?.subjectInfo?.commonName || '')
  const filename = cn ? `${cn}.pem` : 'certificate.pem'
  downloadText(resultData.value.downloads.certPem, filename)
}
```


