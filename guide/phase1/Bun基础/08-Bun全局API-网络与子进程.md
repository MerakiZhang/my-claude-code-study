# 08 - Bun 全局 API：网络服务器与子进程管理

## 学习目标

读完本节后，你将能够：
1. 使用 `Bun.listen()` 创建高性能 TCP 服务器
2. 使用 `Bun.spawn()` 创建和管理子进程
3. 理解 `Bun.spawn()` 与 Node.js `child_process` 的区别
4. 理解网络编程中的"回退策略"
5. 读懂 Claude Code 中上游代理和子进程管理相关源码

---

## 8.1 `Bun.listen()` —— 高性能 TCP 服务器

### 什么是 TCP 服务器？

TCP（Transmission Control Protocol，传输控制协议）是互联网上最常用的通信协议。一个 TCP 服务器就是"监听"某个端口，等待其他程序（客户端）来连接的程序。

**生活中的类比**：
- TCP 服务器就像一家餐厅的服务台
- 端口号就像餐厅的桌号
- 客户端就像来就餐的顾客
- 连接建立后，双方可以持续"对话"（发送和接收数据）

### 为什么 Claude Code 需要 TCP 服务器？

Claude Code 有一个"上游代理中继"（upstream proxy relay）功能。在某些企业网络环境中，Claude Code 需要通过代理服务器连接 Anthropic 的 API。`Bun.listen()` 用于在本地启动一个 TCP 中继，把数据转发到真正的目标服务器。

### `Bun.listen()` 的基本用法

```typescript
// tcp-server-demo.ts

interface SocketState {
  id: number;
  bytesReceived: number;
}

let connectionId = 0;

const server = Bun.listen<SocketState>({
  hostname: '127.0.0.1',  // 只接受本地连接（安全）
  port: 0,                // 0 表示让操作系统分配一个空闲端口

  socket: {
    // 新连接建立时
    open(socket) {
      socket.data = {
        id: ++connectionId,
        bytesReceived: 0,
      };
      console.log(`[连接 ${socket.data.id}] 客户端已连接`);
    },

    // 收到数据时
    data(socket, data) {
      socket.data.bytesReceived += data.length;
      console.log(`[连接 ${socket.data.id}] 收到 ${data.length} 字节`);

      // 把收到的数据原样发回（回声服务器）
      socket.write(data);
    },

    // 连接关闭时
    close(socket) {
      console.log(`[连接 ${socket.data.id}] 连接已关闭，共接收 ${socket.data.bytesReceived} 字节`);
    },

    // 发生错误时
    error(socket, error) {
      console.error(`[连接 ${socket.data.id}] 错误:`, error.message);
    },
  },
});

console.log(`TCP 服务器已启动: 127.0.0.1:${server.port}`);

// 保持运行
setInterval(() => {}, 1000);
```

运行后，你可以用 `telnet` 或 `nc` 测试：

```bash
# 在另一个终端中
nc localhost <端口号>
# 输入任意文字，服务器会原样返回
```

### `Bun.listen()` 的参数详解

```typescript
Bun.listen<T>(options: {
  hostname: string;      // 监听地址，常用 '127.0.0.1'（本地）或 '0.0.0.0'（所有接口）
  port: number;          // 端口号，0 表示自动分配
  tls?: TLSOptions;      // TLS/SSL 加密配置（可选）
  socket: {
    open(socket): void;   // 连接建立回调
    data(socket, data: Buffer): void;  // 收到数据回调
    close(socket): void;  // 连接关闭回调
    error(socket, error): void;  // 错误回调
    drain?(socket): void; // 写缓冲区排空回调（可选）
    end?(socket): void;   // 对端发送 FIN 回调（可选）
  };
}): TCPServer;
```

### 与 Node.js `net.createServer` 的对比

