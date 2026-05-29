# 09 - Bun 全局 API：内存管理与堆快照

## 学习目标

读完本节后，你将能够：
1. 使用 `Bun.gc()` 手动触发垃圾回收
2. 使用 `Bun.generateHeapSnapshot()` 生成内存堆快照
3. 理解垃圾回收（GC）的基本概念
4. 理解内存泄漏的常见原因和排查方法
5. 读懂 Claude Code 中内存管理相关源码

---

## 9.1 什么是垃圾回收（Garbage Collection）？

### 生活中的类比

想象你的办公桌（内存）上堆满了文件（对象）。有些文件是你正在用的，有些是很久以前用过但忘了扔的。垃圾回收就像是**一个勤劳的助手**，他会自动检查每张文件，如果发现某张文件已经没人需要了（没有任何人引用它），就会帮你扔掉，腾出桌面空间。

### 编程中的概念

在 JavaScript 中，当你创建对象、数组、函数时，它们都占用内存：

```typescript
let user = { name: 'Alice', data: new Array(1000000).fill('x') };
// 创建了一个大对象，占用很多内存

user = null;
// user 不再引用那个对象了
// 但对象占用的内存不会立即释放
// 垃圾回收器会在"某个合适的时间"自动释放它
```

垃圾回收器（Garbage Collector，简称 GC）是 JavaScript 引擎内部的一个机制，它会：
1. 定期检查内存中的对象
2. 找出"不再被任何地方引用"的对象
3. 回收这些对象占用的内存

> ⚠️ **重要**：在 JavaScript 中，你不能像 C/C++ 那样直接 `free()` 内存。内存释放是完全自动的。但你可以给 GC "提建议"。

---

## 9.2 `Bun.gc()` —— 建议垃圾回收器干活

### 基本用法

```typescript
// gc-demo.ts

console.log('=== Bun.gc() 演示 ===\n');

// 创建大量临时数据
function createBigData(): number[] {
  return new Array(10_000_000).fill(0).map((_, i) => i);
}

// 获取当前内存使用（Node.js/Bun 通用）
function getMemoryUsage(): number {
  return Math.round(process.memoryUsage().heapUsed / 1024 / 1024);
}

console.log(`初始内存使用: ${getMemoryUsage()} MB`);

// 创建大数据
let bigData = createBigData();
console.log(`创建大数据后: ${getMemoryUsage()} MB`);

// 释放引用
bigData = null as any;
console.log(`释放引用后（GC 前）: ${getMemoryUsage()} MB`);

// 建议垃圾回收
if (typeof Bun !== 'undefined') {
  Bun.gc(true); // true 表示"强制"建议
  console.log(`调用 Bun.gc(true) 后: ${getMemoryUsage()} MB`);
}

console.log('\n注意：内存可能不会立即回落，GC 的具体行为取决于引擎实现。');
```

### `Bun.gc()` 的参数

```typescript
Bun.gc();       // 普通建议："有空的话回收一下吧"
Bun.gc(true);   // 强制建议："请尽快回收"
```

> ⚠️ **重要**：`Bun.gc()` 只是给 GC 一个"建议"（hint），不是命令。JavaScript 引擎可以忽略这个建议。但在实际中，Bun 的 `Bun.gc(true)` 通常会导致立即执行一次完整的垃圾回收。

### 为什么需要手动触发 GC？

既然 GC 是自动的，为什么还要手动触发？主要有以下几个场景：

1. **性能测试时**：确保每次测试前内存状态一致，排除 GC 时机对结果的干扰
2. **内存敏感操作后**：处理完大文件、大批量数据后，主动建议回收
3. **长时间运行的服务**：在空闲时段主动回收，避免高峰期 GC 造成的停顿
4. **调试内存问题时**：确认某块内存是否真的能被回收

---

## 9.3 Claude Code 中的 `Bun.gc()` 用法

### 用法 1：Print 模式的定时 GC

`src/cli/print.ts`：

```typescript
if (typeof Bun !== 'undefined') {
  const gcTimer = setInterval(Bun.gc, 1000);
  gcTimer.unref();
}
```

**场景解释**：
- Claude Code 有一个 `-p` / `--print` 模式，用于非交互式地处理用户请求
- 在这个模式下，程序会快速处理大量数据然后退出
- `setInterval(Bun.gc, 1000)` 每 1 秒触发一次 GC
- `.unref()` 很重要！它告诉事件循环："这个定时器不应该阻止程序退出"

