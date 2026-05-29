# 第六章：异步编程 Promise 与 async/await

> 本章目标：理解 JavaScript 的异步编程模型，能读懂 Claude Code 中大量的 `async` 函数和 `Promise`。

---

## 6.1 为什么需要异步？

### 同步 vs 异步：生活中的比喻

想象你去餐厅吃饭：

- **同步（Synchronous）**：你点了一份炒饭，然后站在柜台前死死盯着厨师炒，什么都不干，一直等到炒好才走。这很浪费时间，对吧？
- **异步（Asynchronous）**：你点了一份外卖，下单后该干嘛干嘛——看电视、打游戏、洗澡。等外卖到了，外卖小哥给你打电话，你再去取餐。

**异步的核心思想**：启动一个耗时的操作后，不等它完成，先去干别的事，等它完成了再回来处理结果。

### JavaScript 是单线程的

JavaScript 就像一家只有**一个厨师**的餐厅。如果这位厨师（线程）一直在等水烧开（等待网络请求），那整个餐厅就停摆了——其他客人的订单都做不了。

所以 JavaScript 需要异步：遇到耗时的任务，先记下来，让厨师继续做别的菜，等水烧开了再回来处理。

### Claude Code 中的异步场景

在 Claude Code 源码中，到处都是异步操作：

- **网络请求**：调用 Claude API 获取回复（要等服务器响应）
- **文件读写**：读取或保存文件到硬盘（要等磁盘操作完成）
- **执行命令**：运行 Bash 命令（要等子进程执行完毕）

如果这些操作都"阻塞"主线程，界面就会卡住，用户什么都做不了。异步编程让 Claude Code 可以同时处理多件事情。

---

## 6.2 回调函数（Callback）：最早的异步方式

在 Promise 出现之前，JavaScript 用**回调函数**处理异步。

### 什么是回调函数？

回调函数就是一个"等事情做完了再调用的函数"。

```js
// setTimeout 是 JS 内置的异步函数
// 意思是：等 1000 毫秒（1秒）后，执行回调函数里的代码
setTimeout(() => {
  console.log('1秒到了！');
}, 1000);

console.log('这行会先执行');
```

输出顺序：
```
这行会先执行
1秒到了！
```

因为 `setTimeout` 不会阻塞后面的代码，JS 引擎先把"等1秒后执行回调"这个任务记在小本本上，然后立刻执行后面的 `console.log`。

### 回调地狱（Callback Hell）

当多个异步操作需要**按顺序**执行时，回调会一层套一层，代码越来越深：

```js
// 假设我们要：先读取文件A，再根据A的内容读取文件B，再处理B
// 这里用 readAsync 代表一个假想的异步读取函数
readAsync('a.txt', (err, dataA) => {
  if (err) {
    console.error('读A失败', err);
    return;
  }
  readAsync(dataA.trim(), (err, dataB) => {
    if (err) {
      console.error('读B失败', err);
      return;
    }
    processData(dataB, (err, result) => {
      if (err) {
        console.error('处理失败', err);
        return;
      }
      console.log('最终结果：', result);
    });
  });
});
```

这就像一个倒着的金字塔，代码越来越向右缩进，可读性极差。这种情况叫做**回调地狱**。

于是，JavaScript 引入了 **Promise** 来拯救我们。

---

## 6.3 Promise：异步操作的"承诺"

Promise 翻译成中文是"承诺"。你可以把它理解为一张"欠条"：

> "我现在还没法给你结果，但我承诺，等事情办完了，一定告诉你是成功还是失败。"

### Promise 的三种状态

Promise 就像一个外卖订单，有三种状态：

| 状态 | 含义 | 比喻 |
|------|------|------|
| `pending` | 进行中，还没结果 | 外卖已下单，骑手正在取餐 |
| `fulfilled` | 已成功完成 | 外卖已送达，很开心 |
| `rejected` | 已失败 | 外卖洒了，很沮丧 |

Promise 一旦从 `pending` 变成 `fulfilled` 或 `rejected`，就**不会再变**了。这叫做"承诺不可变"。

### 创建一个 Promise

```js
const promise = new Promise((resolve, reject) => {
  // 这里放异步操作
  const success = true;

  if (success) {
    resolve('操作成功！');  // 把状态变成 fulfilled
  } else {
    reject('操作失败！');    // 把状态变成 rejected
  }
});
```

