# 第七章：JavaScript 内置对象与工具方法

> 本章目标：掌握 JavaScript 中常用的内置对象和工具方法，能看懂 Claude Code 源码中的 Map、Set、Date、JSON、Error、Math 等用法。

---

## 7.1 Map 和 Set

### Map：键值对集合

`Map` 是一种存储"键值对"的数据结构，就像一本字典：
- 每个词（键）对应一个解释（值）
- 你可以通过词快速找到解释

和普通对象 `{}` 相比，Map 的优势在于：**任何类型的值都可以做键**（包括对象、函数、甚至另一个 Map），而且 Map 会记住键的插入顺序。

```js
// 创建 Map
const dictionary = new Map();

// 添加键值对
dictionary.set('hello', '你好');
dictionary.set('world', '世界');
dictionary.set(42, '一个数字');       // 数字也可以做键
dictionary.set({ name: 'Alice' }, '一个对象');  // 对象也可以做键

// 获取值
console.log(dictionary.get('hello'));  // 输出：你好
console.log(dictionary.get(42));       // 输出：一个数字

// 检查是否存在某个键
console.log(dictionary.has('world'));  // 输出：true
console.log(dictionary.has('bye'));    // 输出：false

// 获取 Map 的大小
console.log(dictionary.size);          // 输出：4

// 删除某个键
dictionary.delete('world');

// 遍历 Map
for (const [key, value] of dictionary) {
  console.log(key, '->', value);
}

// 清空
dictionary.clear();
```

### Claude Code 源码例子：Map

```js
// 来自 src/Tool.ts（简化版，去掉类型注解）
const additionalWorkingDirectories = new Map();

// 添加工作目录
additionalWorkingDirectories.set('/home/user/project', {
  path: '/home/user/project',
  source: 'config',
});

// 读取
const dir = additionalWorkingDirectories.get('/home/user/project');
console.log(dir);  // 输出：{ path: '/home/user/project', source: 'config' }
```

拆解：
- `new Map()` → 创建一个空的 Map
- `.set(key, value)` → 添加键值对
- `.get(key)` → 根据键取值，如果键不存在返回 `undefined`

### Set：不重复值的集合

`Set` 是一堆**不重复的值**的集合，就像你收藏的一盒邮票，每张都是独一无二的。

```js
// 创建 Set
const tags = new Set();

// 添加元素
tags.add('javascript');
tags.add('python');
tags.add('javascript');  // 重复添加不会生效

console.log(tags.size);           // 输出：2（虽然添加了3次，但去重了）
console.log(tags.has('python'));  // 输出：true
console.log(tags.has('java'));    // 输出：false

// 删除
tags.delete('python');

// 遍历
for (const tag of tags) {
  console.log(tag);
}
```

### Claude Code 源码例子：Set

```js
// 来自 src/commands/clear/conversation.ts（简化版）
const preservedAgentIds = new Set();

// 遍历所有任务，收集需要保留的 agentId
for (const task of Object.values(getAppState().tasks)) {
  if (task.isBackgrounded !== false) {
    preservedAgentIds.add(task.agentId);
  }
}

console.log(preservedAgentIds.size);  // 输出保留的 agent 数量
console.log(preservedAgentIds.has('agent-123'));  // 检查某个 agent 是否在保留列表中
```

拆解：
- `new Set()` → 创建一个空集合
- `.add(value)` → 添加元素，如果已存在则忽略（自动去重）
- `.has(value)` → 检查集合中是否包含某个值
- `.size` → 获取集合中元素的数量

Set 特别适合用来：
- 去重数组（`[...new Set(arr)]`）
- 快速判断"某个值是否已存在"

### WeakMap 和 WeakSet（简要了解）

`WeakMap` 和 `WeakSet` 是 Map 和 Set 的"弱引用"版本：
- 键必须是对象
- 当对象不再被其他地方引用时，会自动被垃圾回收
- 不能遍历（没有 `.size` 和 `for...of`）

它们主要用于一些高级场景（如缓存对象私有数据），初学者了解存在即可。

---

## 7.2 Date 对象

### 获取当前时间

