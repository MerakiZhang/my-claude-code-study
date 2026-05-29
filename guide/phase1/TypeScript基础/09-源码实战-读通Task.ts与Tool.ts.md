# 第九章：源码实战——读通 Task.ts 与 Tool.ts

> 本章目标：用前面八章学到的知识，逐行读懂 Claude Code 中两个最核心的类型文件。

---

## 9.1 读源码的方法论

在开始之前，先记住这个**三步法**：

1. **先看 export 了什么**——了解这个文件对外暴露什么
2. **再看类型定义**——理解数据结构
3. **最后看函数实现**——理解逻辑

不要试图一次性读懂每一行注释，先抓骨架，再填血肉。

---

## 9.2 读通 Task.ts

打开 `src/Task.ts`，完整代码只有 **125 行**。我们逐块分析。

### 第一块：导入（第 1-4 行）

```ts
import { randomBytes } from 'crypto'
import type { AppState } from './state/AppState.js'
import type { AgentId } from './types/ids.js'
import { getTaskOutputPath } from './utils/task/diskOutput.js'
```

**知识点回顾**：
- `import { randomBytes } from 'crypto'` → 从 Node.js 内置模块导入**运行时值**（第四章）
- `import type { AppState }` → 只导入**类型**，编译后删除（第四章）
- `./state/AppState.js` → 注意 `.js` 后缀，这是 TypeScript ESM 规范（第四章）

### 第二块：TaskType（第 6-13 行）

```ts
export type TaskType =
  | 'local_bash'
  | 'local_agent'
  | 'remote_agent'
  | 'in_process_teammate'
  | 'local_workflow'
  | 'monitor_mcp'
  | 'dream'
```

**知识点回顾**：
- `export type` → 导出类型（第四章）
- `|` → 联合类型（第三章）
- `'local_bash'` 等 → 字符串字面量类型（第三章）

**含义**：Claude Code 有 7 种任务类型。一个任务只能是其中一种。

### 第三块：TaskStatus（第 15-21 行）

```ts
export type TaskStatus =
  | 'pending'
  | 'running'
  | 'completed'
  | 'failed'
  | 'killed'
```

同样是联合类型。任务生命周期：pending → running → completed/failed/killed。

### 第四块：isTerminalTaskStatus（第 27-29 行）

```ts
export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

**知识点回顾**：
- `status: TaskStatus` → 参数类型标注（第一章）
- `: boolean` → 返回类型标注（第一章）

**作用**：判断任务是否已结束。注意这里没有用自定义类型守卫 `status is ...`，因为返回 `boolean` 就够了，不需要进一步收窄类型。

### 第五块：TaskHandle（第 31-34 行）

```ts
export type TaskHandle = {
  taskId: string
  cleanup?: () => void
}
```

**知识点回顾**：
- `?` → 可选属性（第二章）
- `() => void` → 函数类型（第二章）

**含义**：一个任务句柄，包含任务 ID 和可选的清理函数。

### 第六块：SetAppState（第 36 行）

```ts
export type SetAppState = (f: (prev: AppState) => AppState) => void
```

**这是整章最难的一行**，拆解：

```ts
// 最外层：SetAppState 是一个函数
(f: (prev: AppState) => AppState) => void

// 参数 f 本身也是一个函数
(prev: AppState) => AppState

// f 接收 prev（AppState 类型），返回 AppState
```

**含义**：`SetAppState` 是一个函数，它接收一个"更新函数"`f`，`f` 接收当前状态 `prev`，返回新状态。这是 React 中 `setState` 的经典模式。

### 第七块：TaskContext（第 38-42 行）

```ts
export type TaskContext = {
  abortController: AbortController
  getAppState: () => AppState
  setAppState: SetAppState
}
```

- `abortController: AbortController` → 用于取消任务的 Web API
- `getAppState: () => AppState` → 一个无参函数，返回 AppState
- `setAppState: SetAppState` → 引用上面定义的函数类型

### 第八块：TaskStateBase（第 45-57 行）

```ts
export type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId?: string
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string
  outputOffset: number
  notified: boolean
}
```

**知识点回顾**：
- `type` 而不是 `interface` → Claude Code 偏爱 `type`（第二章）
- `?` 标记了 3 个可选属性：`toolUseId`、`endTime`、`totalPausedMs`

**含义**：所有任务状态共享的基础字段。具体任务类型会在此基础上扩展。

### 第九块：LocalShellSpawnInput（第 59-67 行）

```ts
export type LocalShellSpawnInput = {
  command: string
  description: string
  timeout?: number
  toolUseId?: string
  agentId?: AgentId
  /** UI display variant: description-as-label, dialog title, status bar pill. */
  kind?: 'bash' | 'monitor'
}
```

注意到 `kind?: 'bash' | 'monitor'`——联合类型和字面量类型的结合（第三章）。

### 第十块：Task（第 72-76 行）

```ts
export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

