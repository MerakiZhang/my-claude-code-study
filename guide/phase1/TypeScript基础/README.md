# Claude Code TypeScript 基础教程

> 面向零基础学生，系统讲解 Claude Code 源码中所使用的 TypeScript 语法概念。

---

## 教程定位

本教程不是通用的 TypeScript 入门书，而是**面向 Claude Code 源码的定向语法讲解**。每一章的知识点都直接对应源码中的真实写法，确保你学完就能开始读代码。

**前置要求**：
- 了解 JavaScript 基础语法（变量、函数、对象、数组）
- 如果完全没接触过 JS，建议先花 2-3 小时看一遍 JS 速成教程

---

## 章节导航

| 章节 | 标题 | 核心内容 | 对应源码 |
|------|------|---------|---------|
| 第1章 | [TypeScript 入门与基础类型](./01-TypeScript入门与基础类型.md) | string/number/boolean/数组/对象/void/unknown/null/never | `Task.ts`, `treeify.ts` |
| 第2章 | [函数、接口与类型别名](./02-函数、接口与类型别名.md) | 函数类型、可选参数、rest参数、type vs interface | `Task.ts`, `thinking.ts` |
| 第3章 | [联合类型、字面量类型与类型收窄](./03-联合类型、字面量类型与类型收窄.md) | `\|` 联合、`is` 类型守卫、`in` 收窄 | `permissions.ts`, `treeify.ts` |
| 第4章 | [模块系统 import 与 export](./04-模块系统-import与-export.md) | 命名导出、type-only import、重导出、动态导入、混合导出 | `Tool.ts`, `settings/types.ts` |
| 第5章 | [泛型基础与实战](./05-泛型基础与实战.md) | `<T>`、泛型约束 `extends`、默认值 | `array.ts`, `permissions.ts` |
| 第6章 | [异步编程 Promise 与 async/await](./06-异步编程-Promise与async-await.md) | Promise、async/await、`??`、`?.` | `cli.tsx`, `permissions.ts` |
| 第7章 | [高级类型工具与内置工具类型](./07-高级类型工具与内置工具类型.md) | `keyof`、`typeof`、`infer`、`as const`、Record/Partial/Omit | `permissions.ts`, `thinking.ts` |
| 第8章 | [TSX 与 React 类型基础](./08-TSX与React类型基础.md) | Props 类型、Hooks 类型、类组件、React.ReactNode | `Spinner.tsx`, `cli.tsx` |
| 第9章 | [源码实战——读通 Task.ts 与 Tool.ts](./09-源码实战-读通Task.ts与Tool.ts.md) | 综合运用前八章知识逐行精读 | `Task.ts`, `Tool.ts` |
| 第10章 | [源码实战——追踪一条完整链路](./10-源码实战-追踪一条完整链路.md) | 以 `/clear` 命令追踪执行流程 | `commands/clear/` |

---

## 学习建议

### 顺序

建议**按顺序阅读**，因为后面的章节会用到前面的概念。如果时间有限，可以跳读，但遇到不懂的概念要及时回查。

### 实践

每读完一章，建议做以下事情：

1. **完成练习题**——每章末尾都有 3 道练习题，用来检验理解程度
2. **对照源码**——打开 Claude Code 的源码，找到教程中引用的代码片段，亲自看一遍上下文
3. **动手改代码**——尝试修改某个类型定义，看 TypeScript 会在哪些地方报错（这是最好的学习方式）

### 推荐的学习节奏

| 天数 | 内容 | 预计时间 |
|------|------|---------|
| Day 1 | 第1-2章 | 1-2 小时 |
| Day 2 | 第3-4章 | 1-2 小时 |
| Day 3 | 第5-6章 | 1-2 小时 |
| Day 4 | 第7-8章 | 1-2 小时 |
| Day 5 | 第9章 + 对照源码精读 | 2-3 小时 |
| Day 6 | 第10章 + 自己追踪另一条链路 | 2-3 小时 |

---

## 源码版本

本教程引用的代码均来自 **Claude Code v2.1.88** 源码仓库。

---

## 常见问题

**Q: 学完这十章，我能完全看懂 Claude Code 源码吗？**

A: 不能。这十**章只覆盖了 TypeScript 语法**。Claude Code 还涉及：React/Ink 终端 UI、Node.js 子进程、网络请求、Git 操作、LLM API 调用等。但 TypeScript 语法是你读懂源码的第一块基石。

**Q: 我需要先学 React 吗？**

A: 第8章会讲 TSX 的基础，但只是"能看懂"的程度。如果你想深入理解 UI 组件，建议额外学习 React Hooks。

**Q: 为什么教程里全是 `type` 而不是 `interface`？**

A: 因为 Claude Code 源码中 `type` 占绝对主导。我们教的是"怎么读这个项目"，而不是"TypeScript 的所有写法"。

---

祝你学习愉快！有任何问题，随时回来看对应章节。
