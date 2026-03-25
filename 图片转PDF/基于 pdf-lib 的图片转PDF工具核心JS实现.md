# 基于 pdf-lib 的图片转PDF工具核心JS实现

这篇只讲功能层 JavaScript 实现。这个工具是我用 Vue 组织页面交互，再配合 `pdf-lib` 在浏览器端完成 PDF 生成。`pdf-lib` 是公开可用的通用开源库（MIT 许可），可直接用于浏览器和 Node.js。核心流程是：

`上传图片 -> 读取并建模 -> 调整顺序/旋转 -> 计算页面与布局 -> 写入 PDF -> 下载`

> 在线工具网址：[https://see-tool.com/image-to-pdf](https://see-tool.com/image-to-pdf)  
> 工具截图：  
> ![工具截图](工具截图.png)

## 0）先说下 pdf-lib 在这个工具里的角色

`pdf-lib` 在这里主要负责三件事：创建 PDF 文档、把图片字节嵌入文档、按给定坐标绘制到页面。

它的调用模型很直接：

`create/load -> addPage/embed -> drawImage -> save`

对图片转 PDF 这类场景，它有几个很实用的点：

- API 稳定，浏览器端可直接运行，不依赖后端转换
- 页面坐标和尺寸控制细，适合做“边距、缩放、居中、铺满”这类排版
- 同一套逻辑可扩展到“合并现有 PDF、写文本、填表单”等后续功能

这个工具里最核心的一步是 `embedJpg + drawImage`：前者把二进制图片数据放进 PDF，后者决定图片在页面中的位置和显示尺寸。只要前面把页面尺寸和布局参数算对，最终导出的版式就会稳定可控。

## 1）状态模型：围绕“图片队列 + 转换配置”组织

工具初始化后维护一个状态对象，核心字段如下：

- `images`：当前待转换图片队列
- `settings`：页面尺寸、方向、布局、边距、文件名
- `isConverting`：是否正在生成，防止重复点击

每张图片对象都会保存：`name`、`size`、`format`、`preview(dataURL)`、`width`、`height`、`rotation`。这样后续排序、旋转、预览和生成 PDF 都可以直接复用同一份数据。

## 2）文件接入：先过滤，再读取 DataURL 和尺寸

上传与拖拽最后都进入同一入口，先只保留图片类型，再做数量上限控制。每个文件会经过两步：

1. `FileReader.readAsDataURL` 读成 DataURL
2. 用 `Image` 对象加载，拿到 `naturalWidth/naturalHeight`

核心读取函数：

```js
function readFileAsDataUrl(file) {
  return new Promise(function (resolve, reject) {
    var reader = new FileReader();
    reader.onload = function (event) {
      resolve(
        String(event.target && event.target.result ? event.target.result : ""),
      );
    };
    reader.onerror = function () {
      reject(new Error("file_read_failed"));
    };
    reader.readAsDataURL(file);
  });
}
```

读取成功后把图片对象追加进队列，页面统计和预览区会立即重渲染。

## 3）页面尺寸计算：统一转换到 PDF 点单位

PDF 里统一使用 point（pt），因此先做单位换算：

- `MM_TO_PT = 72 / 25.4`
- `PX_TO_PT = 72 / 96`

固定纸张（A4/A3/Letter/Legal）先由毫米转 pt；如果用户选“原始尺寸”，则按图片像素转 pt。方向为 `auto` 时，会根据图片宽高自动判定横版或竖版。

```js
function getPageSizePt(pageSize, orientation, imageWidth, imageHeight) {
  var widthPt;
  var heightPt;

  if (pageSize === "original") {
    widthPt = imageWidth * PX_TO_PT;
    heightPt = imageHeight * PX_TO_PT;
  } else {
    var pageSizeValue = PAGE_SIZES_MM[pageSize] || PAGE_SIZES_MM.a4;
    widthPt = pageSizeValue.width * MM_TO_PT;
    heightPt = pageSizeValue.height * MM_TO_PT;
  }

  // 省略方向交换逻辑
  return {
    widthPt: widthPt,
    heightPt: heightPt,
    orientation: finalOrientation,
  };
}
```

## 4）布局算法：contain / cover / center

每页先算内容区：

`contentWidth = pageWidth - margin * 2`

`contentHeight = pageHeight - margin * 2`

然后按布局模式计算绘制矩形：

- `contain`：完整显示图片，按最小缩放比适配
- `cover`：铺满内容区，按最大缩放比适配
- `center`：不缩放，居中绘制

当边距过大导致内容区宽高小于等于 0 时，直接抛错，转换流程会给出明确提示。

## 5）旋转处理：先画到 Canvas，再统一转 JPEG

为了让 `pdf-lib` 写入稳定，工具会先把当前图片（含旋转角度）绘制到 `canvas`，再导出 JPEG 字节。角度支持 0/90/180/270。

```js
function drawImageToJpeg(image, rotation) {
  var angle = Number(rotation || 0);
  var swapSide = angle === 90 || angle === 270;
  var canvas = document.createElement("canvas");

  canvas.width = swapSide ? image.naturalHeight : image.naturalWidth;
  canvas.height = swapSide ? image.naturalWidth : image.naturalHeight;

  var context = canvas.getContext("2d", { alpha: false });
  context.fillStyle = "#ffffff";
  context.fillRect(0, 0, canvas.width, canvas.height);
  context.translate(canvas.width / 2, canvas.height / 2);
  context.rotate((angle * Math.PI) / 180);
  context.drawImage(image, -image.naturalWidth / 2, -image.naturalHeight / 2);

  return parseDataUrl(canvas.toDataURL("image/jpeg", 0.92));
}
```

这里先铺白底，可以避免透明图在 PDF 中出现黑底。

## 6）PDF 生成：逐页 addPage + embedJpg + drawImage

转换时按图片队列顺序循环处理，每张图先算页面尺寸和布局，再写入 PDF：

```js
async function buildPdfBlobWithPdfLib(pages, PDFDocument) {
  var pdfDocument = await PDFDocument.create();

  for (var i = 0; i < pages.length; i += 1) {
    var item = pages[i];
    var page = pdfDocument.addPage([item.pageWidthPt, item.pageHeightPt]);
    var jpg = await pdfDocument.embedJpg(item.imageBytes);
    page.drawImage(jpg, {
      x: item.xPt,
      y: item.yPt,
      width: item.drawWidthPt,
      height: item.drawHeightPt,
    });
  }

  var pdfBytes = await pdfDocument.save();
  return new Blob([pdfBytes], { type: "application/pdf" });
}
```

最终通过对象 URL 触发浏览器下载，文件名会先做非法字符清洗。

## 7）交互动作如何影响最终 PDF

工具里“上移/下移/拖拽排序/旋转/删除”都直接改 `images` 队列，因此最终 PDF 页序与图片列表始终一致。

- 排序：交换数组元素或拖拽后 `splice`
- 旋转：`rotation = (rotation + 90) % 360`
- 删除：按 `id` 过滤

由于生成时是按队列顺序遍历，交互结果会无缝反映到导出文件。

## 8）转换过程控制：状态锁 + 分阶段提示

点击“转换”后会先进入 `isConverting = true`：

- 禁用上传、配置项和操作按钮
- 显示“正在生成第 X / N 页”
- 完成后恢复可操作状态

错误按场景给出明确反馈，比如无图片、文件名为空、边距过大、生成失败。整条链路是纯前端闭环：从读取、排版到导出，都在浏览器端完成。
