# 第一章：TypeScript 入门与基础类型

> 本章目标：理解 TypeScript 是什么，以及 Claude Code 中最常用的基础类型。

---

## 1.1 为什么用 TypeScript？

**JavaScript** 是动态类型语言，你可以这样写：

```js
let name = "Claude";
name = 123; // JavaScript 不会报错，但逻辑已经错了
```

这种错误在大型项目中很难发现。**TypeScript** 在 JavaScript 基础上增加了**类型检查**，让错误在写代码时就被发现，而不是运行时才崩溃。

Claude Code 有十几万行代码、几十个模块，如果没有 TypeScript 的类型约束，维护起来会非常困难。

---

## 1.2 你的第一个 TypeScript 代码

TypeScript 文件以 `.ts` 结尾（React 组件用 `.tsx`）。

```ts
// 声明一个变量，并指定它的类型为 string
let userName: string = "Claude";

// 如果你尝试赋值数字，TypeScript 会报错：
// userName = 123; // ❌ 错误：Type 'number' is not assignable to type 'string'
```

在 Claude Code 的源码中，你到处都能看到这种类型声明。比如 `src/utils/theme.ts` 里的颜色名称就是 `string` 类型。

---

## 1.3 基础类型（Primitive Types）

Claude Code 源码中大量使用以下基础类型：

### string（字符串）

```ts
// 来自 src/Task.ts
export type TaskType =
  | 'local_bash'
  | 'local_agent'
  | 'remote_agent'
  | 'in_process_teammate'
  | 'local_workflow'
  | 'monitor_mcp'
  | 'dream'
```

这里 `'local_bash'` 等是**字符串字面量类型**（下一章会讲），但它们的基础都是 `string`。

再看 `src/utils/permissions/PermissionMode.ts`：

```ts
function permissionModeTitle(mode: PermissionMode): string {
  return getModeConfig(mode).title
}
```

`: string` 表示"这个函数**返回**一个字符串"。

### number（数字）

```ts
// 来自 src/Task.ts
export type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  startTime: number        // ← 时间戳是 number
  endTime?: number         // ← ? 表示可选，后面会讲
  totalPausedMs?: number   // ← 毫秒数也是 number
}
```

`number` 在 Claude Code 中常用于：时间戳、token 数量、坐标、持续时间等。

### boolean（布尔值）

```ts
// 来自 src/utils/permissions/PermissionMode.ts
export function isDefaultMode(mode: PermissionMode | undefined): boolean {
  return mode === 'default' || mode === undefined
}
```

`: boolean` 表示函数返回 `true` 或 `false`。

### undefined 和 null

```ts
// endTime?: number 等价于 endTime: number | undefined
```

`?` 是 TypeScript 的简写语法，表示"这个属性可以不存在"。后面第三章会详细讲。

---

## 1.4 数组类型（Array）

在 TypeScript 中有两种写法：

```ts
// 写法一：类型[]
let names: string[] = ["Claude", "GPT", "Gemini"];

// 写法二：Array<类型>（泛型写法）
let ids: Array<string> = ["a1", "b2", "c3"];
```

在 Claude Code 中，两种写法都有使用。看 `src/utils/treeify.ts`：

```ts
const lines: string[] = []
```

再看 `src/utils/thinking.ts`：

```ts
const RAINBOW_COLORS: Array<keyof Theme> = [
  'rainbow_red',
  'rainbow_orange',
  'rainbow_yellow',
  // ...
]
```

这里 `Array<keyof Theme>` 是高级用法，第七章会讲。现在你只需要知道：方括号 `[]` 表示数组。

---

## 1.5 对象类型（Object Type）

TypeScript 中定义对象结构最常见的两种方式：

### 方式一：type（类型别名）

```ts
// 来自 src/Task.ts
export type TaskHandle = {
  taskId: string
  cleanup?: () => void
}
```

这表示：一个 `TaskHandle` 对象必须包含 `taskId`（字符串），可选地包含 `cleanup`（一个函数）。

### 方式二：interface（接口）

```ts
// 假设的例子（Claude Code 中 interface 用得相对少，但很重要）
interface User {
  name: string;
  age: number;
}
```

`type` 和 `interface` 非常相似，第二章会详细对比。

---

## 1.6 函数类型

TypeScript 可以给函数参数和返回值加类型：

