# Vue3项目中集成科学计算器的实现方案

## 前言

在开发在线工具平台时,科学计算器是核心功能之一。本文介绍如何在Vue3项目中优雅地集成一个功能完善的科学计算器,重点讨论外部JavaScript引擎的加载策略和Vue组件的封装方案。

## 工具页面

工具页面地址：https://see-tool.com/calculator

工具页面截图：

<img src="https://i-blog.csdnimg.cn/direct/43e28515e86b4e0e812d00fb7bafedda.png"/>

## 技术栈

- Vue 3 (Composition API) + Vite
- jQuery (计算器引擎依赖)

## 架构设计

### 核心思路

**关注点分离**: 将UI层(Vue组件)与计算引擎(原生JS)分离,通过动态脚本加载和函数桥接实现通信。

```
Vue组件层 ──桥接函数──> 计算引擎层
    ↓                      ↓
  UI渲染               数学运算
  事件处理             表达式解析
  国际化               精度控制
```

### 文件组织

```
src/views/Calculator.vue       # Vue组件(UI+逻辑)
public/js/jquery-2.2.4.min.js  # jQuery依赖
public/js/calculator-original.js # 计算引擎
```

## 核心实现

### 1. 动态脚本加载策略

**关键问题**: 计算器引擎依赖jQuery,必须确保正确的加载顺序。

**解决方案**: 实现链式加载机制

```javascript
// 检测jQuery → 加载jQuery → 加载计算器引擎
if (!window.$) {
  loadScript('/js/jquery.min.js', () => {
    loadScript('/js/calculator.js')
  })
} else {
  loadScript('/js/calculator.js')
}
```

**要点**:
- 使用`onload`回调确保顺序
- 提供`onerror`错误处理
- 添加重试机制应对异步问题

### 2. 函数桥接机制

**核心挑战**: Vue组件的`calc`方法与全局`window.calc`命名冲突。

**解决方案**: 引用保存 + 类型检查

```javascript
let originalCalc = null

const calc = (value) => {
  if (originalCalc) {
    originalCalc(value)
  } else if (window.calc && window.calc !== calc) {
    originalCalc = window.calc
    originalCalc(value)
  }
}
```

**关键点**:
- `originalCalc`保存原始引用避免冲突
- `window.calc !== calc`防止循环引用
- 惰性绑定处理异步加载

### 3. 生命周期管理

```javascript
onMounted(() => {
  // 等待DOM就绪后加载脚本
  nextTick(() => loadScripts())
})

onUnmounted(() => {
  // 清理全局变量和事件监听
  window.calculatorCleanup?.()
  originalCalc = null
})
```

### 4. 角度/弧度模式切换

通过直接操作全局变量和DOM实现:

```javascript
const handleDegRad = (mode) => {
  window.cnDegreeRadians = mode  // 设置计算模式
  updateUIIndicator(mode)         // 更新UI指示器
}
```

## 技术难点与解决方案

### 1. 脚本加载时序

**问题**: `script.onload`触发时函数可能未定义  
**方案**: 延迟50ms后检查,失败则100ms后重试

### 2. 函数命名冲突

**问题**: 组件方法与全局函数同名  
**方案**: 使用闭包变量保存引用,避免直接访问全局

### 3. 内存泄漏

**问题**: 动态添加的script标签和全局变量  
**方案**: onUnmounted钩子清理资源

## 总结

本文介绍了在Vue3项目中集成科学计算器的完整方案,核心要点:

1. **架构设计**: UI与计算引擎分离,通过桥接通信
2. **加载策略**: 链式加载保证依赖顺序,添加重试机制
3. **命名冲突**: 使用引用保存原始函数,避免全局污染
4. **生命周期**: 正确管理资源加载和清理

通过这种方式,我们成功将原生JavaScript计算器引擎集成到现代Vue3项目中,保持了代码的可维护性和扩展性。
