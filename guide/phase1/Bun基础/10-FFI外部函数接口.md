# 10 - FFI：让 JavaScript 直接调用 C 语言库

## 学习目标

读完本节后，你将能够：
1. 理解什么是 FFI（外部函数接口）
2. 使用 `bun:ffi` 模块调用 C 语言写的系统库
3. 理解 `dlopen` 和函数签名的概念
4. 读懂 Claude Code 中调用 `libc` 的源码
5. 理解 FFI 的安全风险和适用场景

---

## 10.1 什么是 FFI？

### 类比：翻译官

想象你是一位中国商人（JavaScript），要去和一个法国工厂（C 语言写的系统库）谈生意。你们语言不通，需要一个翻译官（FFI）在中间传话。

- **你（JavaScript）**：说"调用 prctl 函数，参数是 1 和 0"
- **翻译官（FFI）**：把这句话翻译成法语（C 语言的调用约定），告诉法国工厂
- **法国工厂（C 库）**：执行操作，返回结果
- **翻译官（FFI）**：把法国人的回答翻译回中文，告诉你

### 编程中的定义

FFI（Foreign Function Interface，外部函数接口）是一种机制，允许一种编程语言调用另一种编程语言编写的函数。

JavaScript 是高级语言，运行在虚拟机中，不能直接操作内存、硬件。但操作系统提供了很多底层功能（如进程控制、内存管理、硬件访问），这些功能是用 C 语言写的。

**FFI 让 JavaScript 可以"借用"这些底层能力。**

---

## 10.2 为什么 Claude Code 需要 FFI？

Claude Code 是一个复杂的命令行工具，在某些场景下需要调用操作系统底层功能：

在 `src/upstreamproxy/upstreamproxy.ts` 中：

```typescript
// Linux 上设置进程内存转储保护
const ffi = require('bun:ffi') as typeof import('bun:ffi');
const lib = ffi.dlopen('libc.so.6', {
  prctl: {
    args: [ffi.FFIType.int, ffi.FFIType.int],
    returns: ffi.FFIType.int,
  },
});

// 调用 prctl(PR_SET_DUMPABLE, 0)
// 禁止其他进程读取本进程的内存转储（安全设置）
lib.symbols.prctl(4, 0); // 4 = PR_SET_DUMPABLE
```

**场景解释**：在企业环境中，Claude Code 可能处理敏感代码和数据。通过 `prctl` 系统调用设置进程不可被转储（dump），可以防止恶意程序通过内存转储窃取数据。

---

## 10.3 `bun:ffi` 核心概念

### `dlopen` —— 打开动态链接库

在 Linux/macOS 上，系统库通常是 `.so`（Linux）或 `.dylib`（macOS）文件。`dlopen` 就是"打开"这些文件，让你能调用里面的函数。

```typescript
import { dlopen, FFIType } from 'bun:ffi';

// 打开 C 标准库
const libc = dlopen('libc.so.6', {
  // 列出你想调用的函数及其签名
});
```

### 函数签名 —— 告诉 FFI 如何翻译

C 函数有固定的参数类型和返回值类型。FFI 需要知道这些信息，才能正确地在 JavaScript 和 C 之间转换数据。

```typescript
const lib = dlopen('libc.so.6', {
  getpid: {
    args: [],              // 没有参数
    returns: FFIType.int,  // 返回整数（进程 ID）
  },
  strlen: {
    args: [FFIType.cstring],  // 一个参数：C 字符串
    returns: FFIType.int,     // 返回整数（字符串长度）
  },
});
```

### `FFIType` 类型对照表