```js
// 创建当前时间的 Date 对象
const now = new Date();
console.log(now);  // 输出：Wed May 28 2025 23:18:30 GMT+0800

// 获取时间戳（从 1970年1月1日到现在经过的毫秒数）
const timestamp = Date.now();
console.log(timestamp);  // 输出：1751174310000（一个很大的数字）
```

时间戳是一个数字，非常方便用来：
- 计算两个时间点之间隔了多久
- 记录事件发生的时间
- 作为唯一标识的一部分

### Claude Code 源码例子：Date.now()

```js
// 来自 src/Task.ts（简化版）
function createTask() {
  return {
    id: generateTaskId(),
    startTime: Date.now(),  // 记录任务创建的时间戳
    status: 'running',
  };
}

const task = createTask();
console.log(task.startTime);  // 输出：1751174310000

// 之后可以计算任务运行了多久
const elapsed = Date.now() - task.startTime;
console.log(`任务已运行 ${elapsed} 毫秒`);
```

拆解：
- `Date.now()` → 返回当前时间的时间戳（毫秒数）
- 用两个时间戳相减，就能得到经过的时间

---

## 7.3 JSON

JSON（JavaScript Object Notation）是一种轻量级的数据交换格式，也是 JavaScript 内置的对象。

### JSON.stringify：对象 → 字符串

把 JavaScript 对象转换成 JSON 字符串，方便存储或传输：

```js
const user = {
  name: 'Alice',
  age: 25,
  hobbies: ['读书', '编程'],
};

const jsonString = JSON.stringify(user);
console.log(jsonString);
// 输出：{"name":"Alice","age":25,"hobbies":["读书","编程"]}
```

### JSON.parse：字符串 → 对象

把 JSON 字符串转回 JavaScript 对象：

```js
const jsonString = '{"name":"Alice","age":25}';
const user = JSON.parse(jsonString);
console.log(user.name);  // 输出：Alice
console.log(user.age);   // 输出：25
```

### 注意循环引用的问题

如果对象中存在循环引用（A 引用 B，B 又引用 A），`JSON.stringify` 会报错：

```js
const a = { name: 'A' };
const b = { name: 'B', ref: a };
a.ref = b;  // 循环引用！

JSON.stringify(a);  // ❌ 报错：Converting circular structure to JSON
```

### 函数和 undefined 会被忽略

`JSON.stringify` 还会自动跳过对象的**函数**和 **undefined** 值：

```js
const user = {
  name: 'Alice',
  age: undefined,        // undefined 会被跳过
  sayHi() {              // 函数会被跳过
    console.log('Hi');
  },
};

console.log(JSON.stringify(user));
// 输出：{"name":"Alice"}
// 注意：age 和 sayHi 都没有出现在结果中！
```

> 💡 **为什么？** JSON 格式的设计目标是传递**数据**，而函数是可执行代码、`undefined` 表示"没有值"，它们都不是纯粹的数据，所以 JSON 不支持。

---

## 7.4 正则表达式（RegExp）简要介绍

正则表达式是用来匹配字符串模式的工具。就像你在一篇文章中搜索所有"电话号码"，正则表达式就是描述"电话号码长什么样"的规则。

### 基本语法

```js
// 创建正则表达式：匹配以 "hello" 开头的字符串
const pattern = /^hello/;

// 或者
const pattern2 = new RegExp('^hello');

// test()：测试字符串是否匹配
console.log(pattern.test('hello world'));  // 输出：true
console.log(pattern.test('hi world'));     // 输出：false

// match()：提取匹配的内容
const str = '我的电话是 138-1234-5678';
const phonePattern = /\d{3}-\d{4}-\d{4}/;
console.log(str.match(phonePattern));  // 输出：["138-1234-5678"]
```

正则表达式是处理字符串的利器，但规则比较复杂。初学者只需要知道：
- `/pattern/flags` 是正则的字面量写法
- `.test()` 用来判断是否匹配
- `.match()` 用来提取匹配内容

需要时再深入学习即可。

---

## 7.5 Error 和异常处理

### throw new Error

