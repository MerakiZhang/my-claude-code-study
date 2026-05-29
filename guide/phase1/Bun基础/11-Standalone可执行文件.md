# 11 - Standalone 可执行文件：把代码打包成单个文件

## 学习目标

读完本节后，你将能够：
1. 理解什么是 Standalone Executable（独立可执行文件）
2. 使用 `bun build --compile` 打包项目
3. 使用 `Bun.embeddedFiles` 嵌入额外文件
4. 理解 `process.execPath` 和 `process.argv[1]` 在 bundled 模式下的区别
5. 读懂 Claude Code 中 bundled mode 相关的全部源码逻辑

---

## 11.1 什么是 Standalone Executable？

### 类比：便携式工具箱

想象你是一位维修工程师。传统方式下，你去客户家维修需要带：
- 一个装满工具的大工具箱（`node_modules`）
- 一本厚厚的说明书（源码文件）
- 各种配件和替换零件（配置文件、资源文件）

**Standalone Executable 就像是一把"瑞士军刀"** —— 所有工具都折叠在刀柄里，你只需要带这一把刀就够了。

### 编程中的定义

Standalone Executable 是一个**单独的文件**，包含了运行程序所需的一切：
- JavaScript/TypeScript 代码（编译后的）
- 运行时引擎（Bun 的核心）
- 静态资源（图片、配置文件等）
- 原生模块（可选）

用户不需要安装 Bun、Node.js 或任何依赖。双击这个文件（或在终端中运行），程序就能工作。

### 常见场景

| 场景 | 说明 |
|------|------|
| CLI 工具发布 | 用户下载一个文件就能用，不需要 `npm install` |
| 桌面应用 | Electron/Tauri 应用的内核 |
| 嵌入式系统 | 资源受限的环境，无法安装完整的运行时 |
| 企业部署 | IT 部门可以精确控制运行的软件版本 |

---

## 11.2 `bun build --compile` 基础用法

### 最简单的打包

假设你有一个 `hello.ts`：

```typescript
// hello.ts
console.log('Hello from standalone executable!');
console.log(`运行时: ${process.versions.bun ? 'Bun' : 'Node.js'}`);
```

打包成可执行文件：

```bash
bun build --compile hello.ts --outfile hello
```

运行：

```bash
./hello
# 输出：Hello from standalone executable!
```

> 💡 **Windows 用户**：可执行文件会自动加上 `.exe` 后缀，即 `hello.exe`。

### 打包带依赖的项目

```bash
# 假设项目有 package.json 和多个源文件
bun build --compile src/main.tsx --outfile mycli
```

Bun 会自动：
1. 解析 `src/main.tsx` 的导入树
2. 把所有依赖的代码打包成一个文件
3. 把 Bun 运行时嵌入进去
4. 生成平台原生的可执行文件

### 打包选项

```bash
# 指定目标平台（交叉编译）
bun build --compile --target=bun-linux-x64 main.ts --outfile mycli-linux
bun build --compile --target=bun-darwin-arm64 main.ts --outfile mycli-mac
bun build --compile --target=bun-windows-x64 main.ts --outfile mycli.exe

# 生成更小的文件（实验性）
bun build --compile --minify main.ts --outfile mycli

# 嵌入额外的文件
bun build --compile --embed ./assets/logo.png --embed ./config.json main.ts --outfile mycli
```

---

## 11.3 `Bun.embeddedFiles` —— 访问嵌入的文件

### 什么是嵌入文件？

打包时，你可以把额外的文件"嵌入"到可执行文件中。这些文件不会出现在文件系统里，而是存储在可执行文件内部。

**类比**：就像 MP3 音乐文件中可以嵌入专辑封面图片一样，图片不是单独的文件，而是藏在 MP3 文件内部。

### 嵌入文件

```bash
bun build --compile \
  --embed ./ripgrep \
  --embed ./config.yaml \
  --embed ./templates/welcome.txt \
  main.ts --outfile myapp
```

### 访问嵌入文件

```typescript
// main.ts

// Bun.embeddedFiles 是一个数组，包含所有嵌入的文件信息
if (typeof Bun !== 'undefined' && Array.isArray(Bun.embeddedFiles)) {
  console.log(`嵌入文件数量: ${Bun.embeddedFiles.length}`);

  for (const file of Bun.embeddedFiles) {
    console.log(`- ${file.name} (${file.size} 字节)`);

    // 读取嵌入文件的内容
    const content = await file.text();
    console.log(`  内容预览: ${content.slice(0, 100)}...`);
  }
}
```

