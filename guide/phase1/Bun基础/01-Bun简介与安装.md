# 01 - Bun 简介与安装

## 学习目标

读完本节后，你将能够：
1. 用自己的话说清楚 "Bun 是什么"
2. 在自己的电脑上安装 Bun
3. 运行第一个 Bun 程序
4. 理解为什么 Claude Code 选择用 Bun 开发

---

## 1.1 Bun 是什么？

想象你是一位厨师。以前做一道菜（运行一个程序），你需要：
- 先去市场买食材（安装依赖）
- 用菜刀切菜（编译代码）
- 用燃气灶炒菜（执行程序）
- 每一步都要换不同的工具，很慢

**Bun 就是一把"瑞士军刀"** —— 它把买食材、切菜、炒菜这三件事，都集成在了一把刀里。

具体来说：
- **JavaScript 运行时**：像 Node.js 一样，能执行 JavaScript/TypeScript 代码
- **包管理器**：像 npm/yarn/pnpm 一样，能安装第三方库
- **打包工具**：像 webpack/vite 一样，能把代码打包成可执行文件
- **测试运行器**：像 Jest 一样，能跑单元测试

> 💡 **一句话总结**：Bun = Node.js + npm + webpack + Jest，而且比它们加起来还快。

---

## 1.2 为什么 Claude Code 选择 Bun？

Claude Code 是一个复杂的 AI 命令行工具。它的源码里有几百个文件，需要：

| 需求 | Bun 如何满足 |
|------|-------------|
| 启动速度要快 | Bun 的启动速度是 Node.js 的 4 倍以上 |
| 内置 TypeScript 支持 | 直接运行 `.ts` 和 `.tsx`，不需要额外配置 |
| 打包成单个可执行文件 | `bun build --compile` 能生成独立的 `.exe` 或二进制文件 |
| 高性能哈希/字符串处理 | `Bun.hash()` 等 API 比纯 JavaScript 快几十倍 |
| 编译时条件代码（特性开关） | 通过自定义插件实现编译期树摇优化 |
| 调用系统底层功能（C 库） | `bun:ffi` 可以直接调用 C 语言写的系统库 |

在 Claude Code 的源码中，你会看到大量 `typeof Bun !== 'undefined'` 的判断——这意味着代码在 Bun 下会走"高速公路"，在 Node.js 下会走"普通公路"，但两条路都能走通。

---

## 1.3 安装 Bun

### Windows 安装

打开 PowerShell，粘贴以下命令：

```powershell
powershell -c "irm bun.sh/install.ps1 | iex"
```

安装完成后，关闭并重新打开终端，输入：

```bash
bun --version
```

如果看到版本号（例如 `1.2.0`），说明安装成功！

### macOS / Linux 安装

```bash
curl -fsSL https://bun.sh/install | bash
```

安装完成后，按照提示将 Bun 的路径添加到你的 `~/.bashrc` 或 `~/.zshrc` 中，然后重新打开终端，验证：

```bash
bun --version
```

---

## 1.4 你的第一个 Bun 程序

创建一个叫 `hello.ts` 的文件：

```typescript
// hello.ts
const name: string = "世界";
console.log(`你好，${name}！这是 Bun 在说话。`);
```

运行它：

```bash
bun hello.ts
```

你会看到输出：

```
你好，世界！这是 Bun 在说话。
```

> 🎉 **重要发现**：我们直接运行了 `.ts` 文件！在 Node.js 中，你需要先安装 `typescript` 包，再运行 `tsc` 编译成 `.js`，最后才能用 `node` 运行。Bun 直接帮你省去了这些步骤。

---

## 1.5 Bun 和 Node.js 的直观对比

为了让你更直观地感受差别，我们来做一个"速度比赛"。

### 比赛项目 1：Hello World 启动速度

创建 `speed-test.js`：

```javascript
console.log("Hello!");
```

分别计时：

```bash
# Node.js
time node speed-test.js

# Bun
time bun speed-test.js
```

你会发现 Bun 的 `real` 时间明显更短。

### 比赛项目 2：包安装速度

```bash
# 创建一个新目录测试
mkdir pkg-test && cd pkg-test

# npm 安装 lodash
npm install lodash

# 删除后，用 bun 安装
rm -rf node_modules package-lock.json
bun install lodash
```

Bun 的安装速度通常是 npm 的 10~30 倍。

---

## 1.6 Bun 的核心设计哲学

在后续学习中，你会发现 Bun 有以下几个设计特点：

1. **"如果能更快，就自己做"**：Bun 不依赖 V8（Node.js 用的引擎），而是基于 JavaScriptCore（Safari 浏览器使用的引擎，由 Apple 开发）。Bun 团队对 JSC 进行了深度优化，使其在 macOS、Linux 和 Windows 上都有出色的性能表现。

2. **"API 兼容性优先"**：Bun 尽量兼容 Node.js 的 API。你在 Node.js 里写的 `fs.readFileSync`、`path.join`、`child_process.spawn` 等代码，在 Bun 里基本不用改就能跑。

3. **"额外提供高性能捷径"**：除了兼容 Node.js，Bun 还提供了很多自己的高性能 API（如 `Bun.hash`、`Bun.write`、`Bun.spawn`）。这些 API 是"可选的增强"，不是"必须的替代"。

---

## 1.7 在 Claude Code 源码中的体现

打开 `src/main.tsx`，你会看到它是这样开始的：

```typescript
import { feature } from 'bun:bundle';
import { Command as CommanderCommand } from '@commander-js/extra-typings';
import chalk from 'chalk';
import { readFileSync } from 'fs';
```

注意：
- `bun:bundle` 是 Bun 命名空间模块（后面会详细学）
- `readFileSync` 是 Node.js 的标准 API，在 Bun 里完全兼容
- 整个项目用 TypeScript 写成，但不需要显式编译步骤

---

## 小结

| 概念 | 解释 |
|------|------|
| Bun | 集运行时、包管理器、打包器、测试器于一体的 JavaScript 工具 |
| 运行时 | 执行 JavaScript 代码的环境（类似浏览器里的 JS 引擎） |
| 包管理器 | 安装和管理第三方代码库的工具（类似 npm） |
| 树摇 (Tree Shaking) | 打包时删除未使用代码，让最终文件更小 |
| FFI | 让 JavaScript 直接调用 C/C++ 写的系统库 |

---

## 练习题

1. 在你的电脑上安装 Bun，并运行 `bun --version` 确认安装成功。
2. 创建一个 `intro.ts`，定义一个函数 `greet(name: string)`，打印 `"你好，XXX！"`，然后调用它。用 `bun intro.ts` 运行。
3. （思考题）为什么 Bun 能直接运行 TypeScript，而 Node.js 不能？

> 参考答案：Bun 内置了 TypeScript 编译器，在运行代码之前会自动把 TypeScript 转换成 JavaScript。Node.js 没有内置这个功能，需要用户手动安装 `typescript` 包并配置编译流程。