当程序遇到无法继续执行的情况时，可以"抛出"一个错误：

```js
function divide(a, b) {
  if (b === 0) {
    throw new Error('除数不能为 0');
  }
  return a / b;
}

divide(10, 0);  // ❌ 报错：Error: 除数不能为 0
```

### try/catch/finally

用 `try/catch` 捕获错误，防止程序崩溃：

```js
try {
  const result = divide(10, 0);
  console.log(result);
} catch (error) {
  console.error('捕获到错误：', error.message);
} finally {
  console.log('不管有没有错误，这行都会执行');
}
```

输出：
```
捕获到错误：除数不能为 0
不管有没有错误，这行都会执行
```

### Error 对象的属性

```js
try {
  throw new Error('出错了！');
} catch (error) {
  console.log(error.message);  // 错误信息：出错了！
  console.log(error.name);     // 错误类型：Error
  console.log(error.stack);    // 调用栈：显示错误发生的位置
}
```

### Claude Code 源码例子：自定义错误类

```js
// 来自 src/bridge/bridgeApi.ts（简化版，去掉类型注解）
class BridgeFatalError extends Error {
  constructor(message, status, errorType) {
    super(message);           // 调用父类 Error 的构造函数
    this.name = 'BridgeFatalError';  // 设置错误名称
    this.status = status;     // 自定义属性：HTTP 状态码
    this.errorType = errorType;  // 自定义属性：错误类型
  }
}

// 使用
throw new BridgeFatalError('连接失败', 401, 'unauthorized');
```

拆解：
- `class BridgeFatalError extends Error` → 定义一个继承自 Error 的自定义错误类
- `super(message)` → 调用父类构造函数，设置错误消息
- `this.status = status` → 给错误对象添加自定义属性
- `throw new BridgeFatalError(...)` → 抛出自定义错误

自定义错误的好处是：你可以根据错误类型做不同处理。

```js
try {
  connectToBridge();
} catch (error) {
  if (error instanceof BridgeFatalError) {
    console.error('桥接致命错误，需要重新登录');
  } else {
    console.error('其他错误：', error.message);
  }
}
```

---

## 7.6 全局对象和全局方法

JavaScript 中有一些不需要导入就能直接使用的全局对象和方法。

### console 对象

```js
console.log('普通日志');
console.error('错误信息');   // 通常显示为红色
console.warn('警告信息');    // 通常显示为黄色
console.info('提示信息');

// 表格形式输出
console.table([
  { name: 'Alice', age: 25 },
  { name: 'Bob', age: 30 },
]);

// 计时
console.time('耗时');
for (let i = 0; i < 1000000; i++) {}
console.timeEnd('耗时');  // 输出：耗时: 2.345ms
```

### 定时器

```js
// setTimeout：延迟执行一次
const timerId = setTimeout(() => {
  console.log('3秒后执行');
}, 3000);

// 取消定时器
clearTimeout(timerId);

// setInterval：每隔一段时间执行一次
const intervalId = setInterval(() => {
  console.log('每隔1秒执行一次');
}, 1000);

// 取消循环定时器
clearInterval(intervalId);
```

### 类型转换

```js
// parseInt：字符串转整数
console.log(parseInt('42'));      // 42
console.log(parseInt('42.5'));    // 42
console.log(parseInt('100px'));   // 100（会解析到非数字字符前停止）
console.log(parseInt('abc'));     // NaN（Not a Number）

// parseFloat：字符串转浮点数
console.log(parseFloat('3.14'));  // 3.14

// 注意：NaN 是一个特殊值，表示"不是一个数字"
console.log(isNaN('abc'));        // true（'abc' 不是数字）
console.log(isNaN(42));           // false（42 是数字）

// Number.isFinite：检查是否是有限数字
console.log(Number.isFinite(42));       // true
console.log(Number.isFinite(Infinity)); // false
console.log(Number.isFinite(NaN));      // false
```

---

## 7.7 Math 对象

`Math` 是 JavaScript 内置的数学工具对象，提供了很多常用的数学方法。

### 随机数和取整

