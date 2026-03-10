# 基于 ZXing 的 Vue 在线二维码扫描器实现

这篇只讲功能层 JavaScript：同一个扫描器同时支持“图片上传识别”和“摄像头实时识别”，识别到的内容进入结果列表，并提供复制能力。

> 在线工具网址：[https://see-tool.com/qrcode-scanner](https://see-tool.com/qrcode-scanner)  
> 工具截图：  
> ![工具截图](工具截图.png)

识别依赖 ZXing（`@zxing/library`）的 `BrowserMultiFormatReader`，主要用到两种解码方式：

- 图片：`decodeFromImageElement(img)`
- 摄像头：`decodeFromVideoDevice(deviceId, videoEl, callback)`

下面按功能模块拆开讲核心实现。

## 1）解码器初始化：SSR 下只在客户端创建

Nuxt 有 SSR，`setup` 会先在服务器执行一次生成 HTML。服务器环境没有 `window` / `navigator`，也没有 `navigator.mediaDevices` 这类摄像头 API；而 `BrowserMultiFormatReader` 属于浏览器侧解码器，如果在服务端阶段创建，就可能触发 `window is not defined` / `navigator is undefined` 这类错误。

处理方式：把初始化放进 `onMounted`（只在浏览器端执行），并用 `process.client` 再兜底一次。

```js
import { onMounted, onUnmounted } from "vue";
import { BrowserMultiFormatReader } from "@zxing/library";

let codeReader = null;

onMounted(() => {
  if (process.client) {
    codeReader = new BrowserMultiFormatReader();
  }
});

onUnmounted(() => {
  // 离开页面时释放摄像头相关资源
  if (codeReader) codeReader.reset();
});
```

`reset()` 用于停止当前扫描流程，并释放视频流相关资源（切换模式或离开页面时会用到）。

## 2）上传识别：File -> DataURL -> Image -> decode

上传和拖拽统一走 `handleFiles(files)`：遍历文件，先过滤非图片，再逐个触发识别。

```js
const handleFiles = (files) => {
  if (!files || files.length === 0) return;

  Array.from(files).forEach((file) => {
    if (!file.type.startsWith("image/")) {
      addResult(file.name, "仅支持图片文件", "error");
      return;
    }
    scanImageFile(file);
  });
};
```

`scanImageFile` 的流程是把文件读成 DataURL，加载成 `Image`，再交给 ZXing 解码：

```js
const scanImageFile = (file) => {
  if (!codeReader) return;

  const reader = new FileReader();
  reader.onload = (e) => {
    const img = new Image();
    img.onload = () => {
      codeReader
        .decodeFromImageElement(img)
        .then((result) => addResult(file.name, result.text, result.format))
        .catch(() => addResult(file.name, "未识别到二维码", "error"));
    };
    img.src = e.target.result;
  };
  reader.readAsDataURL(file);
};
```

这里使用 `Image()` 的原因：先让浏览器把 DataURL 解码成像素数据，再由 ZXing 从像素中定位并识别二维码。

## 3）摄像头识别：decodeFromVideoDevice 持续回调

摄像头模式不自行做 `getUserMedia` + `canvas` 截帧，而是让 ZXing 直接接管：它会持续从视频帧中尝试识别。

```js
const isCameraActive = ref(false);
const videoElement = ref(null);

const startCamera = () => {
  if (!codeReader) return;
  isCameraActive.value = true;

  codeReader
    .decodeFromVideoDevice(null, videoElement.value, (result, err) => {
      if (result) {
        addResult("摄像头扫描", result.text, result.format);
      }
      // 识别不到时 err 往往只是“没找到”，不需要每帧都弹提示
    })
    .catch(() => {
      isCameraActive.value = false;
      addResult("摄像头", "摄像头启动失败或无权限", "error");
    });
};

const stopCamera = () => {
  if (codeReader) codeReader.reset();
  isCameraActive.value = false;
};
```

传 `null` 表示用默认摄像头；如果你自己做了设备选择，把 `deviceId` 传进去就行。

## 4）结果结构：只存“来源 + 内容 + 格式 + 时间”

结果列表使用数组保存，元素结构如下：

```js
// { source, content, format, isError, timestamp }
const results = ref([]);
```

字段都很直白：来源是“文件名/摄像头”，`content` 是解出来的文本，`format` 用来展示二维码类型，`timestamp` 用来做去重。

## 5）为什么要去重：摄像头会反复识别同一张码

摄像头模式下，二维码只要还在画面里，就可能被重复识别（可以理解为间隔很短就会再次识别）。如果每次识别成功都写入结果列表，会出现大量重复记录。

这里用“时间窗口去重”：2 秒内内容相同则跳过写入。

```js
const addResult = (source, content, format) => {
  const isError = format === "error";
  const now = Date.now();

  const recentSame = results.value.find(
    (r) => r.content === content && now - r.timestamp < 2000,
  );

  if (recentSame && !isError) return;

  let formatName = format;
  if (!isError && typeof format === "number")
    formatName = getFormatName(format);
  else if (format && format.formatName) formatName = format.formatName;

  results.value.unshift({
    source,
    content,
    format: isError ? "" : String(formatName),
    isError,
    timestamp: now,
  });
};
```

效果是：镜头对准二维码时，结果只会稳定新增一次，不会被重复记录刷屏。

## 6）格式显示：把枚举值映射成常见名字

ZXing 的 `format` 有时候是枚举数字。为了让展示更直观，这里做一个映射表，把常见值转成字符串。

```js
const getFormatName = (format) => {
  const formats = {
    11: "QR_CODE",
    5: "DATA_MATRIX",
    0: "AZTEC",
    10: "PDF_417",
  };
  return formats[format] || format;
};
```

没覆盖到的就原样返回，至少信息不会丢。
