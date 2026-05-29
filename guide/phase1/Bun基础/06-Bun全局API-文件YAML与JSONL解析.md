# 06 - Bun 全局 API：文件操作、YAML 与 JSONL 解析

## 学习目标

读完本节后，你将能够：
1. 使用 `Bun.file()` 和 `Bun.write()` 高效读写文件
2. 使用 `Bun.YAML.parse()` 解析 YAML 配置文件
3. 理解 JSONL 格式并使用 `Bun.JSONL.parseChunk()` 解析
4. 理解为什么这些 API 都有 Node.js 回退方案
5. 读懂 Claude Code 中文件解析相关源码

---

## 6.1 `Bun.file()` —— 高性能文件读取

### 传统方式 vs Bun 方式

在 Node.js 中，读取文件通常这样做：

```typescript
import { readFileSync } from 'fs';

// Node.js 方式：一次性把整个文件读进内存
const content = readFileSync('./data.txt', 'utf-8');
console.log(content);
```

Bun 提供了更现代的方式：

```typescript
// Bun 方式 1：得到一个文件对象
const file = Bun.file('./data.txt');
console.log(`文件大小: ${file.size} 字节`);
console.log(`文件名: ${file.name}`);

// 读取内容
const content = await file.text(); // 返回字符串
const buffer = await file.arrayBuffer(); // 返回二进制数据
const stream = file.stream(); // 返回可读流（适合大文件）
```

### 动手实验

创建 `bun-file-demo.ts`：

```typescript
// bun-file-demo.ts
import { writeFileSync } from 'fs';

// 先准备一个测试文件
writeFileSync('./test-data.txt', 'Hello from Bun!\n第二行内容\n第三行');

// 用 Bun.file() 读取
const file = Bun.file('./test-data.txt');

console.log('=== 文件信息 ===');
console.log(`文件名: ${file.name}`);
console.log(`文件大小: ${file.size} 字节`);
console.log(`MIME 类型: ${file.type}`);

async function readFile() {
  console.log('\n=== 读取内容 ===');
  const text = await file.text();
  console.log(`内容:\n${text}`);

  console.log('\n=== 按行读取 ===');
  const lines = text.split('\n');
  lines.forEach((line, index) => {
    console.log(`  第 ${index + 1} 行: "${line}"`);
  });
}

readFile();
```

运行：

```bash
bun bun-file-demo.ts
```

### `Bun.file()` 的优势

| 特性 | `Bun.file()` | Node.js `fs.readFileSync()` |
|------|-------------|----------------------------|
| API 风格 | 基于 Promise（现代异步） | 同步阻塞 |
| 内存效率 | 支持流式读取（`file.stream()`） | 一次性读入内存 |
| Web API 兼容 | 返回 `Blob`-like 对象 | Node.js 特有的 Buffer |
| 性能 | 经过优化，通常更快 | 标准实现 |

> 💡 **零基础提示**：`Blob` 是浏览器标准 API 中的一个类型，表示不可变的二进制数据。Bun 尽量兼容浏览器 API，这样同一份代码可以在浏览器和服务器上运行。

---

## 6.2 `Bun.write()` —— 高性能文件写入

### 基本用法

```typescript
// bun-write-demo.ts

async function writeDemo() {
  // 写入字符串
  await Bun.write('./output.txt', 'Hello, Bun.write()!');

  // 写入二进制数据
  const buffer = new Uint8Array([0x48, 0x65, 0x6c, 0x6c, 0x6f]); // "Hello" 的二进制
  await Bun.write('./binary.bin', buffer);

  // 写入响应（从网络请求直接写入文件）
  // const response = await fetch('https://example.com/data.json');
  // await Bun.write('./data.json', response);

  console.log('写入完成！');

  // 验证
  const content = await Bun.file('./output.txt').text();
  console.log(`验证内容: "${content}"`);
}

writeDemo();
```

### Claude Code 中的 `Bun.write()`

在 Claude Code 的源码中，文件写入大多使用了 Node.js 的 `writeFileSync`，但也有用到 `Bun.write()` 的地方。值得注意的是 `src/utils/slowOperations.ts` 中有一个被标记为 `DEPRECATED` 的封装：