| `FFIType` | C 类型 | JavaScript 类型 | 说明 |
|-----------|--------|----------------|------|
| `FFIType.int` | `int` | `number` | 32 位有符号整数 |
| `FFIType.uint` | `unsigned int` | `number` | 32 位无符号整数 |
| `FFIType.long` | `long` | `number` / `bigint` | 64 位整数（平台相关）|
| `FFIType.float` | `float` | `number` | 32 位浮点数 |
| `FFIType.double` | `double` | `number` | 64 位浮点数 |
| `FFIType.bool` | `bool` | `boolean` | 布尔值 |
| `FFIType.cstring` | `char*` | `string` | C 字符串（以 `\0` 结尾）|
| `FFIType.pointer` | `void*` | `number` / `BigInt` | 指针地址 |
| `FFIType.int8_t` | `int8_t` | `number` | 8 位有符号整数 |
| `FFIType.uint8_t` | `uint8_t` | `number` | 8 位无符号整数 |
| `FFIType.int64_t` | `int64_t` | `BigInt` | 64 位有符号整数 |

---

## 10.4 动手实验：调用 C 标准库

### 实验 1：获取进程 ID

```typescript
// ffi-getpid.ts

// 注意：FFI 只在 Bun 中可用
if (typeof Bun === 'undefined') {
  console.log('FFI 示例需要 Bun 运行时');
  process.exit(1);
}

// 根据平台选择库文件名
const libName = process.platform === 'darwin'
  ? 'libSystem.dylib'      // macOS
  : process.platform === 'win32'
  ? 'kernel32.dll'         // Windows（简化，实际需不同 API）
  : 'libc.so.6';           // Linux

const { dlopen, FFIType } = require('bun:ffi');

const libc = dlopen(libName, {
  getpid: {
    args: [],
    returns: FFIType.int,
  },
  getppid: {
    args: [],
    returns: FFIType.int,
  },
});

const pid = libc.symbols.getpid();
const ppid = libc.symbols.getppid();

console.log(`当前进程 ID (PID): ${pid}`);
console.log(`父进程 ID (PPID): ${ppid}`);

// 验证：和 Node.js/Bun 的 process.pid 对比
console.log(`process.pid 验证: ${process.pid}`);
console.log(`process.ppid 验证: ${process.ppid}`);
```

### 实验 2：计算字符串长度

```typescript
// ffi-strlen.ts

const { dlopen, FFIType } = require('bun:ffi');

const libName = process.platform === 'darwin' ? 'libSystem.dylib' : 'libc.so.6';

const libc = dlopen(libName, {
  strlen: {
    args: [FFIType.cstring],
    returns: FFIType.int,
  },
});

const testStrings = ['Hello', '你好，世界！', 'Bun FFI is powerful'];

for (const str of testStrings) {
  const cLength = libc.symbols.strlen(str);
  const jsLength = str.length;
  console.log(`"${str}": C strlen=${cLength}, JS length=${jsLength}`);
}
```

> 💡 **注意差异**：对于中文，`strlen` 返回的是字节数（UTF-8 编码下每个汉字占 3 字节），而 JavaScript 的 `length` 返回的是字符数。

### 实验 3：获取当前时间（gettimeofday）

```typescript
// ffi-time.ts

const { dlopen, FFIType, ptr } = require('bun:ffi');

const libName = process.platform === 'darwin' ? 'libSystem.dylib' : 'libc.so.6';

// gettimeofday 需要传入一个结构体指针
// struct timeval {
//   long tv_sec;   // 秒
//   long tv_usec;  // 微秒
// };

const libc = dlopen(libName, {
  gettimeofday: {
    args: [FFIType.pointer, FFIType.pointer],  // tv, tz
    returns: FFIType.int,
  },
});

// 分配一个能容纳两个 long 的内存区域（16 字节，假设 64 位系统）
const tvBuffer = new BigInt64Array(2);  // [tv_sec, tv_usec]
const tvPtr = ptr(tvBuffer);

const result = libc.symbols.gettimeofday(tvPtr, 0);  // tz 传 null

if (result === 0) {
  const seconds = Number(tvBuffer[0]);
  const microseconds = Number(tvBuffer[1]);
  const milliseconds = seconds * 1000 + microseconds / 1000;
  const date = new Date(milliseconds);

  console.log(`C gettimeofday 结果:`);
  console.log(`  秒: ${seconds}`);
  console.log(`  微秒: ${microseconds}`);
  console.log(`  日期: ${date.toISOString()}`);
} else {
  console.log('gettimeofday 调用失败');
}

// 和 JS 的 Date.now() 对比
console.log(`\nJS Date.now() 对比: ${new Date().toISOString()}`);
```

