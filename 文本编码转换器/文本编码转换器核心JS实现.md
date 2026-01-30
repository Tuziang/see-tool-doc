# 工具网址和截图

> 在线工具网址：[https://see-tool.com/encoding-converter](https://see-tool.com/encoding-converter)

> 工具截图：
> ![工具截图](工具截图.png)


# 文本编码转换器功能核心实现解析

本文将深入探讨文本编码转换器（Text Encoding Converter）的核心 JavaScript 实现逻辑。该工具旨在实现普通文本与多种编码格式（如十六进制、二进制、Base64、Unicode 等）之间的相互转换。

## 1. 核心转换机制

整个工具的转换逻辑基于一个统一的入口函数 `convert`，它根据输入和输出格式，通过查找表（Lookup Table）调用相应的转换函数。

核心的字节处理依赖于浏览器原生的 `TextEncoder` 和 `TextDecoder` API，这确保了对 UTF-8 的正确处理。

```javascript
// 字符串转字节数组
const encoder = new TextEncoder();
const bytes = encoder.encode(text);

// 字节数组转字符串
const decoder = new TextDecoder('utf-8');
const text = decoder.decode(new Uint8Array(bytes));
```

## 2. 格式转换实现细节

### 2.1 进制转换 (Hex, Binary, Octal, Decimal)

对于二进制、八进制、十六进制等数字格式，核心思路是将文本转换为字节数组，然后利用 `Number.prototype.toString(radix)` 将每个字节转换为对应的进制字符串。

以**Hex（十六进制）**为例：

```javascript
textToHex: function(text, delimiter, prefix, uppercase) {
    const encoder = new TextEncoder();
    const bytes = encoder.encode(text);
    let hex = Array.from(bytes).map(b => {
        // 每个字节转16进制，并补齐2位
        let h = b.toString(16).padStart(2, '0');
        if (uppercase) h = h.toUpperCase();
        return prefix + h;
    });
    return hex.join(delimiter);
}
```

反向转换则是移除前缀和分隔符后，使用 `parseInt(chunk, 16)` 还原字节。

### 2.2 Base64 编码

JavaScript 原生的 `btoa` 和 `atob` 函数只能处理 ASCII 字符。为了支持中文等 Unicode 字符，我们需要先对字符串进行编码处理。

**文本转 Base64** 的健壮实现：

```javascript
textToBase64: function(text) {
    try {
        // 方法1: 使用 TextEncoder 获取字节，构造二进制字符串
        const encoder = new TextEncoder();
        const bytes = encoder.encode(text);
        let binary = '';
        bytes.forEach(byte => binary += String.fromCharCode(byte));
        return btoa(binary);
    } catch (e) {
        // 方法2: 降级方案，使用 encodeURIComponent 处理
        return btoa(unescape(encodeURIComponent(text)));
    }
}
```

**Base64 转文本**：

```javascript
base64ToText: function(base64) {
    const binary = atob(base64.trim());
    const bytes = new Uint8Array(binary.length);
    for (let i = 0; i < binary.length; i++) {
        bytes[i] = binary.charCodeAt(i);
    }
    const decoder = new TextDecoder('utf-8');
    return decoder.decode(bytes);
}
```

### 2.3 Unicode 转义与码点

处理 Unicode 转义（如 `\u4E2D`）时，关键在于正确处理**代理对（Surrogate Pairs）**。对于超出基本多文种平面（BMP, U+0000 到 U+FFFF）的字符（例如 Emoji），JavaScript 的字符串长度为 2。

我们使用 `codePointAt(0)` 来获取完整的码点值：

```javascript
textToUnicodeEscape: function(text, delimiter, uppercase) {
    let result = [];
    for (let char of text) {
        let code = char.codePointAt(0);
        // 如果码点超过 0xFFFF，说明是代理对，JS 会将其视为两个字符
        if (code > 0xFFFF) {
            // 手动计算代理对（虽然 ES6 for-of 循环会自动正确迭代字符）
            const high = Math.floor((code - 0x10000) / 0x400) + 0xD800;
            const low = (code - 0x10000) % 0x400 + 0xDC00;
            // ... 转换为 \uXXXX\uXXXX 格式
            let h1 = high.toString(16).padStart(4, '0');
            let h2 = low.toString(16).padStart(4, '0');
            result.push('\\u' + h1);
            result.push('\\u' + h2);
        } else {
            // ... 普通字符转换为 \uXXXX
            let h = code.toString(16).padStart(4, '0');
            result.push('\\u' + h);
        }
    }
    return result.join(delimiter);
}
```

注意：使用 `for...of` 循环可以正确遍历字符串中的 Emoji 等宽字符，而普通的 `for(let i=0;...)` 则会把它们拆分成两个。

### 2.4 Punycode 转换

Punycode 是国际化域名（IDN）使用的编码。本项目采用了一个巧妙的利用浏览器原生 API 的方法，避免引入庞大的第三方库：

```javascript
punycode: {
    encode: function(input) {
        try {
            // 利用 URL API 自动进行 Punycode 编码
            const url = new URL('http://' + input);
            return url.hostname.replace(/^xn--/, '');
        } catch (e) {
            // 降级处理...
        }
    },
    decode: function(input) {
        // 利用 URL API 自动解析
        const testUrl = 'http://' + input;
        const url = new URL(testUrl);
        return url.hostname;
    }
}
```

这是一个非常轻量且高效的实现方式。

### 2.5 HTML 实体

HTML 实体的转换相对直接，主要将字符转换为其对应的十进制或十六进制引用：

```javascript
textToHtmlDecimal: function(text, delimiter) {
    let result = [];
    for (let char of text) {
        let code = char.codePointAt(0);
        result.push('&#' + code + ';');
    }
    return result.join(delimiter);
}
```

## 3. 字符详情分析

工具还提供了一个 `getCharacterInfo` 函数，用于分析单个字符的详细信息。它不仅返回字符本身，还计算其 Unicode 码点、UTF-8 字节序列等。

```javascript
function getCharacterInfo(char) {
    const codePoint = char.codePointAt(0);
    const encoder = new TextEncoder();
    const utf8Bytes = encoder.encode(char);
    
    return {
        char: char,
        codePoint: codePoint, // 数字形式
        hex: codePoint.toString(16).toUpperCase(), // Hex 形式
        utf8: Array.from(utf8Bytes) // UTF-8 字节序列
              .map(b => b.toString(16).toUpperCase().padStart(2, '0'))
              .join(' ')
    };
}
```

## 总结

本项目的文本编码转换器通过充分利用 `TextEncoder`/`TextDecoder`、`URL` API 以及 ES6+ 的字符串处理特性（如 `codePointAt`、`for...of`），以原生 JavaScript 实现了高效、轻量的多格式转换，无需依赖任何重型第三方库。