```typescript
// 这是 Claude Code 源码中的简化示意
export function writeFileSync_DEPRECATED(path: string, data: string, encoding: 'utf8') {
  // 实际可能根据环境选择不同的写入方式
  // 在 Bun 下可能使用 Bun.write，在 Node.js 下使用 fs.writeFileSync
}
```

这提醒我们一个工程实践：**即使是高性能的 API，也需要根据项目整体架构统一封装，方便后续替换和维护。**

---

## 6.3 `Bun.YAML.parse()` —— 内置 YAML 解析器

### 什么是 YAML？

YAML（YAML Ain't Markup Language）是一种人类可读的数据序列化格式，常用于配置文件。

JSON 写法：
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "dependencies": {
    "react": "^18.0.0"
  }
}
```

YAML 写法（更简洁，支持注释）：
```yaml
name: my-app
version: 1.0.0
dependencies:
  react: ^18.0.0
  # 这是注释，JSON 不支持
```

### 为什么要用 `Bun.YAML.parse()`？

YAML 解析是一个复杂的工作。一个完整的 YAML 解析器（如 npm 上的 `yaml` 包）大约有 **270KB**。

Bun 把 YAML 解析器**内置**到了运行时中，所以：
- **不需要安装额外的包**
- **解析速度更快**（原生实现 vs JavaScript 实现）
- **不增加产物体积**

### 基本用法

```typescript
// yaml-parse-demo.ts

const yamlContent = `
name: Claude Code
version: 2.1.88
features:
  - ssh_remote
  - direct_connect
  - web_browser
settings:
  theme: dark
  auto_update: true
  max_tokens: 4096
`;

if (typeof Bun !== 'undefined') {
  const data = Bun.YAML.parse(yamlContent);

  console.log('=== 解析结果 ===');
  console.log(`名称: ${data.name}`);
  console.log(`版本: ${data.version}`);
  console.log(`功能列表: ${data.features.join(', ')}`);
  console.log(`主题设置: ${data.settings.theme}`);
  console.log(`最大 Token: ${data.settings.max_tokens}`);

  console.log('\n=== 完整结构 ===');
  console.log(JSON.stringify(data, null, 2));
} else {
  console.log('当前不在 Bun 运行时，无法使用 Bun.YAML.parse()');
}
```

### Claude Code 中的 YAML 解析

`src/utils/yaml.ts` 是 Claude Code 的 YAML 解析封装：

```typescript
export function parseYaml(input: string): unknown {
  if (typeof Bun !== 'undefined') {
    return Bun.YAML.parse(input);
  }
  // 回退到 npm 的 yaml 包（约 270KB，懒加载）
  return getNpmYamlPackage().parse(input);
}
```

这段代码展示了典型的"Bun 优先，Node.js 回退"模式：

1. **优先路径**：如果运行在 Bun 下，直接用内置的 `Bun.YAML.parse()`
2. **回退路径**：如果运行在 Node.js 下，动态加载 npm 的 `yaml` 包
3. **懒加载**：`getNpmYamlPackage()` 不会在模块加载时执行，只在真正需要时才加载

> 💡 **为什么用 `unknown` 作为返回类型？** `unknown` 是 TypeScript 中的"最安全的 any"。它强制调用者在使用返回值之前先做类型检查，避免运行时错误。相比 `any`，`unknown` 能帮助编译器发现潜在的类型问题。

### 动手实验：解析 MCP 配置

Claude Code 支持 MCP（Model Context Protocol）配置，这些配置常常以 YAML 格式存储。我们来模拟解析一个 MCP 服务器配置：

```typescript
// mcp-config-demo.ts

const mcpConfigYaml = `
servers:
  filesystem:
    command: npx
    args:
      - -y
      - "@modelcontextprotocol/server-filesystem"
      - /home/user/projects
    env:
      NODE_ENV: production
  
  github:
    command: docker
    args:
      - run
      - -i
      - --rm
      - "mcp/github"
    disabled: false

default_timeout: 30000
`;

interface McpServerConfig {
  command: string;
  args: string[];
  env?: Record<string, string>;
  disabled?: boolean;
}

interface McpConfig {
  servers: Record<string, McpServerConfig>;
  default_timeout: number;
}

