# 密码生成器技术实现分析

本文将深入探讨本项目中“密码生成器”工具的前端技术实现，主要聚焦于 Vue 页面交互逻辑与底层的 JavaScript 核心算法。


> 在线工具网址：[https://see-tool.com/password-generator](https://see-tool.com/password-generator)

> 工具截图：
> ![工具截图](工具截图.png)


## 1. 核心逻辑实现 (Composable)

密码生成器的核心逻辑被封装在 `composables/usePasswordGenerator.js` 中，主要包含随机数生成、字符集构建、密码生成策略以及强度估算算法。

### 1.1 安全随机数生成

为了确保密码的安全性，我们没有使用 `Math.random()`，而是采用了浏览器提供的 Web Crypto API。

```javascript
// 生成随机整数
const getRandomInt = (max) => {
    // 使用加密安全的随机数生成器
    const array = new Uint32Array(1);
    crypto.getRandomValues(array);
    return array[0] % max;
};
```

`crypto.getRandomValues` 提供了加密级别的随机性，避免了伪随机数生成器可能存在的预测风险。

### 1.2 随机密码生成策略

随机密码的生成过程主要分为两步：构建字符集和抽取字符。

1.  **构建字符集 (`buildCharset`)**：根据用户勾选的选项（大写、小写、数字、符号、自定义符号），动态拼接可用字符字符串。同时支持正则替换，用于排除易混淆字符（如 `l` 和 `1`）或有歧义的字符。
2.  **生成密码 (`generateRandomPassword`)**：利用 `getRandomInt` 从构建好的字符集中随机抽取指定长度的字符。

```javascript
const generateRandomPassword = (length, options) => {
    const charset = buildCharset(options); // 根据选项构建字符集
    if (!charset) return '';

    let password = '';
    for (let i = 0; i < length; i++) {
        password += charset[getRandomInt(charset.length)];
    }
    return password;
};
```

### 1.3 密码短语模式 (Passphrase)

除了传统随机字符密码，工具还支持 XKCD 风格的“密码短语”。它通过从预置的单词列表（`WORD_LIST`）中随机抽取单词并使用分隔符连接来实现。

```javascript
const generatePassphrase = (options) => {
    const { wordCount, wordSeparator, capitalizeWords, includeNumber } = options;
    const words = [];

    for (let i = 0; i < wordCount; i++) {
        let word = WORD_LIST[getRandomInt(WORD_LIST.length)];
        // 处理首字母大写
        if (capitalizeWords) {
            word = word.charAt(0).toUpperCase() + word.slice(1);
        }
        words.push(word);
    }
    
    // 逻辑省略：在随机位置插入数字...
    
    return words.join(wordSeparator);
};
```

### 1.4 强度估算算法

密码强度主要通过计算“熵 (Entropy)”来评估。计算公式为 $E = L \times \log_2(R)$，其中 $L$ 是密码长度，$R$ 是字符集大小。

```javascript
const getStrengthLevel = (password) => {
    // 估算字符集大小 poolSize
    let poolSize = 0;
    if (/[a-z]/.test(password)) poolSize += 26;
    if (/[A-Z]/.test(password)) poolSize += 26;
    if (/[0-9]/.test(password)) poolSize += 10;
    if (/[^a-zA-Z0-9]/.test(password)) poolSize += 32;

    // 计算熵
    const entropy = Math.log2(Math.pow(poolSize, password.length));

    // 根据熵值划分等级（弱、中、强、极佳）
    // ...
};
```

## 2. Vue 页面交互实现

前端页面 (`pages/password-generator.vue`) 使用 Vue 3 的 Composition API (`<script setup>`) 构建，实现了高度响应式的交互体验。

### 2.1 响应式状态管理

页面维护了多种状态来控制生成逻辑，包括模式选择（随机/短语）、字符长度、字符类型开关等。

```javascript
// 模式控制
const currentMode = ref('random')

// 随机模式设置
const passwordLength = ref(16)
const uppercase = ref(true)
// ... 其他开关

// 密码短语设置
const wordCount = ref(4)
// ... 其他设置
```

### 2.2 实时预览与自动生成

为了提供流畅的用户体验，利用 `watch` 监听所有相关配置的变化。一旦用户调整了滑块长度或勾选了某个选项，密码会立即重新生成。

```javascript
// 监听主要设置变化，实时更新单个密码
watch([passwordLength, uppercase, lowercase, ..., currentMode], () => {
    if (batchQuantity.value === 1) {
        generatePassword()
    }
})
```

这种即时反馈机制避免了用户频繁点击“生成”按钮的操作。

### 2.3 批量生成逻辑

工具支持批量生成密码。逻辑上通过简单的循环复用 `generateOne` 内部函数来实现，结果存储在 `batchPasswords` 数组中供列表渲染。

```javascript
const generatePassword = () => {
    // ...
    if (quantity === 1) {
        // 单个生成逻辑，并计算强度
        // ...
    } else {
        // 批量循环生成
        batchPasswords.value = []
        for (let i = 0; i < quantity; i++) {
            const pwd = generateOne()
            if (pwd) batchPasswords.value.push(pwd)
        }
        // ...
    }
}
```


## 总结

本工具通过将核心算法逻辑抽离为 Composable (`usePasswordGenerator`)，保持了代码的清晰和可复用性。Vue 层则专注于状态管理和交互反馈，利用响应式系统实现了配置修改后的即时预览功能。
