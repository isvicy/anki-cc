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

## 推荐使用的卡片类型（优先级排序）

### 1. Better Markdown : Basic
- **特点**：支持 Markdown 语法、代码高亮、LaTeX 公式
- **适用场景**：普通问答题、概念解释、代码示例
- **字段**：Front, Back, Extra, Difficulty

### 2. Quizify - 格致 (@chehil)
- **特点**：支持填空题和选择题
- **适用场景**：需要互动的学习内容
- **字段**：Front, Back, Add Reverse
- **格式要求**：
  - 填空题：使用 `{{内容}}`
  - 选择题：Front 必须以 `[` 开头，以 `]::(答案)` 结尾

## Quizify 选择题格式错误总结

### 常见错误
1. **错误**：选择题 Front 字段没有以 `[` 开头
   - ❌ 错误：`当 io.Reader 的 Read 方法返回...`
   - ✅ 正确：`[当 io.Reader 的 Read 方法返回...]::(B)`

2. **错误**：选择题 Front 字段没有以 `]::(答案)` 结尾
   - ❌ 错误：`[问题内容<br>A. 选项A<br>B. 选项B`
   - ✅ 正确：`[问题内容<br>A. 选项A<br>B. 选项B]::(A)`

## 精华卡片样例

### Better Markdown : Basic - 带代码示例

**Front:**
```
Go 语言中的 io.Reader 接口定义是什么？
```

**Back:**
```markdown
```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

**解释：**
- Read 方法从数据源读取数据到字节切片 p 中
- 返回读取的字节数 n 和可能的错误 err
- 当读取到数据末尾时，返回 `io.EOF` 错误
```

### Better Markdown : Basic - 不带代码示例

**Front:**
```
什么是 Reader 接口模式？
```

**Back:**
```markdown
## Reader 接口模式

Reader 接口模式是 Go 语言中的一种重要设计模式，它提供了一种统一的方式来读取数据。

### 主要特点：

1. **统一接口**：无论数据来源于文件、网络、内存还是其他地方，都使用相同的 Read 方法
2. **组合性**：Reader 可以被嵌套和组合，如 bufio.Reader、io.TeeReader 等
3. **流式处理**：支持处理大文件或无限流，无需一次性加载到内存

### 常见应用场景：

- 文件读取
- 网络通信
- 数据转换和处理
- 测试和模拟
```

### Quizify - 填空题示例

**Front:**
```
bufio.NewReader 创建一个带缓冲的 Reader，默认缓冲区大小是 {{4096}} 字节。使用 {{ReadString}} 方法可以读取到指定的分隔符，使用 {{ReadLine}} 方法可以读取一行数据。
```

**Back:**
```
（留空）
```

### Quizify - 选择题示例（单选）

**Front:**
```
[当 io.Reader 的 Read 方法返回 n > 0 且 err != nil 时，下列哪种处理方式是正确的？<br>A. 立即返回错误，忽略读取的数据<br>B. 先处理读取到的 n 个字节的数据，然后再处理错误<br>C. 认为这是无效状态，panic<br>D. 重试读取操作]::(B)
```

**Back:**
```
((选项 A::不正确。即使有错误，已读取的数据仍然是有效的，不应该忽略。))<br>((选项 B::✓ 正确！根据 io.Reader 的约定，当 n > 0 时，应该先处理这 n 个字节的有效数据，然后再处理错误。这是因为错误可能发生在读取部分数据之后。))<br>((选项 C::不正确。这是 io.Reader 的正常行为模式，不应该 panic。))<br>((选项 D::不正确。不应该自动重试，应该先处理已读取的数据。))
```

### Quizify - 选择题示例（多选）

**Front:**
```
[以下哪些类型实现了 io.Reader 接口？<br>A. *os.File<br>B. *bytes.Buffer<br>C. *strings.Reader<br>D. *bufio.Scanner]::(ABC)
```

**Back:**
```
((选项 A::✓ 正确！*os.File 实现了 Read 方法，可以从文件读取数据。))<br>((选项 B::✓ 正确！*bytes.Buffer 实现了 Read 方法，可以从内存缓冲区读取数据。))<br>((选项 C::✓ 正确！*strings.Reader 实现了 Read 方法，可以从字符串读取数据。))<br>((选项 D::✗ 不正确。*bufio.Scanner 不是 Reader，它是基于 Reader 的更高级抽象，用于按行或按 token 扫描。))
```
