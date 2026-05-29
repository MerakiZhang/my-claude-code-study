# 04 - Bun 内置模块 bun命名空间bundle 与编译时特性标记

## 学习目标

读完本节后，你将能够：
1. 理解 `bun:` 命名空间模块的概念
2. 理解什么是编译时特性标记（Feature Flags）
3. 理解什么是"树摇优化"（Dead Code Elimination）
4. 读懂 Claude Code 源码中 `feature('XXX')` 的用法
5. 明白为什么 `feature()` 只能在 `if` 或三元表达式中调用

---

## 4.1 什么是 `bun:` 命名空间？

在 Bun 中，有一类特殊的模块，它们以 `bun:` 开头。这就像是图书馆里的"特藏区"——只有这个图书馆（Bun 运行时）才有这些书。

```typescript
import { something } from 'bun:xxx';
```

Bun 官方提供的 `bun:` 模块包括：

| 模块名 | 用途 |
|--------|------|
| `bun:ffi` | 调用 C/C++ 写的系统库（ Foreign Function Interface ） |
| `bun:sqlite` | 内置的高性能 SQLite 数据库 |
| `bun:test` | 内置测试框架 |
| `bun:jsc` | 直接操作 JavaScriptCore 引擎 |

这些模块的特点是：
1. **不需要安装**：它们是 Bun 内置的，不在 `node_modules` 里
2. **速度极快**：通常用 Zig 或 C++ 实现，比纯 JavaScript 实现快得多
3. **Bun 专属**：在 Node.js 中运行会报错（模块找不到）

### 动手实验：尝试导入 `bun:` 模块

创建 `bun-modules.ts`：

```typescript
// bun-modules.ts

// 测试 SQLite 模块
import { Database } from 'bun:sqlite';

const db = new Database(':memory:'); // 内存数据库

// 创建表
db.run(`CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)`);

// 插入数据
db.run(`INSERT INTO users (name) VALUES ('Alice'), ('Bob')`);

// 查询
const users = db.query(`SELECT * FROM users`).all();
console.log("用户列表:", users);

// 关闭
db.close();
```

运行：

```bash
bun bun-modules.ts
```

> 💡 **零基础提示**：`:memory:` 表示数据库只在内存中存在，程序结束后数据就消失了。这对于测试和学习非常有用。

---

## 4.2 Claude Code 的 `bun:bundle` 是什么？

在 Claude Code 的源码中，你会看到这样一个导入：

```typescript
import { feature } from 'bun:bundle';
```

**注意**：`bun:bundle`**不是 Bun 官方的标准模块**。它是 Claude Code 开发团队利用 Bun 的**插件系统**自己创建的一个**虚拟模块**。

### 为什么要自己造一个模块？

想象 Claude Code 是一个大型软件，里面有上百个功能：SSH 远程连接、语音模式、浏览器工具、WebView、MCP 协议……不是所有用户都需要所有功能，也不是所有功能都同时开发完成。

开发团队需要一种机制来：
1. **控制哪些代码被编译进最终产品**（减少体积）
2. **在编译期就删除未启用的功能代码**（提高安全性，防止未发布代码泄露）
3. **让同一份源码支持不同的产品形态**（CLI 版、桌面版、企业版等）

这就是 `bun:bundle` 的作用——它是一个**编译时特性开关系统**。

### 它是如何工作的？

在 Bun 的构建配置中（你可能在一个叫 `build.ts` 或类似的地方找到），开发团队注册了一个 Bun 插件：

```typescript
// 这是 Claude Code 构建系统的大致逻辑（非源码原文）
import { plugin } from 'bun';

plugin({
  name: 'bundle-feature-flags',
  setup(build) {
    // 当代码中 import 'bun:bundle' 时，Bun 不会真的去文件系统找这个模块
    // 而是调用这个回调，动态生成模块内容
    build.onResolve({ filter: /^bun:bundle$/ }, () => ({
      path: 'bun:bundle',
      namespace: 'bun',
    }));

    build.onLoad({ filter: /.*/, namespace: 'bun' }, () => {
      // 从环境变量或配置文件中读取特性列表
      const features = ['DIRECT_CONNECT', 'KAIROS', 'LODESTONE', 'SSH_REMOTE', 'WEB_BROWSER_TOOL'];

      // 生成一个模块，导出一个 feature 函数
      return {
        contents: `
          const ENABLED_FEATURES = new Set(${JSON.stringify(features)});
          export function feature(name) {
            return ENABLED_FEATURES.has(name);
          }
        `,
        loader: 'js',
      };
    });
  },
});
```

> 📝 注意：上面的代码是**为了教学目的简化的示例**，不是 Claude Code 的真实构建代码。真实的实现可能更复杂，可能从 JSON 配置文件或环境变量中读取特性列表。

---

## 4.3 特性标记（Feature Flags）的使用

