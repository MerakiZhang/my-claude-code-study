# 第一章：JavaScript 入门与基础语法

> 本章目标：理解 JavaScript 是什么，学会写第一个程序，掌握变量、数据类型、数组和对象。

---

## 1.1 什么是 JavaScript？

### 生活中的比喻

想象你买了一台全新的扫地机器人。刚买回来的时候，它什么都不会——不会扫地、不会回充、不会避障。你需要通过**手机 App** 告诉它："先扫客厅""遇到障碍要绕开""电量低了要回去充电"。

**JavaScript 就是你和电脑之间的"App 语言"**。你写一段 JavaScript 代码，告诉电脑："把这句话打印出来""计算这两个数的和""把这个名字存下来"。

### 三个关键特点

1. **动态类型**：你不需要提前告诉电脑"这是一个数字"或"这是一个文字"，JavaScript 自己会判断。
   ```js
   let name = "Claude";  // JavaScript 一看：哦，这是文字（字符串）
   let age = 3;          // JavaScript 一看：哦，这是数字
   ```

2. **解释执行**：你写的代码不需要先"编译"成机器语言，而是直接一行一行地执行。就像你读小说一样，从左到右、从上到下逐行阅读。

3. **Node.js 环境**：以前 JavaScript 只能在浏览器里运行（用来让网页动起来），现在有了 **Node.js**，你可以在电脑上直接运行 JavaScript，就像运行 Python 或其他编程语言一样。Claude Code 就是在 Node.js 环境中运行的。

---

## 1.2 你的第一个 JavaScript 程序

打开你的终端（Windows 上是 PowerShell 或 Git Bash，Mac 上是 Terminal），输入：

```js
console.log("Hello, Claude Code!");
```

你会看到屏幕上打印出：

```
Hello, Claude Code!
```

### 这行代码是什么意思？

- `console` 是电脑里的"控制台"，可以理解为一块显示屏
- `.log()` 是"记录、打印"的意思
- `"Hello, Claude Code!"` 是你想打印的文字
- `;` 是语句结束符（就像中文里的句号），JavaScript 中大部分情况下可以省略

> 💡 **小技巧**：在 Node.js 中运行代码时，先创建一个文件（比如 `hello.js`），把代码写进去，然后在终端运行 `node hello.js`。

---

## 1.3 变量声明：let 和 const

### 什么是变量？

想象你有一个带标签的盒子。盒子里可以装东西，标签上写着盒子的名字。

```js
let age = 25;           // 有一个叫 age 的盒子，里面装了数字 25
const name = "Claude";  // 有一个叫 name 的盒子，里面装了文字 "Claude"
```

### let vs const

| 关键字 | 含义 | 比喻 |
|--------|------|------|
| `let` | 可以重新赋值 | 一个可重复使用的盒子，这次装苹果，下次可以装香蕉 |
| `const` | 不能重新赋值 | 一个封死的盒子，装好后就锁死了，不能再换里面的东西 |

```js
let count = 0;
count = 1;      // ✅ 可以，let 允许重新赋值
count = 2;      // ✅ 可以

const pi = 3.14;
pi = 3.15;      // ❌ 报错！const 不允许重新赋值

// ⚠️ 但是，const 对对象和数组有一点"特殊"
const user = { name: "Claude" };
user.name = "GPT";   // ✅ 可以！修改对象属性是允许的
user = {};           // ❌ 报错！不能重新赋值整个对象

const nums = [1, 2, 3];
nums.push(4);        // ✅ 可以！修改数组内容是允许的
nums = [4, 5, 6];    // ❌ 报错！不能重新赋值整个数组
```

> 💡 **为什么呢？** 对象和数组是**引用类型**。变量里保存的其实不是对象本身，而是对象的"地址"（就像一张指向房子的地图）。`const` 锁住的是"你不能换一张地图"，但你可以沿着地图找到房子，然后重新装修它。数字、字符串等**基础类型**则直接存在变量里，所以 `const` 连内容一起锁住了。

> ⚠️ **重要规则**：本教程（以及 Claude Code 源码）中**不使用 `var`**。`var` 是 JavaScript 早期的变量声明方式，有一些容易让人困惑的特性。记住：能用 `const` 就用 `const`，需要重新赋值时才用 `let`。

### 变量命名规则

- 只能包含字母、数字、下划线 `_` 和美元符 `$`
- **不能以数字开头**
- 区分大小写：`name` 和 `Name` 是两个不同的变量
- 推荐使用**驼峰命名法**（第一个单词小写，后面每个单词首字母大写）：

```js
const userName = "Claude";      // ✅ 推荐
const task_id = "abc123";       // ✅ 也可以（下划线风格）
const 123name = "error";        // ❌ 不能以数字开头
const user-name = "error";      // ❌ 不能包含横线
```

---

