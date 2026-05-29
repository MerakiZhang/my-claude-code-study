# 第四章：模块系统 import 与 export

> 本章目标：彻底搞懂 Claude Code 源码中各种 `import` 和 `export` 的写法，包括 `type` 导入、重导出、动态导入等。

---

## 4.1 为什么需要模块系统？

Claude Code 有上千个文件。如果没有模块系统，所有代码都写在一个文件里，根本无法维护。

**模块系统**让你可以把代码拆分成多个文件，每个文件只暴露需要暴露的内容，其他内容对外隐藏。

---

## 4.2 基础导入导出

### export（导出）

一个文件默认不对外暴露任何内容，你需要显式 `export`：

```ts
// src/Task.ts（简化）
export type TaskType = 'local_bash' | 'local_agent' | 'remote_agent';

export type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed';

export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}

// 没有 export 的变量，其他文件无法访问
const TASK_ID_ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyz';
```

### import（导入）

```ts
// 另一个文件想使用 Task.ts 的内容
import { TaskType, TaskStatus, isTerminalTaskStatus } from './Task.js';

const status: TaskStatus = 'running';
console.log(isTerminalTaskStatus(status)); // false
```

注意：Claude Code 中使用的是 **`.js` 后缀**的导入路径（如 `./Task.js`），这是 TypeScript 的 ESM 规范要求——虽然文件实际是 `.ts`，但导入时要写 `.js`。

---

## 4.3 默认导出（default export）

一个文件可以有一个"默认导出"：

```ts
// 导出方
export default function main() {
  console.log('Hello');
}

// 导入方
import main from './main.js';
main();
```

但在 Claude Code 中，**极少使用 `export default`**。团队更偏好**命名导出**（即带名字的 `export`），因为：

1. 命名导出在 IDE 中更容易搜索和跳转
2. 重命名时更安全（IDE 可以自动重构）
3. 导入时必须用正确的名字，不容易混淆

---

## 4.4 type-only import（仅类型导入）

这是 Claude Code 源码中**极其常见**的写法：

```ts
// 来自 src/Tool.ts
import type { Command } from './commands.js'
import type { CanUseToolFn } from './hooks/useCanUseTool.js'
import type { ThinkingConfig } from './utils/thinking.js'
```

### `import` 和 `import type` 的区别

| 写法 | 作用 | 是否生成到运行时 |
|------|------|----------------|
| `import { foo } from '...'` | 导入值（函数、对象、类等） | ✅ 会生成 |
| `import type { Foo } from '...'` | 只导入类型 | ❌ 编译后删除 |

**为什么要区分？**

TypeScript 的类型在编译后会被完全擦除。如果你只使用某个类型（比如作为函数参数类型标注），用 `import type` 可以明确告诉编译器："我只在类型层面依赖这个模块"。

这有助于：
1. **避免循环依赖问题**
2. **加速编译**（类型导入可以被编译器更好地优化）
3. **明确意图**（读代码的人一眼就知道这是类型依赖）

### 混合导入

也可以把类型和非类型一起导入：

```ts
// 来自 src/utils/permissions/PermissionMode.ts
import {
  EXTERNAL_PERMISSION_MODES,
  type ExternalPermissionMode,
  PERMISSION_MODES,
  type PermissionMode,
} from '../../types/permissions.js'
```

这里 `EXTERNAL_PERMISSION_MODES` 和 `PERMISSION_MODES` 是**运行时值**，`ExternalPermissionMode` 和 `PermissionMode` 是**类型**。

---

## 4.5 重导出（Re-export）

有时一个文件只是作为"中转站"，把其他模块的内容集中导出：

```ts
// 来自 src/utils/settings/types.ts
export {
  type AgentHook,
  type BashCommandHook,
  type HookCommand,
  HookCommandSchema,
  type HookMatcher,
  HookMatcherSchema,
  HooksSchema,
  type HooksSettings,
  type HttpHook,
  type PromptHook,
} from '../../schemas/hooks.js'
```

