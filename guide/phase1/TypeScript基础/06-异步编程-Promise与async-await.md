# 第六章：异步编程 Promise 与 async/await

> 本章目标：理解 JavaScript/TypeScript 的异步编程模型，能读懂 Claude Code 中大量的 `async` 函数和 `Promise` 类型。

---

## 6.1 为什么需要异步？

在 Claude Code 中，有很多操作是"耗时的"：
- 调用 Claude API（网络请求）
- 执行 Bash 命令（子进程）
- 读写文件（磁盘 IO）

如果这些操作都"阻塞"主线程，界面就会卡住。JavaScript 用**异步编程**来解决这个问题——启动操作后不等它完成，先去干别的，等完成了再回来处理。

---

## 6.2 Promise：异步操作的"承诺"

`Promise` 是一个表示"未来会完成的操作"的对象：

```ts
const promise = fetch('https://api.example.com/data');
// promise 的状态：pending（进行中）

promise.then(data => {
  // 成功时执行
  console.log(data);
}).catch(err => {
  // 失败时执行
  console.error(err);
});
```

在 TypeScript 中，`Promise` 是**泛型**的：

```ts
// Promise<string> 表示"这个异步操作最终会返回一个字符串"
function readFile(path: string): Promise<string> {
  return new Promise((resolve, reject) => {
    // ... 异步操作
    resolve("file content");
  });
}
```

---

## 6.3 async/await：更优雅的异步写法

`async/await` 是 `Promise` 的语法糖，让异步代码看起来像同步代码：

```ts
// 来自 src/entrypoints/cli.tsx
async function main(): Promise<void> {
  const args = process.argv.slice(2);

  if (args.length === 1 && args[0] === '--version') {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;
  }

  const { profileCheckpoint } = await import('../utils/startupProfiler.js');
  profileCheckpoint('cli_entry');
}
```

拆解：
- `async function` → 声明这是一个异步函数
- `Promise<void>` → 返回类型，表示"这个异步函数最终什么都不返回"
- `await import(...)` → 等待动态导入完成，拿到结果后再继续

**规则**：
1. `await` 只能在 `async` 函数中使用（或在顶层模块中，ES2022+）
2. `await` 后面必须是一个 `Promise`（或 thenable）
3. `async` 函数**总是**返回一个 `Promise`

---

## 6.4 Claude Code 中的异步模式

### 模式 1：async 函数返回 Promise

```ts
// 来自 src/Task.ts
export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

`kill` 方法返回 `Promise<void>`，表示这是一个异步操作，完成后不返回有意义的数据。

### 模式 2：await 多个并行操作

```ts
// 假设的例子
const [user, settings] = await Promise.all([
  fetchUser(),
  fetchSettings(),
]);
```

`Promise.all` 接收一个 Promise 数组，等待**所有** Promise 完成。这在 Claude Code 的初始化流程中很常见。

### 模式 3：try/catch 处理错误

```ts
async function safeOperation() {
  try {
    const result = await riskyAsyncCall();
    return result;
  } catch (error) {
    console.error('出错了:', error);
    return null;
  }
}
```

---

## 6.5 Promise 的类型

| 类型 | 含义 | 例子 |
|------|------|------|
| `Promise<void>` | 异步完成，无返回值 | `kill(): Promise<void>` |
| `Promise<string>` | 异步完成，返回字符串 | `readFile(): Promise<string>` |
| `Promise<T\|null>` | 异步完成，可能返回 null | `findUser(): Promise<User\|null>` |

---

## 6.6 空值合并运算符 `??`

Claude Code 中大量使用 `??`：

```ts
// 来自 src/entrypoints/cli.tsx
process.env[k] ??= '1';
```

`??` 是**空值合并运算符**：如果左边的值是 `null` 或 `undefined`，就使用右边的值。

```ts
const value = userInput ?? 'default';
// 等价于：
const value = userInput !== null && userInput !== undefined ? userInput : 'default';
```

与 `||` 的区别：
- `||`：左边是"假值"（0, '', false, null, undefined, NaN）就用右边
- `??`：只有左边是 `null` 或 `undefined` 才用右边

```ts
const num = 0 ?? 100;  // 0（因为 0 不是 null/undefined）
const num2 = 0 || 100; // 100（因为 0 是假值）
```

在配置处理中，`??` 更安全，因为合法的配置值可能是 `0`、`false` 或空字符串。

---

## 6.7 可选链 `?.`

```ts
// 假设的例子
const userCity = user?.address?.city;
// 等价于：
const userCity = user && user.address ? user.address.city : undefined;
```

`?.` 在 Claude Code 中也大量使用，用于安全地访问嵌套属性：

```ts
// 如果 update.updatedInput 存在，取它的 command 属性
const cmd = update.updatedInput?.command;
```

---

## 6.8 本章小结

| 概念 | 写法 | 作用 |
|------|------|------|
| Promise | `new Promise((resolve, reject) => {...})` | 表示异步操作 |
| async 函数 | `async function foo()` | 声明异步函数 |
| await | `await promise` | 等待 Promise 完成 |
| Promise.all | `Promise.all([p1, p2])` | 等待多个 Promise |
| `??` | `a ?? b` | null/undefined 时用默认值 |
| `?.` | `a?.b` | 安全访问嵌套属性 |

---

## 练习题

1. 下面代码的输出顺序是什么？
   ```ts
   async function test() {
     console.log('A');
     await Promise.resolve();
     console.log('B');
   }
   console.log('C');
   test();
   console.log('D');
   ```

2. 把下面的 Promise 链改写成 async/await：
   ```ts
   fetchUser()
     .then(user => fetchOrders(user.id))
     .then(orders => console.log(orders))
     .catch(err => console.error(err));
   ```

3. `??` 和 `||` 在什么情况下结果不同？举例说明。

---

> 下一章：高级类型工具与内置工具类型 —— 理解 `keyof`、`typeof`、`as const` 和 `Record` 等工具。