```js
// Math.random()：生成 0（包含）到 1（不包含）之间的随机小数
console.log(Math.random());  // 例如：0.734562

// Math.floor()：向下取整
console.log(Math.floor(3.7));  // 3
console.log(Math.floor(3.1));  // 3

// Math.ceil()：向上取整
console.log(Math.ceil(3.1));   // 4
console.log(Math.ceil(3.9));   // 4

// Math.round()：四舍五入
console.log(Math.round(3.4));  // 3
console.log(Math.round(3.5));  // 4

// 生成 1 到 10 的随机整数
const randomInt = Math.floor(Math.random() * 10) + 1;
console.log(randomInt);
```

### 最大最小值

```js
console.log(Math.max(5, 10, 3));  // 10
console.log(Math.min(5, 10, 3));  // 3

// 也可以用在数组上（配合展开运算符）
const numbers = [5, 10, 3, 8];
console.log(Math.max(...numbers));  // 10
console.log(Math.min(...numbers));  // 3
```

### Claude Code 源码例子：生成随机任务 ID

```js
// 来自 src/tasks/LocalMainSessionTask.ts（简化版）
import { randomBytes } from 'crypto';

const TASK_ID_ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyz';

function generateMainSessionTaskId() {
  const bytes = randomBytes(8);  // 生成 8 个随机字节
  let id = 's';  // 任务 ID 以 's' 开头

  for (let i = 0; i < 8; i++) {
    // bytes[i] 是一个 0-255 的数字
    // % TASK_ID_ALPHABET.length 确保索引在字母表范围内
    id += TASK_ID_ALPHABET[bytes[i] % TASK_ID_ALPHABET.length];
  }

  return id;
}

console.log(generateMainSessionTaskId());  // 例如：s4k9m2p7
```

拆解：
- `randomBytes(8)` → 生成 8 个随机字节，每个字节是 0-255 的整数
- `bytes[i] % TASK_ID_ALPHABET.length` → 取余运算，把随机数映射到字母表索引范围内
  - `TASK_ID_ALPHABET.length` 是 36（0-9 和 a-z）
  - 所以索引范围是 0-35
- `TASK_ID_ALPHABET[...]` → 根据索引从字母表中取字符

这就像摇骰子：骰子有 36 个面（字母表），每次摇一个随机数，取对应面的字符，摇 8 次就组成了一个随机 ID。

---

## 7.8 从 JavaScript 值理解 TypeScript 类型工具

> 本节目标：理解 TypeScript 中那些"类型工具"在 JavaScript 运行时对应的底层知识。TypeScript 的类型系统就像给 JavaScript 代码画了一张"设计图"，而真正的代码运行时，这些类型全部消失，只剩下纯粹的 JavaScript 值和对象操作。

---

### 7.8.1 `keyof` —— 获取对象的所有键

TypeScript 中的 `keyof` 用来获取一个类型的所有键。在 JavaScript 中，对应的是 **`Object.keys()`**：

```js
const theme = {
  rainbow_red: '#ff0000',
  rainbow_blue: '#0000ff',
};

// JavaScript 中获取对象的所有键
const keys = Object.keys(theme);
console.log(keys);  // ['rainbow_red', 'rainbow_blue']
```

**对应关系**：
- TS 的 `keyof Theme` → 编译时检查你只能写 `'rainbow_red'` 或 `'rainbow_blue'`
- JS 的 `Object.keys(theme)` → 运行时真的去对象里找出所有键名

---

### 7.8.2 `typeof`（类型层面）—— 从值推断形状

JavaScript 有运行时的 `typeof`（返回 `'string'`、`'number'` 等字符串），而 TypeScript 的类型 `typeof` 可以从**已有的值**推断出类型形状。

```js
const config = {
  port: 3000,
  host: 'localhost',
  debug: true,
};

// JavaScript 中，这个对象的结构就是"类型"本身
// config 有 port（数字）、host（字符串）、debug（布尔值）
// TypeScript 的 `type Config = typeof config` 就是把这个结构提取出来
```

**核心理解**：在 JavaScript 中，**值的结构就是类型**。TypeScript 只是把这个结构用正式的语法写出来。

