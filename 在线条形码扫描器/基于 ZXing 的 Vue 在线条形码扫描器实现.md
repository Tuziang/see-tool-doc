# 基于 ZXing 的 Vue 在线条形码扫描器实现

这篇文章只讲本项目里“在线条形码扫描器”工具的功能 JavaScript 实现。它的目标很直接：让用户通过上传图片或调用摄像头，识别常见条形码内容，并把结果整理成可复制的文本。

> 在线工具网址：[https://see-tool.com/barcode-scanner](https://see-tool.com/barcode-scanner)  
> 工具截图：  
> ![工具截图](工具截图.png)

整个实现可以拆成 5 段：状态组织、扫描器初始化、图片识别、摄像头识别、结果整理与复制。

## 1）先围绕“扫描流程”组织状态

这个工具的前端状态不复杂，核心就是把一次扫描过程拆开管理：

- `activeTab`：当前是图片上传还是摄像头扫描
- `fileInput`：文件选择框引用
- `videoElement`：摄像头视频元素引用
- `isCameraActive`：摄像头是否已开启
- `results`：扫描结果列表
- `copiedIndex`：当前已复制的是哪一条结果
- `codeReader`：条形码解码实例

这样做的好处是，上传、扫描、展示、复制这几段逻辑彼此独立，但都围绕同一份数据流运行。

## 2）扫描器实例只在浏览器端初始化

识别能力来自 `@zxing/library` 提供的 `BrowserMultiFormatReader`。页面挂载后再创建实例，页面卸载时顺手关闭摄像头。

```js
let codeReader = null;

onMounted(() => {
  if (process.client) {
    codeReader = new BrowserMultiFormatReader();
  }
});

onUnmounted(() => {
  stopCamera();
});
```

这里的关键点不是“创建一个对象”，而是让识别器的生命周期和页面保持一致，避免用户离开页面后摄像头还占着。

## 3）图片识别是“文件 -> Base64 -> Image -> 解码结果”

上传和拖拽最终都会进入同一套处理逻辑。先判断文件是不是图片，再逐个读取。

```js
const handleFiles = (files) => {
  if (!files || files.length === 0) return;

  Array.from(files).forEach((file) => {
    if (file.type.startsWith("image/")) {
      scanImageFile(file);
    }
  });
};
```

真正的识别过程分两步：

1. 用 `FileReader` 把本地图片读成 Data URL
2. 创建 `Image` 对象，等图片加载完成后交给 ZXing 解码

```js
const scanImageFile = (file) => {
  if (!codeReader) return;

  const reader = new FileReader();
  reader.onload = (e) => {
    const img = new Image();
    img.onload = () => {
      codeReader
        .decodeFromImageElement(img)
        .then((result) => {
          addResult(file.name, result.text, result.format);
        })
        .catch(() => {
          addResult(file.name, "未识别到条形码", "error");
        });
    };
    img.src = e.target.result;
  };
  reader.readAsDataURL(file);
};
```

这条链路的重点是：页面并不是直接“上传文件就扫”，而是先把文件转换成浏览器可处理的图像对象，再统一交给解码器。

## 4）摄像头识别是持续监听视频帧

和图片扫描一次不同，摄像头模式是持续从视频流中取帧，再不断尝试解码。

```js
const startCamera = () => {
  if (!codeReader) return;
  isCameraActive.value = true;

  codeReader
    .decodeFromVideoDevice(null, videoElement.value, (result) => {
      if (result) {
        addResult("摄像头扫描", result.text, result.format);
      }
    })
    .catch(() => {
      isCameraActive.value = false;
    });
};
```

这里用了 `decodeFromVideoDevice`。它不是返回一次结果就结束，而是不断回调：

- 当前帧识别到了条码，就立刻写入结果列表
- 当前帧没识别到，就继续等下一帧

关闭摄像头时则统一调用重置方法：

```js
const stopCamera = () => {
  if (codeReader) {
    codeReader.reset();
  }
  isCameraActive.value = false;
};
```

这样上传模式和摄像头模式可以随时切换，不会互相干扰。

## 5）结果要做一次归一化，方便展示和复制

解码器返回的原始结果里，最重要的是两部分：

- `text`：条形码实际内容
- `format`：条码格式

工具不会把原始对象直接丢给界面，而是先整理成统一结构：

```js
const addResult = (source, content, format) => {
  const isError = format === "error";
  let formatName = format;

  if (!isError && typeof format === "number") {
    formatName = getFormatName(format);
  } else if (format && format.formatName) {
    formatName = format.formatName;
  }

  results.value.unshift({
    source,
    content,
    format: isError ? "" : String(formatName),
    isError,
  });
};
```

其中 `getFormatName` 的作用，是把数字格式码转换成更直观的名字，例如 `CODE_128`、`EAN_13`、`UPC_A`。这样用户看到结果时，不只是拿到内容，也能知道识别出的条码属于哪一类。

结果区还支持两个常用动作：

- `clearResults()`：清空历史结果
- `copyToClipboard()`：把指定结果复制到剪贴板

```js
const copyToClipboard = async (text, index) => {
  await navigator.clipboard.writeText(text);
  copiedIndex.value = index;
  setTimeout(() => {
    copiedIndex.value = -1;
  }, 2000);
};
```

这段逻辑很轻，但很实用。对用户来说，识别完成不是终点，复制出去继续使用才是完整闭环。

## 6）这套功能 JS 的核心思路

这个工具的实现，本质上就是一条很清晰的数据链：

`选择输入方式 -> 获取图片或视频帧 -> ZXing 解码 -> 统一整理结果 -> 复制或清空`

从实现角度看，真正的重点不是界面，而是把“上传图片识别”和“摄像头实时识别”这两条路径收拢到同一套结果模型里。这样工具既容易维护，也能让普通用户用最少操作完成条形码识别。