---

## 10.5 Claude Code 中的 FFI 用法详解

### 完整源码分析

```typescript
// src/upstreamproxy/upstreamproxy.ts（高度简化的教学示意）
// ⚠️ 注意：真实的 prctl 签名更复杂（5个参数：int + 4*u64），
// 这里简化为2个参数以便理解核心概念。

// Linux 上设置进程内存转储保护
function setDumpable(dumpable: boolean): void {
  if (process.platform !== 'linux') {
    return; // 只在 Linux 上需要这个设置
  }

  try {
    // 使用 require 而不是 import，因为 bun:ffi 是 Bun 专属模块
    // 在 Node.js 环境中这段代码不会执行（有平台判断保护）
    const ffi = require('bun:ffi') as typeof import('bun:ffi');

    // 打开 C 标准库
    // 真实源码中 args/returns 使用字符串字面量（如 'int', 'u64'），
    // FFIType.int 和 'int' 两种写法在 Bun 中等价
    const lib = ffi.dlopen('libc.so.6', {
      prctl: {
        // 简化示意：prctl 实际接受可变数量的参数
        args: [ffi.FFIType.int, ffi.FFIType.int],
        returns: ffi.FFIType.int,
      },
    });

    // PR_SET_DUMPABLE = 4
    // 0 = 禁止转储，1 = 允许转储
    const PR_SET_DUMPABLE = 4;
    const result = lib.symbols.prctl(PR_SET_DUMPABLE, dumpable ? 1 : 0);

    if (result !== 0) {
      console.warn(`prctl(PR_SET_DUMPABLE) 失败，返回码: ${result}`);
    }
  } catch (err) {
    // FFI 调用可能失败（如权限不足），优雅地忽略
    console.warn('无法设置进程转储保护:', err);
  }
}
```

### 这段代码的工程智慧

1. **平台保护**：`process.platform !== 'linux'` 时直接返回，避免在非 Linux 系统上出错
2. **try-catch 保护**：FFI 调用涉及底层系统调用，可能因权限、库缺失等原因失败
3. **使用 `require` 而非 `import`**：
   - `import` 是静态的，在模块加载时就解析
   - `require` 是动态的，在代码执行到这行时才加载
   - 如果这行代码不会执行（因为平台判断），模块加载时不会报错
4. **类型断言**：`as typeof import('bun:ffi')` 让 TypeScript 知道 `ffi` 的类型，提供代码提示

---

## 10.6 FFI 的安全注意事项

### ⚠️ 危险操作

FFI 非常强大，但也极其危险：

| 风险 | 说明 |
|------|------|
| **内存崩溃** | C 语言没有边界检查，错误的指针操作会导致段错误（Segmentation Fault），整个程序崩溃 |
| **安全漏洞** | 缓冲区溢出、格式化字符串漏洞等 C 语言经典问题，在 FFI 中同样存在 |
| **平台依赖** | `libc.so.6` 是 Linux 特有的，macOS 和 Windows 需要不同的库 |
| **ABI 不兼容** | 参数类型、调用约定不匹配会导致未定义行为 |

### ✅ 安全使用原则

1. **只在必要时使用**：能用 JavaScript/Bun API 实现的，不要走 FFI
2. **充分测试**：在不同平台和版本上测试
3. **错误处理**：始终用 `try-catch` 包裹 FFI 调用
4. **平台检查**：用 `process.platform` 判断当前系统
5. **类型匹配**：确保 `FFIType` 和 C 函数的参数类型完全一致

---

## 10.7 FFI vs 其他方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| FFI (`bun:ffi`) | 零开销，直接调用 | 危险，平台依赖 | 调用系统底层 API |
| 子进程 (`Bun.spawn`) | 安全，隔离性好 | 有进程创建开销 | 调用外部命令行工具 |
| NAPI / Node-API | 稳定，生态成熟 | 需要编译 C++ 插件 | 编写可复用的原生扩展 |
| WASM | 安全，跨平台 | 有运行时开销 | 高性能计算，游戏引擎 |