---

### 7.8.3 `as const` —— 让值不可变

TypeScript 的 `as const` 可以把一个值变成"最严格的字面量类型"。在 JavaScript 中，对应的观念是**不重新赋值**和 **`Object.freeze()`**：

```js
// 普通数组：你可以重新赋值、push 新元素
const modes = ['default', 'plan'];
modes.push('auto');  // ✅ JavaScript 允许

// 如果想"冻结"它，让它完全不能变
const frozenModes = Object.freeze(['default', 'plan']);
frozenModes.push('auto');  // ❌ 报错（严格模式下）或静默失败

// 在 JavaScript 中，const + 不修改 + 团队约定 = 类似 as const 的效果
```

**对应关系**：
- TS 的 `as const` → 编译时阻止你修改值
- JS 的 `Object.freeze()` → 运行时阻止你修改值
- JS 的 `const` + 约定 → 程序员承诺不修改

---

### 7.8.4 `Record`、`Partial`、`Pick`、`Omit` —— 对象的构造与变形

这些 TypeScript 工具类型在底层都是**对对象进行构造、筛选或变形**。JavaScript 中没有这些关键字，但可以用对象字面量和展开运算符实现完全一样的逻辑。

**`Record<K, V>` —— 构造一个对象**：

```js
// TypeScript: Record<string, number> 表示"键是字符串，值是数字的对象"

// JavaScript 中，这就是普通对象：
const scores = {
  alice: 95,
  bob: 87,
};
// 键是 'alice'、'bob'（字符串），值是 95、87（数字）
```

**`Partial<T>` —— 所有属性变为可选**：

```js
// 在 JavaScript 中，对象的属性本来就"可以有也可以没有"
const user1 = { name: 'Alice', age: 25 };  // 有 age
const user2 = { name: 'Bob' };              // 没有 age

// Partial 在 TS 中的作用是：编译时允许你省略某些属性
// 在 JS 中，你本来就可以省略任何属性，运行时不检查
```

**`Pick<T, K>` 和 `Omit<T, K>` —— 挑选和排除属性**：

```js
const user = {
  name: 'Alice',
  age: 25,
  password: 'secret',
};

// Pick：只保留 name 和 age（排除 password）
const publicInfo = {
  name: user.name,
  age: user.age,
};

// Omit：排除 password，保留其他
const publicInfo2 = { ...user };
delete publicInfo2.password;  // 不推荐直接删除，但展示了"排除"的概念

// 更优雅的 Omit（用解构）
const { password, ...publicInfo3 } = user;
console.log(publicInfo3);  // { name: 'Alice', age: 25 }
```

**`Required<T>` —— 所有属性变为必选**：

与 `Partial` 相反。在 JavaScript 中，这意味着你要确保对象**一定包含**所有需要的属性。通常通过默认值或初始化逻辑来保证：

```js
function createUser(name, age) {
  // 确保返回的对象一定有 name 和 age
  return {
    name: name || '匿名',
    age: age !== undefined ? age : 0,
  };
}
```

**对应关系**：
- `Pick` → 从对象中取出指定属性
- `Omit` → 用解构 `const { a, b, ...rest } = obj` 排除属性
- `Partial` → JavaScript 对象属性本来就可选
- `Required` → 通过默认值或断言确保属性存在
- `Record` → 普通对象字面量 `{}`

---

### 7.8.5 `Exclude` 和 `Extract` —— 条件筛选

```js
const allStatuses = ['pending', 'running', 'completed', 'failed'];

// Exclude：排除某些值（筛选出"未完成"的状态）
const unfinished = allStatuses.filter(s => s !== 'completed' && s !== 'failed');
console.log(unfinished);  // ['pending', 'running']

// Extract：只保留符合某些条件的值
const terminal = allStatuses.filter(s => s === 'completed' || s === 'failed');
console.log(terminal);  // ['completed', 'failed']
```

**对应关系**：
- TS 的 `Exclude<T, U>` → JS 的 `filter` 排除某些值
- TS 的 `Extract<T, U>` → JS 的 `filter` 只保留某些值