| 特性 | `Bun.listen()` | Node.js `net.createServer()` |
|------|---------------|------------------------------|
| API 风格 | 基于回调对象 | 基于 EventEmitter |
| 性能 | 更高（原生优化）| 标准 |
| 类型安全 | 支持泛型 `<SocketState>` | 无内置泛型 |
| 每个连接的状态 | `socket.data` 属性 | 需要手动维护 Map |
| 零拷贝发送 | 支持 | 不支持 |

---

## 8.2 Claude Code 中的 `Bun.listen()` 用法

### `src/upstreamproxy/relay.ts`

这是 Claude Code 中使用 `Bun.listen()` 最核心的文件。它的作用是建立一个本地 TCP 中继，把流量转发到上游代理服务器。

```typescript
// 简化后的核心逻辑
import { createServer } from 'net'; // Node.js 回退

function createRelayServer(): Server | ReturnType<typeof Bun.listen> {
  // 优先使用 Bun.listen（高性能）
  if (typeof Bun !== 'undefined') {
    return Bun.listen<BunState>({
      hostname: '127.0.0.1',
      port: 0,
      socket: {
        open(socket) {
          // 连接建立，设置状态
          // ...
        },
        data(socket, chunk) {
          // 收到数据，转发到上游
          // ...
        },
        close(socket) {
          // 清理资源
          // ...
        },
        error(socket, err) {
          // 错误处理
          // ...
        },
      },
    });
  }

  // Node.js 回退路径
  // CCR 容器在 Node 下运行 CLI，因此需要回退
  return createServer((clientSocket) => {
    // 使用 Node.js 的 net 模块实现相同逻辑
    // ...
  });
}
```

**注释中的关键信息**：`CCR 容器在 Node 下运行 CLI，因此需要回退到 Node 的 net.createServer`。

这说明 Claude Code 的部署场景非常多样：
- 普通用户：用 Bun 编译的独立二进制运行，走 `Bun.listen()` 高性能路径
- CCR（Claude Code Remote）容器环境：可能运行在 Node.js 下，需要回退路径

---

## 8.3 `Bun.spawn()` —— 高性能子进程

### 什么是子进程？

子进程就是"程序中的程序"。当你的程序需要执行另一个程序（如 `git status`、`ripgrep pattern`）时，就创建一个子进程。

**生活中的类比**：
- 主进程就像餐厅经理
- 子进程就像经理临时雇佣的帮工
- 经理可以告诉帮工做什么（传入参数），帮工做完后汇报结果（输出数据）

### `Bun.spawn()` 的基本用法

```typescript
// spawn-basic-demo.ts

console.log('=== Bun.spawn() 基本用法 ===\n');

// 1. 运行一个简单的命令，捕获其输出
const proc1 = Bun.spawn(['echo', 'Hello from Bun.spawn!'], {
  stdout: 'pipe',
});
const output1 = await new Response(proc1.stdout).text();
console.log(`echo 输出: ${output1.trim()}`);

// 2. 运行一个需要处理输出的命令
const proc2 = Bun.spawn(['ls', '-la'], {
  stdout: 'pipe',    // 捕获标准输出
  stderr: 'pipe',    // 捕获标准错误
});

const stdout2 = await new Response(proc2.stdout).text();
console.log(`ls 输出:\n${stdout2.slice(0, 500)}...`);

// 3. 等待进程退出并获取退出码
const exitCode2 = await proc2.exited;
console.log(`ls 退出码: ${exitCode2}`);
```

### 高级用法：完整的子进程控制