## 1.4 基础数据类型

数据类型就是"盒子里的东西是什么种类的"。JavaScript 有几种基础类型：

### string（字符串）：文字

字符串就是一串文字，用引号包起来：

```js
const name = "Claude";          // 双引号
const greeting = 'Hello';       // 单引号
const message = `Hi, ${name}`;  // 反引号（模板字符串），可以把变量嵌入进去

console.log(message);  // 输出：Hi, Claude
```

三种引号的区别：
- 单引号和双引号**功能完全一样**，选一种一直用下去就行
- 反引号（`` ` ``）是**模板字符串**，可以在里面用 `${变量名}` 插入变量

在 Claude Code 源码中，字符串随处可见。比如 `src/Task.ts` 中的任务类型：

```js
// 来自 src/Task.ts（去掉类型注解后的纯 JS 版本）
const TASK_ID_PREFIXES = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}
```

这里的 `'b'`、`'a'`、`'r'` 等都是字符串。

### number（数字）：整数和小数

```js
const age = 25;           // 整数
const price = 19.99;      // 小数
const negative = -5;      // 负数

console.log(age + 5);     // 输出：30
console.log(price * 2);   // 输出：39.98
```

特殊的数字值：

```js
const notANumber = NaN;   // NaN = Not a Number，表示"不是一个有效的数字"
console.log("hello" * 2); // 输出：NaN（文字不能乘数字）
```

在 Claude Code 源码中，数字常用于时间戳、token 数量等。比如 `src/Task.ts` 中的时间：

```js
// 来自 src/Task.ts（简化版）
const task = {
  id: "task_001",
  startTime: Date.now(),    // 当前时间的时间戳，是一个数字
  totalPausedMs: 0,         // 暂停的毫秒数，也是数字
}
```

### boolean（布尔值）：true 或 false

布尔值只有两个：`true`（真）或 `false`（假）。

```js
const isActive = true;
const isDeleted = false;

console.log(isActive);    // 输出：true
```

布尔值通常用来表示"是/否"的状态。比如 Claude Code 中判断任务是否已结束：

```js
// 来自 src/Task.ts（去掉类型注解后的纯 JS 版本）
function isTerminalTaskStatus(status) {
  return status === 'completed' || status === 'failed' || status === 'killed'
}

console.log(isTerminalTaskStatus('completed'));  // 输出：true
console.log(isTerminalTaskStatus('running'));    // 输出：false
```

拆解这行代码：
- `status === 'completed'` → 判断 `status` 是否等于 `'completed'`
- `||` 是"或者"的意思
- 整句意思是：如果状态是 `'completed'` **或者** `'failed'` **或者** `'killed'`，就返回 `true`

### undefined：未定义

`undefined` 表示"这个东西存在，但我还没给它赋值"。

```js
let name;
console.log(name);  // 输出：undefined

name = "Claude";
console.log(name);  // 输出：Claude
```

### null：空值

`null` 表示"这里故意什么都不放"。

```js
let user = null;    // 明确表示"现在还没有用户"
user = { name: "Claude" };  // 后来有了用户
```

`undefined` 和 `null` 的区别：
- `undefined` = "我还没想好放什么"（无意的）
- `null` = "我故意不放东西"（有意的）

### typeof 运算符

`typeof` 可以告诉你一个变量是什么类型：

```js
console.log(typeof "hello");    // 输出：string
console.log(typeof 42);         // 输出：number
console.log(typeof true);       // 输出：boolean
console.log(typeof undefined);  // 输出：undefined
console.log(typeof null);       // 输出：object（这是一个历史遗留 bug，记住就行）
```

---

## 1.5 数组（Array）

### 什么是数组？

想象你有一个带很多格子的收纳盒，每个格子里可以放一样东西。数组就是这样的"多格盒子"。

```js
const fruits = ["apple", "banana", "cherry"];
```

### 创建数组

用方括号 `[]` 创建数组，里面的每个东西叫"元素"，用逗号分隔：

```js
const empty = [];                              // 空数组
const numbers = [1, 2, 3, 4, 5];              // 数字数组
const mixed = ["hello", 42, true];            // 混合数组（不推荐，但合法）
```

### 访问元素

数组中的每个元素都有一个编号，叫做"索引"（index），**从 0 开始**：

```js
const fruits = ["apple", "banana", "cherry"];

console.log(fruits[0]);   // 输出：apple（第1个元素，索引是0）
console.log(fruits[1]);   // 输出：banana（第2个元素，索引是1）
console.log(fruits[2]);   // 输出：cherry（第3个元素，索引是2）
```

> ⚠️ **重要**：数组索引从 0 开始，不是从 1 开始！这是几乎所有编程语言的通用规则。

### 常用方法

```js
const fruits = ["apple", "banana"];

