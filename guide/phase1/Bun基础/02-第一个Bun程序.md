# 02 - 第一个 Bun 程序：TypeScript、包管理与脚本

## 学习目标

读完本节后，你将能够：
1. 理解 Bun 内置的 TypeScript 支持是如何工作的
2. 用 `bun init` 创建一个新项目
3. 用 Bun 安装和管理依赖包
4. 运行测试和自定义脚本
5. 理解 `tsconfig.json` 的基本配置

---

## 2.1 Bun 的 TypeScript 支持 —— "即写即跑"

在上一节中，你直接运行了 `.ts` 文件。这是 Bun 最吸引人的特性之一。

### 回顾 Node.js 的工作流程

如果你是零基础，不知道 Node.js 的历史也没关系。但为了让你理解 Bun 的便利，我们简单看一下传统流程：

```
写 TypeScript 代码 (hello.ts)
    ↓
安装 typescript 包 (npm install -D typescript)
    ↓
配置 tsconfig.json
    ↓
运行编译器 (npx tsc)
    ↓
生成 JavaScript 代码 (hello.js)
    ↓
运行生成的代码 (node hello.js)
```

**Bun 把这个流程压缩成了一步：**

```
写 TypeScript 代码 (hello.ts)
    ↓
直接运行 (bun hello.ts) ← Bun 在内存中自动编译并执行
```

### 动手实验：体验即写即跑

创建一个稍微复杂一点的 TypeScript 文件 `math-utils.ts`：

```typescript
// math-utils.ts

// 定义一个接口（Interface）—— TypeScript 的类型系统
interface Point {
  x: number;
  y: number;
}

// 带类型标注的函数
function distance(a: Point, b: Point): number {
  const dx = a.x - b.x;
  const dy = a.y - b.y;
  return Math.sqrt(dx * dx + dy * dy);
}

// 泛型函数 —— TypeScript 的高级特性
function identity<T>(arg: T): T {
  return arg;
}

// 运行
const p1: Point = { x: 0, y: 0 };
const p2: Point = { x: 3, y: 4 };

console.log(`两点间距离: ${distance(p1, p2)}`); // 输出 5
console.log(`identity 测试: ${identity("Bun 真好用")}`);
```

直接运行：

```bash
bun math-utils.ts
```

你看到输出了吗？**完全不需要配置任何东西。**

---

## 2.2 用 `bun init` 创建正式项目

虽然可以直接运行单个 `.ts` 文件，但真实项目通常有多个文件、依赖库和配置。Bun 提供了 `bun init` 来快速初始化项目。

### 步骤

```bash
# 创建一个新目录
mkdir my-bun-project
cd my-bun-project

# 初始化项目
bun init
```

你会看到一系列提示，按回车使用默认值即可。最终生成的文件包括：

```
my-bun-project/
├── bunfig.toml       # Bun 的配置文件（类似 .npmrc）
├── package.json      # 项目信息和依赖列表
├── tsconfig.json     # TypeScript 编译配置
├── README.md         # 项目说明
└── index.ts          # 入口文件
```

### package.json 的结构

打开生成的 `package.json`：

```json
{
  "name": "my-bun-project",
  "module": "index.ts",
  "type": "module",
  "devDependencies": {
    "@types/bun": "latest"
  },
  "peerDependencies": {
    "typescript": "^5.0.0"
  }
}
```

关键字段解释：

| 字段 | 含义 |
|------|------|
| `"type": "module"` | 使用 ES Module 格式（`import` 语法），而不是旧的 CommonJS（`require` 语法） |
| `"module": "index.ts"` | 项目的入口文件 |
| `devDependencies` | 开发时需要的依赖（不会随项目发布） |
| `@types/bun` | Bun 的类型定义文件，让 TypeScript 认识 `Bun` 这个全局对象 |

### tsconfig.json 的结构

```json
{
  "compilerOptions": {
    "lib": ["ES2020"],
    "module": "esnext",
    "target": "esnext",
    "moduleResolution": "bundler",
    "strict": true
  }
}
```

> 💡 **零基础提示**：`tsconfig.json` 是 TypeScript 的"说明书"，告诉编译器如何理解你的代码。Bun 会自动读取这个文件。`"strict": true` 表示启用所有类型检查，帮你尽早发现错误。

---

## 2.3 安装依赖包

假设你的项目需要处理颜色（就像 Claude Code 用 `chalk` 一样）。

### 安装生产依赖

```bash
bun install chalk
```

这会把 `chalk` 安装到 `node_modules` 中，并在 `package.json` 的 `dependencies` 中记录。

### 使用安装的包

修改 `index.ts`：

```typescript
import chalk from 'chalk';

console.log(chalk.green('成功！'));
console.log(chalk.red('错误！'));
console.log(chalk.blue.bold('重要信息'));
```

运行：

```bash
bun index.ts
```

### 安装开发依赖

开发依赖是你写代码时需要，但用户运行你的程序时不需要的包。例如类型定义：

```bash
bun install -d @types/node
```