### 基本用法

在 Claude Code 源码中，`feature()` 的使用方式非常统一：

```typescript
import { feature } from 'bun:bundle';

if (feature('DIRECT_CONNECT')) {
  // 只有启用了 DIRECT_CONNECT 功能，这段代码才会被编译进最终产物
  console.log("直接连接功能已启用");
}
```

### 实际源码中的例子

**例子 1：`src/main.tsx` 中的 SSH 远程功能**

```typescript
if (feature('SSH_REMOTE') && _pendingSSH) {
  const rawCliArgs = process.argv.slice(2);
  // ... SSH 相关的参数解析逻辑
}
```

如果编译时 `SSH_REMOTE` 特性被关闭，整个 `if` 块里的代码都会被删除。

**例子 2：`src/main.tsx` 中的 WebView 检测**

```typescript
const hint = feature('WEB_BROWSER_TOOL') && typeof Bun !== 'undefined' && 'WebView' in Bun
  ? CLAUDE_IN_CHROME_SKILL_HINT_WITH_WEBBROWSER
  : CLAUDE_IN_CHROME_SKILL_HINT;
```

这段代码同时检查了三个条件：
1. 编译时是否启用了 `WEB_BROWSER_TOOL` 特性
2. 运行时是否在 Bun 环境中
3. 运行时 Bun 是否支持 `WebView`

> 💡 **`'WebView' in Bun` 是什么意思？** 这是 JavaScript 的 `in` 运算符，用来检查一个对象是否拥有某个属性。这里检查全局的 `Bun` 对象是否有 `WebView` 属性。这是一种**运行时特性检测**——即使 Bun 版本相同，不同构建配置或平台可能支持不同的内置功能。通过 `in` 运算符检测，代码可以在不支持 `WebView` 的环境中优雅降级。

**例子 3：`src/query.ts` 中的注释说明**

```typescript
// feature() only works in if/ternary conditions (bun:bundle
// tree-shaking constraint), so the collapse check is nested
// rather than composed.
```

这段注释非常重要！它告诉我们一个关键限制：

> ⚠️ **`feature()` 只能在 `if` 语句或三元表达式（`?:`）中使用，才能被正确树摇。**

---

## 4.4 什么是"树摇优化"（Tree Shaking / Dead Code Elimination）？

这是现代 JavaScript 打包工具的核心能力之一。

### 生活中的类比

想象你要搬家：
- **没有树摇**：把所有东西（包括三年没穿过的衣服、坏掉的电器）全部打包，卡车很重，搬家很慢
- **有树摇**：提前检查哪些东西真的需要，把不需要的扔掉，卡车轻了，搬家快了

### 代码层面的类比

```typescript
// 原始代码
function featureA() { /* 100 行代码 */ }
function featureB() { /* 200 行代码 */ }
function featureC() { /* 150 行代码 */ }

function main() {
  featureA();
  // featureB 和 featureC 从来没被调用
}
```

经过树摇优化后，最终产物只包含：

```typescript
function featureA() { /* 100 行代码 */ }

function main() {
  featureA();
}
```

`featureB` 和 `featureC` 被"摇掉了"，因为没有任何代码引用它们。

### `feature()` 如何参与树摇？

这是关键所在。如果 `feature('X')` 在编译期就能确定返回 `false`，打包工具就会把整个分支当作"死代码"删除：

```typescript
// 编译前
import { feature } from 'bun:bundle';

if (feature('EXPERIMENTAL_MODE')) {
  // 假设编译时这个特性被关闭
  console.log("实验模式");
  enableExperimentalStuff();
  loadExperimentalModule();
}

console.log("正常模式");
```

打包工具在编译时会"执行"`feature('EXPERIMENTAL_MODE')`，发现它返回 `false`，于是把 `if` 块整个删除：

```typescript
// 编译后（概念上）
console.log("正常模式");
```

### 为什么 `feature()` 只能在 `if` / 三元表达式中使用？

因为只有这样，打包工具才能**静态分析**出哪些代码是"死代码"。

❌ **错误用法**（无法树摇）：

```typescript
const hasFeature = feature('SSH_REMOTE'); // 赋值给变量

function check() {
  if (hasFeature) { // 打包工具不知道 hasFeature 的值
    // ... 这段代码不会被删除
  }
}
```

✅ **正确用法**（可以树摇）：

```typescript
function check() {
  if (feature('SSH_REMOTE')) { // 直接在 if 中调用
    // ... 如果特性关闭，这段代码会被删除
  }
}
```

❌ **错误用法**（无法树摇）：

```typescript
const modules = [
  feature('A') ? import('./a.ts') : null,
  feature('B') ? import('./b.ts') : null,
];
```

✅ **正确用法**（可以树摇）：

```typescript
if (feature('A')) {
  const m = await import('./a.ts');
}
if (feature('B')) {
  const m = await import('./b.ts');
}
```

