# 07 - Bun 全局 API：语义版本比较与命令查找

## 学习目标

读完本节后，你将能够：
1. 使用 `Bun.semver` 进行语义版本号的比较和匹配
2. 使用 `Bun.which()` 快速查找系统命令
3. 理解语义版本控制（SemVer）规范
4. 理解为什么 `Bun.which()` 比子进程执行 `which` 命令更高效
5. 读懂 Claude Code 中版本管理和命令查找相关源码

---

## 7.1 语义版本控制（SemVer）简介

### 什么是 SemVer？

软件开发中，版本号通常采用 **MAJOR.MINOR.PATCH** 的格式：

```
1.2.3
│ │ │
│ │ └── 补丁版本：修复 bug，向下兼容
│ └──── 次要版本：新增功能，向下兼容
└────── 主要版本：重大变更，可能不兼容
```

**例子**：
- `1.0.0` → `1.0.1`：修了一个 bug，用户升级无感知
- `1.0.0` → `1.1.0`：加了一个新功能，旧代码还能跑
- `1.0.0` → `2.0.0`：改了 API，旧代码可能要改

### 版本范围表达式

在实际开发中，我们常常需要判断"某个版本是否满足某个范围"：

| 表达式 | 含义 |
|--------|------|
| `>=1.2.0` | 1.2.0 或更高版本 |
| `^1.2.0` | 兼容 1.2.0（即 >=1.2.0, <2.0.0）|
| `~1.2.0` | 约等于 1.2.0（即 >=1.2.0, <1.3.0）|
| `>=1.0.0 <2.0.0` | 1.0.0 到 2.0.0 之间（不含 2.0.0）|
| `1.2.3 || 2.0.0` | 精确匹配 1.2.3 或 2.0.0 |

Claude Code 需要判断：
- 当前 Bun/Node 版本是否支持某个功能
- 依赖包的版本是否满足要求
- 系统的工具（如 Git、ripgrep）版本是否足够新

---

## 7.2 `Bun.semver` —— 内置语义版本工具

### 为什么不用 npm 的 `semver` 包？

npm 上有一个非常流行的 `semver` 包（约 50KB+），几乎每个人都在用。但 Bun 选择内置这个功能：

| 对比项 | `Bun.semver` | npm `semver` 包 |
|--------|-------------|----------------|
| 安装 | 内置，无需安装 | `npm install semver` |
| 速度 | 快约 20 倍（原生实现）| JavaScript 实现 |
| 体积影响 | 0 | +50KB node_modules |
| API 风格 | 函数式 | 面向对象（`new SemVer()`）|

### 核心 API

```typescript
// semver-demo.ts

if (typeof Bun === 'undefined' || !Bun.semver) {
  console.log('需要 Bun 运行时使用此示例');
  process.exit(1);
}

const { semver } = Bun;

console.log('=== 版本比较 ===');

// order: 比较两个版本的先后顺序
// 返回 -1（a < b）、0（相等）、1（a > b）
console.log(`order('1.0.0', '2.0.0'): ${semver.order('1.0.0', '2.0.0')}`);  // -1
console.log(`order('2.0.0', '1.0.0'): ${semver.order('2.0.0', '1.0.0')}`);  // 1
console.log(`order('1.0.0', '1.0.0'): ${semver.order('1.0.0', '1.0.0')}`);  // 0

console.log('\n=== 用 order() 实现常用比较 ===');

// 注意：Bun.semver 只提供 order() 和 satisfies() 两个方法
// gt/gte/lt/lte 需要自己用 order() 的结果封装

function gt(a: string, b: string): boolean {
  return semver.order(a, b) === 1;   // a > b
}
function gte(a: string, b: string): boolean {
  return semver.order(a, b) >= 0;    // a >= b
}
function lt(a: string, b: string): boolean {
  return semver.order(a, b) === -1;  // a < b
}
function lte(a: string, b: string): boolean {
  return semver.order(a, b) <= 0;    // a <= b
}

console.log(`gt('2.0.0', '1.0.0'): ${gt('2.0.0', '1.0.0')}`);   // true
console.log(`gte('1.0.0', '1.0.0'): ${gte('1.0.0', '1.0.0')}`); // true
console.log(`lt('1.0.0', '2.0.0'): ${lt('1.0.0', '2.0.0')}`);   // true
console.log(`lte('1.0.0', '1.0.0'): ${lte('1.0.0', '1.0.0')}`); // true

console.log('\n=== 范围匹配 ===');

// satisfies: 检查版本是否满足范围表达式
console.log(`satisfies('1.2.3', '>=1.0.0'): ${semver.satisfies('1.2.3', '>=1.0.0')}`);      // true
console.log(`satisfies('1.2.3', '>=2.0.0'): ${semver.satisfies('1.2.3', '>=2.0.0')}`);      // false
console.log(`satisfies('1.2.3', '^1.0.0'): ${semver.satisfies('1.2.3', '^1.0.0')}`);        // true
console.log(`satisfies('2.0.0', '^1.0.0'): ${semver.satisfies('2.0.0', '^1.0.0')}`);        // false
console.log(`satisfies('1.2.3', '>=1.0.0 <2.0.0'): ${semver.satisfies('1.2.3', '>=1.0.0 <2.0.0')}`); // true
```

