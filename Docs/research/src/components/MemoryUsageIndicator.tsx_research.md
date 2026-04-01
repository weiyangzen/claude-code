# MemoryUsageIndicator.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`MemoryUsageIndicator.tsx` 是一个 React 组件，用于在 Claude Code 的终端界面中显示 Node.js 进程的内存使用状态。它是一个内部调试工具，仅在 "ant" 构建版本中启用，用于帮助开发团队监控和诊断内存问题。

### 1.2 使用场景
- **内存泄漏检测**：当进程内存使用异常增长时提供视觉提示
- **调试辅助**：提供 `/heapdump` 端点链接，方便生成堆快照进行分析
- **性能监控**：实时监控堆内存使用趋势

### 1.3 设计哲学
组件采用**保守显示策略**：
- 仅在内存使用超过阈值时才显示（避免干扰正常用户体验）
- 通过构建时常量完全排除外部版本（零运行时开销）
- 使用颜色编码区分警告级别（warning vs error）

---

## 2. 功能点目的

### 2.1 条件渲染策略
组件实现了三层条件渲染：

| 层级 | 条件 | 行为 |
|------|------|------|
| **构建时** | `USER_TYPE === 'ant'` | 外部构建直接返回 `null`，Hook 完全不被调用 |
| **运行时状态** | `memoryUsage === null` | 状态正常，不显示指示器 |
| **状态级别** | `status === 'normal'` | 内存使用正常，不显示指示器 |

### 2.2 内存状态分级

| 状态 | 阈值 | 显示颜色 | 含义 |
|------|------|----------|------|
| `normal` | < 1.5GB | 不显示 | 内存使用正常 |
| `high` | 1.5GB - 2.5GB | `warning` (黄色) | 内存使用较高，需关注 |
| `critical` | ≥ 2.5GB | `error` (红色) | 内存使用危险，可能即将 OOM |

### 2.3 堆转储提示
当指示器显示时，同时显示 `/heapdump` 链接，提示开发者可以通过该端点生成堆内存快照进行进一步分析。

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
// 内存使用状态（来自 useMemoryUsage hook）
interface MemoryUsageInfo {
  heapUsed: number;           // 堆内存使用量（字节）
  status: MemoryUsageStatus;  // 状态分级
}

type MemoryUsageStatus = 'normal' | 'high' | 'critical';

// 阈值常量（来自 useMemoryUsage.ts）
const HIGH_MEMORY_THRESHOLD = 1.5 * 1024 * 1024 * 1024;      // 1.5GB
const CRITICAL_MEMORY_THRESHOLD = 2.5 * 1024 * 1024 * 1024;  // 2.5GB
```

### 3.2 关键流程

#### 3.2.1 构建时排除机制
```typescript
// 硬编码检查 - 构建时确定
if ("external" !== 'ant') {
  return null;
}
```
这段代码在构建时就会被评估：
- **Ant 构建**：条件为 `false`，继续执行 Hook 调用
- **外部构建**：条件为 `true`，直接返回，Hook 代码被死码消除（dead-code elimination）

#### 3.2.2 Hook 调用与状态获取
```typescript
// eslint-disable-next-line react-hooks/rules-of-hooks
// biome-ignore lint/correctness/useHookAtTopLevel: USER_TYPE is a build-time constant
const memoryUsage = useMemoryUsage();
```

虽然 Hook 调用在条件语句之后，但由于 `USER_TYPE` 是构建时常量，实际上：
- 外部构建：Hook 调用不可达，不会违反 React Hooks 规则
- Ant 构建：Hook 总是被执行，符合规则

#### 3.2.3 状态渲染决策
```typescript
if (!memoryUsage) return null;           // Hook 返回 null（正常状态）
if (status === 'normal') return null;    // 内存使用正常

// 显示指示器
const color = status === 'critical' ? 'error' : 'warning';
return (
  <Box>
    <Text color={color} wrap="truncate">
      High memory usage ({formattedSize}) · /heapdump
    </Text>
  </Box>
);
```

### 3.3 内存监控机制（useMemoryUsage Hook）

```typescript
export function useMemoryUsage(): MemoryUsageInfo | null {
  const [memoryUsage, setMemoryUsage] = useState<MemoryUsageInfo | null>(null);

  useInterval(() => {
    const heapUsed = process.memoryUsage().heapUsed;
    const status: MemoryUsageStatus =
      heapUsed >= CRITICAL_MEMORY_THRESHOLD ? 'critical' :
      heapUsed >= HIGH_MEMORY_THRESHOLD ? 'high' : 'normal';
    
    setMemoryUsage(prev => {
      if (status === 'normal') return prev === null ? prev : null;
      return { heapUsed, status };
    });
  }, 10_000);  // 每 10 秒轮询

  return memoryUsage;
}
```

**关键设计决策**：
- 状态为 `normal` 时返回 `null`，避免不必要的重渲染
- 10 秒轮询间隔平衡了实时性和性能开销
- 使用 `useInterval`（来自 `usehooks-ts`）简化定时器管理

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件
| 文件 | 职责 |
|------|------|
| `src/components/MemoryUsageIndicator.tsx` | 指示器 UI 组件 |
| `src/hooks/useMemoryUsage.ts` | 内存监控 Hook |
| `src/components/PromptInput/Notifications.tsx` | 调用方，嵌入到底部通知栏 |
| `src/utils/format.ts` | `formatFileSize` 工具函数 |

### 4.2 关键代码片段

#### 4.2.1 构建时排除（行 10-12）
```typescript
if ("external" !== 'ant') {
  return null;
}
```
注意：这里使用字符串字面量 `"external"` 与 `'ant'` 比较，在构建时就能确定结果。

#### 4.2.2 Hook 调用（行 16）
```typescript
// biome-ignore lint/correctness/useHookAtTopLevel: USER_TYPE is a build-time constant
const memoryUsage = useMemoryUsage();
```
注释解释了为何可以忽略 lint 规则：条件分支基于构建时常量。

#### 4.2.3 条件渲染（行 17-28）
```typescript
if (!memoryUsage) {
  return null;
}
const { heapUsed, status } = memoryUsage;