---

## 4.5 动态导入（Dynamic Import）与特性标记

在 Claude Code 中，你会看到大量这样的代码：

```typescript
if (feature('COORDINATOR_MODE')) {
  const coordinatorModeModule = require('./coordinator/coordinatorMode.js') as typeof import('./coordinator/coordinatorMode.js');
}
```

以及：

```typescript
if (feature('TRANSCRIPT_CLASSIFIER')) {
  const autoModeStateModule = require('./utils/permissions/autoModeState.js') as typeof import('./utils/permissions/autoModeState.js');
}
```

这里使用了 `require()` 而不是 `import` 语句。为什么？

### `require()` vs `import()`

| 写法 | 加载时机 | 能否条件加载 | 树摇 |
|------|---------|-------------|------|
| `import { x } from './a'` | 模块加载时（顶部） | ❌ 不能 | ✅ 能 |
| `const m = require('./a')` | 运行时（执行到这行时） | ✅ 能 | ❌ 不能（自身） |
| `const m = await import('./a')` | 运行时 | ✅ 能 | ✅ 能（配合 if） |

在 Claude Code 中：
- 顶部的 `import` 语句用于核心依赖（React、 Commander 等）
- `require()` 用于可选模块，配合 `feature()` 做条件加载
- `await import()` 用于异步条件加载（在 `async` 函数中）

> 💡 **设计意图**：`require()` 是同步的，可以在非 async 函数中使用。它配合 `feature()` 时，如果特性被关闭，整个 `require()` 调用都不会被执行，对应的模块文件也不会被打包进产物。

---

## 4.6 在 Claude Code 源码中的全貌

根据我们的统计，`import { feature } from 'bun:bundle'` 出现在 **150+ 个文件**中。以下是按模块分类的概览：

| 模块 | 涉及的特性示例 |
|------|--------------|
| 核心功能 | `DIRECT_CONNECT`, `KAIROS`, `SSH_REMOTE`, `LODESTONE` |
| 工具系统 | `WEB_BROWSER_TOOL`, `COMPUTER_USE_TOOL` |
| 权限与策略 | `TRANSCRIPT_CLASSIFIER` |
| 遥测与分析 | `UPLOAD_USER_SETTINGS` |
| UI 组件 | 各种新功能对应的界面开关 |

这意味着：同一份 Claude Code 源码，可以通过调整特性列表，编译出不同版本的产品：
- **标准 CLI 版**：关闭 `KAIROS`、`SSH_REMOTE` 等高级功能
- **完整功能版**：启用所有特性
- **企业定制版**：只启用企业需要的特性

---

## 小结

| 概念 | 解释 |
|------|------|
| `bun:` 命名空间 | Bun 内置模块的专属前缀，如 `bun:ffi`、`bun:sqlite` |
| `bun:bundle` | Claude Code 自定义的编译时虚拟模块，用于特性标记 |
| Feature Flag（特性标记） | 编译时开关，控制哪些代码被包含在最终产物中 |
| Tree Shaking（树摇） | 打包时自动删除未被使用的代码 |
| Dead Code Elimination | 同上，另一个叫法 |
| `feature()` 的使用限制 | 只能在 `if` 语句或三元表达式中调用，才能被正确树摇 |
| `require()` | 同步动态加载，常用于配合 `feature()` 做条件模块加载 |
| `await import()` | 异步动态加载，同样支持条件加载 |

---

## 练习题

1. 创建一个 `feature-demo.ts`（不需要真的实现 `bun:bundle`，用普通变量模拟）：
   ```typescript
   const ENABLED_FEATURES = new Set(['DARK_MODE', 'NOTIFICATIONS']);
   function feature(name: string): boolean {
     return ENABLED_FEATURES.has(name);
   }

   if (feature('DARK_MODE')) {
     console.log("暗黑模式已启用");
   }
   if (feature('VOICE_MODE')) {
     console.log("语音模式已启用");
   }
   ```
   运行后观察输出，理解为什么 "语音模式" 没有打印。

2. （思考题）为什么 `feature()` 返回布尔值，而不是返回一个配置对象？如果返回配置对象，会有什么问题？

> 参考答案：如果 `feature()` 返回对象，打包工具无法静态确定哪些属性被使用，就无法进行有效的树摇。返回布尔值可以让打包工具在编译期确定 `if` 分支的走向，从而删除整个死代码分支。

3. （思考题）在 Claude Code 源码中，为什么有些地方用 `require()` 而有些地方用 `import`？

> 参考答案：`import` 是静态的，在模块加载时就执行，适合核心依赖；`require()` 是动态的，在代码执行到该语句时才加载，适合配合 `feature()` 做条件加载。如果特性被关闭，`require()` 不会执行，对应的模块不会被加载也不会被打包。