```ts
// 来自 src/Task.ts
export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

拆解：
- `status: TaskStatus` → 参数 `status` 的类型是 `TaskStatus`
- `: boolean` → 函数返回值类型是 `boolean`

再看一个更复杂的，来自 `src/utils/array.ts`：

```ts
export function count<T>(arr: readonly T[], pred: (x: T) => unknown): number {
  let n = 0
  for (const x of arr) n += +!!pred(x)
  return n
}
```

- `arr: readonly T[]` → 参数 `arr` 是一个只读数组，元素类型为 `T`（泛型，第五章讲）
- `pred: (x: T) => unknown` → 参数 `pred` 是一个函数，接收一个 `T` 类型的参数，返回 `unknown`
- `: number` → 函数返回数字

---

## 1.7 any 与 unknown

### any（尽量避免）

`any` 表示"任何类型"，TypeScript 不会对 `any` 做类型检查：

```ts
let data: any = "hello";
data = 123;      // 不报错
data.toFixed();  // 不报错（但运行时会崩溃，因为字符串没有 toFixed）
```

Claude Code 源码中很少用 `any`，因为它会破坏类型安全。

### unknown（安全版的 any）

```ts
let data: unknown = "hello";
data = 123;      // 可以赋值
data.toFixed();  // ❌ 报错！unknown 不允许直接调用方法
```

使用 `unknown` 时，你必须先**类型收窄**（第三章讲）才能操作它。

在 Claude Code 中，你会看到 `unknown` 用于不确定的数据：

```ts
// 来自 src/types/permissions.ts
| {
    type: 'permissionPromptTool'
    permissionPromptToolName: string
    toolResult: unknown  // ← 外部工具返回的结果，类型不确定
  }
```

---

## 1.8 null 与 undefined

`null` 和 `undefined` 都表示"空"，但含义略有不同：

```ts
let user: string | null = null;  // 明确表示"还没有用户"
let name: string | undefined;     // 可能不存在
```

在 Claude Code 中，两者都有使用。`undefined` 常用于可选属性，`null` 常用于需要明确表达"空值"的场景。

---

## 1.9 never

`never` 是一种特殊的类型，表示"**不可能存在的值**"。最常见的用法是：函数永远不正常返回（总是抛出异常或退出进程）。

```ts
// 来自 src/cli/exit.ts
export function cliError(msg?: string): never {
  // ... 打印错误并退出进程
  process.exit(1);
}
```

`: never` 告诉 TypeScript："调用这个函数后，后面的代码不会执行。"

初学者遇到 `never` 不多，看到时知道它是"永远不会返回"的意思即可。

---

## 1.10 void

`void` 表示"没有返回值"，常用于函数：

```ts
// 来自 src/Task.ts
export type SetAppState = (f: (prev: AppState) => AppState) => void
```

这个函数类型表示：接收一个函数作为参数，返回 `void`（什么都不返回）。

---

## 1.9 本章小结

| 类型 | 含义 | Claude Code 中的例子 |
|------|------|---------------------|
| `string` | 字符串 | 任务描述、ID、颜色名 |
| `number` | 数字 | 时间戳、token数、坐标 |
| `boolean` | 真/假 | `isTerminalTaskStatus` 返回值 |
| `T[]` / `Array<T>` | 数组 | `lines: string[]` |
| `{ ... }` | 对象结构 | `TaskHandle`、`TaskStateBase` |
| `any` | 任意类型（避免使用） | 很少出现 |
| `unknown` | 未知类型（安全的any） | `toolResult: unknown` |
| `void` | 无返回值 | `setAppState` 的返回类型 |
| `null` | 空值 | 明确表示"没有" |
| `never` | 永不返回 | `cliError(): never` |
| `?` | 可选属性 | `endTime?: number` |

---

## 练习题

1. 下面代码有什么问题？
   ```ts
   let taskId: string = 123;
   ```

2. 为下面的对象写类型定义：
   ```ts
   const user = {
     name: "Claude",
     age: 3,
     isAdmin: true
   };
   ```

3. `endTime?: number` 和 `endTime: number \| undefined` 有什么区别？（提示：几乎没有区别，`?` 是简写）

---

> 下一章：函数、接口与类型别名 —— 我们会深入理解 `type` 和 `interface` 的区别，以及 Claude Code 为什么更偏爱 `type`。
