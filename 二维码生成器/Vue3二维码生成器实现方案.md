# Vue3 二维码生成器实现方案（本项目实战拆解）

本文基于本项目的「二维码生成器」工具，拆解一套在 Vue3 / Nuxt3 项目中实现可视化二维码生成器的完整方案，重点放在页面结构与功能 JavaScript 的协作方式上，代码均来源于实际线上工具。

> 在线工具网址：[https://see-tool.com/qr-code-generator](https://see-tool.com/qr-code-generator)
> 工具截图：
> ![工具截图](工具截图.png)

## 一、整体架构设计

本工具采用「Vue 负责结构和状态容器、独立 JS 负责绘制与交互」的分层思路：

- Vue 页面组件：`pages/qr-code-generator.vue`，负责：
  - 渲染所有表单控件（文本/URL/邮箱/电话/SMS/WiFi/vCard 等类型）
  - 输出带有 `data-*` 标记的 DOM 结构，作为 JS 工具层的「挂载点」
  - 提供预览区域、下载按钮区域，以及文档说明和相关工具组件
- 功能脚本一：`public/js/qr-code-generator-ui.js`，只处理：
  - 类型卡片的选中态切换（`.qr-type-card`）
  - 不同类型表单块的显隐（`.qr-input-form`）
  - 监听 `render-event` / 路由变化重新绑定
- 功能脚本二：`public/js/qr-code-generator-tool.js`，负责核心能力：
  - 动态加载第三方 `qrcode-generator` 库
  - 解析不同类型表单内容并组装成最终字符串
  - 根据参数（尺寸、容错等级、前景色/背景色、点样式、定位角样式、中心 Logo）绘制二维码
  - 生成 Canvas，并支持导出 PNG / SVG
  - 监听输入变化，已生成过一次后自动「实时预览重绘」

这种拆分的好处是：Vue 只关心「结构和语义」，复杂的 Canvas 绘制逻辑完全放在独立 JS 中，通过 `data-qr-generator` 这一属性完成耦合。

## 二、Vue 页面：用 data-* 标记出「能力插槽」

在 `pages/qr-code-generator.vue` 中，最重要的不是具体 HTML，而是各种 `data-*` 标记，它们定义了功能 JS 可以操作的「协议」：

- `data-qr-generator`：整个工具根节点，JS 通过它找到一整块 DOM
- 类型卡片：`.qr-type-card` + `data-type="text|url|email|phone|sms|wifi|vcard"`
- 各类型表单：`.qr-input-form` + `data-form="text|url|..."`
- 表单字段：`data-field="text" | "url" | "email-address" | "wifi-ssid" | ...`
- 自定义设置：
  - 尺寸：`data-field="qr-size"`（range）+ `data-field="qr-size-value"`（显示文字）
  - 容错等级：`data-field="qr-error-level"`
  - 颜色：`data-field="qr-color-dark"` / `qr-color-light` 以及对应的文本输入
  - 点样式：`.pattern-selector` + `data-style="square|rounded|dots"`
  - 定位角样式：`.eye-frame-selector` + `data-eye-style="square|diamond|leaf|dot|circle|extra-rounded|classy"`
  - 中心 Logo 上传相关：`data-field="logo-upload"`、`data-field="logo-filename"`、`data-action="select-logo"`、`data-action="clear-logo"`
- 预览与下载区域：
  - 预览容器：`#qr-preview`
  - 占位文案：`.qr-preview` 内初始提示
  - 下载提示占位：`.qr-download-placeholder`
  - 下载按钮包裹：`.qr-download-buttons`
  - 具体按钮：`data-action="download-png"` / `download-svg"`

Vue 组件本身并没有写任何二维码业务逻辑，通过这些标记，把「能调节哪些参数、有哪些输入类型」暴露给 JS 工具层即可。

## 三、UI 交互层：类型切换与表单显隐

`public/js/qr-code-generator-ui.js` 是一个自执行函数，仅做 UI 层交互：

- `setActiveType(root, type)`：
  - 遍历所有 `.qr-type-card`，给当前类型加上 `active`，其它移除
  - 遍历 `.qr-input-form`，`data-form === type` 的移除 `hidden`，其余加上 `hidden`
- `bind(root)`：
  - 找到默认 `active` 的卡片，或回退到 `text` 类型
  - 给每张卡片绑定 `click` 事件，切换类型
- `init()`：
  - 通过 `document.querySelectorAll('[data-qr-generator]')` 找到所有实例并绑定
  - 监听 `DOMContentLoaded`、`render-event`、`hashchange`、`popstate`，确保在 SSR 渲染完成、前端路由切换后都能重新扫描并绑定

这一层刻意不碰任何二维码算法或 Canvas，只负责「用户点哪里，看到哪个表单」。

## 四、核心工具层：从表单到二维码的完整流水线

`public/js/qr-code-generator-tool.js` 是真正的核心，实现步骤可以拆成几块。

### 1. 环境准备与工具函数

- `loadScriptOnce(src)`：按 src 检查是否已经插入 `<script>`，避免重复加载；如果脚本标签存在但库尚未就绪，则监听 `load` / `error`。
- `hexToRgba` / `isValidHexColor`：处理颜色合法性与转换。
- `normalizeUrl(url)`：为 URL 类型做 https 默认补全。
- `getActiveType / getActiveDotStyle / getActiveEyeStyle`：根据当前 DOM 选中态读出类型和样式。

### 2. 按类型组装内容字符串

`getContentByType(root, type)` 是「业务协议」的核心，它根据类型从表单里取值，并组装不同的目标格式：

- `text`：直接取纯文本；
- `url`：调用 `normalizeUrl`，自动补 `https://`；
- `email`：拼接为 `mailto:` 协议，支持 subject / body 参数；
- `phone`：`tel:` 协议；
- `sms`：`sms:号码?body=内容`；
- `wifi`：按照标准 WiFi QR 协议拼出 `WIFI:T:WPA;S:SSID;P:PASSWORD;H:true;;`；
- `vcard`：生成一个 `BEGIN:VCARD` / `END:VCARD` 的多行 vCard 文本，包含姓名、公司、电话、邮箱、网址等。

如果必填字段为空（如邮箱地址、WiFi SSID、vCard 姓名），函数会返回空字符串，后续生成逻辑会弹出「请填写必填字段」的通知而不是生成无效二维码。

### 3. Canvas 绘制：点阵 + 定位角 + Logo

生成二维码视觉效果的关键在 `buildQrCanvas`：

1. 通过第三方库 `window.qrcode(0, errorLevel)` 生成二维码矩阵：
   - `addData(text)` / `make()` 得到 `qr.getModuleCount()` 和 `qr.isDark(row, col)`。
2. 创建 Canvas 并先用背景色铺底。
3. 遍历矩阵：
   - 通过 `isInEyeFrame(row, col, moduleCount)` 判断当前 cell 是否属于三个位于角落的定位图形；
   - 非定位区域调用 `drawDots(ctx, x, y, cellSize, dotStyle)`：
     - `square`：纯方块；
     - `rounded`：通过 `roundRect` 或手写 path 实现圆角矩形；
     - `dots`：圆点样式。
4. 三个定位角通过 `drawEyeFrame` 单独绘制，支持多种样式：
   - 通过 path / arc / 手写 roundRect 实现 diamond / leaf / dot / circle / extra-rounded / classy 等；
   - 外框、内框、中心块用深浅色组合绘制。
5. 如果用户上传了中心 Logo：
   - 把图片读成 DataURL，`img.onload` 后在中心开一块浅底矩形，再绘制缩放后的 logo 图片。

最终返回一个 Promise，resolve 出完整绘制好的 Canvas。

### 4. 从 Canvas 导出 SVG

`canvasToSvg(canvas, colorLight, colorDark)` 负责把渲染结果转成矢量 SVG：

- 读取整张 Canvas 的 `imageData`；
- 先输出一个背景 `rect`，用浅色填充；
- 再按像素遍历，凡是足够「黑」的像素，就在 path 中追加 `M x,y h1 v1 h-1 z` 这种 1×1 小方块；
- 最终得到一个单 path 的 SVG，方便下载和后续放大使用。

### 5. 绑定页面：生成、实时重绘与下载

`bindQrTool(root)` 负责把整条流水线串起来：

- 先通过各种 `data-field`、`.pattern-selector`、`.eye-frame-selector` 把 DOM 节点引用缓存下来；
- 同步尺寸滑块与文字显示、同步颜色选择器与文本输入；
- 处理 Logo 上传、清除，以及文件名展示。

核心的生成逻辑在 `generate()`：

1. `ensureLib()`：按需加载 `qrcode-generator.min.js`，只加载一次；
2. 读取当前类型与内容，若为空则调用 `window.showNotification` 提示并终止；
3. 读取尺寸、容错等级、颜色、点样式、定位角样式、Logo 等参数；
4. 清空预览 DOM，调用 `buildQrCanvas` 得到 Canvas；
5. 把 Canvas append 到预览容器中，并展示下载按钮；
6. 记录 `lastCanvas` 与 `hasGeneratedOnce` 标记，后续即可支持「改参数自动重绘」。

为了让体验更「所见即所得」，还提供了防抖重绘：

- `debounce(generate, 250)` 得到 `debouncedRegenerate`；
- 在尺寸滑块、容错等级选择、颜色输入、点样式/定位角样式、Logo 上传与清除、以及所有 `.qr-input-form` 中的 `input/textarea/select/checkbox` 变动时触发；
- 如果尚未生成过一次，则不会重绘，避免无意义计算。

最后是下载逻辑：

- PNG 下载：`lastCanvas.toDataURL('image/png')` 赋给临时 `<a>` 的 `href`，设置 `download='qrcode.png'` 后触发点击；
- SVG 下载：先用 `canvasToSvg` 拿到字符串，再用 Blob + `URL.createObjectURL` 生成临时链接，设置 `download='qrcode.svg'` 后点击并释放 URL。

## 五、在 Vue 项目中复用这套方案的要点

总结这套实现的关键点，其实就是一句话：**Vue 负责结构和「可操作的标记」，具体的绘制逻辑用独立 JS 模块承载，通过 data- 属性「协议」打通两层**。在实际项目里，如果你也需要类似的「复杂 Canvas 工具」，可以沿用这套模式，把 Vue 页面当作「数据+状态容器」，再用一个专门的工具脚本去操作 DOM 和 Canvas，这样既不和框架生命周期打架，又方便在多个项目中迁移与复用逻辑。

