# 05 - Bun 全局 API：哈希与字符串处理

## 学习目标

读完本节后，你将能够：
1. 使用 `Bun.hash()` 计算高性能哈希值
2. 使用 `Bun.stringWidth()` 计算字符串在终端中的显示宽度
3. 使用 `Bun.wrapAnsi()` 处理 ANSI 彩色文本的换行
4. 理解为什么这些 API 比 JavaScript 纯实现更快
5. 读懂 Claude Code 中哈希和终端渲染相关源码

---

## 5.1 `Bun.hash()` —— 高性能哈希函数

### 什么是哈希？

想象你有一个巨大的图书馆，里面有几十万本书。如果你想快速找到某本书，不能一本一本地翻。**哈希**就像是给每本书一个独一无二的"身份证号"——不管书名多长，都能变成一串固定长度的数字。

在编程中，哈希函数把任意长度的输入（字符串、文件内容等）转换成固定长度的数字。相同输入永远得到相同输出，不同输入大概率得到不同输出。

### 为什么要用 `Bun.hash()`？

JavaScript 本身没有内置的哈希函数（除非你用 Node.js 的 `crypto` 模块）。`Bun.hash()` 是 Bun 提供的一个**超高速哈希函数**，基于一种叫 **wyhash** 的算法。

| 对比项 | `Bun.hash()` | Node.js `crypto.createHash()` |
|--------|-------------|------------------------------|
| 速度 | 极快（每秒数 GB） | 较快（每秒数百 MB） |
| 是否为加密安全 | ❌ 不是（不适合密码） | ✅ `sha256` 是 |
| 用法 | 一行代码 | 需要创建流、更新、消化 |
| 返回值 | `number` | `Buffer`（二进制数据） |

> ⚠️ **重要**：`Bun.hash()` 不是加密安全的哈希！不要用它在密码、签名、证书等安全场景。它适合缓存键、数据去重、快速比较等场景。

### 基本用法

```typescript
// bun-hash-basic.ts

// 计算单个字符串的哈希
const hash1 = Bun.hash("Hello, World!");
console.log(`"Hello, World!" 的哈希值: ${hash1}`);

// 相同输入得到相同输出
const hash2 = Bun.hash("Hello, World!");
console.log(`再次计算: ${hash2}`);
console.log(`两次是否相同: ${hash1 === hash2}`);

// 不同输入得到不同输出（大概率）
const hash3 = Bun.hash("Hello, Bun!");
console.log(`"Hello, Bun!" 的哈希值: ${hash3}`);
```

运行：

```bash
bun bun-hash-basic.ts
```

### 高级用法：链式哈希

`Bun.hash()` 支持第二个参数作为"种子"（seed），这允许你做链式哈希：

```typescript
// bun-hash-chain.ts

function hashPair(a: string, b: string): string {
  // 先算 a 的哈希，再用结果作为种子算 b 的哈希
  const combined = Bun.hash(b, Bun.hash(a));
  return combined.toString();
}

console.log(hashPair("Alice", "Bob"));
console.log(hashPair("Bob", "Alice")); // 不同！顺序会影响结果
```

### 转成不同进制的字符串

`Bun.hash()` 返回的是 `number`（准确说是 `bigint` 或 `number`，取决于平台）。通常需要转成字符串使用：

```typescript
// bun-hash-formats.ts

const hash = Bun.hash("example");

console.log(`默认数字: ${hash}`);
console.log(`十进制字符串: ${hash.toString()}`);
console.log(`十六进制: ${hash.toString(16)}`);
console.log(`三十六进制: ${hash.toString(36)}`); // 更短，适合做文件名
```

---

## 5.2 Claude Code 中的 `Bun.hash()` 用法

### 用法 1：内容哈希（缓存键）

`src/utils/hash.ts`：

```typescript
export function hashContent(content: string): string {
  if (typeof Bun !== 'undefined') {
    return Bun.hash(content).toString();
  }
  // Node.js 回退...
}
```

**场景**：Claude Code 需要缓存很多中间结果（如文件内容、API 响应等）。用哈希值作为缓存键，比用原始内容更短、更高效。

### 用法 2：会话存储哈希

`src/utils/sessionStoragePortable.ts`：

```typescript
const hash = typeof Bun !== 'undefined' ? Bun.hash(name).toString(36) : simpleHash(name);
```

注意这里用了 `toString(36)`。为什么用 36 进制？

- 36 进制使用 `0-9` + `a-z`，可以表示更大的数用更少的字符
- 例如：十进制的 `1000000` 在 36 进制中是 `"lfls"`（4 个字符 vs 7 个字符）
- 适合做文件名、URL 路径等长度敏感的场景

### 用法 3：提示词缓存破坏检测

`src/services/api/promptCacheBreakDetection.ts`：

```typescript
const hash = Bun.hash(str);
```

Claude Code 会计算提示词（prompt）的哈希，检测是否发生了变化。如果变化了，就刷新缓存。

### 用法 4：随机数生成（Buddy 功能）

`src/buddy/companion.ts`：