function parseMcpConfig(yaml: string): McpConfig {
  if (typeof Bun === 'undefined') {
    throw new Error('此示例需要 Bun 运行时');
  }
  return Bun.YAML.parse(yaml) as McpConfig;
}

const config = parseMcpConfig(mcpConfigYaml);

console.log('=== MCP 配置解析结果 ===');
for (const [name, server] of Object.entries(config.servers)) {
  const status = server.disabled ? '❌ 已禁用' : '✅ 已启用';
  console.log(`\n${name}: ${status}`);
  console.log(`  命令: ${server.command}`);
  console.log(`  参数: ${server.args.join(' ')}`);
  if (server.env) {
    console.log(`  环境变量: ${JSON.stringify(server.env)}`);
  }
}
console.log(`\n默认超时: ${config.default_timeout}ms`);
```

---

## 6.4 `Bun.JSONL.parseChunk()` —— JSON Lines 高性能解析

### 什么是 JSONL（JSON Lines）？

JSONL 是一种数据格式，每行是一个独立的 JSON 对象：

```jsonl
{"name": "Alice", "age": 30}
{"name": "Bob", "age": 25}
{"name": "Charlie", "age": 35}
```

相比普通 JSON 数组 `[{"name":...}, {"name":...}]`，JSONL 的优势：
- **流式处理**：可以一行一行读，不需要等整个文件下载完
- **容错性**：如果某一行坏了，不影响其他行
- **追加友好**：可以在文件末尾直接添加新行

Claude Code 在处理流式 API 响应（如 SSE / streaming JSON）时，需要解析 JSONL 数据。

### `Bun.JSONL.parseChunk()` 的用法

```typescript
// jsonl-demo.ts

const chunk1 = '{"event": "start", "timestamp": 1000}\n{"event": "progress", ';
const chunk2 = '"progress": 0.5}\n{"event": "complete", "timestamp": 2000}\n';

function parseChunk(chunk: string): void {
  if (typeof Bun === 'undefined') {
    console.log('需要 Bun 运行时');
    return;
  }

  // Bun.JSONL.parseChunk 可能不是所有 Bun 版本都有
  // 源码中做了防御性检查
  const b = Bun as Record<string, unknown>;
  if (typeof b.JSONL?.parseChunk !== 'function') {
    console.log('当前 Bun 版本不支持 Bun.JSONL.parseChunk');
    return;
  }

  const result = (b.JSONL.parseChunk as (chunk: string) => {
    values: unknown[];
    remainder: string;
  })(chunk);

  console.log('解析出的完整对象:', result.values);
  console.log('剩余未完整解析的片段:', result.remainder);
}

console.log('=== 解析第一个 chunk ===');
parseChunk(chunk1);

console.log('\n=== 解析第二个 chunk ===');
parseChunk(chunk2);
```

### Claude Code 中的 JSONL 解析

`src/utils/json.ts` 中有 JSONL 解析的封装：

```typescript
// 简化示意
export function parseJsonlChunk(chunk: string): { lines: unknown[]; remainder: string } | false {
  if (typeof Bun === 'undefined') return false;

  const b = Bun as Record<string, unknown>;
  if (typeof b.JSONL?.parseChunk !== 'function') return false;

  const parsed = (b.JSONL.parseChunk as Function)(chunk);
  return {
    lines: parsed.values,
    remainder: parsed.remainder,
  };
}
```

这里有几个值得学习的工程实践：

1. **渐进式功能检测**：不是直接 `typeof Bun.JSONL.parseChunk === 'function'`，而是先检查 `Bun`，再检查 `Bun.JSONL`，最后检查方法
2. **类型体操**：`Bun as Record<string, unknown>` 绕过了 TypeScript 的类型检查，因为 `JSONL` 可能是较新版本的 Bun 才支持的 API，旧版本的类型定义中可能没有
3. **返回 `false` 表示不支持**：调用者可以判断 `if (result === false)` 来回退到纯 JavaScript 实现

### 为什么需要 `parseChunk` 而不是直接 `parse`？

因为网络数据是**流式到达**的。API 响应不会一次性给你完整的 JSONL 文件，而是分批发送：

```
时间线:
t=0    收到: '{"event": "start", "t'
t=100ms 收到: 'imestamp": 1000}\n{"event": "'
t=200ms 收到: 'progress", "progress": 0.5}\n'
```

`parseChunk` 会：
1. 解析出所有完整的 JSON 对象（`values`）
2. 把不完整的尾部保存下来（`remainder`）
3. 下次收到新数据时，把 `remainder` 和新数据拼接后再解析

---

## 6.5 综合练习：配置文件加载器

让我们综合运用本节学到的知识，写一个 Claude Code 风格的配置文件加载器：

```typescript
// config-loader.ts