### 动手实验

运行上面的代码：

```bash
bun semver-demo.ts
```

---

## 7.3 Claude Code 中的 `Bun.semver` 用法

### 用法 1：Windows 终端 VT 模式检测

`src/keybindings/defaultBindings.ts` 中有这样一段代码：

```typescript
import { satisfies } from 'semver'; // 注意：这是 npm 的 semver 包作为回退

const SUPPORTS_TERMINAL_VT_MODE =
  getPlatform() !== 'windows' ||
  (isRunningWithBun()
    ? satisfies(process.versions.bun, '>=1.2.23')
    : satisfies(process.versions.node, '>=22.17.0 <23.0.0 || >=24.2.0'));
```

**场景解释**：
- VT 模式是 Windows 终端的一种高级控制模式，支持更丰富的 ANSI 转义序列
- Bun 在 `1.2.23` 版本修复了相关支持
- Node.js 在 `22.17.0` 和 `24.2.0` 中引入了相关支持
- 这段代码精确判断当前运行时版本是否包含该修复

### 用法 2：Claude Code 的版本比较工具

`src/utils/semver.ts` 是 Claude Code 对 `Bun.semver` 的封装：

```typescript
// 简化后的结构
export function gt(a: string, b: string): boolean {
  if (typeof Bun !== 'undefined') {
    return Bun.semver.order(a, b) === 1;
  }
  return getNpmSemver().gt(a, b, { loose: true });
}

export function gte(a: string, b: string): boolean {
  if (typeof Bun !== 'undefined') {
    return Bun.semver.order(a, b) >= 0;
  }
  return getNpmSemver().gte(a, b, { loose: true });
}

export function lt(a: string, b: string): boolean {
  if (typeof Bun !== 'undefined') {
    return Bun.semver.order(a, b) === -1;
  }
  return getNpmSemver().lt(a, b, { loose: true });
}

export function lte(a: string, b: string): boolean {
  if (typeof Bun !== 'undefined') {
    return Bun.semver.order(a, b) <= 0;
  }
  return getNpmSemver().lte(a, b, { loose: true });
}

export function satisfies(version: string, range: string): boolean {
  if (typeof Bun !== 'undefined') {
    return Bun.semver.satisfies(version, range);
  }
  return getNpmSemver().satisfies(version, range, { loose: true });
}
```

**设计亮点**：
1. **统一接口**：无论底层用 `Bun.semver` 还是 npm `semver`，对外暴露的函数签名完全相同
2. **loose 模式**：npm `semver` 的 `{ loose: true }` 允许更宽松的版本号格式（如 `1.2` 被理解为 `1.2.0`）
3. **函数封装**：`order()` 返回 `-1/0/1`，上层封装成更直观的 `gt/gte/lt/lte`

### 动手实验：版本检查工具