> 💡 **为什么用 `.unref()`？** 在 Node.js/Bun 中，`setInterval` 默认会阻止进程退出（因为事件循环认为还有任务要做）。`.unref()` 让定时器变成"非引用"状态——如果其他所有事情都做完了，进程可以直接退出，不需要等这个定时器。

### 用法 2：堆快照前的强制 GC

`src/utils/heapDumpService.ts`：

```typescript
Bun.gc(true);
```

在生成堆快照之前调用 `Bun.gc(true)`，目的是：
1. 先回收所有能回收的垃圾
2. 堆快照中只保留"真正存活"的对象
3. 避免垃圾数据干扰内存分析

### 用法 3：源码注释中的 GC 讨论

`src/utils/sessionStorage.ts` 的注释提到：

```typescript
// Bun.gc(true) 在释放大 ArrayBuffer 后 RSS 仍无法回落的问题
```

这揭示了一个重要的内存知识：

| 内存指标 | 含义 | 说明 |
|---------|------|------|
| Heap Used | JavaScript 堆上实际使用的内存 | GC 主要管理这块 |
| RSS（Resident Set Size）| 进程占用的总物理内存 | 包含堆、代码、栈、库等 |
| External Memory | V8/JSC 外部分配的内存（如 Buffer、ArrayBuffer）| 不由 JS GC 直接管理 |

**关键发现**：即使 `Bun.gc(true)` 把堆内存回收了，RSS 可能仍然很高。这是因为：
1. 操作系统不会立即把释放的物理内存收回
2. 大 ArrayBuffer 的内存可能来自堆外分配
3. 内存分配器（allocator）可能保留了空闲内存池以备后用

---

## 9.4 `Bun.generateHeapSnapshot()` —— 内存堆快照

### 什么是堆快照？

堆快照（Heap Snapshot）是当前时刻内存中所有对象的"照片"。通过分析这张照片，你可以：
- 看到哪些对象占用了最多内存
- 找到内存泄漏的源头
- 理解对象的引用关系

**生活中的类比**：堆快照就像是给凌乱的办公桌拍一张照片。你可以事后仔细研究照片上有什么、它们之间的关系是什么。

### 基本用法

```typescript
// heap-snapshot-demo.ts

import { writeFileSync } from 'fs';

function createSampleData() {
  // 创建一些会被引用的对象
  const users = [];
  for (let i = 0; i < 1000; i++) {
    users.push({
      id: i,
      name: `User ${i}`,
      data: new Array(100).fill(`data-${i}`),
    });
  }
  return users;
}

async function generateSnapshot() {
  if (typeof Bun === 'undefined' || !Bun.generateHeapSnapshot) {
    console.log('需要 Bun 运行时支持堆快照功能');
    return;
  }

  console.log('准备生成堆快照...');

  // 先创建一些数据
  const users = createSampleData();
  console.log(`创建了 ${users.length} 个用户对象`);

  // 强制 GC，确保快照干净
  Bun.gc(true);

  // 生成堆快照
  const snapshot = Bun.generateHeapSnapshot('v8', 'arraybuffer');
  // 第一个参数 'v8' 表示使用 V8 格式的堆快照
  // 第二个参数 'arraybuffer' 表示输出格式（Bun 新版本支持）

  // 保存到文件
  const filename = `heap-snapshot-${Date.now()}.heapsnapshot`;
  writeFileSync(filename, snapshot, { mode: 0o600 });
  // mode: 0o600 表示只有文件所有者可读写，保护敏感内存数据

  console.log(`堆快照已保存到: ${filename}`);
  console.log(`快照大小: ${(snapshot.byteLength / 1024 / 1024).toFixed(2)} MB`);

  // 可以用 Chrome DevTools 打开 .heapsnapshot 文件进行分析
  console.log('提示：在 Chrome 中打开 DevTools → Memory 面板 → Load 加载此文件');
}

generateSnapshot().catch(console.error);
```

### 堆快照参数详解