```typescript
// spawn-advanced-demo.ts

async function runAdvancedSpawn() {
  // 运行一个子进程，完全控制其输入输出
  const proc = Bun.spawn(['node', '-e', `
    console.log('标准输出第1行');
    console.error('标准错误第1行');
    console.log('标准输出第2行');
  `], {
    stdout: 'pipe',
    stderr: 'pipe',
    stdin: 'pipe',     // 可以向子进程写入数据
    env: {             // 自定义环境变量
      ...process.env,
      MY_VAR: 'hello',
    },
    cwd: process.cwd(), // 设置工作目录
  });

  // 读取标准输出
  const stdout = await new Response(proc.stdout).text();
  console.log('=== 标准输出 ===');
  console.log(stdout);

  // 读取标准错误
  const stderr = await new Response(proc.stderr).text();
  console.log('=== 标准错误 ===');
  console.log(stderr);

  // 等待退出
  const exitCode = await proc.exited;
  console.log(`退出码: ${exitCode}`);
}

runAdvancedSpawn().catch(console.error);
```

### `Bun.spawn()` 的返回对象

```typescript
const proc = Bun.spawn(['command', 'arg1', 'arg2'], {
  stdout: 'pipe',
  stderr: 'pipe',
});

// proc 包含以下属性：
proc.pid        // 子进程 ID
proc.stdout     // ReadableStream（如果 stdout: 'pipe'）
proc.stderr     // ReadableStream（如果 stderr: 'pipe'）
proc.stdin      // WritableStream（如果 stdin: 'pipe'）
proc.killed     // 是否已被杀死
proc.exitCode   // 退出码（进程结束后）
proc.signalCode // 信号码（如果被信号终止）

// 等待进程结束的方法：
await proc.exited        // Promise<number>，进程结束后 resolve 退出码

// 终止进程：
proc.kill();             // 发送 SIGTERM
proc.kill('SIGKILL');    // 强制终止
```

### `Bun.spawn()` 与 Node.js `spawn()` 的对比

| 特性 | `Bun.spawn()` | Node.js `child_process.spawn()` |
|------|--------------|--------------------------------|
| API 风格 | 基于 Web Streams | 基于 EventEmitter + Stream |
| stdout/stderr | `ReadableStream`（Web 标准）| `Readable`（Node 特有）|
| 获取输出 | `new Response(proc.stdout).text()` | 监听 `data` 事件拼接字符串 |
| 进程结束 | `await proc.exited` | `proc.on('exit', cb)` |
| 默认行为 | stdout/stderr 继承父进程 | stdout/stderr 继承父进程 |
| 性能 | 更快（更少内存拷贝）| 标准 |

---

## 8.4 Claude Code 中的 `Bun.spawn()` 用法

### `src/utils/ripgrep.ts` —— 检测内嵌 ripgrep 版本

```typescript
// 简化后的逻辑
if (isInBundledMode()) {
  // bundled 模式下，ripgrep 被静态编译进 binary
  // 通过 argv0='rg' 来调度内嵌的 ripgrep
  const proc = Bun.spawn([config.command, '--version'], {
    argv0: config.argv0,  // 设置为 'rg'，让内嵌二进制知道要以 ripgrep 模式运行
    stderr: 'ignore',     // 不需要错误输出
    stdout: 'pipe',       // 捕获标准输出
  });

  // 读取子进程输出：两种方式都有效
  // 方式1（教程推荐）：new Response(proc.stdout).text()
  // 方式2（源码实际使用）：(proc.stdout as Blob).text() —— Bun 给 ReadableStream 扩展了 .text()
  const output = await new Response(proc.stdout).text();
  const version = output.trim();
  // ...
}
```

**关键点**：
- `argv0: 'rg'`：这是一个非常巧妙的技巧。Bun 编译的独立二进制文件内部可能嵌入了多个工具（如 ripgrep）。通过修改 `argv0`，可以告诉内嵌二进制"你现在应该扮演 ripgrep 的角色"。
- `stderr: 'ignore'`：不需要的错误输出直接丢弃，减少不必要的流处理
- `stdout: 'pipe'`：必须显式设置才能读取输出（Bun.spawn 默认 stdout 继承父进程）

### 子进程参数生成的差异

在 Claude Code 中，`isInBundledMode()` 影响了子进程的启动方式：