interface AppConfig {
  name: string;
  version: string;
  features: string[];
}

async function loadConfig(path: string): Promise<AppConfig> {
  // 1. 检查文件是否存在
  const file = Bun.file(path);
  if (!(await file.exists())) {
    throw new Error(`配置文件不存在: ${path}`);
  }

  // 2. 读取内容
  const content = await file.text();

  // 3. 根据扩展名选择解析器
  if (path.endsWith('.json')) {
    return JSON.parse(content) as AppConfig;
  }

  if (path.endsWith('.yaml') || path.endsWith('.yml')) {
    if (typeof Bun !== 'undefined') {
      return Bun.YAML.parse(content) as AppConfig;
    }
    throw new Error('YAML 解析需要 Bun 运行时');
  }

  throw new Error(`不支持的配置文件格式: ${path}`);
}

// 测试
async function main() {
  // 创建一个测试 YAML 文件
  await Bun.write('./test-config.yaml', `
name: MyApp
version: 1.0.0
features:
  - dark_mode
  - auto_save
  - cloud_sync
`);

  const config = await loadConfig('./test-config.yaml');
  console.log('=== 加载的配置 ===');
  console.log(JSON.stringify(config, null, 2));
}

main().catch(console.error);
```

---

## 小结

| API | 用途 | 输入 | 输出 | Claude Code 中的场景 |
|-----|------|------|------|---------------------|
| `Bun.file(path)` | 获取文件对象 | 文件路径 | `BunFile` 对象 | 读取配置文件、缓存文件 |
| `Bun.write(path, data)` | 写入文件 | 路径 + 字符串/Buffer/Response | `Promise<number>` | 写入日志、保存状态 |
| `Bun.YAML.parse(str)` | 解析 YAML | YAML 字符串 | JavaScript 对象 | MCP 配置、项目设置 |
| `Bun.JSONL.parseChunk(str)` | 流式解析 JSONL | JSONL 字符串片段 | `{ values, remainder }` | 流式 API 响应解析 |

| 工程实践 | 说明 |
|---------|------|
| Bun 优先，Node.js 回退 | 优先使用 Bun API，提供纯 JS 回退方案 |
| 渐进式功能检测 | 逐步检查 `Bun` → `Bun.JSONL` → `Bun.JSONL.parseChunk` |
| 类型体操 | 用 `as Record<string, unknown>` 访问可能不存在的属性 |
| `unknown` 返回类型 | 强制调用者做类型检查，提高代码安全性 |
| 懒加载 | 大体积回退包（如 yaml 270KB）只在需要时加载 |

---

## 练习题

1. 编写 `file-analyzer.ts`，用 `Bun.file()` 读取当前目录下的 `package.json`，打印文件大小、修改时间，并解析内容。
2. 编写 `yaml-to-json.ts`，读取一个 YAML 文件，用 `Bun.YAML.parse()` 解析，然后用 `JSON.stringify()` 输出成格式化的 JSON。
3. 编写 `jsonl-stream-simulator.ts`，模拟接收分块的 JSONL 数据，使用 `Bun.JSONL.parseChunk()` 逐步解析。如果没有 `Bun.JSONL.parseChunk`，用纯 JavaScript 实现回退方案（基于 `indexOf('\n')` 的分割）。
4. （思考题）Claude Code 的 `parseYaml` 函数返回类型是 `unknown` 而不是 `any`。如果改成 `any` 会有什么风险？

> 参考答案：`any` 会关闭 TypeScript 的所有类型检查。调用者可以对 `any` 类型的值做任何操作（如访问不存在的属性、调用不存在的方法），编译器不会报错，但运行时可能崩溃。`unknown` 强制调用者在使用前先做类型断言或类型收窄（如 `if (typeof data === 'object' && data !== null)`），从而提前发现潜在错误。