`kill` 是一个方法签名：
- 参数：`taskId`（string）、`setAppState`（SetAppState 类型）
- 返回：`Promise<void>` → 异步函数，完成后不返回数据（第六章）

### 第十一块：TASK_ID_PREFIXES（第 79-87 行）

```ts
const TASK_ID_PREFIXES: Record<string, string> = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}
```

**知识点回顾**：
- `Record<string, string>` → 内置工具类型（第七章）
- `const` 但没有 `as const` → 这里不需要最严格的字面量类型

### 第十二块：generateTaskId（第 98-106 行）

```ts
export function generateTaskId(type: TaskType): string {
  const prefix = getTaskIdPrefix(type)
  const bytes = randomBytes(8)
  let id = prefix
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % TASK_ID_ALPHABET.length]
  }
  return id
}
```

注意 `bytes[i]!` 中的 `!` ——这是 TypeScript 的**非空断言**（Non-null assertion），表示"我确定这里不是 `undefined`"。因为 `bytes` 有 8 个元素，`i < 8`，所以 `bytes[i]` 一定存在。

### 第十三块：createTaskStateBase（第 108-125 行）

```ts
export function createTaskStateBase(
  id: string,
  type: TaskType,
  description: string,
  toolUseId?: string,
): TaskStateBase {
  return {
    id,
    type,
    status: 'pending',
    description,
    toolUseId,
    startTime: Date.now(),
    outputFile: getTaskOutputPath(id),
    outputOffset: 0,
    notified: false,
  }
}
```

- `toolUseId?: string` → 函数的可选参数（第二章）
- 返回值类型 `: TaskStateBase` → 对象字面量的类型检查

---

## 9.3 读通 Tool.ts（前半部分）

`Tool.ts` 有 792 行，我们只读**类型定义部分**（前 340 行），这足以展示所有重要语法。

### 导入区（第 1-88 行）

```ts
import type {
  ToolResultBlockParam,
  ToolUseBlockParam,
} from '@anthropic-ai/sdk/resources/index.mjs'
```

**知识点**：花括号换行、多行导入——代码风格问题，不影响类型理解。

注意这一行：

```ts
import type { z } from 'zod/v4'
```

只导入 `zod` 的**类型命名空间**，用于 `z.infer<...>`（Schema 校验库）。

### ToolInputJSONSchema（第 15-21 行）

```ts
export type ToolInputJSONSchema = {
  [x: string]: unknown
  type: 'object'
  properties?: {
    [x: string]: unknown
  }
}
```

**新知识点**：`[x: string]: unknown` 是**索引签名**，表示"这个对象可以有任意多个字符串键，值类型是 `unknown`"。

同时它又有 `type: 'object'` 这个固定属性。所以这是一个："必须有 `type: 'object'`，还可以有其他任意属性"的对象。

### ValidationResult（第 95-101 行）

```ts
export type ValidationResult =
  | { result: true }
  | {
      result: false
      message: string
      errorCode: number
    }
```

**可辨识联合**（第三章）！用 `result: true` 和 `result: false` 作为标签。

### SetToolJSXFn（第 103-114 行）

```ts
export type SetToolJSXFn = (
  args: {
    jsx: React.ReactNode | null
    shouldHidePromptInput: boolean
    shouldContinueAnimation?: true
    showSpinner?: boolean
    // ...更多属性
  } | null,
) => void
```

拆解：
- `SetToolJSXFn` 是一个函数类型
- 参数 `args` 的类型是一个联合：`{ ... } | null`
- 返回 `void`

`React.ReactNode | null` 表示"React 节点或 null"。

### ToolPermissionContext（第 123-138 行）

```ts
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  // ...
}>
```

`DeepImmutable<T>` 是一个自定义泛型类型，表示"把 T 以及 T 的所有嵌套属性都变成只读"。这是递归泛型的高级用法，初学者只需知道它让对象不可变。

### ToolUseContext（第 158-300 行）