```typescript
// src/tools/shared/spawnMultiAgent.ts
// src/utils/swarm/spawnUtils.ts

function getExecutablePath(): string {
  return isInBundledMode()
    ? process.execPath    // bundled 模式：使用当前可执行文件自身
    : process.argv[1]!;   // 普通模式：使用 main.tsx 文件路径
}
```

**解释**：
- **Bundled 模式**：`claude.exe` 本身包含了所有代码。启动子进程时，再次执行 `claude.exe`，通过参数告诉它"这次你以子 agent 模式运行"。
- **普通模式**：代码分散在多个文件中。启动子进程时，需要指定入口文件 `main.tsx` 的完整路径。

### 动手实验：模拟 Claude Code 的子进程启动

```typescript
// spawn-agent-demo.ts

function getAgentCommand(): string {
  // 模拟 Claude Code 的逻辑
  if (typeof Bun !== 'undefined' && Array.isArray(Bun.embeddedFiles) && Bun.embeddedFiles.length > 0) {
    return process.execPath; // bundled 模式
  }
  return process.argv[1] || 'index.ts'; // 普通模式
}

async function spawnAgent(mode: string): Promise<void> {
  const command = getAgentCommand();
  console.log(`启动子 agent，使用命令: ${command}`);
  console.log(`运行模式: ${mode}`);

  // 模拟启动一个子进程（实际用 echo 代替）
  const proc = Bun.spawn([
    'echo',
    `模拟启动: ${command} --agent-mode ${mode}`
  ], {
    stdout: 'pipe',
  });

  const output = await new Response(proc.stdout).text();
  console.log(`输出: ${output.trim()}`);
}

console.log(`当前模式: ${typeof Bun !== 'undefined' && Bun.embeddedFiles?.length > 0 ? 'bundled' : 'normal'}`);
console.log(`process.execPath: ${process.execPath}`);
console.log(`process.argv[1]: ${process.argv[1]}`);
console.log('');

spawnAgent('reviewer').catch(console.error);
```

---

## 8.5 TCP 客户端：`Bun.connect()`

与 `Bun.listen()` 对应，`Bun.connect()` 用于创建 TCP 客户端连接：

```typescript
// tcp-client-demo.ts

async function testTcpClient() {
  const socket = await Bun.connect({
    hostname: 'example.com',
    port: 80,
    socket: {
      open(socket) {
        console.log('已连接到服务器');
        socket.write('GET / HTTP/1.1\r\nHost: example.com\r\n\r\n');
      },
      data(socket, data) {
        console.log(`收到 ${data.length} 字节`);
        console.log(data.toString().slice(0, 500));
        socket.end(); // 关闭连接
      },
      close(socket) {
        console.log('连接已关闭');
      },
      error(socket, error) {
        console.error('连接错误:', error);
      },
    },
  });
}

testTcpClient().catch(console.error);
```

> 📝 这个 API 在 Claude Code 源码中没有直接使用，但了解它有助于完整理解 Bun 的网络编程能力。

---

## 8.6 综合练习：简单的端口转发器

让我们综合运用 `Bun.listen()` 和 `Bun.connect()`，写一个端口转发器（类似 Claude Code 的上游代理中继）：