// push：在数组末尾添加元素
fruits.push("cherry");
console.log(fruits);  // 输出：["apple", "banana", "cherry"]

// pop：删除并返回数组末尾的元素
const last = fruits.pop();
console.log(last);    // 输出：cherry
console.log(fruits);  // 输出：["apple", "banana"]

// length：获取数组的长度（元素个数）
console.log(fruits.length);  // 输出：2
```

### 遍历数组：for...of

如果你想对数组中的每个元素都执行一次操作，用 `for...of`：

```js
const fruits = ["apple", "banana", "cherry"];

for (const fruit of fruits) {
  console.log(fruit);
}
// 输出：
// apple
// banana
// cherry
```

### 源码中的数组例子

在 `src/utils/treeify.ts` 中，Claude Code 用数组来收集输出的每一行文字：

```js
// 来自 src/utils/treeify.ts（去掉类型注解后的纯 JS 版本）
const lines = []

// ... 后面的代码会不断往 lines 里添加内容
lines.push(prefix + colorize(node, treeCharColors.value))
```

拆解：
- `const lines = []` → 创建一个空数组，名字叫 `lines`
- `lines.push(...)` → 把新的内容添加到数组末尾

你可以想象 `lines` 是一个记事本，代码一边执行，一边往记事本上写一行行的内容。

---

## 1.6 对象（Object）

### 什么是对象？

对象是 JavaScript 中最重要的概念之一。如果把数组比作"按编号排列的收纳格"，对象就是"带名字的标签收纳盒"——每个东西都有一个名字（键，key）和一个值（value）。

```js
const person = {
  name: "Claude",
  age: 3,
  isAI: true
};
```

> 💡 还记得 `const` 的特殊行为吗？和数组一样，对象也是**引用类型**。`const` 保证 `person` 变量本身不能被替换成另一个对象，但对象内部的属性是可以修改的。

### 创建对象

用花括号 `{}` 创建对象，里面是 `键: 值` 对，用逗号分隔：

```js
const book = {
  title: "JavaScript 入门",
  author: "张老师",
  price: 59.9,
  inStock: true
};
```

### 点号访问和括号访问

```js
const person = {
  name: "Claude",
  age: 3
};

// 点号访问（推荐，更简洁）
console.log(person.name);   // 输出：Claude

// 括号访问（当键名包含特殊字符或是变量时使用）
console.log(person["age"]); // 输出：3

const key = "name";
console.log(person[key]);   // 输出：Claude
```

### 对象方法：函数作为属性值

对象的值不仅可以是数字、字符串，还可以是**函数**：

```js
const calculator = {
  name: "简易计算器",
  add: function(a, b) {
    return a + b;
  },
  subtract: function(a, b) {
    return a - b;
  }
};

console.log(calculator.add(3, 5));       // 输出：8
console.log(calculator.subtract(10, 4)); // 输出：6
```

> 这里 `add` 和 `subtract` 是 `calculator` 对象的"方法"——也就是"属于这个对象的函数"。

#### 函数可以没有返回值

函数不一定非要返回东西。如果函数里没有写 `return`，或者只写了 `return;`，它会返回 `undefined`：

```js
function sayHello(name) {
  console.log("你好，" + name);
  // 没有 return 语句
}

const result = sayHello("Claude");
console.log(result);  // 输出：undefined
```

在 TypeScript 中，这种"不返回有意义值"的函数会被标记为 `void` 类型。在 JavaScript 里，你只需要知道：没写 `return` 的函数，返回值就是 `undefined`。

#### 访问不存在的属性

如果你访问一个对象中不存在的属性，JavaScript 会返回 `undefined`：

```js
const person = { name: "Claude" };
console.log(person.name);     // 输出：Claude
console.log(person.age);      // 输出：undefined（属性不存在）
```

这就像查字典时，字典里没有这个词，你会得到一个"空"的结果。这个特性非常重要——TypeScript 的"可选属性"（`?`）就是基于这个底层行为的。

### 源码中的对象例子

Claude Code 中大量使用了对象。来看 `src/Task.ts` 中的 `TaskHandle`：

```js
// 来自 src/Task.ts（去掉类型注解后的纯 JS 版本）
const taskHandle = {
  taskId: 'abc123',
  cleanup: () => {}
}
```

拆解：
- `taskId: 'abc123'` → 这个对象有一个属性 `taskId`，值是字符串 `'abc123'`
- `cleanup: () => {}` → 这个对象有一个属性 `cleanup`，值是一个空函数（箭头函数的写法，下一章会详细讲）

你可以把 `taskHandle` 想象成一张"任务处理卡"，上面写着任务编号（`taskId`），以及一个"清理操作"（`cleanup`）。

再看一个更复杂的例子：

```js
// 来自 src/Task.ts（简化版，去掉类型注解）
const task = {
  id: "task_001",
  type: "local_bash",
  status: "running",
  description: "执行 shell 命令",
  startTime: Date.now(),
}
```

这个对象有 5 个属性，分别描述了任务的 ID、类型、状态、描述和开始时间。

---

## 1.7 注释

注释是写给人看的文字，电脑会忽略它。好的注释能帮助你和别人理解代码。

```js
// 这是单行注释，从 // 到行尾都是注释

