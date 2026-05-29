# 第八章：TSX 与 React 类型基础

> 本章目标：理解 `.tsx` 文件的基本结构，能看懂 Claude Code 中 React 组件的类型写法。

---

## 8.1 什么是 TSX？

TSX = TypeScript + JSX。JSX 是一种语法扩展，允许你在 JavaScript/TypeScript 中写类似 HTML 的代码：

```tsx
// 这是一个 TSX 文件
import * as React from 'react';

function Hello() {
  return <div>Hello, Claude Code!</div>;
}
```

Claude Code 使用 **Ink** 库在终端中渲染 React 组件，所以它的 UI 代码全部是 TSX。

---

## 8.2 组件的类型定义

### 函数组件

```tsx
// 来自 src/components/Spinner.tsx（简化）
import * as React from 'react';

type Props = {
  mode: SpinnerMode;
  loadingStartTimeRef: React.RefObject<number>;
  spinnerTip?: string;
  verbose: boolean;
};

export function SpinnerWithVerb(props: Props): React.ReactNode {
  const isBriefOnly = useAppState(s => s.isBriefOnly);
  // ...
  return <SpinnerWithVerbInner {...props} />;
}
```

拆解：
- `type Props = { ... }` → 定义组件的属性类型
- `(props: Props)` → 参数 `props` 的类型是 `Props`
- `: React.ReactNode` → 返回类型是 React 节点（可以是 JSX、string、null 等）

### Props 中的常见类型

```tsx
type Props = {
  // 普通属性
  title: string;
  
  // 可选属性
  subtitle?: string;
  
  // number 类型
  count: number;
  
  // boolean 类型
  isActive: boolean;
  
  // 回调函数
  onClick: () => void;
  
  // 带参数的回调
  onSelect: (id: string) => void;
  
  // React 的 ref（引用 DOM 或组件）
  inputRef: React.RefObject<HTMLInputElement>;
  
  // children（子元素）
  children: React.ReactNode;
};
```

---

## 8.3 React Hooks 的类型

Claude Code 使用 React Hooks 管理状态，以下是常见 Hooks 的类型：

### useState

```tsx
import { useState } from 'react';

// state 类型自动推断为 boolean
const [isOpen, setIsOpen] = useState(false);

// 显式指定类型
const [user, setUser] = useState<User | null>(null);
```

### useRef

```tsx
import { useRef } from 'react';

// 引用一个 number 值（不是 DOM 元素）
const loadingStartTimeRef = useRef<number>(0);

// 引用 DOM 元素
const inputRef = useRef<HTMLInputElement>(null);
```

在 `Spinner.tsx` 中：

```tsx
type Props = {
  loadingStartTimeRef: React.RefObject<number>;
  pauseStartTimeRef: React.RefObject<number | null>;
  // ...
};
```

### useMemo

```tsx
import { useMemo } from 'react';

const briefEnvEnabled = useMemo(() => isEnvTruthy(process.env.CLAUDE_CODE_BRIEF), []);
// 返回值类型自动推断
```

### useEffect

```tsx
import { useEffect } from 'react';

useEffect(() => {
  // 副作用逻辑
  const timer = setInterval(() => setCount(c => c + 1), 1000);
  
  // 清理函数
  return () => clearInterval(timer);
}, []); // 依赖数组
```

---

## 8.4 组件中的类型断言

有时 TypeScript 无法自动推断类型，需要手动断言：

```tsx
// 来自 src/components/Spinner.tsx（简化）
export function SpinnerWithVerb(props: Props): React.ReactNode {
  const isBriefOnly = useAppState(s => s.isBriefOnly);
  const viewingAgentTaskId = useAppState(s_0 => s_0.viewingAgentTaskId);
  
  if (/* 某些条件 */) {
    return <BriefSpinner mode={props.mode} overrideMessage={props.overrideMessage} />;
  }
  return <SpinnerWithVerbInner {...props} />;
}
```

这里 `s` 和 `s_0` 的类型是 `AppState`，是通过 `useAppState` 的泛型推断出来的。

---

## 8.5 展开运算符传递 Props

```tsx
// {...props} 把父组件收到的所有 props 传给子组件
return <SpinnerWithVerbInner {...props} />;
```

等价于：

```tsx
return <SpinnerWithVerbInner 
  mode={props.mode} 
  loadingStartTimeRef={props.loadingStartTimeRef}
  // ... 所有属性都写一遍
/>;
```

---

## 8.6 自定义 Hooks 的类型

Claude Code 中有很多自定义 Hook：

```tsx
// 假设来自 src/hooks/useAppState.ts
export function useAppState<T>(selector: (state: AppState) => T): T {
  // ...
}
```

使用时：

```tsx
const isBriefOnly = useAppState(s => s.isBriefOnly);
// TypeScript 自动推断 T = boolean
```

---

## 8.7 类的简要概念（遇到自定义错误类时）

Claude Code 源码中虽然函数组件占主导，但你会遇到一些**类（class）**，尤其是**自定义错误类**：

```ts
// 来自 src/bridge/bridgeApi.ts
export class BridgeFatalError extends Error {
  constructor(message: string) {
    super(message);
  }
}

// 使用
throw new BridgeFatalError('连接失败');
```

对零基础学生来说，只需要理解：
- `class` 定义一个"蓝图"，可以创建对象
- `extends Error` 表示"继承自 JavaScript 内置的 Error 类"
- `new` 关键字创建类的实例

再看一个带属性的例子：

```ts
// 来自 src/services/mcp/client.ts
export class McpToolCallError extends Error {
  constructor(
    message: string,
    public readonly code: number,  // 自动成为实例属性
  ) {
    super(message);
  }
}
```

`public readonly code: number` 是 TypeScript 的简写语法，等价于：

```ts
class McpToolCallError extends Error {
  readonly code: number;
  constructor(message: string, code: number) {
    super(message);
    this.code = code;
  }
}
```

> **建议**：不要在读源码的初期深入学 class。遇到时知道"这是一个自定义错误"就够了。

### override 关键字

当子类重写父类的方法时，TypeScript 推荐加上 `override`：

```ts
class MyError extends Error {
  override message: string;  // 明确表示这是重写父类的属性
  
  constructor(message: string) {
    super(message);
    this.message = message;
  }
}
```

`override` 的作用是：如果父类没有这个方法，TypeScript 会报错，防止你拼错方法名。

---

## 8.8 本章小结

| 概念 | 写法 | 作用 |
|------|------|------|
| TSX | `.tsx` 文件 | 在 TS 中写 JSX |
| Props | `type Props = {...}` | 定义组件属性 |
| React.ReactNode | 返回类型 | 任何可渲染内容 |
| useState | `useState<T>(init)` | 状态管理 |
| useRef | `useRef<T>(init)` | 引用保存 |
| useEffect | `useEffect(fn, deps)` | 副作用 |
| useMemo | `useMemo(fn, deps)` | 缓存计算 |
| {...props} | 展开运算 | 传递所有属性 |
| class | `class X extends Error` | 自定义错误类 |
| override | `override method()` | 标记重写父类方法 |

---

## 练习题

1. 为一个按钮组件写 Props 类型：需要 `label`（字符串，必填）、`onClick`（回调，必填）、`disabled`（布尔，可选）。

2. `useRef<number>(0)` 和 `useRef<number>(null)` 有什么区别？

3. 为什么 Claude Code 用 Ink 而不是浏览器里的 React？（提示：它是 CLI 工具，跑在终端里）

---

> 下一章：源码实战 —— 读通 Task.ts 和 Tool.ts。
