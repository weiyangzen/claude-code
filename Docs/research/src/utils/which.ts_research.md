# which.ts 研究文档

## 场景与职责

`which.ts` 提供跨平台的命令查找功能，用于定位系统可执行文件的完整路径。这是 Claude Code CLI 中执行外部命令（如 git、tmux 等）的基础依赖模块。

**核心使用场景：**
- 查找 git 可执行文件路径（`gitExe()` 函数使用）
- 查找其他系统命令（如 tmux、bash 等）
- 为 Windows 和 POSIX 系统提供统一的命令查找接口

## 功能点目的

### 1. 异步命令查找 (`which`)
- **目的**：异步查找命令路径，不阻塞事件循环
- **实现**：优先使用 Bun 原生 `Bun.which`（如果可用），否则使用 `whichNodeAsync`

### 2. 同步命令查找 (`whichSync`)
- **目的**：在需要同步上下文中查找命令路径
- **实现**：优先使用 Bun 原生 `Bun.which`，否则使用 `whichNodeSync`

### 3. 跨平台支持
- **Windows**：使用 `where.exe` 命令
- **POSIX**（macOS/Linux/WSL）：使用 `which` 命令

## 具体技术实现

### 关键流程

```
which(command: string) → Promise<string | null>
├── 如果 Bun.which 存在 → 直接使用 Bun.which(command)
└── 否则 → whichNodeAsync(command)
    ├── Windows (win32):
    │   └── execa(`where.exe ${command}`, { shell: true, reject: false })
    │       └── 返回第一行结果（where.exe 可能返回多行）
    └── POSIX:
        └── execa(`which ${command}`, { shell: true, reject: false })

whichSync(command: string) → string | null
├── 如果 Bun.which 存在 → 直接使用 Bun.which(command)
└── 否则 → whichNodeSync(command)
    ├── Windows: execSync_DEPRECATED(`where.exe ${command}`)
    └── POSIX: execSync_DEPRECATED(`which ${command}`)
```

### 数据结构

```typescript
// 函数签名
export const which: (command: string) => Promise<string | null>
export const whichSync: (command: string) => string | null

// Bun.which 类型守卫
const bunWhich = typeof Bun !== 'undefined' && typeof Bun.which === 'function'
  ? Bun.which
  : null
```

### 平台差异处理

| 平台 | 命令 | 输出处理 |
|------|------|----------|
| Windows | `where.exe ${command}` | 分割多行结果，取第一行 |
| POSIX | `which ${command}` | 直接使用输出 |

## 关键代码路径与文件引用

### 导出函数
- `src/utils/which.ts:71` - `which` 异步查找函数
- `src/utils/which.ts:81` - `whichSync` 同步查找函数

### 内部实现
- `src/utils/which.ts:4-31` - `whichNodeAsync` 异步 Node.js 实现
- `src/utils/which.ts:33-56` - `whichNodeSync` 同步 Node.js 实现

### 依赖文件
- `src/utils/execSyncWrapper.ts` - 提供 `execSync_DEPRECATED` 包装器
- `execa` npm 包 - 跨平台进程执行库

### 调用方
- `src/utils/git.ts:212` - `gitExe()` 函数使用 `whichSync` 缓存 git 路径
- `src/utils/windowsPaths.ts:29` - `findExecutable()` 使用 `execSync_DEPRECATED` 查找可执行文件

## 依赖与外部交互

### 外部依赖
| 依赖 | 用途 |
|------|------|
| `execa` | 跨平台异步进程执行 |
| `Bun.which` | Bun 运行时原生快速查找 |

### 内部依赖
| 文件 | 导入内容 |
|------|----------|
| `src/utils/execSyncWrapper.ts` | `execSync_DEPRECATED` |

## 风险、边界与改进建议

### 已知风险

1. **Windows 多结果问题**
   - `where.exe` 可能返回多个匹配路径（如 `git.exe` 在多个目录）
   - 当前实现取第一行，可能与预期不符
   - 代码注释已说明此行为

2. **Bun 运行时依赖**
   - 依赖 `Bun.which` 进行快速查找
   - 在非 Bun 环境（Node.js）会降级到 spawn 实现

3. **同步执行阻塞**
   - `whichNodeSync` 使用 `execSync_DEPRECATED`，会阻塞事件循环
   - 文档已标记为 `@deprecated`，建议使用异步版本

### 边界情况

1. **命令不存在**
   - 返回 `null` 而非抛出错误
   - 调用方需要处理 `null` 情况

2. **权限问题**
   - 如果命令存在但无执行权限，行为取决于底层 `which`/`where.exe`

3. **路径包含空格**
   - 使用 `shell: true` 选项，由 shell 处理参数解析

### 改进建议

1. **缓存机制**
   - 考虑添加 LRU 缓存，避免重复查找相同命令
   - 参考 `git.ts` 中的 `memoize` 使用模式

2. **错误信息增强**
   - 当前仅返回 `null`，可考虑添加调试日志记录查找失败原因

3. **Windows PATH 优先级**
   - 考虑实现更智能的 PATH 优先级处理，而非简单取第一行

4. **类型安全**
   - 考虑使用 branded type 区分普通字符串和已验证的路径字符串
