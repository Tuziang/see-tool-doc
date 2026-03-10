# 文字/数字提取 在线工具核心JS实现

这个工具看起来简单，实际开发时我先做了一件事：不急着写界面，先把“提取逻辑”做好。因为用户输入可能会非常杂，可能一段话里同时有中文、英文、数字、空格和符号，如果核心函数不稳，前端交互再漂亮也没用。

> 在线工具网址：[https://see-tool.com/text-number-extractor](https://see-tool.com/text-number-extractor)  
> 工具截图：  
> ![工具截图](工具截图.png)


我最后把功能拆成两块：

- `extractTextNumbers`：只负责提取。
- `countChars`：只负责统计。

页面层只做参数传递和按钮动作，避免把业务细节塞进组件里。

## 先说提取函数是怎么定下来的

函数签名是这样：

```javascript
export function extractTextNumbers(
  text,
  extractChinese = true,
  extractEnglish = true,
  extractNumbers = true,
  extractByLine = false,
  removeDuplicates = false,
  sortChars = false
) {
  if (!text || !text.trim()) return ''
}
```

这几个布尔值其实对应页面上的勾选项。这样有个好处：页面不需要做复杂映射，用户怎么选，函数就怎么跑。

我一开始想过用一条“大正则”把三种字符一次性提取完，但后来放弃了。原因很简单：可读性太差，后面再加规则会麻烦。所以改成逐字符判断，逻辑更长一点，但直观。

## 逐字符提取，比花哨写法更稳

核心循环没有技巧，就是老老实实遍历：

```javascript
for (let i = 0; i < line.length; i++) {
  const char = line[i]

  if (extractChinese && /[\u4e00-\u9fff]/.test(char)) {
    lineResult += char
  } else if (extractEnglish && /[a-zA-Z]/.test(char)) {
    lineResult += char
  } else if (extractNumbers && /[0-9]/.test(char)) {
    lineResult += char
  }
}
```

这里我保留了 `if / else if` 结构，目的就是“一个字符只走一个分支”。这能减少边界行为，输出顺序也和原文一致，用户更容易理解结果。

## 按行提取和整体提取，走两条分支

`extractByLine` 打开时，流程是：先按换行切分，再逐行处理，最后把每行结果再拼回去。

```javascript
const lines = text.split('\n')
const processedLines = []
// 每一行独立提取
return processedLines.join('\n')
```

这个模式很实用。比如用户在整理多行数据时，通常希望“第几行对第几行”，不希望全部揉成一串。

关闭按行模式时就简单了，直接整段提取，适合一次性清洗长文本。

## 去重和排序，顺序不能乱

提取完之后我把后处理固定成：先去重，再排序。

```javascript
if (removeDuplicates) {
  result = [...new Set(result)].join('')
}

if (sortChars) {
  result = result.split('').sort().join('')
}
```

如果先排序再去重，结果也能对，但中间会多做无效工作，而且阅读代码时不如“先减量再整理”自然。

## 统计函数单独维护，页面就轻很多

统计逻辑我没有混进提取函数，而是单独写在 `countChars`。

- 普通模式：直接看长度。
- 按行模式：行数只算非空行，总字符数不含换行。

页面里通过 `computed` 串起来：输入一变，提取结果自动变；提取结果一变，统计自动变。用户看起来就是实时更新，代码上也比较干净。

## 按钮动作只做三件事

页面脚本里和结果相关的动作只有三个：复制、下载、清空。

- 复制：`navigator.clipboard.writeText(extractedText)`
- 下载：`Blob + URL.createObjectURL + a.click()`
- 清空：只改输入值，结果和统计由计算属性自动归零

到这里，这个工具的核心 JS 基本就闭环了：输入文本进来，规则提取，按需后处理，统计结果，再做复制或导出。没有绕太多层，但每个环节都能单独看懂、单独改动。