```typescript
// port-forwarder.ts

interface RelayState {
  // Bun.connect() 返回 Promise<Socket>，await 后得到 Socket 对象
  targetSocket: any | null;
  buffer: Buffer[];
}

async function createPortForwarder(localPort: number, targetHost: string, targetPort: number) {
  console.log(`启动端口转发: localhost:${localPort} → ${targetHost}:${targetPort}`);

  const server = Bun.listen<RelayState>({
    hostname: '127.0.0.1',
    port: localPort,
    socket: {
      async open(clientSocket) {
        console.log(`[${localPort}] 新客户端连接`);

        clientSocket.data = {
          targetSocket: null,
          buffer: [],
        };

        // 连接到目标服务器
        const targetSocket = await Bun.connect({
          hostname: targetHost,
          port: targetPort,
          socket: {
            open(target) {
              console.log(`[${localPort}] 已连接到目标`);
              // 发送缓冲的数据
              for (const data of clientSocket.data.buffer) {
                target.write(data);
              }
              clientSocket.data.buffer = [];
            },
            data(target, data) {
              // 把目标服务器的响应转发回客户端
              clientSocket.write(data);
            },
            close(target) {
              clientSocket.end();
            },
            error(target, err) {
              console.error(`[${localPort}] 目标错误:`, err.message);
              clientSocket.end();
            },
          },
        });

        clientSocket.data.targetSocket = targetSocket;
      },

      data(clientSocket, data) {
        if (clientSocket.data.targetSocket) {
          clientSocket.data.targetSocket.write(data);
        } else {
          // 目标连接还没建立，先缓冲数据
          clientSocket.data.buffer.push(Buffer.from(data));
        }
      },

      close(clientSocket) {
        console.log(`[${localPort}] 客户端断开`);
        clientSocket.data.targetSocket?.end();
      },

      error(clientSocket, err) {
        console.error(`[${localPort}] 客户端错误:`, err.message);
      },
    },
  });

  return server;
}

// 使用示例：把本地的 8080 端口转发到 httpbin.org 的 80 端口
// createPortForwarder(8080, 'httpbin.org', 80);

console.log('端口转发器已定义。取消注释上面的代码来运行。');
console.log('运行后可以用: curl http://localhost:8080/get');
```

---

## 小结

| API | 用途 | 关键特性 | Claude Code 中的场景 |
|-----|------|---------|---------------------|
| `Bun.listen()` | 创建 TCP 服务器 | 高性能、类型安全、零拷贝 | 上游代理中继 |
| `Bun.connect()` | 创建 TCP 客户端 | 异步连接、基于 Promise | 端口转发 |
| `Bun.spawn()` | 创建子进程 | Web Streams、高性能 | ripgrep 版本检测、子 agent 启动 |

| 子进程配置项 | 说明 |
|-------------|------|
| `stdout: 'pipe'` | 捕获标准输出 |
| `stderr: 'pipe'` | 捕获标准错误 |
| `stdin: 'pipe'` | 允许向子进程写入 |
| `argv0: 'xxx'` | 设置进程看到的 argv[0] |
| `env: {...}` | 自定义环境变量 |
| `cwd: '...'` | 设置工作目录 |

| 设计模式 | 说明 |
|---------|------|
| 服务器/客户端对称 API | `listen()` 和 `connect()` 结构相似 |
| socket.data | 每个连接的状态存储 |
| 数据缓冲 | 连接未就绪时先缓冲数据 |
| 双向转发 | 中继服务器的核心逻辑 |

---

## 练习题

1. 编写 `echo-server.ts`，用 `Bun.listen()` 创建一个回声服务器。客户端发送什么，服务器就返回什么。支持同时处理多个客户端连接。
2. 编写 `command-runner.ts`，用 `Bun.spawn()` 执行 `git log --oneline -10`，解析输出并打印每条提交的哈希和前 50 个字符的说明。
3. 编写 `health-checker.ts`，并行对多个服务（如 `localhost:3000`、`localhost:8080`）做 TCP 连接检查，报告每个服务的连通状态。
4. （思考题）Claude Code 的 `src/upstreamproxy/relay.ts` 中为什么需要 Node.js 回退路径？什么场景下会走到回退路径？

> 参考答案：Claude Code 的某些部署环境（如 CCR - Claude Code Remote 容器）可能使用 Node.js 运行 CLI，而不是 Bun。为了保证在这些环境下上游代理功能仍然可用，源码提供了基于 Node.js `net.createServer` 的回退实现。这种设计确保了功能在各种运行环境下的可用性。