```typescript
Bun.generateHeapSnapshot(format?, type?);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `format` | `'v8'` | 快照格式，目前主要支持 V8 格式（与 Chrome DevTools 兼容）|
| `type` | `'string'` \| `'arraybuffer'` | 输出格式。`string` 返回 JSON 字符串；`arraybuffer` 返回二进制数据（更高效）|

### 如何分析堆快照？

1. **Chrome DevTools**：
   - 打开 Chrome，按 F12 打开 DevTools
   - 切换到 Memory 面板
   - 点击 Load 按钮，选择 `.heapsnapshot` 文件
   - 可以看到对象列表、保留树、支配者树等

2. **查找内存泄漏**：
   - 在 Snapshot 1 时生成快照
   - 执行怀疑有泄漏的操作
   - 在 Snapshot 2 时再次生成快照
   - 对比两个快照，找出增长最多的对象类型

---

## 9.5 Claude Code 中的堆快照服务

### `src/utils/heapDumpService.ts`

Claude Code 内置了一个堆快照服务，用于诊断内存问题：

```typescript
// 简化后的核心逻辑
class HeapDumpService {
  generateSnapshot(filepath: string): void {
    if (typeof Bun === 'undefined') {
      throw new Error('堆快照需要 Bun 运行时');
    }

    // 1. 先强制 GC，清理垃圾
    Bun.gc(true);

    // 2. 生成快照
    const snapshot = Bun.generateHeapSnapshot('v8', 'arraybuffer');

    // 3. 写入文件，设置权限为 0o600（仅所有者可读写）
    writeFileSync(filepath, snapshot, { mode: 0o600 });
  }
}
```

**工程实践亮点**：
1. **GC 前置**：生成快照前先 `Bun.gc(true)`，确保快照反映的是"真实存活"的对象
2. **安全权限**：`mode: 0o600` 防止堆快照文件被其他用户读取（可能包含敏感数据）
3. **ArrayBuffer 格式**：使用二进制格式而不是字符串，更高效

---

## 9.6 内存管理最佳实践

### 1. 避免内存泄漏的常见模式

```typescript
// ❌ 错误：事件监听器未移除
function setup() {
  const bigData = new Array(1000000).fill('x');
  process.on('exit', () => {
    console.log(bigData.length); // bigData 被闭包引用，无法释放
  });
}

// ✅ 正确：使用弱引用或及时移除监听器
function setupFixed() {
  const bigData = new Array(1000000).fill('x');
  const handler = () => {
    console.log('exiting');
  };
  process.on('exit', handler);
  // 用完后移除
  process.removeListener('exit', handler);
}
```

### 2. 大数组和 ArrayBuffer

```typescript
// 处理大文件时，使用流而不是一次性读入内存

// ❌ 错误：一次性读入
const hugeFile = await Bun.file('10gb.log').text(); // 内存爆炸

// ✅ 正确：使用流
for await (const chunk of Bun.file('10gb.log').stream()) {
  // 处理每个 chunk（通常几 KB）
}
```

### 3. 定时器清理

```typescript
// ❌ 错误：定时器泄漏
setInterval(() => {
  checkStatus();
}, 1000);
// 如果这段代码被多次调用，会创建多个定时器

// ✅ 正确：保存引用以便清理
const timer = setInterval(() => {
  checkStatus();
}, 1000);

// 不需要时清理
clearInterval(timer);
```

---

## 9.7 综合练习：内存监控工具

```typescript
// memory-monitor.ts

class MemoryMonitor {
  private snapshots: Array<{ time: number; heapUsed: number; rss: number }> = [];
  // ReturnType<typeof setInterval> 是 setInterval 返回类型的通用写法，兼容所有环境
  private timer: ReturnType<typeof setInterval> | null = null;

  start(intervalMs: number = 5000): void {
    if (this.timer) return;

    console.log(`启动内存监控（每 ${intervalMs}ms 采样一次）`);

    this.timer = setInterval(() => {
      const usage = process.memoryUsage();
      const sample = {
        time: Date.now(),
        heapUsed: Math.round(usage.heapUsed / 1024 / 1024),
        rss: Math.round(usage.rss / 1024 / 1024),
      };
      this.snapshots.push(sample);
      console.log(`[内存] Heap: ${sample.heapUsed}MB | RSS: ${sample.rss}MB`);
    }, intervalMs);
  }

