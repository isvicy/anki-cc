# Quizify 模板 HTML 实体显示问题

## 问题描述

在使用 Quizify 模板创建包含代码示例的卡片时，`<` `>` `&` 等特殊字符显示异常：
- `&lt;` `&gt;` `&amp;` 字面显示，而不是转换为 `<` `>` `&`
- 正则表达式如 `(?<!...)` 被破坏成 `(?<!--...)`
- 卡片末尾出现多余的 `</div>` 或 `</body-->` 等垃圾内容
- 代码块内容格式混乱

## 环境信息

- OS: NixOS (Linux 6.12.60)
- Anki: 桌面版
- 模板: Quizify - 格致 (@chehil) + Markdown Integration (marked.js + highlight.js)

## 根本原因

### 1. Anki 将字段内容当作 HTML 解析

Anki 的字段内容是 HTML 格式。当存储包含 `<` `>` 的内容时：
- `</\1>` 被解析为 HTML 闭合标签，导致内容被截断
- `(?<!...)` 中的 `<!` 被解析为 HTML 注释开始，变成 `(?<!--...)`
- 未闭合的标签会导致 Anki 自动添加闭合标签如 `</div>`

### 2. marked.js 二次转义

当内容存储为 `&lt;` 时，marked.js 在解析 Markdown 代码块时会进行二次转义：
- 输入: `&lt;div&gt;`
- 输出: `&amp;lt;div&amp;gt;`
- 显示: `&lt;div&gt;`（字面显示实体）

### 3. AnkiConnect API 的 `update_note` 行为

通过 AnkiConnect API 使用 `update_note` 更新卡片内容时，返回成功但实际内容未更新（原因未明确）。需要删除后重新创建卡片。

## 诊断过程

### 1. 确认存储内容

```bash
curl -s localhost:8765 -X POST -d '{
  "action": "notesInfo",
  "version": 6,
  "params": {"notes": [NOTE_ID]}
}' | jq '.result[0].fields'
```

观察到：
- `(?<!...)` 变成了 `(?<!--...)`
- 末尾有 `</body-->` 等垃圾内容
- `&lt;` 显示为字面文本

### 2. 测试 `create_note` vs `update_note`

创建测试卡片验证：
- `create_note` 可以正确存储 `<` `>` 字符
- `update_note` 返回成功但内容未变化

```bash
# 创建测试卡片
curl -s localhost:8765 -X POST -d '{
  "action": "addNote",
  "params": {
    "note": {
      "deckName": "Test",
      "modelName": "Quizify - 格致 (@chehil)",
      "fields": {
        "Front": "测试 `&lt;div&gt;`",
        "Back": "```html\n&lt;div&gt;test&lt;/div&gt;\n```"
      }
    }
  }
}'
```

### 3. 分析模板渲染流程

marked.js 处理流程：
1. 读取 `el.innerHTML` (包含 `&lt;`)
2. `marked.parse()` 将 `&lt;` 二次转义为 `&amp;lt;`
3. 输出到 DOM 后显示为字面的 `&lt;`

## 解决方案

### 方案概述

1. **存储**: 使用 HTML 实体 (`&lt;` `&gt;` `&amp;`) 存储特殊字符
2. **模板**: 修改模板，在 marked.js 解析前先解码 HTML 实体

### 步骤 1: 修改 Quizify 背面模板

找到 Markdown 渲染部分，修改为：

```javascript
// 对 Back 字段且不包含 Quizify 语法的内容进行 Markdown 渲染
if (el.classList.contains('quizify-field--back') && !hasQuizifySyntax) {
    // 先解码 HTML 实体，再传给 marked.js
    var tempDiv = document.createElement('div');
    tempDiv.innerHTML = el.innerHTML.replace(/<br\s*\/?>/gi, '\n');
    var decodedContent = tempDiv.textContent;

    el.innerHTML = marked.parse(decodedContent);

    // 代码高亮
    el.querySelectorAll('pre code').forEach((block) => {
        hljs.highlightElement(block);
    });

    // 规范化内联代码
    el.querySelectorAll('code').forEach((code) => {
        if (!code.closest('pre')) {
            normalizeHtmlEntities(code);
        }
    });
} else {
    // 对有 Quizify 语法的内容，也规范化内联代码
    el.querySelectorAll('code').forEach(normalizeHtmlEntities);
}
```

添加 `normalizeHtmlEntities` 函数：

```javascript
// HTML 实体规范化函数：利用 textContent 自动解码实体
function normalizeHtmlEntities(element) {
    element.textContent = element.textContent;
}
```

### 步骤 2: 修改正面模板中的内联代码处理

确保 Front 字段在背面显示时也正确渲染内联代码：