/*
  这是多行注释
  可以写很多行
  电脑会忽略这些内容
*/

const name = "Claude"; // 行尾也可以加注释
```

---

## 1.8 运算符

### 算术运算符

```js
const a = 10;
const b = 3;

console.log(a + b);   // 13（加）
console.log(a - b);   // 7（减）
console.log(a * b);   // 30（乘）
console.log(a / b);   // 3.333...（除）
console.log(a % b);   // 1（取余：10 除以 3 余 1）
console.log(a ** b);  // 1000（幂：10 的 3 次方）
```

### 比较运算符

```js
const a = 5;
const b = 10;

console.log(a == b);   // false（相等，不严格，不推荐）
console.log(a === b);  // false（严格相等，推荐）
console.log(a != b);   // true（不相等）
console.log(a !== b);  // true（严格不相等）
console.log(a < b);    // true（小于）
console.log(a > b);    // false（大于）
console.log(a <= b);   // true（小于等于）
console.log(a >= b);   // false（大于等于）
```

> ⚠️ **永远使用 `===` 和 `!==`**，不要用 `==` 和 `!=`。`===` 不仅比较值，还比较类型，更安全。

```js
console.log(5 == "5");   // true（不好的结果！字符串和数字被认为相等）
console.log(5 === "5");  // false（正确！类型不同）
```

### 逻辑运算符

```js
const a = true;
const b = false;

console.log(a && b);   // false（与：两个都为 true，结果才为 true）
console.log(a || b);   // true（或：只要有一个为 true，结果就为 true）
console.log(!a);       // false（非：取反）
```

逻辑运算符常用于条件判断：

```js
const isLoggedIn = true;
const isAdmin = false;

// 只有登录了 并且 是管理员，才能进入后台
if (isLoggedIn && isAdmin) {
  console.log("欢迎进入后台");
} else {
  console.log("权限不足");
}
```

### 赋值运算符

```js
let x = 10;

x += 5;   // 等价于 x = x + 5，x 现在是 15
x -= 3;   // 等价于 x = x - 3，x 现在是 12
x *= 2;   // 等价于 x = x * 2，x 现在是 24
x /= 4;   // 等价于 x = x / 4，x 现在是 6
```

---

## 1.9 本章小结

| 概念 | 含义 | 例子 |
|------|------|------|
| `let` | 可重新赋值的变量 | `let count = 0;` |
| `const` | 不可重新赋值的变量 | `const name = "Claude";` |
| `string` | 字符串（文字） | `"Hello"`, `'World'`, `` `Hi` `` |
| `number` | 数字 | `42`, `3.14` |
| `boolean` | 布尔值 | `true`, `false` |
| `undefined` | 未定义 | `let x;` |
| `null` | 空值 | `let user = null;` |
| `typeof` | 查看类型 | `typeof "hello"` → `"string"` |
| `[]` | 数组 | `const arr = [1, 2, 3];` |
| `{}` | 对象 | `const obj = { name: "Claude" };` |
| `.` / `[]` | 访问对象属性 | `obj.name`, `obj["name"]` |
| `push` | 数组末尾添加 | `arr.push(4)` |
| `pop` | 数组末尾删除 | `arr.pop()` |
| `length` | 数组长度 | `arr.length` |
| `===` | 严格相等 | `5 === 5` → `true` |
| `&&` / `\|\|` / `!` | 逻辑与/或/非 | `true && false` → `false` |

---

## 练习题

1. **变量交换**
   下面代码的输出是什么？为什么？
   ```js
   let a = 5;
   let b = 10;
   let temp = a;
   a = b;
   b = temp;
   console.log(a, b);
   ```

2. **数组操作**
   已知 `const fruits = ["apple", "banana"]`，请写代码：
   - 在末尾添加 `"cherry"`
   - 打印数组的长度
   - 用 `for...of` 遍历并打印每个水果

3. **对象访问**
   给定下面的对象：
   ```js
   const task = {
     id: "task_001",
     status: "running",
     handler: {
       name: "bash",
       pid: 12345
     }
   };
   ```
   - 用点号访问打印 `task.id`
   - 用括号访问打印 `task["status"]`
   - 打印 `task.handler.name`（嵌套对象的属性）

---

> 下一章：函数与对象 —— 我们会学习函数的三种写法、参数的高级用法、解构赋值和展开运算符，这些都是 Claude Code 源码中最常见的模式。