```typescript
// version-checker.ts

function checkRuntimeCompatibility(): void {
  console.log('=== 运行时版本检查 ===\n');

  const checks = [
    {
      name: 'Bun 基础功能',
      version: process.versions.bun,
      required: '>=1.0.0',
      isBunOnly: true,
    },
    {
      name: 'Bun YAML 解析',
      version: process.versions.bun,
      required: '>=1.1.0',
      isBunOnly: true,
    },
    {
      name: 'Node.js 基础功能',
      version: process.versions.node,
      required: '>=18.0.0',
      isBunOnly: false,
    },
  ];

  for (const check of checks) {
    if (check.isBunOnly && !process.versions.bun) {
      console.log(`⏭️  ${check.name}: 跳过（需要 Bun）`);
      continue;
    }

    const version = check.version || '0.0.0';
    let supported: boolean;

    if (typeof Bun !== 'undefined') {
      supported = Bun.semver.satisfies(version, check.required);
    } else {
      // 简化回退：只做简单的大版本比较
      const [actualMajor] = version.split('.').map(Number);
      const requiredMajor = parseInt(check.required.replace(/>=\s*/, ''));
      supported = actualMajor >= requiredMajor;
    }

    const icon = supported ? '✅' : '❌';
    console.log(`${icon} ${check.name}: ${version} ${supported ? '满足' : '不满足'} ${check.required}`);
  }
}

checkRuntimeCompatibility();
```

---

## 7.4 `Bun.which()` —— 快速查找可执行文件

### 为什么要查找命令？

Claude Code 需要调用很多外部工具：
- `git`：版本控制
- `rg`（ripgrep）：文本搜索
- `node` / `bun`：运行子进程
- `gh`：GitHub CLI
- `docker`：容器管理

在调用这些工具之前，程序需要先确认它们是否安装在系统中，以及它们的具体路径。

### 传统方式：子进程执行 `which`

在 Node.js 中，通常这样查找：

```typescript
import { execSync } from 'child_process';

function findCommandOld(cmd: string): string | null {
  try {
    // 在 Unix/macOS 上
    const result = execSync(`which ${cmd}`, { encoding: 'utf-8' });
    return result.trim();
  } catch {
    return null;
  }
}
```

**问题**：
- 需要创建一个子进程，开销很大（几十到几百毫秒）
- 不同操作系统命令不同（Unix 用 `which`，Windows 用 `where`）
- 需要处理输出解析和错误情况

### Bun 的方式：`Bun.which()`

`Bun.which()` 直接在 Bun 内部查找 PATH 环境变量，**不需要创建子进程**：

```typescript
// which-demo.ts

console.log('=== Bun.which() 演示 ===\n');

const commands = ['git', 'node', 'bun', 'rg', 'gh', 'docker', 'not-likely-to-exist'];

for (const cmd of commands) {
  if (typeof Bun !== 'undefined') {
    const path = Bun.which(cmd);
    if (path) {
      console.log(`✅ ${cmd}: ${path}`);
    } else {
      console.log(`❌ ${cmd}: 未找到`);
    }
  } else {
    console.log(`⏭️ ${cmd}: 跳过（需要 Bun）`);
  }
}
```

### 性能对比

```typescript
// which-performance.ts

import { execSync } from 'child_process';

function benchmark(label: string, fn: () => void, iterations: number) {
  const start = performance.now();
  for (let i = 0; i < iterations; i++) {
    fn();
  }
  const end = performance.now();
  console.log(`${label}: ${(end - start).toFixed(2)}ms (${iterations} 次)`);
}

const target = 'git';
const iterations = 1000;

console.log('=== which 性能对比 ===\n');

if (typeof Bun !== 'undefined') {
  benchmark('Bun.which()', () => {
    Bun.which(target);
  }, iterations);
}

benchmark('execSync("which git")', () => {
  try {
    execSync(`which ${target}`, { encoding: 'utf-8', stdio: 'pipe' });
  } catch {
    // ignore
  }
}, iterations);
```

运行后你会发现，`Bun.which()` 比 `execSync('which ...')` 快几十倍甚至上百倍。