```javascript
// 对不进行 Markdown 渲染的字段，处理内联代码
if (hasQuizifySyntax || !el.classList.contains('quizify-field--back')) {
    el.innerHTML = el.innerHTML.replace(/`([^`]+)`/g, '<code>$1</code>');
}
```

### 步骤 3: 重建有问题的卡片

由于 `update_note` 不可靠，需要删除并重新创建有问题的卡片：

```bash
# 删除有问题的卡片
curl -s localhost:8765 -X POST -d '{
  "action": "deleteNotes",
  "version": 6,
  "params": {"notes": [NOTE_ID1, NOTE_ID2, ...]}
}'

# 创建新卡片，Back 字段使用 HTML 实体
curl -s localhost:8765 -X POST -d @card.json
```

卡片内容示例（card.json）：

```json
{
  "action": "addNote",
  "version": 6,
  "params": {
    "note": {
      "deckName": "Linux::Regex",
      "modelName": "Quizify - 格致 (@chehil)",
      "fields": {
        "Front": "正则 `<div>` 匹配...",
        "Back": "**示例：**\n`&lt;div&gt;hello&lt;/div&gt;`"
      }
    }
  }
}
```

**注意**: Front 字段可以直接使用 `<` `>`（在反引号内），Back 字段必须使用 `&lt;` `&gt;`。

## 需要特别注意的正则模式

以下模式容易被 Anki HTML 解析破坏，必须用 HTML 实体存储：

| 正则模式 | 问题 | 存储方式 |
|---------|------|---------|
| `(?<=...)` | `<=` 可能被解析 | `(?&lt;=...)` |
| `(?<!...)` | `<!` 被解析为注释开始 | `(?&lt;!...)` |
| `</tag>` | 被解析为闭合标签 | `&lt;/tag&gt;` |
| `<div>` | 被解析为 HTML 标签 | `&lt;div&gt;` |
| `[!@#$%^&*]` | `&` 需要转义 | `[!@#$%^&amp;*]` |

## 完整模板参考

### 背面模板关键部分

```html
<script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
<script src="https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.9.0/build/highlight.min.js"></script>

<script type="text/javascript">
    // HTML 实体规范化函数
    function normalizeHtmlEntities(element) {
        element.textContent = element.textContent;
    }

    document.querySelectorAll('.quizify-field').forEach((el, index) => {
        const hasQuizifySyntax = /\{\{[^\}]+\}\}|\(\(.*?::.*?\)\)|\[\[.*?::.*?\]\]|\[[\s\S]+?\]::\([A-Z]+\)/.test(el.innerHTML);

        // 对不进行 Markdown 渲染的字段，处理内联代码
        if (hasQuizifySyntax || !el.classList.contains('quizify-field--back')) {
            el.innerHTML = el.innerHTML.replace(/`([^`]+)`/g, '<code>$1</code>');
        }

        // Quizify 语法处理...

        // Markdown 渲染
        if (el.classList.contains('quizify-field--back') && !hasQuizifySyntax) {
            var tempDiv = document.createElement('div');
            tempDiv.innerHTML = el.innerHTML.replace(/<br\s*\/?>/gi, '\n');
            var decodedContent = tempDiv.textContent;

            el.innerHTML = marked.parse(decodedContent);

            el.querySelectorAll('pre code').forEach((block) => {
                hljs.highlightElement(block);
            });

            el.querySelectorAll('code').forEach((code) => {
                if (!code.closest('pre')) {
                    normalizeHtmlEntities(code);
                }
            });
        } else {
            el.querySelectorAll('code').forEach(normalizeHtmlEntities);
        }
    });
</script>
```

## 验证清单

修复完成后，检查以下内容是否正确显示：

- [ ] 内联代码中的 `<div>` 显示为 `<div>`（不是 `&lt;div&gt;`）
- [ ] 代码块中的 HTML 标签正确显示并有语法高亮
- [ ] 正则表达式 `(?<=...)` 和 `(?<!...)` 正确显示
- [ ] `&` 字符正确显示（不是 `&amp;`）
- [ ] 卡片末尾没有多余的 `</div>` 等垃圾内容
- [ ] 背面显示的 Front 字段内联代码有正确样式

## 相关文件

- Quizify 模板: Anki → Tools → Manage Note Types → Quizify - 格致 → Cards...
- AnkiConnect API: http://localhost:8765

## 总结

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| `&lt;` 字面显示 | marked.js 二次转义 | 解码后再传给 marked.js |
| `(?<!` 变成 `(?<!--` | Anki HTML 解析 | 存储时用 `&lt;!` |
| 末尾多余 `</div>` | Anki 自动闭合标签 | 用 HTML 实体避免 |
| `update_note` 无效 | API 问题 | 删除后重新创建 |