这是整个 Claude Code 中**最大的对象类型之一**。它描述了"工具执行时能获得的所有上下文"。

我们只挑几个重点：

```ts
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    tools: Tools
    // ...
  }
  abortController: AbortController
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  // ...
}
```

注意两种函数定义的写法：
- `getAppState(): AppState` → 方法简写（等价于 `getAppState: () => AppState`）
- `setAppState(f: ...): void` → 同上

### Tool 泛型类型（第 362-695 行）

```ts
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // ... 很多属性
}
```

**这是本项目最复杂的类型定义。** 拆解泛型参数：

- `Input extends AnyObject = AnyObject` → 输入 Schema 类型，约束必须是对象，默认任意对象
- `Output = unknown` → 输出类型，默认 unknown
- `P extends ToolProgressData = ToolProgressData` → 进度数据类型，约束必须是 ToolProgressData 子类型

这意味着每个工具都可以定制自己的输入、输出和进度类型：

```ts
// BashTool 可能这样用：
type BashTool = Tool<BashInputSchema, BashOutput, BashProgress>;
```

再看 `Tool` 内部的一些属性：

```ts
aliases?: string[]
searchHint?: string
call(args: z.infer<Input>, context: ToolUseContext, ...): Promise<ToolResult<Output>>
description(input: z.infer<Input>, options: {...}): Promise<string>
readonly inputSchema: Input
// ...
```

- `readonly inputSchema: Input` → 只读属性（第七章）
- `z.infer<Input>` → 从 Zod Schema 提取 TypeScript 类型（第七章讲过 `infer`）

`z.infer<Input>` 在 `Tool.ts` 中出现了十几次。它的作用是把 Zod 的 Schema 对象转换成普通的 TypeScript 类型。比如 `Input` 是一个 Zod Schema，`z.infer<Input>` 就是这个 Schema 校验通过后的数据类型。这样工具的 `call` 方法就能在编译时知道参数的结构，同时在运行时由 Zod 做数据校验。

### ToolDef 与 buildTool（第 721-792 行）

```ts
export type ToolDef<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = Omit<Tool<Input, Output, P>, DefaultableToolKeys> &
  Partial<Pick<Tool<Input, Output, P>, DefaultableToolKeys>>
```

**知识点大杂烩**：
- `Omit<T, K>` → 从 T 中去掉 K 属性（第七章）
- `Pick<T, K>` → 从 T 中挑选 K 属性（第七章）
- `Partial<T>` → 所有属性变可选（第七章）
- `&` → 交叉类型，把两个类型合并

**含义**：`ToolDef` 是 `Tool` 的"宽松版"——某些属性可以省略，因为 `buildTool` 函数会填充默认值。

```ts
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

- `D extends AnyToolDef` → 泛型约束（第五章）
- `...TOOL_DEFAULTS, ...def` → 展开运算符，把默认值和用户定义合并
- `as BuiltTool<D>` → 类型断言（告诉 TypeScript"相信我，返回值的类型是对的"）

---

## 9.4 本章总结

读完这两个文件，你应该已经看到了：

| 知识点 | 出现位置 |
|--------|---------|
| 基础类型 (`string`, `number`, `boolean`) | `TaskStateBase` |
| 联合类型 (`\|`) | `TaskType`, `TaskStatus`, `ValidationResult` |
| 字面量类型 | `'pending'`, `'object'` |
| 可选属性 (`?`) | `endTime?`, `cleanup?` |
| 函数类型 | `SetAppState`, `SetToolJSXFn` |
| 泛型 | `Tool<Input, Output, P>` |
| 泛型约束 | `Input extends AnyObject` |
| 内置工具类型 | `Omit`, `Pick`, `Partial`, `Record` |
| 类型导入 | `import type { AppState }` |
| 类型导出 | `export type TaskType` |
| 类型断言 | `as BuiltTool<D>` |
| 索引签名 | `[x: string]: unknown` |
| 交叉类型 | `A & B` |
| `readonly` | `readonly inputSchema` |

---

## 练习题

1. 在 `TaskStateBase` 中，为什么 `startTime` 没有 `?` 而 `endTime` 有？

2. `SetAppState` 的类型中，为什么参数 `f` 要接收 `prev` 而不是直接接收新状态？

3. `Tool` 类型为什么要设计三个泛型参数 `Input`、`Output`、`P`？如果只用一个泛型会有什么缺点？

---

> 下一章：源码实战——追踪 `/clear` 命令的完整执行链路。