### `EmbeddedFile` 对象的属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | 文件的原始路径 |
| `size` | `number` | 文件大小（字节）|
| `text()` | `Promise<string>` | 读取文件内容为字符串 |
| `json()` | `Promise<any>` | 读取文件并解析为 JSON |
| `arrayBuffer()` | `Promise<ArrayBuffer>` | 读取文件内容为二进制 |
| `stream()` | `ReadableStream` | 读取文件为流 |

---

## 11.4 `isInBundledMode()` —— 检测是否以独立可执行文件运行

这是 Claude Code 中最重要的 bundled mode 检测逻辑：

```typescript
// src/utils/bundledMode.ts

/**
 * 检测当前运行时是否为 Bun。
 */
export function isRunningWithBun(): boolean {
  return process.versions.bun !== undefined;
}

/**
 * 检测当前是否以 Bun 编译的独立可执行文件运行。
 * 通过检查是否存在嵌入的文件来判断。
 */
export function isInBundledMode(): boolean {
  return (
    typeof Bun !== 'undefined' &&
    Array.isArray(Bun.embeddedFiles) &&
    Bun.embeddedFiles.length > 0
  );
}
```

### 为什么用三重检查？

```typescript
typeof Bun !== 'undefined' &&      // 1. 确保是 Bun 运行时
Array.isArray(Bun.embeddedFiles) && // 2. 确保属性存在且是数组
Bun.embeddedFiles.length > 0         // 3. 确保确实有嵌入的文件
```

这三重检查是"防御性编程"的典范：
- 如果只检查 `typeof Bun !== 'undefined'`，普通 `bun run` 也会返回 `true`
- 如果只检查 `Bun.embeddedFiles`，在 Node.js 下会抛出 `ReferenceError`
- 三重检查确保只有在"Bun 运行时 + 编译产物 + 有嵌入文件"时才返回 `true`

---

## 11.5 Claude Code 中的 Bundled Mode 应用

### 应用 1：启动子 Agent 的路径选择

```typescript
// src/tools/shared/spawnMultiAgent.ts
// src/utils/swarm/spawnUtils.ts

function getExecutablePath(): string {
  return isInBundledMode()
    ? process.execPath      // bundled: 使用当前可执行文件自身，如 /usr/bin/claude
    : process.argv[1]!;     // 普通: 使用入口文件，如 /path/to/src/main.tsx
}
```

**场景**：Claude Code 的 Multi-Agent 功能需要启动子进程来运行其他 agent。子进程需要知道"我要运行哪个程序"。

- **Bundled 模式**：`claude.exe` 本身包含了所有逻辑。启动子进程就是再执行一次 `claude.exe`，通过参数区分主进程和子 agent。
- **普通模式**：代码分散在多个文件中。启动子进程需要指定 TypeScript 入口文件的路径。

### 应用 2：内嵌 ripgrep

```typescript
// src/utils/ripgrep.ts（简化）

function getRipgrepConfig() {
  if (isInBundledMode()) {
    // bundled 模式下，ripgrep 被静态编译进 binary
    // 通过设置 argv0='rg' 来告诉内嵌二进制以 ripgrep 模式运行
    return {
      mode: 'embedded',
      command: process.execPath,  // 运行自己
      args: ['--no-config'],
      argv0: 'rg',               // 伪装成 rg
    };
  }

  // 普通模式：在 PATH 中查找 rg
  return {
    mode: 'system',
    command: 'rg',
    args: ['--no-config'],
    argv0: undefined,
  };
}
```

> 💡 **`argv0` 是什么？** 在 Unix 系统中，程序可以通过 `argv[0]`（程序名）来判断自己应该以什么模式运行。Claude Code 的编译产物内部可能嵌入了多个工具（主程序、ripgrep 等），它们通过 `argv0` 来识别自己的身份。

### 应用 3：内嵌原生图像处理器