这样其他文件只需要从 `src/utils/settings/types.ts` 导入，而不需要关心实际定义在哪里。

再看 `src/Tool.ts` 中的重导出：

```ts
// Re-export progress types for backwards compatibility
export type {
  AgentToolProgress,
  BashProgress,
  MCPProgress,
  REPLToolProgress,
  SkillToolProgress,
  TaskOutputProgress,
  WebSearchProgress,
}
```

这行代码没有 `from`！它表示"**从当前文件已经导入的类型中**，重新导出这些"。

完整版本应该是：

```ts
import type { AgentToolProgress, BashProgress } from './types/tools.js';
export type { AgentToolProgress, BashProgress };
```

Claude Code 把 `import` 和 `export` 分开写了（因为还有其他逻辑需要用到这些类型）。

### 混合重导出

也可以同时重导出类型和非类型：

```ts
// 来自 src/utils/permissions/PermissionMode.ts
export {
  EXTERNAL_PERMISSION_MODES,
  PERMISSION_MODES,
  type ExternalPermissionMode,
  type PermissionMode,
} from '../../types/permissions.js'
```

这里 `EXTERNAL_PERMISSION_MODES` 和 `PERMISSION_MODES` 是运行时值，`ExternalPermissionMode` 和 `PermissionMode` 是类型。一行代码同时重导出两者。

---

## 4.6 动态导入（Dynamic Import）

有时你不知道是否需要某个模块，或者想按需加载，可以用 `import()` 函数：

```ts
// 来自 src/entrypoints/cli.tsx
async function main(): Promise<void> {
  // 只在需要时加载 startupProfiler
  const { profileCheckpoint } = await import('../utils/startupProfiler.js');
  profileCheckpoint('cli_entry');
}
```

动态导入返回一个 `Promise`，所以必须在 `async` 函数中使用 `await`。

Claude Code 的 `main.tsx` 中还有更复杂的用法：

```ts
// Dead code elimination: conditional import for COORDINATOR_MODE
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') as typeof import('./coordinator/coordinatorMode.js')
  : null;
```

这里用了 `require()` 和 `typeof import()`。对于初学者，你只需要理解：**`import()` 是异步加载模块的方式**。

---

## 4.7 通配符导入（Namespace Import）

```ts
import * as React from 'react';

// 使用时
const element = React.createElement('div');
```

在 Claude Code 的 TSX 文件中很常见：

```ts
import * as React from 'react';
```

这表示"导入 `react` 模块的**所有导出**，并放在 `React` 这个名字空间下"。

---

## 4.8 本章小结

| 写法 | 含义 | Claude Code 使用频率 |
|------|------|---------------------|
| `export { foo }` | 命名导出 | ⭐⭐⭐ 极多 |
| `export default foo` | 默认导出 | ⭐ 很少 |
| `import { foo } from '...'` | 命名导入 | ⭐⭐⭐ 极多 |
| `import type { Foo }` | 仅类型导入 | ⭐⭐⭐ 极多 |
| `import { type Foo, bar }` | 混合导入 | ⭐⭐⭐ 极多 |
| `export { foo } from '...'` | 重导出 | ⭐⭐ 较多 |
| `export type { Foo }` | 重导出类型 | ⭐⭐ 较多 |
| `export { type Foo, bar }` | 混合重导出 | ⭐⭐ 较多 |
| `import * as X from '...'` | 通配符导入 | ⭐⭐ 较多（React） |
| `await import('...')` | 动态导入 | ⭐ 按需加载用 |

---

## 练习题

1. 下面两个导入有什么区别？
   ```ts
   import { Task } from './Task.js';
   import type { Task } from './Task.js';
   ```

2. 为什么 Claude Code 几乎不用 `export default`？说出至少两个理由。

3. 写出重导出的语法，把 `A.ts` 中的 `foo` 和 `bar` 通过 `B.ts` 重新导出。

---

> 下一章：泛型基础与实战 —— 理解 `function foo<T>()` 中的 `<T>` 到底是什么。