```typescript
return Number(BigInt(Bun.hash(s)) & 0xffffffffn);
```

这里用哈希值生成伪随机数，用于"Buddy"（Claude Code 的 mascot 动画）的行为决策。

> 💡 **注意 `0xffffffffn`**：这是 32 位全 1 的二进制数（`4294967295`）。`&` 是按位与操作，作用是把哈希值截取到 32 位范围内，确保结果是一个有效的 32 位整数。

### 动手实验：模拟 Claude Code 的缓存键生成

```typescript
// cache-key-demo.ts

function generateCacheKey(input: string): string {
  if (typeof Bun !== 'undefined') {
    // Claude Code 风格：用 36 进制缩短长度
    return Bun.hash(input).toString(36);
  }

  // Node.js 回退：用简单的字符码求和
  let sum = 0;
  for (let i = 0; i < input.length; i++) {
    sum = (sum * 31 + input.charCodeAt(i)) >>> 0;
  }
  return sum.toString(36);
}

// 测试
const inputs = [
  "src/utils/hash.ts",
  "src/main.tsx",
  "这是一个很长的文件路径/src/components/PromptInput/PromptInputFooterLeftSide.tsx",
];

for (const input of inputs) {
  const key = generateCacheKey(input);
  console.log(`原始长度: ${input.length}, 缓存键: ${key}, 键长度: ${key.length}`);
}
```

---

## 5.3 `Bun.stringWidth()` —— 计算终端显示宽度

### 为什么要专门计算字符串宽度？

在终端（命令行窗口）中显示文字时，不是所有字符都占一样的宽度：

- 普通英文字母：`A`、`B`、`1` → 占 **1** 列
- 中文汉字：`你`、`好` → 占 **2** 列
- 表情符号：`👍`、`🎉` → 占 **2** 列（有时更多）
- ANSI 颜色代码：`\x1b[31m`（红色）→ 占 **0** 列（控制字符不显示）

如果你只是用 `str.length` 计算长度，会得到错误的结果：

```typescript
const str = "你好";
console.log(str.length); // 输出 2
// 但在终端中实际占 4 列！
```

### `Bun.stringWidth()` 的基本用法

```typescript
// string-width-demo.ts

const examples = [
  "Hello",
  "你好",
  "👍 Great!",
  "\x1b[31mRed Text\x1b[0m", // ANSI 红色文本
  "日本語テスト",
];

for (const str of examples) {
  const jsLength = str.length;
  const bunWidth = Bun.stringWidth(str);
  console.log(`文本: "${str}" | JS length: ${jsLength} | 终端宽度: ${bunWidth}`);
}
```

运行后你会发现，`Bun.stringWidth()` 能正确处理：
- 中文字符占 2 列
- ANSI 颜色代码不占列
- Emoji 占 2 列

### Claude Code 中的应用

`src/ink/stringWidth.ts`：

```typescript
const bunStringWidth = typeof Bun !== 'undefined' && typeof Bun.stringWidth === 'function'
  ? Bun.stringWidth
  : null;
```

Claude Code 使用 Ink（一个基于 React 的命令行 UI 库）来渲染终端界面。在终端 UI 中，精确计算每个字符的宽度至关重要：
- 文本框的对齐
- 表格的边框绘制
- 输入光标的定位
- 长文本的截断显示

这段代码的优化技巧也值得学习：
- 在**模块顶部**（模块作用域）就判断 `typeof Bun.stringWidth === 'function'`
- 把结果缓存到 `bunStringWidth` 变量中
- 后续调用直接使用缓存的变量，避免每次调用都做 `typeof` 检查

> 💡 **为什么这是热路径优化？** Claude Code 的终端 UI 每帧可能需要计算成千上万个字符的宽度。如果每次都用 `typeof` 检查，累积的开销会很可观。提前解析一次可以节省大量 CPU 时间。

---

## 5.4 `Bun.wrapAnsi()` —— ANSI 文本智能换行

### 什么是 ANSI 转义序列？

ANSI 转义序列是一组特殊的字符代码，用来控制终端的显示效果：

```typescript
console.log("\x1b[31m这是红色\x1b[0m");
console.log("\x1b[1m这是粗体\x1b[0m");
console.log("\x1b[32m\x1b[4m绿色下划线\x1b[0m");
```

这些 `\x1b[...` 是"控制字符"，不会被显示出来，但会影响后续文本的样式。

### 换行的难题

当你在终端中显示一段彩色文字，并且需要换行时，问题来了：

```typescript
const coloredText = "\x1b[31m这是一段很长的红色文字，需要在特定宽度处换行\x1b[0m";
```

如果用普通的字符串分割，ANSI 控制字符会被切断，导致：
- 第二行变成了红色（因为关闭颜色的 `\x1b[0m` 被切到了第二行末尾）
- 或者颜色泄漏到后续内容

### `Bun.wrapAnsi()` 的用法