```typescript
// src/tools/FileReadTool/imageProcessor.ts（简化）

async function loadImageProcessor() {
  if (isInBundledMode()) {
    // bundled 模式下优先加载原生图像处理模块
    try {
      const imageProcessor = await import('image-processor-napi');
      return imageProcessor;
    } catch {
      console.warn('原生图像处理器不可用，回退到 sharp');
    }
  }

  // 普通模式或非 bundled 回退：使用 npm 的 sharp 包
  const sharp = await import('sharp');
  return sharp.default;
}
```

### 应用 4：诊断工具中的安装类型检测

```typescript
// src/utils/doctorDiagnostic.ts（简化）

function detectInstallationType(): string {
  if (isInBundledMode()) {
    // 检查是否通过包管理器安装的
    if (detectHomebrew() || detectWinget() || detectMise()) {
      return 'package-manager';  // 通过 brew/winget/mise 安装
    }
    return 'native';              // 直接下载的二进制文件
  }

  // 非 bundled 模式
  return 'source';                // 从源码运行
}
```

### 应用 5：跳过 npm 弃用通知

```typescript
// src/hooks/notifs/useNpmDeprecationNotification.tsx（简化）

function useNpmDeprecationNotification() {
  if (isInBundledMode()) {
    // Native binary 不需要 npm 弃用通知
    return null;
  }

  // 从 npm/bun 运行时：检查是否需要显示弃用警告
  // ...
}
```

### 应用 6：Computer Use MCP 参数传递

```typescript
// src/utils/computerUse/setup.ts（简化）

function getComputerUseArgs(): string[] {
  return isInBundledMode()
    ? ['--computer-use-mcp']           // bundled: 直接传 flag
    : [join(__dirname, 'cli.js'), '--computer-use-mcp'];  // 普通: 指定 cli.js 路径
}
```

---

## 11.6 `process.execPath` vs `process.argv[1]`

这是在 bundled mode 编程中最容易混淆的两个概念。

### 定义

| 属性 | 普通模式 (`bun run main.ts`) | Bundled 模式 (`./claude`) |
|------|---------------------------|--------------------------|
| `process.execPath` | `/path/to/bun`（Bun 可执行文件的路径）| `/path/to/claude`（可执行文件本身）|
| `process.argv[0]` | `/path/to/bun` | `/path/to/claude` |
| `process.argv[1]` | `/path/to/main.ts`（入口脚本）| `undefined`（没有脚本文件）|

### 实验验证

创建 `argv-demo.ts`：

```typescript
// argv-demo.ts

console.log('=== 进程参数信息 ===');
console.log(`process.execPath: ${process.execPath}`);
console.log(`process.argv[0]: ${process.argv[0]}`);
console.log(`process.argv[1]: ${process.argv[1]}`);
console.log(`process.argv: ${JSON.stringify(process.argv)}`);
console.log(`isInBundledMode: ${typeof Bun !== 'undefined' && Array.isArray(Bun.embeddedFiles) && Bun.embeddedFiles.length > 0}`);
```

分别用两种方式运行：

```bash
# 方式 1：普通运行
bun argv-demo.ts

# 方式 2：编译后运行
bun build --compile argv-demo.ts --outfile argv-demo
./argv-demo
```

对比输出，你会清楚看到差异。

### Claude Code 中的应用

```typescript
// src/bridge/bridgeMain.ts（简化）

function getSpawnArgs(): string[] {
  if (isInBundledMode() || !process.argv[1]) {
    // bundled 模式下 process.argv[1] 不存在
    // 直接返回空数组，子进程用 process.execPath 启动
    return [];
  }

  // 普通模式：返回入口文件路径作为子进程的参数
  return [process.argv[1]];
}
```

---

## 11.7 综合练习：创建一个可部署的 CLI 工具

### 项目结构

```
my-cli/
├── package.json
├── src/
│   ├── main.ts
│   └── utils.ts
├── assets/
│   └── welcome.txt
└── build.ts
```

### 代码