拆解：
- `new Promise(...)` → 创建一个新的 Promise 对象
- `(resolve, reject) => {...}` → Promise 构造函数接收一个函数，这个函数又有两个参数
  - `resolve`：调用它表示"成功了"，并把结果传出去
  - `reject`：调用它表示"失败了"，并把错误原因传出去

### .then() 和 .catch()

Promise 创建后，用 `.then()` 处理成功，用 `.catch()` 处理失败：

```js
const fetchData = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('数据拿到了！');
  }, 1000);
});

fetchData
  .then((result) => {
    console.log(result);  // 输出：数据拿到了！
  })
  .catch((error) => {
    console.error(error);
  });
```

### .finally()

无论成功还是失败，`.finally()` 里的代码都会执行。就像你无论外卖好不好吃，吃完后都要扔垃圾：

```js
fetchData
  .then((result) => {
    console.log('成功：', result);
  })
  .catch((error) => {
    console.log('失败：', error);
  })
  .finally(() => {
    console.log('不管成功失败，我都要执行');
  });
```

### Promise 链式调用

`.then()` 返回的仍然是一个 Promise，所以可以一直链式调用下去：

```js
function fetchUser() {
  return Promise.resolve({ id: 1, name: 'Alice' });
}

function fetchOrders(userId) {
  return Promise.resolve(['订单A', '订单B']);
}

fetchUser()
  .then((user) => {
    console.log('用户：', user.name);
    return fetchOrders(user.id);  // 返回一个新的 Promise
  })
  .then((orders) => {
    console.log('订单：', orders);
  })
  .catch((err) => {
    console.error('出错了：', err);
  });
```

输出：
```
用户： Alice
订单： [ '订单A', '订单B' ]
```

这样代码就从"横向缩进"变成了"纵向链条"，清晰多了！

### Promise.all()：并行等待多个 Promise

有时候你需要同时做几件事，等**所有**事情都做完再处理：

```js
const p1 = fetch('/api/user');
const p2 = fetch('/api/settings');
const p3 = fetch('/api/messages');

Promise.all([p1, p2, p3])
  .then(([user, settings, messages]) => {
    console.log('三个请求都完成了！');
  })
  .catch((err) => {
    console.error('有一个失败了：', err);
  });
```

`Promise.all()` 接收一个 Promise 数组：
- 当**所有** Promise 都成功时，返回一个结果数组
- 只要**有一个**失败，整个 `Promise.all()` 就失败

这就像你点了三份外卖，`Promise.all()` 会等三份都到了才开饭，只要有一份永远送不到，你就一直等不到。

### Promise.race() 和 Promise.allSettled()（简要了解）

```js
// Promise.race：谁先完成就返回谁的结果（赛跑）
Promise.race([fetchFast(), fetchSlow()])
  .then((result) => console.log('最快的完成了：', result));

// Promise.allSettled：等所有 Promise 都有结果（无论成功失败）
Promise.allSettled([p1, p2, p3])
  .then((results) => {
    // results 里每个元素都有 status: 'fulfilled' 或 'rejected'
    console.log(results);
  });
```

### Promise 与类型（运行时的真相）

在 TypeScript 中，你可能会看到 `Promise<string>`、`Promise<void>` 这样的类型标注。但在 JavaScript 运行时，Promise **不携带任何类型信息**——它只是一个普通的对象，内部状态可以是 `pending`、`fulfilled` 或 `rejected`。`Promise<string>` 和 `Promise<number>` 在运行时完全没有区别，JS 引擎根本不会检查 resolve 出来的值是不是"字符串"。

TypeScript 的 `Promise<T>` 只是在编译阶段给开发者一个提示："这个 Promise 成功时，resolve 的值应该是 T 类型。" 编译成 JavaScript 后，所有的 `<T>` 都会被擦除，只剩下最原始的 Promise 对象。

---

## 6.4 async/await：更优雅的异步写法

Promise 的链式调用虽然比回调清晰，但还是有一堆 `.then()`。`async/await` 是 Promise 的"语法糖"，让异步代码**看起来像同步代码**。

### async function

在函数前面加 `async`，这个函数就变成了异步函数：

```js
async function sayHello() {
  return '你好';
}

// 调用 async 函数，它总是返回一个 Promise
sayHello().then((result) => {
  console.log(result);  // 输出：你好
});
```

**重要规则**：`async` 函数**总是返回一个 Promise**，即使你写的是 `return '你好'`，它也会自动包装成 `Promise.resolve('你好')`。

### await 等待 Promise

在 `async` 函数内部，可以用 `await` 关键字"等待"一个 Promise 完成：