> 💡 **为什么快这么多？** `Bun.which()` 直接在 Bun 进程内部遍历 `PATH` 环境变量中的目录，检查文件是否存在。而 `execSync` 需要：创建新进程 → 加载 shell → shell 解析命令 → shell 执行 `which` → 等待输出 → 解析输出。这个过程涉及多次系统调用和进程切换。

---

## 7.5 Claude Code 中的 `Bun.which()` 用法

### `src/utils/which.ts` —— 统一的命令查找封装

```typescript
// 简化后的核心逻辑
const bunWhich = typeof Bun !== 'undefined' && typeof Bun.which === 'function'
  ? Bun.which
  : null;

export async function which(command: string): Promise<string | null> {
  // 优先使用 Bun.which（无子进程开销）
  if (bunWhich) {
    const result = bunWhich(command);
    if (result) return result;
  }

  // 回退：使用平台原生的 which/where 命令
  const platformCmd = process.platform === 'win32' ? 'where' : 'which';
  try {
    const { stdout } = await spawnAsync(platformCmd, [command]);
    return stdout.trim().split('\n')[0] || null;
  } catch {
    return null;
  }
}
```

这段代码的设计非常精妙：
1. **模块级缓存 `bunWhich`**：在文件加载时就确定 `Bun.which` 是否可用
2. **快速路径**：Bun 环境下直接调用，微秒级响应
3. **慢速回退**：Node.js 环境下启动子进程查找
4. **跨平台**：Windows 用 `where`，Unix/macOS 用 `which`

### 间接使用场景

很多模块通过 `which()` 函数间接受益于 `Bun.which`：

| 文件 | 用途 |
|------|------|
| `src/utils/Shell.ts` | 查找 zsh/bash 等 shell 的路径 |
| `src/utils/github/ghAuthStatus.ts` | 查找 `gh` 命令的位置 |
| `src/utils/ripgrep.ts` | 查找 `rg` 命令的位置 |

### 动手实验：命令可用性检查器

```typescript
// command-checker.ts

const REQUIRED_TOOLS = [
  { name: 'git', required: true, minVersion: '>=2.0.0' },
  { name: 'rg', required: false, minVersion: '>=13.0.0' },
  { name: 'node', required: false, minVersion: '>=18.0.0' },
  { name: 'gh', required: false, minVersion: '>=2.0.0' },
];

async function checkCommands(): Promise<void> {
  console.log('=== 命令行工具检查 ===\n');

  for (const tool of REQUIRED_TOOLS) {
    let path: string | null = null;

    if (typeof Bun !== 'undefined') {
      path = Bun.which(tool.name);
    } else {
      // Node.js 回退
      const { execSync } = await import('child_process');
      try {
        const platformCmd = process.platform === 'win32' ? 'where' : 'which';
        path = execSync(`${platformCmd} ${tool.name}`, { encoding: 'utf-8' }).trim().split('\n')[0];
      } catch {
        path = null;
      }
    }

    if (path) {
      const icon = tool.required ? '✅' : '✓';
      console.log(`${icon} ${tool.name}: ${path}`);
      if (tool.required) {
        console.log(`   最低版本要求: ${tool.minVersion}`);
      }
    } else {
      const icon = tool.required ? '❌' : '⚠️';
      const note = tool.required ? '（必需）' : '（可选）';
      console.log(`${icon} ${tool.name}: 未找到 ${note}`);
    }
  }
}

checkCommands();
```

---

## 7.6 综合应用：环境诊断工具

让我们综合运用 `Bun.semver` 和 `Bun.which()`，写一个 Claude Code 风格的系统环境诊断工具：