```typescript
// wrap-ansi-demo.ts

const longText = "\x1b[31m这是一段很长的红色文字，我们需要在 10 个字符的宽度处把它折断成多行，同时保持每行都有正确的颜色\x1b[0m";

// 普通换行（会破坏 ANSI 颜色）
console.log("=== 普通分割（错误示范） ===");
for (let i = 0; i < longText.length; i += 10) {
  console.log(longText.slice(i, i + 10));
}

console.log("\n=== Bun.wrapAnsi（正确示范） ===");
const wrapped = Bun.wrapAnsi(longText, 10);
console.log(wrapped);
```

运行后你会发现，`Bun.wrapAnsi()` 会在正确的位置换行，同时保持 ANSI 颜色代码的完整性。

### Claude Code 中的应用

`src/ink/wrapAnsi.ts`：

```typescript
const wrapAnsiBun = typeof Bun !== 'undefined' && typeof Bun.wrapAnsi === 'function'
  ? Bun.wrapAnsi
  : null;
```

Claude Code 的终端界面需要正确渲染：
- 带有语法高亮的代码块
- 带颜色的错误信息
- Markdown 渲染后的富文本

所有这些场景都需要精确处理 ANSI 颜色代码的换行。

---

## 5.5 性能对比实验

让我们实际感受一下 Bun 这些 API 的速度优势。

```typescript
// performance-comparison.ts

function benchmark(name: string, fn: () => void, iterations: number): void {
  const start = performance.now();
  for (let i = 0; i < iterations; i++) {
    fn();
  }
  const end = performance.now();
  console.log(`${name}: ${(end - start).toFixed(2)}ms (${iterations} 次)`);
}

const testString = "The quick brown fox jumps over the lazy dog. ";
const longString = testString.repeat(1000);

console.log("=== 哈希性能对比 ===");
if (typeof Bun !== 'undefined') {
  benchmark("Bun.hash()", () => Bun.hash(longString), 10000);
}

import { createHash } from 'crypto';
benchmark("crypto.createHash('sha256')", () => {
  createHash('sha256').update(longString).digest('hex');
}, 10000);

console.log("\n=== 字符串宽度计算对比 ===");
const wideString = "你好，世界！Hello World! 👍🎉";

if (typeof Bun !== 'undefined') {
  benchmark("Bun.stringWidth()", () => Bun.stringWidth(wideString), 100000);
}

// 纯 JS 实现（简化版）
function simpleStringWidth(str: string): number {
  let width = 0;
  for (const char of str) {
    const code = char.charCodeAt(0);
    width += (code >= 0x4e00 && code <= 0x9fff) ? 2 : 1;
  }
  return width;
}
benchmark("简单 JS 实现", () => simpleStringWidth(wideString), 100000);
```

运行这个文件，你会看到 Bun 的 API 通常比纯 JavaScript 或 Node.js 替代方案快数倍到数十倍。

---

## 小结

| API | 用途 | 返回值 | Claude Code 中的使用场景 |
|-----|------|--------|------------------------|
| `Bun.hash(input, seed?)` | 高性能非加密哈希 | `number` / `bigint` | 缓存键生成、内容去重、伪随机数 |
| `Bun.stringWidth(str)` | 计算终端显示宽度 | `number` | 终端 UI 对齐、光标定位、文本截断 |
| `Bun.wrapAnsi(str, width)` | ANSI 文本智能换行 | `string` | 彩色文本的自动换行 |

| 设计模式 | 说明 |
|---------|------|
| 模块级缓存 | 在文件顶部判断 `typeof Bun.xxx`，避免每次调用都检查 |
| 36 进制编码 | `toString(36)` 可以缩短哈希字符串长度 |
| 种子链式哈希 | `Bun.hash(b, Bun.hash(a))` 用于组合多个值的哈希 |
| 按位截断 | `& 0xffffffffn` 把大数限制在 32 位整数范围内 |

---

## 练习题

1. 编写 `content-deduplicate.ts`，读取一个字符串数组，用 `Bun.hash()` 找出重复的内容。
2. 编写 `terminal-table.ts`，用 `Bun.stringWidth()` 实现一个简单的终端表格对齐工具。输入几个中英文混合的字符串，确保它们在终端中对齐。
3. 编写 `ansi-paragraph.ts`，创建一段带有 ANSI 颜色的长文本，用 `Bun.wrapAnsi()` 在 30 列宽度处换行，观察颜色是否正确保持。
4. （思考题）`src/utils/cachePaths.ts` 的注释说 "使用 djb2Hash 而非 Bun.hash，以保证缓存目录名在运行时升级后保持稳定"。为什么使用不同的哈希算法会影响缓存目录名的稳定性？

> 参考答案：不同的哈希算法可能产生不同的输出。如果升级 Bun 后 `Bun.hash()` 的内部实现发生变化（例如 wyhash 版本升级），同样的输入会得到不同的哈希值。这意味着缓存键会全部失效，之前的缓存数据就找不到了。djb2Hash 是一个纯 JavaScript 实现的确定性算法，不会因为 Bun 版本升级而改变行为，所以更适合做长期缓存的目录名。