```js
async function getData() {
  console.log('开始获取数据...');

  // await 会"暂停"这里，等 Promise 完成后再继续
  const result = await Promise.resolve('数据来了！');

  console.log(result);
  console.log('获取完成');
}

getData();
```

输出：
```
开始获取数据...
数据来了！
获取完成
```

注意：`await` 只能在 `async` 函数内部使用（或者在 ES2022+ 的模块顶层）。

### 用 try/catch 处理错误

异步操作也会失败，用 `try/catch` 来捕获错误：

```js
async function riskyOperation() {
  try {
    const result = await fetch('/api/data');
    console.log('成功：', result);
  } catch (error) {
    console.error('出错了：', error);
  }
}
```

这就像你外卖下单时加了保险：如果外卖洒了（catch），你能得到赔偿，而不是程序崩溃。

### Claude Code 源码例子：main() 函数

来看看 Claude Code 入口文件中的 `main` 函数（已简化为纯 JavaScript）：

```js
// 来自 src/entrypoints/cli.tsx（简化版）
async function main() {
  const args = process.argv.slice(2);

  // 快速路径：如果用户输入 --version，直接输出版本号并返回
  if (args.length === 1 && args[0] === '--version') {
    console.log('v2.1.88 (Claude Code)');
    return;
  }

  // 动态导入其他模块，await 等待导入完成
  const { profileCheckpoint } = await import('../utils/startupProfiler.js');
  profileCheckpoint('cli_entry');
}
```

拆解：
- `async function main()` → 声明 `main` 是一个异步函数
- `return` → 虽然写了 `return`，但因为函数是 `async` 的，实际返回的是 `Promise.resolve(undefined)`
- `await import(...)` → 等待模块加载完成，拿到导出的对象后再继续执行

如果没有 `await`，代码就会在模块还没加载完的时候继续往下跑，可能导致 `profileCheckpoint` 还没定义就被调用，从而报错。

### Claude Code 源码例子：clearConversation

再看一个更复杂的异步函数：

```js
// 来自 src/commands/clear/conversation.ts（简化版）
export async function clearConversation({
  setMessages,
  readFileState,
  getAppState,
  setAppState,
}) {
  // 先执行 SessionEnd hooks，await 等待它完成
  await executeSessionEndHooks('clear', {
    getAppState,
    setAppState,
    signal: AbortSignal.timeout(1500),
    timeoutMs: 1500,
  });

  // 计算需要保留的任务
  const preservedAgentIds = new Set();

  if (getAppState) {
    for (const task of Object.values(getAppState().tasks)) {
      if (task.isBackgrounded !== false) {
        preservedAgentIds.add(task.agentId);
      }
    }
  }

  // 清空消息
  setMessages(() => []);

  // 清除会话缓存
  clearSessionCaches(preservedAgentIds);

  // 恢复原始工作目录
  setCwd(getOriginalCwd());
  readFileState.clear();
}
```

拆解：
- `export async function clearConversation(...)` → 导出异步函数
- `await executeSessionEndHooks(...)` → **必须等** hooks 执行完才能继续，因为后面要清除数据，如果 hooks 还没跑完就清除了，hooks 可能会用到已删除的数据
- 后面的操作（收集保留任务、清空消息、清除缓存）不需要等待网络，所以不用 `await`

---

## 6.5 空值合并运算符 `??`

### `a ?? b` 的含义

`??` 叫做**空值合并运算符**。它的作用是：

> 如果左边的值是 `null` 或 `undefined`，就使用右边的值。

```js
const userInput = null;
const value = userInput ?? '默认值';
console.log(value);  // 输出：默认值

const userName = 'Alice';
const name = userName ?? '匿名用户';
console.log(name);   // 输出：Alice
```

### `??` 与 `||` 的区别

`||`（逻辑或）和 `??` 很像，但有一个重要区别：

- `||`：左边是**假值**（`0`、`''`、`false`、`null`、`undefined`、`NaN`）就用右边
- `??`：只有左边是 `null` 或 `undefined` 才用右边

```js
const num1 = 0 ?? 100;
console.log(num1);  // 输出：0（0 不是 null/undefined）

const num2 = 0 || 100;
console.log(num2);  // 输出：100（0 是假值）

const str1 = '' ?? 'hello';
console.log(str1);  // 输出：（空字符串，不是 null/undefined）

const str2 = '' || 'hello';
console.log(str2);  // 输出：hello（空字符串是假值）
```

在配置处理中，`??` 更安全。因为合法的配置值可能是 `0`、空字符串或 `false`，用 `||` 会误判为"没设置"。

