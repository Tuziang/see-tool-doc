# Vue3 进制转换器 JS 实现方案

> 在线工具网址：[https://see-tool.com/base-converter](https://see-tool.com/base-converter)
> 工具截图：
> ![工具截图](工具截图.png)

## 一、核心设计思路

### 1.1 十进制中转策略
所有进制转换都通过**十进制作为中转站**:
```
任意进制 → parseInt(value, base) → 十进制
十进制 → decimal.toString(base) → 目标进制
```

这样避免了 N×(N-1) 种转换函数的组合爆炸,复杂度从 O(n²) 降至 O(n)。

### 1.2 防死锁机制
四个输入框实时同步时,必须防止循环触发:
```javascript
let isUpdating = false

const onSyncInput = (source) => {
  if (isUpdating) return  // 阻止递归调用
  isUpdating = true
  
  // ... 执行转换逻辑
  
  isUpdating = false
}
```

## 二、同步转换模式实现

### 2.1 核心转换函数

```javascript
const onSyncInput = (source) => {
  if (isUpdating) return
  isUpdating = true

  try {
    let decimalNum = 0
    let cleanValue = ''

    // 第一步: 数据清洗 + 转十进制
    switch (source) {
      case 'binary':
        // 去除空格、下划线、0b前缀
        cleanValue = binaryValue.value.replace(/[\s_]/g, '').replace(/^0[bB]/, '')
        // 正则验证是否为合法二进制
        if (cleanValue && /^[01]+$/.test(cleanValue)) {
          decimalNum = parseInt(cleanValue, 2)
        } else {
          clearOthers('binary')  // 清空其他框
          isUpdating = false
          return
        }
        break
        
      case 'octal':
        cleanValue = octalValue.value.replace(/[\s_]/g, '').replace(/^0[oO]/, '')
        if (cleanValue && /^[0-7]+$/.test(cleanValue)) {
          decimalNum = parseInt(cleanValue, 8)
        } else {
          clearOthers('octal')
          isUpdating = false
          return
        }
        break
        
      case 'decimal':
        cleanValue = decimalValue.value.replace(/[\s_,]/g, '')
        if (cleanValue && /^\d+$/.test(cleanValue)) {
          decimalNum = parseInt(cleanValue, 10)
        } else {
          clearOthers('decimal')
          isUpdating = false
          return
        }
        break
        
      case 'hex':
        cleanValue = hexValue.value.replace(/[\s_]/g, '').replace(/^0[xX]/, '')
        if (cleanValue && /^[0-9A-Fa-f]+$/.test(cleanValue)) {
          decimalNum = parseInt(cleanValue, 16)
        } else {
          clearOthers('hex')
          isUpdating = false
          return
        }
        break
    }

    // 第二步: 广播更新其他进制(跳过源输入框)
    if (source !== 'binary') {
      binaryValue.value = formatValue(decimalNum.toString(2), 2)
    }
    if (source !== 'octal') {
      octalValue.value = formatValue(decimalNum.toString(8), 8)
    }
    if (source !== 'decimal') {
      decimalValue.value = decimalNum.toString()
    }
    if (source !== 'hex') {
      let hex = decimalNum.toString(16)
      if (uppercaseHex.value) hex = hex.toUpperCase()
      hexValue.value = formatValue(hex, 16)
    }
  } catch (err) {
    console.error('Conversion error:', err)
  }

  isUpdating = false
}
```

**关键点**:
1. **数据清洗**: 用正则去除空格、下划线、前缀等干扰字符
2. **严格验证**: 每种进制都有对应的正则验证规则
3. **跳过源框**: 避免回填源输入框导致光标跳动
4. **异常处理**: try-catch 防止非法输入崩溃

### 2.2 数据验证正则

```javascript
/^[01]+$/          // 二进制: 只允许 0 和 1
/^[0-7]+$/         // 八进制: 只允许 0-7
/^\d+$/            // 十进制: 只允许数字
/^[0-9A-Fa-f]+$/   // 十六进制: 允许 0-9 和 A-F(大小写)
```

### 2.3 格式化显示函数

```javascript
const formatValue = (value, base) => {
  let formatted = value
  
  // 1. 添加前缀(可选)
  let prefix = ''
  if (showPrefix.value) {
    switch (base) {
      case 2:  prefix = '0b'; break
      case 8:  prefix = '0o'; break
      case 16: prefix = '0x'; break
    }
  }
  
  // 2. 数字分组(提升可读性)
  if (digitGrouping.value && formatted.length > 4) {
    let groupSize = 4
    if (base === 2)  groupSize = 4  // 二进制: 1010 1100
    else if (base === 8)  groupSize = 3  // 八进制: 123 456
    else if (base === 10) groupSize = 3  // 十进制: 123 456
    else if (base === 16) groupSize = 4  // 十六进制: 1A2B 3C4D
    
    // 从右向左分组
    const groups = []
    for (let i = formatted.length; i > 0; i -= groupSize) {
      groups.unshift(formatted.substring(Math.max(0, i - groupSize), i))
    }
    formatted = groups.join(' ')
  }
  
  return prefix + formatted
}
```

**分组规则**:
- 二进制/十六进制: 每 4 位一组
- 八进制/十进制: 每 3 位一组

### 2.4 辅助函数

```javascript
// 清除其他输入框
const clearOthers = (except) => {
  if (except !== 'binary') binaryValue.value = ''
  if (except !== 'octal') octalValue.value = ''
  if (except !== 'decimal') decimalValue.value = ''
  if (except !== 'hex') hexValue.value = ''
}

// 清除全部
const clearAll = () => {
  binaryValue.value = ''
  octalValue.value = ''
  decimalValue.value = ''
  hexValue.value = ''
}

// 复制文本(去除格式化空格)
const copyText = async (text) => {
  if (!text) return
  
  try {
    // 去除空格、下划线、前缀,复制纯净数据
    const cleanText = text.replace(/[\s_]/g, '')
    await navigator.clipboard.writeText(cleanText)
    safeMessage.success(t('baseConverter.notifications.copySuccess'))
  } catch (err) {
    console.error('Failed to copy:', err)
    safeMessage.error(t('baseConverter.notifications.copyFailed'))
  }
}
```

## 三、自定义进制模式实现

### 3.1 核心转换函数

```javascript
const convertCustom = () => {
  if (!customInput.value.trim()) {
    customOutput.value = ''
    return
  }

  try {
    // 利用原生 API 支持 2-36 进制
    const decimal = parseInt(customInput.value, customFromBase.value)
    if (isNaN(decimal)) {
      customOutput.value = t('common.invalidInput')
      return
    }
    customOutput.value = decimal.toString(customToBase.value).toUpperCase()
  } catch (err) {
    customOutput.value = t('common.conversionError')
  }
}
```

**原生 API 优势**:
- `parseInt(str, radix)`: 支持 2-36 进制解析
- `num.toString(radix)`: 支持 2-36 进制输出
- 无需自己实现进制转换算法

### 3.2 进制交换函数

```javascript
const swapBases = () => {
  // 交换源进制和目标进制
  const temp = customFromBase.value
  customFromBase.value = customToBase.value
  customToBase.value = temp
  
  // 将输出作为新的输入
  customInput.value = customOutput.value
  convertCustom()
}
```

## 四、响应式配置实现

### 4.1 大小写切换

```javascript
watch(uppercaseHex, () => {
  if (hexValue.value) {
    hexValue.value = uppercaseHex.value 
      ? hexValue.value.toUpperCase() 
      : hexValue.value.toLowerCase()
  }
})
```

### 4.2 前缀/分组切换

```javascript
// 监听前缀切换
watch(showPrefix, () => {
  if (decimalValue.value) {
    onSyncInput('decimal')  // 重新格式化所有输入框
  }
})

// 监听数字分组切换
watch(digitGrouping, () => {
  if (decimalValue.value) {
    onSyncInput('decimal')
  }
})
```

### 4.3 自定义进制变化

```javascript
// 监听自定义进制变化,自动重新转换
watch([customFromBase, customToBase], () => {
  convertCustom()
})
```

## 五、状态管理

### 5.1 响应式状态定义

```javascript
// Tab 切换
const activeTab = ref('sync')

// 同步转换模式的四个输入框
const binaryValue = ref('')
const octalValue = ref('')
const decimalValue = ref('')
const hexValue = ref('')

// 配置选项
const uppercaseHex = ref(false)    // 十六进制大写
const showPrefix = ref(false)      // 显示前缀 (0x, 0b)
const digitGrouping = ref(true)    // 数字分组

// 自定义进制模式
const customFromBase = ref(10)     // 源进制
const customToBase = ref(16)       // 目标进制
const customInput = ref('')        // 输入值
const customOutput = ref('')       // 输出值

// 防死锁标志位
let isUpdating = false
```

## 六、已知限制与改进方向

### 6.1 大数精度问题

当前使用 `Number` 类型,超过 `Number.MAX_SAFE_INTEGER` (2^53 - 1) 会丢失精度。

**改进方案**:
```javascript
// 使用 BigInt 替代 Number
const decimalNum = BigInt(`0b${cleanValue}`)
const result = decimalNum.toString(16)
```

**注意事项**:
- `BigInt` 不支持小数
- `BigInt` 和 `Number` 不能直接运算
- 需要特殊处理边界情况

### 6.2 性能优化

对于频繁输入场景,可添加防抖:
```javascript
import { debounce } from 'lodash-es'

const onSyncInput = debounce((source) => {
  // ... 转换逻辑
}, 100)
```

## 七、核心流程总结

```
用户输入 → 数据清洗 → 正则验证 → parseInt(转十进制)
                                        ↓
                                   十进制中转
                                        ↓
    格式化显示 ← toString(转目标进制) ← 广播更新
```

**核心原则**:
1. **十进制中转**: 避免组合爆炸
2. **防死锁机制**: 阻止循环触发
3. **数据清洗**: 确保输入合法
4. **格式化显示**: 提升可读性
5. **原生 API**: 避免重复造轮子
