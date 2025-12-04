## intro

这是一个专门用来生成 anki flashcard 的目录。在生成 anki flashcard
的时候，请遵循以下约定：

- 每一个问题的答案除了答案本身之外，还需要有对应的解释，以帮助理解答案的来源和背景；这样一个卡片本身是自解释的；
- 每一个问题的答案中，如果有代码示例，请使用代码块进行包裹；

## 创建流程

**重要：在执行实际创建之前，必须先展示待创建的卡片设计内容，获得用户同意后再执行创建。**

## 笔记模板选择

### 推荐：KaTeX and Markdown Basic (Color)

这是支持代码高亮的模板，内置了 highlight.js 和 markdown-it。

**使用 Markdown 语法编写内容：**

```markdown
## 标题

这是正文内容。

```bash
# 这是代码块，会自动语法高亮
echo "hello"
```

**解释：**
- `行内代码` 使用反引号
- 列表项
```

### 不推荐：KaTeX and Markdown Basic（无 Color 后缀）

这个模板不支持代码高亮，需要使用 HTML 格式编写（`<h2>`, `<pre><code>` 等），且代码没有语法高亮。

## 特殊字符转义

### $ 符号必须转义

KaTeX and Markdown Basic (Color) 模板使用 `$...$` 作为 LaTeX 数学公式定界符。

如果内容中包含 `$` 字符（如 shell 变量 `$!`, `$?`, `$0` 等），**必须转义为 `\$`**，否则会被错误解析为数学公式。

**错误示例：**
```
`$!` 表示最后一个后台进程的 PID
```
→ 会被 KaTeX 解析为数学公式，渲染出乱码

**正确示例：**
```
`\$!` 表示最后一个后台进程的 PID
```

### 代码块中的 $ 也需要转义

```bash
SERVER_PID=\$!
echo \$HOME
```

## 常见问题

### Q: 代码没有语法高亮？
A: 检查是否使用了 **KaTeX and Markdown Basic (Color)** 模板（带 Color 后缀）。不带 Color 后缀的模板没有内置 highlight.js。

### Q: 内容显示为 HTML 标签或 MathML？
A:
1. 检查是否使用了正确的语法格式（Color 模板用 Markdown，非 Color 模板用 HTML）
2. 检查是否有未转义的 `$` 符号

### Q: 如何查看/编辑笔记模板？
A: Anki → Tools → Manage Note Types → 选择模板 → Cards...