---

### 7.8.6 `ReturnType` 和 `Awaited` —— 函数返回值与 Promise 解包

**`ReturnType` —— 获取函数返回什么**：

```js
function greet() {
  return 'hello';
}

// JavaScript 中，函数的"返回类型"就是实际返回的值
const result = greet();
console.log(typeof result);  // 'string'

// 如果你想知道一个函数返回什么，直接调用它看看结果就知道了
```

**`Awaited` —— 解开 Promise 拿到里面的值**：

```js
async function fetchUser() {
  return { name: 'Alice', age: 25 };
}

// Promise 就像一个"包裹"，Awaited 就是拆开包裹拿到里面的东西
const user = await fetchUser();
console.log(user);  // { name: 'Alice', age: 25 }

// 在 JavaScript 中，await 就是 "Awaited" 的运行时实现
```

---

### 7.8.7 `Readonly` 和 `readonly` —— 只读约定

TypeScript 的 `readonly` 修饰符表示"这个属性不能被重新赋值"。在 JavaScript 中，对应的约定是**程序员自己承诺不修改**：

```js
const config = {
  apiUrl: 'https://api.example.com',
  maxRetries: 3,
};

// JavaScript 中，const 保证 config 不能被重新赋值
// config = {};  // ❌ 报错

// 但对象内部的属性仍然可以修改！
config.maxRetries = 10;  // ⚠️ JavaScript 允许，但团队可能约定"不许改"

// 如果真想让它完全只读，用 Object.freeze
const frozenConfig = Object.freeze({
  apiUrl: 'https://api.example.com',
  maxRetries: 3,
});
frozenConfig.maxRetries = 10;  // ❌ 严格模式下报错
```

**对应关系**：
- TS 的 `readonly` → 编译时阻止修改
- JS 的 `Object.freeze()` → 运行时阻止修改
- JS 的 `const` + 团队规范 → 约定不修改

---

### 7.8.8 `satisfies` —— 运行时断言

TypeScript 的 `satisfies` 用于检查一个值是否满足某种类型约束。在 JavaScript 中，对应的运行时概念是**手动断言检查**：

```js
const modes = ['default', 'plan'];

// JavaScript 中的"satisfies"：手动检查
function assertValidModes(modes) {
  const validModes = ['default', 'plan', 'auto'];
  for (const mode of modes) {
    if (!validModes.includes(mode)) {
      throw new Error(`Invalid mode: ${mode}`);
    }
  }
}

assertValidModes(modes);  // 运行时检查：如果传了非法值就报错
```

**核心区别**：
- TS 的 `satisfies` → **编译时**检查，不生成任何运行时代码
- JS 的断言函数 → **运行时**检查，真的会执行代码并可能报错

---

### 7.8.9 `infer` —— 从已有结构中提取信息

TypeScript 的 `infer` 用于从复杂类型中提取子类型。在 JavaScript 中，对应的观念是**从已有数据中提取信息**：

```js
// 假设有一个 Zod Schema（运行时校验库）
// const UserSchema = z.object({ name: z.string(), age: z.number() });

// TypeScript 的 z.infer<typeof UserSchema> 提取出类型 { name: string; age: number }

// 在 JavaScript 中，Schema 本身就描述了数据的形状
// 运行时，Zod 会根据这个 Schema 检查数据是否符合
const userData = { name: 'Alice', age: 25 };
// UserSchema.parse(userData) 会验证 userData 是否符合 Schema 定义的形状
```

**初学者只需记住**：`infer` 在 TS 中的作用类似于"从一个复杂的定义中提取出简单的那部分"。在 JS 中，你通常直接操作值本身，不需要"提取类型"。

---

## 7.9 本章小结

### 运行时内置对象

