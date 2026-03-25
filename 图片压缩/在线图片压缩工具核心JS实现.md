# 在线图片压缩工具核心JS实现

这篇只讲本项目“图片压缩”工具的功能层 JavaScript 实现。整体链路是：

`选择图片 -> 读取原图信息 -> 提交压缩参数 -> 获取压缩结果 -> 更新预览与统计 -> 下载`

> 在线工具网址：[https://see-tool.com/image-compressor](https://see-tool.com/image-compressor)  
> 工具截图：  
> ![工具截图](工具截图.png)

## 1. 前端状态模型：围绕“多图批处理”设计

工具初始化后维护一个轻量状态对象，核心是 `images` 数组和当前预览索引。

```javascript
const state = {
  images: [],
  currentIndex: 0,
  recompressTimer: null,
};
```

每张图片在内存里都是一个独立对象，既保存原图信息，也保存压缩结果信息：

- 原图：`originalUrl`、`originalSize`、`originalWidth`、`originalHeight`、`originalType`
- 压缩图：`compressedUrl`、`compressedSize`、`compressedWidth`、`compressedHeight`、`compressedMimeType`
- 控制字段：`compressController`、`compressVersion`

这种结构的好处是，单图更新不会影响其他图片，批量场景下逻辑更稳定。

## 2. 文件接入：先过滤，再补齐元信息

拖拽和文件选择最终都走同一入口 `processFiles`，先过滤非图片文件，再逐个处理。

```javascript
const processFiles = async (files) => {
  const imageFiles = files.filter(
    (file) => file.type && file.type.startsWith("image/"),
  );
  if (imageFiles.length === 0) return;

  for (const file of imageFiles) {
    await addImageFile(file);
  }
};
```

`addImageFile` 会为文件创建对象 URL，并用 `Image` 对象读取像素尺寸，形成完整的图片对象后立即触发一次压缩。

```javascript
const addImageFile = async (file) => {
  const url = URL.createObjectURL(file);
  const img = await loadImage(url);

  const imageObj = {
    id: `${Date.now()}-${Math.random()}`,
    name: file.name,
    file,
    originalUrl: url,
    originalIsObjectUrl: true,
    originalSize: file.size,
    originalType: file.type || "image/jpeg",
    originalWidth: img.width,
    originalHeight: img.height,
    compressedUrl: null,
    compressedSize: null,
    compressController: null,
    compressVersion: 0,
  };

  state.images.push(imageObj);
  state.currentIndex = state.images.length - 1;
  compressImage(imageObj);
};
```

## 3. 参数解析：输出格式和“直出原图”判定

压缩前先计算输出格式配置。工具支持 `original/jpeg/png/webp/avif`，并统一映射成 `mimeType + extension`。

同时有一个关键分支：当用户选择 `original` 且质量 100，并且没有勾选移除元数据时，直接复用原图，不发请求。

```javascript
const shouldUseOriginal = (imageObj) => {
  const format = elements.outputFormat.value;
  const quality = Number(elements.qualitySlider.value);
  if (format !== "original") return false;
  if (quality < 100) return false;
  if (elements.removeMetadata && elements.removeMetadata.checked) return false;
  return true;
};
```

这个分支能保证结果与用户设置一致：既然参数不要求二次编码，就直接返回原始文件。

## 4. 压缩主流程：可取消请求 + 版本号防乱序

`compressImage` 是核心函数，负责把当前图片和参数提交到接口并回写结果。

```javascript
const formData = new FormData();
formData.append("image", imageObj.file, imageObj.name);
formData.append("format", elements.outputFormat.value);
formData.append("quality", String(Number(elements.qualitySlider.value)));
formData.append(
  "removeMetadata",
  String(Boolean(elements.removeMetadata?.checked)),
);
formData.append("progressive", String(Boolean(elements.progressive?.checked)));

const controller = new AbortController();
const compressVersion = Number(imageObj.compressVersion || 0) + 1;
imageObj.compressVersion = compressVersion;
imageObj.compressController = controller;

const response = await fetch("/api/image-compress", {
  method: "POST",
  body: formData,
  signal: controller.signal,
});
```

结果返回后不会立刻写入，而是先校验两件事：

1. 这张图还在列表里（没被删除）
2. 当前响应版本号仍是最新

只有通过校验才更新 `compressedUrl`、`compressedSize`、`compressedWidth`、`compressedHeight`。

这能避免“用户连续拖动质量滑块”时旧请求覆盖新结果。

## 5. 重压缩触发：延迟调度统一入口

质量滑块、输出格式、元数据开关变化后，都不会直接逐张同步请求，而是进入 `scheduleRecompress`。

```javascript
const scheduleRecompress = (delay = 120) => {
  if (state.recompressTimer) {
    clearTimeout(state.recompressTimer);
    state.recompressTimer = null;
  }
  state.recompressTimer = setTimeout(() => {
    state.recompressTimer = null;
    updateCompressedImages();
  }, delay);
};
```

`updateCompressedImages` 内部遍历 `state.images`，逐张调用 `compressImage`，让参数变化后的行为保持一致。

## 6. 服务端压缩：字段校验 + Sharp 编码

接口只接收 `POST multipart/form-data`。先做参数清洗：

- 文件体积上限（30MB）
- 目标格式白名单
- 质量范围（1~100）
- 布尔参数标准化（`true/false/1/0/yes/no`）

随后读取输入元信息，解析目标格式，再按格式进入不同编码分支：

```typescript
switch (targetFormat) {
  case "jpeg":
    transformer = transformer.jpeg({ quality, mozjpeg: true, progressive });
    break;
  case "png":
    transformer = transformer.png({
      quality,
      compressionLevel: 9,
      progressive,
    });
    break;
  case "webp":
    transformer = transformer.webp({ quality, effort: 6 });
    break;
  case "avif":
    transformer = transformer.avif({ quality, effort: 6 });
    break;
}
```

返回体是压缩后的二进制数据，同时附带输出信息头：

- `X-Output-Ext`
- `X-Output-Width`
- `X-Output-Height`
- `X-Output-Size`

前端据此展示压缩图信息，不需要再额外解析文件。

## 7. 结果展示与资源回收

统计区通过聚合 `originalSize` 与 `compressedSize` 计算总原始体积、总压缩体积、节省体积和压缩率。预览区按 `currentIndex` 展示当前图的原图/压缩图与像素信息。

批量下载按顺序触发，清空或删除图片时会执行两步清理：

1. 中断进行中的压缩请求（`AbortController`）
2. 释放对象 URL（`URL.revokeObjectURL`）

这样整条链路能长期稳定运行：上传、重压缩、预览切换、删除、清空都在同一套状态模型里闭环。
