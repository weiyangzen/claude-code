# useAfterFirstRender.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useAfterFirstRender.ts` 是一个极简的 React Hook，用于在组件首次渲染完成后执行特定的诊断操作。它是 Claude Code 内部开发/测试基础设施的一部分。

### 1.2 使用场景
| 场景 | 描述 |
|------|------|
| 性能基准测试 | 测量并输出启动时间 |
| CI/CD 集成 | 自动化测试后自动退出进程 |
| 内部开发 | Ant 员工快速验证渲染性能 |

### 1.3 调用方
- `src/screens/REPL.tsx` - 主应用界面

---

## 2. 功能点目的

### 2.1 启动时间测量
- **目的**：精确测量从进程启动到首次渲染完成的时间
- **实现**：使用 `process.uptime()` 获取高精度时间
- **输出**：写入 stderr 避免污染 stdout

### 2.2 自动退出机制
- **目的**：自动化测试场景下渲染完成后立即退出
- **条件**：
  - `USER_TYPE=ant`（仅限内部员工）
  - `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER=true`

---

## 3. 具体技术实现

### 3.1 核心实现

```typescript
import { useEffect } from 'react'
import { isEnvTruthy } from '../utils/envUtils.js'

export function useAfterFirstRender(): void {
  useEffect(() => {
    // 双重条件检查：内部员工 + 显式启用
    if (
      process.env.USER_TYPE === 'ant' &&
      isEnvTruthy(process.env.CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER)
    ) {
      // 输出启动时间（毫秒）
      process.stderr.write(
        `\nStartup time: ${Math.round(process.uptime() * 1000)}ms\n`,
      )
      // 强制退出进程
      // eslint-disable-next-line custom-rules/no-process-exit
      process.exit(0)
    }
  }, [])  // 空依赖数组：仅在挂载时执行
}
```

### 3.2 环境变量工具

```typescript
// utils/envUtils.ts
export function isEnvTruthy(value: string | undefined): boolean {
  if (!value) return false
  const lower = value.toLowerCase()
  return lower === 'true' || lower === '1' || lower === 'yes'
}
```

### 3.3 执行流程

```
REPL.tsx 挂载
  └── useAfterFirstRender()
      └── useEffect 回调
          ├── 检查 USER_TYPE === 'ant'
          ├── 检查 CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER
          ├── 输出 Startup time
          └── process.exit(0)
```

---

## 4. 关键代码路径与文件引用

### 4.1 依赖图

```
useAfterFirstRender.ts
├── react (useEffect)
└── utils/envUtils.js (isEnvTruthy)
```

### 4.2 调用位置

```typescript
// REPL.tsx
import { useAfterFirstRender } from '../hooks/useAfterFirstRender.js'

function REPL() {
  useAfterFirstRender()  // 在组件顶层调用
  // ...
}
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 | 类型 |
|------|------|------|
| `react` | useEffect Hook | npm 包 |

### 5.2 环境变量

| 变量 | 值 | 作用 |
|------|-----|------|
| `USER_TYPE` | `ant` | 限制为内部员工 |
| `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` | `true/1/yes` | 启用自动退出 |

### 5.3 Node.js API

| API | 用途 |
|-----|------|
| `process.uptime()` | 获取进程运行时间（秒） |
| `process.stderr.write()` | 输出到标准错误 |
| `process.exit(0)` | 正常退出进程 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解 |
|------|------|------|
| 过早退出 | 可能中断其他初始化逻辑 | 依赖数组为空，确保最后执行 |
| 环境泄露 | `USER_TYPE` 检查防止外部滥用 | 硬编码检查 |
| 测试干扰 | 可能中断集成测试 | 仅在特定环境变量下启用 |

### 6.2 边界条件

1. **非 ant 用户**：Hook 完全无操作
2. **未设置环境变量**：Hook 完全无操作
3. **服务端渲染**：`useEffect` 不会在 SSR 执行
4. **快速卸载**：组件快速卸载不会触发副作用

### 6.3 改进建议

1. **更精确测量**：
   ```typescript
   // 可添加更多性能指标
   const metrics = {
     startupTime: process.uptime() * 1000,
     memoryUsage: process.memoryUsage(),
     cpuUsage: process.cpuUsage(),
   }
   ```

2. **可配置退出码**：
   ```typescript
   const exitCode = parseInt(process.env.CLAUDE_CODE_EXIT_CODE || '0', 10)
   process.exit(exitCode)
   ```

3. **回调支持**：
   ```typescript
   export function useAfterFirstRender(callback?: () => void): void {
     useEffect(() => {
       if (callback) callback()
       // ...
     }, [])
   }
   ```

4. **性能标记集成**：
   ```typescript
   // 使用 Performance API
   performance.mark('first-render')
   performance.measure('startup', 'navigationStart', 'first-render')
   ```

### 6.4 代码质量

- **优点**：
  - 极简实现，无副作用
  - 明确的安全限制（USER_TYPE 检查）
  - 使用 stderr 避免污染正常输出
  
- **潜在改进**：
  - 添加 JSDoc 说明用途
  - 考虑重命名为更描述性的名称（如 `useStartupBenchmark`）
  - 添加日志级别控制

### 6.5 安全考虑

该 Hook 虽然调用 `process.exit()`，但有多重保护：
1. **USER_TYPE 限制**：仅限内部员工
2. **显式启用**：需要设置特定环境变量
3. **开发专用**：明显是开发和测试工具