  stop(): void {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
      console.log('内存监控已停止');
    }
  }

  report(): void {
    if (this.snapshots.length === 0) {
      console.log('没有采样数据');
      return;
    }

    const heaps = this.snapshots.map(s => s.heapUsed);
    const rsses = this.snapshots.map(s => s.rss);

    console.log('\n=== 内存使用报告 ===');
    console.log(`采样次数: ${this.snapshots.length}`);
    console.log(`Heap 最小: ${Math.min(...heaps)}MB | 最大: ${Math.max(...heaps)}MB | 平均: ${Math.round(heaps.reduce((a, b) => a + b, 0) / heaps.length)}MB`);
    console.log(`RSS  最小: ${Math.min(...rsses)}MB | 最大: ${Math.max(...rsses)}MB | 平均: ${Math.round(rsses.reduce((a, b) => a + b, 0) / rsses.length)}MB`);

    // 计算增长率
    if (this.snapshots.length >= 2) {
      const firstHeap = this.snapshots[0].heapUsed;
      const lastHeap = this.snapshots[this.snapshots.length - 1].heapUsed;
      const growth = lastHeap - firstHeap;
      console.log(`Heap 增长: ${growth > 0 ? '+' : ''}${growth}MB`);

      if (growth > 50) {
        console.log('⚠️ 警告：内存增长较快，可能存在内存泄漏');
      }
    }
  }

  async takeHeapSnapshot(filepath?: string): Promise<void> {
    if (typeof Bun === 'undefined' || !Bun.generateHeapSnapshot) {
      console.log('堆快照功能需要 Bun 运行时');
      return;
    }

    const filename = filepath || `heap-${Date.now()}.heapsnapshot`;

    // 强制 GC 后生成快照
    Bun.gc(true);
    const snapshot = Bun.generateHeapSnapshot('v8', 'arraybuffer');

    const { writeFileSync } = await import('fs');
    writeFileSync(filename, snapshot, { mode: 0o600 });

    console.log(`堆快照已保存: ${filename} (${(snapshot.byteLength / 1024 / 1024).toFixed(2)}MB)`);
  }
}

// 使用示例
const monitor = new MemoryMonitor();
monitor.start(2000);

// 模拟一些内存操作
setTimeout(() => {
  const waste = new Array(1_000_000).fill('memory waste');
  console.log('创建了一些大对象');
}, 3000);

setTimeout(() => {
  monitor.report();
  monitor.takeHeapSnapshot();
  monitor.stop();
}, 10000);
```

---

## 小结

| API | 用途 | 参数 | Claude Code 中的场景 |
|-----|------|------|---------------------|
| `Bun.gc()` | 建议垃圾回收 | `force?: boolean` | print 模式定时 GC |
| `Bun.gc(true)` | 强制建议垃圾回收 | - | 堆快照前清理 |
| `Bun.generateHeapSnapshot()` | 生成内存快照 | `format`, `type` | 内存诊断、泄漏排查 |

| 内存指标 | 含义 | 管理方 |
|---------|------|--------|
| Heap Used | JS 对象占用的内存 | JavaScript GC |
| RSS | 进程总物理内存 | 操作系统 |
| External | 堆外内存（Buffer 等）| 部分由 JS 管理 |

| 工程实践 | 说明 |
|---------|------|
| `.unref()` 定时器 | 避免阻止进程退出 |
| GC 前置 | 快照前清理，数据更准确 |
| `0o600` 权限 | 保护敏感内存数据 |
| ArrayBuffer 输出 | 比字符串格式更高效 |

---

## 练习题

1. 编写 `memory-stress-test.ts`，创建一个不断分配内存的程序。每 500ms 创建一个 10MB 的数组，同时用 `MemoryMonitor` 监控。观察 Heap 和 RSS 的变化。
2. 编写 `gc-comparison.ts`，对比 `Bun.gc()` 和 `Bun.gc(true)` 的效果。创建大量临时对象后分别调用，测量内存回收的差异。
3. 编写 `heap-analyzer-cli.ts`，一个命令行工具，接受 `--snapshot` 参数生成堆快照，接受 `--monitor` 参数启动内存监控。
4. （思考题）`src/cli/print.ts` 中的 `gcTimer.unref()` 如果去掉 `.unref()` 会发生什么？

> 参考答案：如果去掉 `.unref()`，`setInterval` 会成为事件循环中的"引用"任务。这意味着即使 print 模式的所有工作都已完成（输出了结果），进程也不会退出，因为事件循环认为还有一个活跃的定时器。用户会看到程序"卡住"不结束，必须手动按 Ctrl+C 终止。