### `??=` 赋值运算符

`??=` 是 `??` 的赋值版本：

```js
let a = null;
a ??= '默认值';
console.log(a);  // 输出：默认值

let b = '已有值';
b ??= '新值';
console.log(b);  // 输出：已有值（因为 b 不是 null/undefined）
```

### Claude Code 源码例子

```js
// 来自 src/entrypoints/cli.tsx
for (const k of ['CLAUDE_CODE_SIMPLE', 'CLAUDE_CODE_DISABLE_THINKING']) {
  process.env[k] ??= '1';
}
```

拆解：
- `process.env[k]` → 读取环境变量
- `??= '1'` → 如果环境变量还没设置（是 `undefined`），就把它设为 `'1'`
- 如果已经设置过了，就保持原样不变

这就像进房间前先检查灯有没有开，没开才打开，避免重复操作。

---

## 6.6 可选链 `?.`

### 安全地访问嵌套属性

在实际编程中，我们经常需要访问嵌套对象的属性，但中间的某一层可能是 `null` 或 `undefined`：

```js
const user = {
  address: {
    city: '北京'
  }
};

// 传统写法：要一层层判断，很繁琐
const city = user && user.address ? user.address.city : undefined;

// 可选链写法：简洁优雅
const city2 = user?.address?.city;
```

`?.` 的意思是：
- 如果左边的值存在（不是 `null` 或 `undefined`），就继续访问右边的属性
- 如果左边的值不存在，就停止，返回 `undefined`

### 三种用法

```js
// 1. 访问对象属性
const city = user?.address?.city;

// 2. 访问数组元素
const firstItem = arr?.[0];

// 3. 调用函数（如果函数存在的话）
const result = obj?.callback?.();
```

### Claude Code 源码例子

```js
// 如果 update.updatedInput 存在，取它的 command 属性
const cmd = update?.updatedInput?.command;
```

拆解：
- `update` → 可能是一个更新对象
- `update?.updatedInput` → 如果 `update` 存在，就访问 `updatedInput`；如果 `update` 是 `null`/`undefined`，返回 `undefined`
- `?.command` → 如果 `updatedInput` 存在，就访问 `command`；否则返回 `undefined`

如果没有可选链，传统写法会很啰嗦：

```js
const cmd = update && update.updatedInput ? update.updatedInput.command : undefined;
```

可选链让代码更安全、更简洁。

---

## 6.7 本章小结

| 概念 | 写法 | 作用 |
|------|------|------|
| 回调函数 | `setTimeout(() => {...}, 1000)` | 最早的异步处理方式 |
| Promise | `new Promise((resolve, reject) => {...})` | 表示一个异步操作的"承诺" |
| Promise 状态 | `pending` → `fulfilled`/`rejected` | 记录异步操作的进行状态 |
| .then() | `promise.then(result => {...})` | 处理 Promise 成功 |
| .catch() | `promise.catch(err => {...})` | 处理 Promise 失败 |
| .finally() | `promise.finally(() => {...})` | 无论成功失败都执行 |
| Promise.all() | `Promise.all([p1, p2])` | 等待多个 Promise 全部完成 |
| Promise.allSettled() | `Promise.allSettled([p1, p2])` | 等待多个 Promise 都有结果，无论成功失败 |
| async 函数 | `async function foo()` | 声明异步函数，总是返回 Promise |
| await | `await promise` | 在 async 函数中等待 Promise 完成 |
| try/catch | `try { await x() } catch(e) {...}` | 捕获异步错误 |
| `??` | `a ?? b` | 左边为 null/undefined 时用右边 |
| `??=` | `a ??= b` | 空值合并的赋值版本 |
| `?.` | `obj?.prop` | 安全访问嵌套属性 |

---

## 练习题

1. 下面代码的输出顺序是什么？请说明原因。
   ```js
   async function test() {
     console.log('A');
     await Promise.resolve();
     console.log('B');
   }
   console.log('C');
   test();
   console.log('D');
   ```

2. 把下面的 Promise 链改写成 `async/await` 形式：
   ```js
   fetchUser()
     .then(user => fetchOrders(user.id))
     .then(orders => console.log(orders))
     .catch(err => console.error(err));
   ```

3. `??` 和 `||` 在什么情况下结果不同？请分别举一个例子说明。

---

> 下一章：JavaScript 内置对象与工具方法 —— 理解 Map、Set、Date、JSON、Error、Math 等常用工具。
