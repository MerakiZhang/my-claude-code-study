# Claude Code JavaScript 基础教程

> 面向完全零基础学生，用最通俗的语言讲解 Claude Code 源码中所使用的 JavaScript 语法概念。

---

## 教程定位

本教程不是通用的 JavaScript 入门书，而是**面向 Claude Code 源码的定向语法讲解**。每一章的知识点都直接对应源码中的真实写法，确保你学完就能开始读代码。

**前置要求**：
- **完全零基础，不需要任何前置知识**
- 你只需要会基本的电脑操作（打字、打开文件、运行程序）
- 不需要懂 TypeScript、不需要懂 React、不需要懂任何其他编程语言

---

## 与 TypeScript 教程的对应关系

本项目同时提供 JavaScript 基础教程和 TypeScript 进阶教程。两者的对应关系如下：

| JS 章节 | TS 章节 | 核心内容 |
|---------|---------|---------|
| [第1章：JavaScript 入门与基础语法](./01-JavaScript入门与基础语法.md) | 第1章：TypeScript 入门与基础类型 | 变量、数据类型、数组、对象、运算符 |
| [第2章：函数与对象](./02-函数与对象.md) | 第2章：函数、接口与类型别名 | 函数三种写法、参数、对象进阶、解构、展开 |
| [第3章：条件判断与类型检查](./03-条件判断与类型检查.md) | 第3章：联合类型、字面量类型与类型收窄 | `if`/`switch`、`typeof`、`in`、严格相等、短路求值 |
| [第4章：模块系统](./04-模块系统.md) | 第4章：模块系统 import 与 export | ESM 导入导出、默认导出、重导出、动态导入 |
| [第5章：数组与对象的高级操作](./05-数组与对象的高级操作.md) | 第5章：泛型基础与实战 | 高阶函数、`map`/`filter`/`reduce`、`Object` 静态方法 |
| [第6章：异步编程 Promise 与 async/await](./06-异步编程-Promise与async-await.md) | 第6章：异步编程 Promise 与 async/await | Promise、`async`/`await`、`??`、`?.` |
| [第7章：JavaScript 内置对象与工具方法](./07-JavaScript内置对象与工具方法.md) | 第7章：高级类型工具与内置工具类型 | `Map`/`Set`、`Date`、`JSON`、`Error`、`Math` |
| [第8章：类与面向对象基础](./08-类与面向对象基础.md) | 第8章：TSX 与 React 类型基础 | `class`、`constructor`、`extends`、JSX 简介 |
| [第9章：源码实战——读通 Task.ts 与 Tool.ts](./09-源码实战-读通Task.ts与Tool.ts.md) | 第9章：源码实战——读通 Task.ts 与 Tool.ts | 综合运用前八章知识逐行精读 |
| [第10章：源码实战——追踪一条完整链路](./10-源码实战-追踪一条完整链路.md) | 第10章：源码实战——追踪一条完整链路 | 以 `/clear` 命令追踪执行流程 |

> **学习路径建议**：先读完本教程的 JavaScript 基础章节，再进入 TypeScript 教程。因为 TypeScript 本质上就是"带类型注释的 JavaScript"，先学会 JS 再学 TS 会事半功倍。

---

## 章节导航