// Only show indicator when memory usage is high or critical
if (status === 'normal') {
  return null;
}
```

### 4.3 调用关系
```
Notifications.tsx (底部通知栏)
  └── MemoryUsageIndicator
        ├── useMemoryUsage()
        │     ├── useState()
        │     └── useInterval(10s)
        │           └── process.memoryUsage()
        └── formatFileSize() → "1.5GB" 等格式
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖
| 包名 | 用途 |
|------|------|
| `react` | React 核心 |
| `usehooks-ts` | `useInterval` Hook |

### 5.2 内部依赖
| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `../hooks/useMemoryUsage.js` | `useMemoryUsage` | 内存监控逻辑 |
| `../ink.js` | `Box`, `Text` | Ink UI 组件 |
| `../utils/format.js` | `formatFileSize` | 文件大小格式化 |

### 5.3 格式化函数
```typescript
// src/utils/format.ts
export function formatFileSize(sizeInBytes: number): string {
  const kb = sizeInBytes / 1024;
  if (kb < 1) return `${sizeInBytes} bytes`;
  if (kb < 1024) return `${kb.toFixed(1).replace(/\.0$/, '')}KB`;
  const mb = kb / 1024;
  if (mb < 1024) return `${mb.toFixed(1).replace(/\.0$/, '')}MB`;
  const gb = mb / 1024;
  return `${gb.toFixed(1).replace(/\.0$/, '')}GB`;
}
```

示例输出：
- `1536` → `"1536 bytes"`
- `1536000` → `"1.5MB"`
- `1610612736` (1.5GB) → `"1.5GB"`

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 硬编码构建检查
**问题**：组件使用硬编码字符串 `"external" !== 'ant'` 进行构建时检查，这可能与实际的构建系统配置不同步。

**风险**：如果构建系统改变 `USER_TYPE` 的定义方式，可能导致：
- 外部构建意外启用内存监控（性能开销）
- Ant 构建意外禁用功能（调试困难）

**建议**：使用明确的构建时常量或环境变量替代硬编码字符串。

#### 6.1.2 Lint 抑制注释
**问题**：代码包含 ESLint 和 Biome 的抑制注释，可能掩盖真正的 Hooks 使用问题。

```typescript
// eslint-disable-next-line react-hooks/rules-of-hooks
// biome-ignore lint/correctness/useHookAtTopLevel: USER_TYPE is a build-time constant
```

**风险**：如果条件逻辑被修改，可能引入真正的 Hooks 规则违反。

#### 6.1.3 单点阈值配置
**问题**：阈值（1.5GB/2.5GB）在 `useMemoryUsage.ts` 中硬编码，无法动态调整。

**风险**：不同环境（开发机 vs CI）可能有不同的内存限制需求。

### 6.2 边界情况

| 场景 | 当前行为 | 潜在问题 |
|------|----------|----------|
| 内存使用刚好在阈值边界 | 状态切换可能频繁触发 | 可能导致指示器闪烁 |
| 内存快速增长 | 10 秒轮询可能延迟发现 | 可能错过关键增长阶段 |
| 多进程环境 | 只监控当前进程 | Worker 进程内存不被统计 |
| 32 位系统 | 2.5GB 阈值可能接近上限 | 可能提前触发 OOM |

### 6.3 改进建议

#### 6.3.1 配置化阈值
```typescript
// 建议：从环境变量或配置读取
const HIGH_MEMORY_THRESHOLD = 
  Number(process.env.MEMORY_HIGH_THRESHOLD) || 1.5 * 1024 * 1024 * 1024;
```

#### 6.3.2 状态变化防抖
添加状态变化的防抖或滞后（hysteresis）机制，避免在阈值边界频繁切换：
```typescript
setMemoryUsage(prev => {
  if (status === 'normal') {
    // 添加滞后：只有当低于 1.4GB 才恢复 normal
    if (heapUsed < HIGH_MEMORY_THRESHOLD * 0.93) return null;
    return prev;
  }
  return { heapUsed, status };
});
```

#### 6.3.3 内存趋势显示
除了当前使用量，显示内存增长趋势：
```
High memory usage (2.1GB ↑ 0.3GB/min) · /heapdump
```

#### 6.3.4 堆转储快捷操作
支持点击或快捷键直接触发堆转储，而非仅显示链接：
```typescript
// 监听特定按键组合
useInput((input, key) => {
  if (key.ctrl && input === 'h') {
    fetch('/heapdump');
  }
});
```

#### 6.3.5 测试覆盖
当前缺乏针对以下场景的测试：
- 构建时排除逻辑验证
- 阈值边界条件
- 状态转换序列
- 格式化输出验证

建议添加单元测试：
```typescript
describe('MemoryUsageIndicator', () => {
  it('returns null for external builds', () => {
    // 模拟外部构建环境
  });
  
  it('shows warning at 1.5GB', () => {
    // 模拟内存使用 1.5GB
  });
  
  it('shows error at 2.5GB', () => {
    // 模拟内存使用 2.5GB
  });
});
```

### 6.4 相关代码参考
- 类似条件渲染模式见 `VoiceIndicator`（`Notifications.tsx` 行 37）
- 构建时特性开关模式见 `feature('VOICE_MODE')` 和 `feature('KAIROS')`