```typescript
// src/utils.ts
export function getRuntimeInfo() {
  return {
    isBun: process.versions.bun !== undefined,
    isBundled: typeof Bun !== 'undefined' && Array.isArray(Bun.embeddedFiles) && Bun.embeddedFiles.length > 0,
    execPath: process.execPath,
    scriptPath: process.argv[1] || 'N/A (bundled)',
  };
}

// src/main.ts
import { getRuntimeInfo } from './utils';

async function main() {
  const info = getRuntimeInfo();

  console.log('╔════════════════════════════════════╗');
  console.log('║     My CLI Tool v1.0.0             ║');
  console.log('╚════════════════════════════════════╝');
  console.log();

  console.log('运行时信息:');
  console.log(`  Bun 运行时: ${info.isBun ? '✅' : '❌'}`);
  console.log(`  Bundled 模式: ${info.isBundled ? '✅' : '❌'}`);
  console.log(`  可执行文件: ${info.execPath}`);
  console.log(`  脚本路径: ${info.scriptPath}`);

  // 如果有嵌入文件，列出它们
  if (info.isBundled && Bun.embeddedFiles) {
    console.log('\n嵌入文件:');
    for (const file of Bun.embeddedFiles) {
      console.log(`  📦 ${file.name}`);
    }
  }
}

main().catch(console.error);

// build.ts
// Bun.$ 是 Bun 的 shell 脚本功能（Bun v1.1+），可以直接执行模板字符串中的命令
import { $ } from 'bun';

async function build() {
  console.log('开始打包...');

  // 使用 Bun.$ 执行打包命令，自动处理输出和错误
  await $`bun build --compile \
    --embed ./assets/welcome.txt \
    ./src/main.ts \
    --outfile ./dist/my-cli`;

  console.log('打包完成！');
  console.log('运行 ./dist/my-cli 测试');
}

build().catch(console.error);
```

### 打包和运行

```bash
# 创建目录结构
mkdir -p my-cli/src my-cli/assets my-cli/dist
cd my-cli

# 写入上面的代码...

# 打包
bun run build.ts

# 运行
./dist/my-cli
```

---

## 小结

| 概念 | 解释 |
|------|------|
| Standalone Executable | 独立可执行文件，包含所有运行时和代码 |
| `bun build --compile` | Bun 的编译命令，生成独立可执行文件 |
| `Bun.embeddedFiles` | 嵌入到可执行文件中的资源文件列表 |
| `isInBundledMode()` | 检测当前是否以独立可执行文件运行 |
| `process.execPath` | 当前可执行文件的完整路径 |
| `process.argv[1]` | 入口脚本路径（bundled 模式下为 `undefined`）|
| `argv0` | 进程看到的程序名，用于多模式二进制 |

| Claude Code 中的 bundled mode 应用 | 说明 |
|-----------------------------------|------|
| 子 Agent 启动路径 | bundled 用 `process.execPath`，普通用 `process.argv[1]` |
| 内嵌 ripgrep | 通过 `argv0='rg'` 调度内嵌搜索工具 |
| 原生图像处理器 | bundled 下优先加载 `image-processor-napi` |
| 安装类型诊断 | 区分 native binary / package-manager / source |
| npm 弃用通知 | bundled 模式跳过 |
| MCP 参数传递 | bundled 直接传 flag，普通需指定 cli.js 路径 |

---

## 练习题

1. 编写 `self-info.ts`，打印当前进程的所有运行时信息（`process.execPath`、`process.argv`、`process.versions.bun`、`Bun.embeddedFiles` 等），然后分别用 `bun run` 和 `bun build --compile` 运行，对比输出差异。
2. 编写 `embed-demo.ts`，创建几个测试文件（`config.json`、`template.txt`），用 `bun build --compile --embed` 打包，在程序中读取并打印嵌入文件的内容。
3. 编写 `multi-mode-binary.ts`，模拟 Claude Code 的 `argv0` 技巧。程序根据 `process.argv0` 的值输出不同信息（如果是 `rg` 模式就模拟 ripgrep，如果是默认模式就显示主程序信息）。
4. （思考题）为什么 `src/bridge/bridgeMain.ts` 中判断 bundled mode 的条件是 `isInBundledMode() || !process.argv[1]`？`!process.argv[1]` 这个条件在什么情况下会触发？

> 参考答案：`!process.argv[1]` 处理了一种边缘情况——即使由于某些原因 `Bun.embeddedFiles` 检测不准确（例如未来的 Bun 版本改变了这个 API），只要 `process.argv[1]` 不存在，就可以安全地假设当前是 bundled 模式。这增加了一层防御。`process.argv[1]` 不存在的情况包括：直接运行编译后的可执行文件、某些特殊的启动方式。