| 章节 | 标题 | 核心内容 | 对应源码 |
|------|------|---------|---------|
| 第1章 | [JavaScript 入门与基础语法](./01-JavaScript入门与基础语法.md) | 变量、string/number/boolean/undefined/null、数组、对象、运算符 | `Task.ts`, `treeify.ts` |
| 第2章 | [函数与对象](./02-函数与对象.md) | 函数声明/表达式/箭头函数、默认参数/剩余参数、解构赋值、展开运算符 | `Task.ts`, `osc.ts`, `treeify.ts` |
| 第3章 | [条件判断与类型检查](./03-条件判断与类型检查.md) | `if`/`else`、`switch`、`typeof`、`in`、三元运算符、`===` vs `==`、短路求值 | `permissions.ts`, `treeify.ts` |
| 第4章 | [模块系统](./04-模块系统.md) | `export`/`import`、默认导出、通配符导入、重导出、动态导入 | `Tool.ts`, `settings/types.ts` |
| 第5章 | [数组与对象的高级操作](./05-数组与对象的高级操作.md) | `forEach`/`map`/`filter`/`reduce`/`find`、`Set` 去重、`Object.keys`/`values`/`entries` | `array.ts`, `permissions.ts` |
| 第6章 | [异步编程 Promise 与 async/await](./06-异步编程-Promise与async-await.md) | Promise、async/await、try/catch、`??`、`?.` | `cli.tsx`, `permissions.ts` |
| 第7章 | [JavaScript 内置对象与工具方法](./07-JavaScript内置对象与工具方法.md) | `Map`/`Set`、`Date`、`JSON`、`Error`、正则、`Math`、`console` | `permissions.ts`, `Task.ts` |
| 第8章 | [类与面向对象基础](./08-类与面向对象基础.md) | `class`、`constructor`、继承 `extends`、`super`、getter/setter、static、私有字段 | `bridgeApi.ts`, `mcp/client.ts` |
| 第9章 | [源码实战——读通 Task.ts 与 Tool.ts](./09-源码实战-读通Task.ts与Tool.ts.md) | 综合运用前八章知识逐行精读核心类型文件 | `Task.ts`, `Tool.ts` |
| 第10章 | [源码实战——追踪一条完整链路](./10-源码实战-追踪一条完整链路.md) | 以 `/clear` 命令追踪完整执行流程 | `commands/clear/` |

---

## 学习建议

### 顺序

建议**按顺序阅读**，因为后面的章节会用到前面的概念。如果时间有限，可以跳读，但遇到不懂的概念要及时回查。

### 实践

编程不是"看"出来的，是"练"出来的。每读完一章，建议做以下事情：

1. **完成练习题**——每章末尾都有 3 道练习题，用来检验理解程度
2. **动手敲代码**——把教程中的代码示例自己敲一遍（不要复制粘贴），你会发现很多"看起来懂"其实"动手就错"的地方
3. **对照源码**——打开 Claude Code 的源码，找到教程中引用的代码片段，亲自看一遍上下文

### 推荐的学习节奏

| 天数 | 内容 | 预计时间 |
|------|------|---------|
| Day 1 | 第1-2章 | 2-3 小时 |
| Day 2 | 第3-4章 | 2-3 小时 |
| Day 3 | 第5-6章 | 2-3 小时 |
| Day 4 | 第7-8章 | 2-3 小时 |
| Day 5 | 第9章 + 对照源码精读 | 2-3 小时 |
| Day 6 | 第10章 + 自己追踪另一条链路 | 2-3 小时 |

---

## 什么是 JavaScript？一句话解释

JavaScript 是一种**让电脑按照你的指令做事**的语言。

你可以把它想象成训练一只非常听话的狗狗：你发出"坐下"的指令，狗狗就坐下；你发出"握手"的指令，狗狗就握手。JavaScript 就是你和电脑之间的"指令语言"——你写代码（发指令），电脑执行（做动作）。

Claude Code 这个项目，就是用 JavaScript（准确地说是 TypeScript，一种 JavaScript 的扩展）写出来的。

---

## 源码版本

本教程引用的代码均来自 **Claude Code v2.1.88** 源码仓库。

---

## 常见问题

**Q: 我完全没学过编程，能看懂吗？**

A: 能。本教程假设你是第一次接触编程，所有概念都会用生活中的比喻来解释。

**Q: 为什么先学 JavaScript 而不是直接学 TypeScript？**

A: 因为 TypeScript = JavaScript + 类型系统。如果你不懂 JavaScript，直接学 TypeScript 就像还没学会走路就想学跑步。先学会 JavaScript 的基础语法，再学 TypeScript 的类型注释，会顺畅很多。

**Q: 学完这十章，我能完全看懂 Claude Code 源码吗？**

A: 不能。这十章只覆盖了 **JavaScript 语法**。Claude Code 还涉及：TypeScript 类型系统、React/Ink 终端 UI、Node.js 子进程、网络请求、Git 操作、LLM API 调用等。但 JavaScript 语法是你读懂源码的第一块基石。

**Q: 我需要安装什么软件？**

A: 只需要安装 **Node.js**（JavaScript 的运行环境）。安装后，你就可以在终端中运行 `node 文件名.js` 来执行 JavaScript 代码了。

---

祝你学习愉快！有任何问题，随时回来看对应章节。