```typescript
// system-diagnostic.ts

interface DiagnosticResult {
  category: string;
  name: string;
  status: 'ok' | 'warn' | 'error';
  message: string;
}

async function runDiagnostics(): Promise<DiagnosticResult[]> {
  const results: DiagnosticResult[] = [];

  // 1. 运行时检测
  const runtime = process.versions.bun ? 'Bun' : 'Node.js';
  const runtimeVersion = process.versions.bun || process.versions.node || 'unknown';
  results.push({
    category: '运行时',
    name: runtime,
    status: 'ok',
    message: `版本 ${runtimeVersion}`,
  });

  // 2. Bun 版本检查
  if (process.versions.bun && typeof Bun !== 'undefined') {
    const bunVersion = process.versions.bun;
    const isRecent = Bun.semver.satisfies(bunVersion, '>=1.1.0');
    results.push({
      category: '运行时',
      name: 'Bun 版本检查',
      status: isRecent ? 'ok' : 'warn',
      message: isRecent ? `${bunVersion} 满足 >=1.1.0` : `${bunVersion} 较旧，建议升级`,
    });
  }

  // 3. 核心工具检查
  const tools = [
    { name: 'git', required: true },
    { name: 'node', required: false },
    { name: 'bun', required: false },
  ];

  for (const tool of tools) {
    let path: string | null = null;
    if (typeof Bun !== 'undefined') {
      path = Bun.which(tool.name);
    }

    results.push({
      category: '工具',
      name: tool.name,
      status: path ? 'ok' : (tool.required ? 'error' : 'warn'),
      message: path || `未找到${tool.required ? '（必需）' : ''}`,
    });
  }

  return results;
}

async function main() {
  const results = await runDiagnostics();

  console.log('=== Claude Code 环境诊断 ===\n');

  const categories = [...new Set(results.map(r => r.category))];
  for (const category of categories) {
    console.log(`[${category}]`);
    for (const result of results.filter(r => r.category === category)) {
      const icon = result.status === 'ok' ? '✅' : result.status === 'warn' ? '⚠️' : '❌';
      console.log(`  ${icon} ${result.name}: ${result.message}`);
    }
    console.log('');
  }
}

main().catch(console.error);
```

---

## 小结

| API | 用途 | 示例 | Claude Code 中的场景 |
|-----|------|------|---------------------|
| `Bun.semver.order(a, b)` | 比较版本顺序 | 返回 `-1/0/1` | 封装成 `gt/gte/lt/lte` |
| `Bun.semver.satisfies(v, r)` | 版本是否满足范围 | `satisfies('1.2', '>=1.0')` → `true` | Windows VT 模式支持检测 |
| `gt/gte/lt/lte`（需封装）| 通过 `order()` 结果判断 | `order(a,b)===1` → `gt` | Claude Code 统一接口 |
| `Bun.which(cmd)` | 查找命令路径 | `which('git')` → `/usr/bin/git` | 查找外部工具位置 |

| 设计模式 | 说明 |
|---------|------|
| 模块级缓存查找函数 | `bunWhich` 在模块加载时解析，避免运行时重复判断 |
| 统一封装层 | `semver.ts` / `which.ts` 提供统一接口，隐藏底层差异 |
| 渐进降级 | 优先用 Bun API，回退到子进程执行 |
| 版本守卫 | 精确到补丁版本的特性可用性判断 |

---

## 练习题

1. 编写 `version-manager.ts`，实现一个简单的版本管理类 `Version`，支持 `compareTo(other)`、`isGreaterThan(other)`、`satisfies(range)` 方法。在 Bun 下用 `Bun.semver`，在 Node.js 下用简化实现。
2. 编写 `tool-finder.ts`，查找系统中所有常见的开发工具（git、node、bun、docker、python、code 等），打印它们的路径和版本号。
3. 编写 `dependency-checker.ts`，读取一个 `package.json`，检查 `dependencies` 中的每个包是否满足 `package.json` 中指定的版本范围（用 `Bun.semver.satisfies`）。
4. （思考题）`src/utils/which.ts` 在模块顶部缓存了 `bunWhich`。如果用户在程序运行过程中修改了 `PATH` 环境变量，这个缓存会导致什么问题？如何修复？

> 参考答案：如果 `PATH` 被修改，`bunWhich` 缓存的仍然是旧的 `Bun.which` 函数引用，但 `Bun.which` 本身每次调用都会读取当前的 `PATH`，所以实际上不会有问题。但如果缓存的是某个具体路径结果（如 `const gitPath = Bun.which('git')`），那就会在 `PATH` 修改后返回过时的路径。修复方法是每次调用都重新执行 `Bun.which(command)`，或者提供一个 `clearCache()` 方法。