| 对象/方法 | 常用写法 | 作用 |
|-----------|----------|------|
| Map | `new Map()` | 键值对集合，任何类型可做键 |
| Map.set | `map.set(key, value)` | 添加键值对 |
| Map.get | `map.get(key)` | 根据键取值 |
| Map.has | `map.has(key)` | 检查键是否存在 |
| Set | `new Set()` | 不重复值的集合 |
| Set.add | `set.add(value)` | 添加元素（自动去重） |
| Set.has | `set.has(value)` | 检查元素是否存在 |
| Date.now | `Date.now()` | 获取当前时间戳 |
| JSON.stringify | `JSON.stringify(obj)` | 对象转 JSON 字符串 |
| JSON.parse | `JSON.parse(str)` | JSON 字符串转对象 |
| RegExp.test | `/pattern/.test(str)` | 测试字符串是否匹配 |
| String.match | `str.match(/pattern/)` | 提取匹配内容 |
| Error | `throw new Error('msg')` | 抛出错误 |
| try/catch | `try {...} catch(e) {...}` | 捕获并处理错误 |
| console.log | `console.log(...)` | 打印日志 |
| setTimeout | `setTimeout(fn, ms)` | 延迟执行 |
| setInterval | `setInterval(fn, ms)` | 定时重复执行 |
| parseInt | `parseInt('42')` | 字符串转整数 |
| parseFloat | `parseFloat('3.14')` | 字符串转浮点数 |
| isNaN | `isNaN(value)` | 检查是否为 NaN |
| Math.random | `Math.random()` | 生成随机小数 |
| Math.floor | `Math.floor(x)` | 向下取整 |
| Math.ceil | `Math.ceil(x)` | 向上取整 |
| Math.round | `Math.round(x)` | 四舍五入 |
| Math.max | `Math.max(a, b, c)` | 取最大值 |
| Math.min | `Math.min(a, b, c)` | 取最小值 |

### TypeScript 类型工具的 JavaScript 底层对应

| TS 工具 | JS 底层对应 | 核心思想 |
|---------|------------|---------|
| `keyof` | `Object.keys()` | 获取对象的所有键 |
| `typeof`（类型） | 值的结构即类型 | 从已有值推断形状 |
| `as const` | `const` + `Object.freeze()` | 让值不可变 |
| `Record<K, V>` | 普通对象 `{}` | 键值对集合 |
| `Partial<T>` | 对象属性可省略 | 某些属性可有可无 |
| `Pick<T, K>` | 取指定属性构造新对象 | 挑选属性 |
| `Omit<T, K>` | 解构 `const {a, ...rest} = obj` | 排除属性 |
| `Exclude<T, U>` | `arr.filter(...)` | 排除某些值 |
| `Extract<T, U>` | `arr.filter(...)` | 只保留某些值 |
| `ReturnType<T>` | 调用函数看返回值 | 函数的输出 |
| `Awaited<T>` | `await promise` | 解开 Promise |
| `Readonly` / `readonly` | `Object.freeze()` + 约定 | 阻止修改 |
| `satisfies` | 断言检查函数 | 运行时校验 |
| `infer` | 从结构中提取信息 | 类型层面的"提取" |

---

## 练习题

1. 给定一个数组 `[1, 2, 2, 3, 3, 3]`，如何用 `Set` 去重？请写出代码。

2. 下面的代码输出什么？为什么？
   ```js
   const map = new Map();
   map.set('a', 1);
   map.set('a', 2);
   console.log(map.get('a'));
   console.log(map.size);
   ```

3. 用 `Math.random()` 和 `Math.floor()` 写一个函数 `rollDice()`，模拟掷一个六面骰子（返回 1-6 的整数）。

4. **对应 TypeScript 练习**：在 JavaScript 中，下面哪种写法最接近 TypeScript 的 `Omit<User, 'password'>`？
   ```js
   const user = { name: 'Alice', age: 25, password: 'secret' };
   
   // 选项 A
   const a = { name: user.name, age: user.age };
   
   // 选项 B
   const { password, ...b } = user;
   
   // 选项 C
   const c = { ...user };
   delete c.password;
   ```

5. **对应 TypeScript 练习**：`Object.freeze(['default', 'plan'])` 和 TypeScript 的 `as const` 有什么相似之处？又有什么区别？（提示：一个发生在编译时，一个发生在运行时）

---

> 下一章：类与面向对象基础 —— 理解 class、继承、getter/setter 和静态方法。