Claude Code 的 `src/tools/FileReadTool/imageProcessor.ts` 中，使用了 **NAPI** 模块（`image-processor-napi`）来处理图像。这说明 FFI 和 NAPI 在项目中是互补的：
- **FFI**：快速调用简单的系统函数（如 `prctl`）
- **NAPI**：调用复杂的第三方原生库（如图像处理）

---

## 10.8 综合练习：系统信息获取器

```typescript
// sysinfo-ffi.ts

// 注意：此示例仅限 Linux/macOS

if (process.platform === 'win32') {
  console.log('此示例不支持 Windows');
  process.exit(1);
}

if (typeof Bun === 'undefined') {
  console.log('需要 Bun 运行时');
  process.exit(1);
}

const { dlopen, FFIType } = require('bun:ffi');
const libName = process.platform === 'darwin' ? 'libSystem.dylib' : 'libc.so.6';

const libc = dlopen(libName, {
  getpid: {
    args: [],
    returns: FFIType.int,
  },
  getuid: {
    args: [],
    returns: FFIType.int,
  },
  getgid: {
    args: [],
    returns: FFIType.int,
  },
});

console.log('=== 系统信息（通过 FFI）===');
console.log(`进程 ID: ${libc.symbols.getpid()}`);
console.log(`用户 ID (UID): ${libc.symbols.getuid()}`);
console.log(`组 ID (GID): ${libc.symbols.getgid()}`);

// 和 process 对象对比
console.log('\n=== 验证（通过 process 对象）===');
console.log(`process.pid: ${process.pid}`);
console.log(`process.uid: ${process.getuid?.() || 'N/A'}`);
console.log(`process.gid: ${process.getgid?.() || 'N/A'}`);
```

---

## 小结

| 概念 | 解释 |
|------|------|
| FFI | Foreign Function Interface，允许 JavaScript 调用 C 函数 |
| `dlopen` | 打开动态链接库文件（`.so` / `.dylib` / `.dll`）|
| 函数签名 | 描述 C 函数的参数类型和返回值类型 |
| `FFIType` | Bun FFI 的类型系统，用于 JS 类型和 C 类型的映射 |
| `prctl` | Linux 进程控制系统调用 |
| PR_SET_DUMPABLE | 控制进程是否允许被转储（内存快照）|

| 工程实践 | 说明 |
|---------|------|
| `require` 动态加载 | 避免在模块加载时就解析 Bun 专属模块 |
| `try-catch` | FFI 调用可能失败，必须做好错误处理 |
| `process.platform` 检查 | 避免在不支持的平台上调用平台专属 API |
| `as typeof import(...)` | TypeScript 类型断言，保持类型安全 |

---

## 练习题

1. 编写 `ffi-clock.ts`，使用 FFI 调用 C 的 `clock()` 函数获取 CPU 时间，和 JavaScript 的 `performance.now()` 对比。
2. 编写 `ffi-safe-wrapper.ts`，封装一个 `safeFfiCall<T>(fn: () => T): T | null` 函数，自动处理 try-catch 和平台检查。
3. （思考题）Claude Code 为什么在 `src/upstreamproxy/upstreamproxy.ts` 中使用 `require('bun:ffi')` 而不是 `import { dlopen } from 'bun:ffi'`？

> 参考答案：`import` 是静态导入，在模块加载时就会尝试解析 `bun:ffi`。如果这段代码在 Node.js 环境中运行（虽然此处有 `process.platform !== 'linux'` 保护，但 `import` 会在模块加载阶段就执行），会导致模块找不到的错误。`require` 是动态导入，只有代码执行到这行时才会加载模块。由于前面有平台判断和 Bun 检测，这段代码在不需要的环境中不会执行，因此不会报错。这种写法是"防御性编程"的体现。