`-d` 是 `--dev` 的缩写。

### 同时安装多个包

```bash
bun install lodash commander zod
```

Bun 会并行下载，速度极快。

### 卸载包

```bash
bun remove lodash
```

---

## 2.4 运行脚本（Scripts）

在 `package.json` 中，你可以定义脚本命令：

```json
{
  "scripts": {
    "start": "bun index.ts",
    "dev": "bun --watch index.ts",
    "build": "bun build index.ts --outdir ./dist"
  }
}
```

然后用以下命令运行：

```bash
bun run start    # 运行项目
bun run dev      # 开发模式（文件变化自动重启）
bun run build    # 打包
```

> 💡 **技巧**：`bun run` 比 `npm run` 快得多，因为它不需要先解析大量的 npm 内部逻辑。

### `--watch` 模式

开发时最常用的命令：

```bash
bun --watch index.ts
```

这会让 Bun 监视你的文件。一旦保存修改，Bun 会自动重新运行程序。**这是开发命令行工具的利器。**

---

## 2.5 Bun 的内置测试运行器

Bun 自带了测试框架，语法和 Jest 很像。

创建 `math.test.ts`：

```typescript
import { describe, test, expect } from "bun:test";

function add(a: number, b: number): number {
  return a + b;
}

describe("数学函数", () => {
  test("1 + 1 = 2", () => {
    expect(add(1, 1)).toBe(2);
  });

  test("负数相加", () => {
    expect(add(-5, 3)).toBe(-2);
  });
});
```

运行测试：

```bash
bun test
```

> 📝 注意：虽然 Claude Code 源码中没有直接使用 `bun:test`，但这是 Bun 生态的重要部分，值得了解。

---

## 2.6 模块导入的两种写法

在 JavaScript/TypeScript 中，导入模块有两种主要方式：

### ES Module（现代标准）

```typescript
import { readFileSync } from 'fs';
import chalk from 'chalk';
import type { Point } from './types.js';
```

这是目前推荐的方式。Claude Code 的源码全部使用这种方式。

### CommonJS（旧标准）

```typescript
const fs = require('fs');
const chalk = require('chalk');
```

在 Claude Code 的源码中，你偶尔会看到 `require`：

```typescript
// src/main.tsx
const getTeammateUtils = () => require('./utils/teammate.js') as typeof import('./utils/teammate.js');
```

这是**懒加载（Lazy Loading）**的技巧——在需要时才加载模块，避免循环依赖或加速启动。

> 💡 **零基础提示**：`as typeof import('./utils/teammate.js')` 是 TypeScript 的类型断言，告诉编译器 "这个 `require` 返回的东西，类型和 `import` 一样"。这样 TypeScript 就能正确提供代码提示了。

---

## 2.7 在 Claude Code 源码中的体现

### 包管理

Claude Code 的 `package.json`（虽然这个学习仓库已清理）原本包含大量依赖，如：

```json
{
  "dependencies": {
    "@commander-js/extra-typings": "...",
    "chalk": "...",
    "react": "...",
    "lodash-es": "..."
  }
}
```

### 入口文件

`src/main.tsx` 是 CLI 的入口。它导入了大量模块：

```typescript
import { feature } from 'bun:bundle';
import { Command as CommanderCommand } from '@commander-js/extra-typings';
import chalk from 'chalk';
import { readFileSync } from 'fs';
import mapValues from 'lodash-es/mapValues.js';
import React from 'react';
```

注意这些导入来源：
- `bun:bundle` —— Bun 的内置/虚拟模块
- `@commander-js/extra-typings` —— npm 包
- `fs` —— Node.js 内置模块（Bun 兼容）
- `lodash-es/mapValues.js` —— 直接导入子路径

---

## 小结

| 命令/概念 | 作用 |
|-----------|------|
| `bun init` | 初始化新项目 |
| `bun install <pkg>` | 安装依赖 |
| `bun install -d <pkg>` | 安装开发依赖 |
| `bun remove <pkg>` | 卸载依赖 |
| `bun run <script>` | 运行 package.json 中定义的脚本 |
| `bun --watch <file>` | 监视模式开发 |
| `bun test` | 运行测试 |
| `import` / `require` | 导入模块的两种方式 |
| `@types/bun` | Bun 的类型定义包 |

---

## 练习题

1. 用 `bun init` 创建一个新项目，然后安装 `zod` 库（一个用于数据校验的流行库）。
2. 在项目中创建一个 `greeter.ts`，导出一个函数 `greet(name: string): string`，返回 `"你好，${name}"`。在 `index.ts` 中导入并使用它。
3. 给 `greeter.ts` 写一个简单的测试文件 `greeter.test.ts`，测试 `greet("Bun")` 的返回值。
4. （思考题）为什么 `bun install` 比 `npm install` 快？

> 参考答案：Bun 使用 Zig 语言编写了高效的包管理器核心，支持并行下载、全局缓存、符号链接优化等。npm 基于 JavaScript 编写，架构较老，并发能力有限。
