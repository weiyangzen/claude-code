# Research: src/commands/version.ts

## 场景与职责

`version.ts` 实现了 `/version` 斜杠命令，用于显示当前 Claude Code CLI 会话运行的版本信息。这是一个简单但重要的诊断工具，帮助用户和开发者确认当前运行的软件版本。

**核心场景：**
1. 用户需要确认当前安装的 Claude Code 版本
2. 调试时确认是否使用了预期版本
3. 区分当前运行版本与自动更新下载的版本（重要提示：显示的是当前运行版本，而非 autoupdate 下载的版本）

**定位：** 这是一个 `local` 类型的命令，内部仅对 ant 构建启用（`process.env.USER_TYPE === 'ant'`），支持非交互式模式。

---

## 功能点目的

### 1. 版本信息展示
- 显示当前运行的版本号（`MACRO.VERSION`）
- 如果可用，同时显示构建时间（`MACRO.BUILD_TIME`）
- 格式：`VERSION (built BUILD_TIME)` 或仅 `VERSION`

### 2. 构建时宏替换
- 使用 `MACRO.*` 占位符，在构建时被实际值替换
- 这是 CLI 项目中常见的版本管理模式

---

## 具体技术实现

### 代码结构

```typescript
import type { Command, LocalCommandCall } from '../types/command.js'

const call: LocalCommandCall = async () => {
  return {
    type: 'text',
    value: MACRO.BUILD_TIME
      ? `${MACRO.VERSION} (built ${MACRO.BUILD_TIME})`
      : MACRO.VERSION,
  }
}

const version = {
  type: 'local',
  name: 'version',
  description:
    'Print the version this session is running (not what autoupdate downloaded)',
  isEnabled: () => process.env.USER_TYPE === 'ant',
  supportsNonInteractive: true,
  load: () => Promise.resolve({ call }),
} satisfies Command

export default version
```

### 关键实现点

1. **命令类型**: `local` - 本地执行，不发送给模型
2. **返回类型**: `LocalCommandResult` 中的 `{ type: 'text', value: string }`
3. **懒加载**: `load` 函数直接返回 resolved Promise，无需异步加载
4. **启用控制**: `isEnabled` 仅在 `USER_TYPE === 'ant'` 时返回 true
5. **非交互式支持**: `supportsNonInteractive: true` 允许在 CI/脚本中使用

### MACRO 占位符

| 占位符 | 说明 | 示例值 |
|--------|------|--------|
| `MACRO.VERSION` | 版本号 | `0.2.56` |
| `MACRO.BUILD_TIME` | 构建时间戳（可选） | `2025-01-15T10:30:00Z` |

这些宏在构建过程中被替换为实际值，运行时已是字符串字面量。

---

## 依赖与外部交互

### 内部依赖

| 模块 | 用途 |
|------|------|
| `../types/command.js` | `Command`, `LocalCommandCall` 类型定义 |

### 无外部依赖

此命令完全自包含，不调用任何外部 API、不访问文件系统、不依赖其他模块的功能。

---

## 风险、边界与改进建议

### 风险分析

1. **构建时宏未替换**
   - 如果构建系统未正确配置，`MACRO.VERSION` 可能保持原样
   - **影响**: 用户看到字面量 "MACRO.VERSION" 而非实际版本号
   - **可能性**: 低（构建系统已稳定运行）

2. **类型检查**
   - 使用了 `MACRO` 全局变量但未声明类型
   - 依赖构建系统的全局注入
   - **注意**: 代码中未看到 `declare const MACRO` 类型声明

### 边界情况

1. **BUILD_TIME 为空**
   - 代码已处理：`MACRO.BUILD_TIME ? ... : MACRO.VERSION`
   - 仅显示版本号，不显示构建时间

2. **非交互式模式**
   - `supportsNonInteractive: true` 确保可在脚本中使用
   - 输出直接写入 stdout

### 改进建议

1. **添加类型声明**
   ```typescript
   declare const MACRO: {
     VERSION: string;
     BUILD_TIME?: string;
   };
   ```
   或添加到项目的全局类型定义文件中。

2. **扩展版本信息**
   可考虑添加更多诊断信息：
   ```typescript
   // 可能的扩展
   value: [
     `Version: ${MACRO.VERSION}`,
     `Build: ${MACRO.BUILD_TIME || 'unknown'}`,
     `Platform: ${process.platform}`,
     `Node: ${process.version}`,
   ].join('\n')
   ```

3. **与 autoupdate 集成**
   - 当前描述明确说明 "not what autoupdate downloaded"
   - 可考虑添加检查：对比当前版本与最新可用版本，提示更新

4. **命令别名**
   - 可考虑添加 `ver` 或 `-v` 别名，符合常见 CLI 惯例
   - 需要在 `aliases` 字段中定义

---

## 文件引用汇总

| 文件路径 | 引用关系 |
|---------|---------|
| `src/commands/version.ts` | 本文件 |
| `src/types/command.ts` | 类型定义导入 |
| `src/commands.ts` | 命令注册（`INTERNAL_ONLY_COMMANDS` 列表） |

---

## 与其他版本相关功能的对比

| 功能 | 位置 | 用途 |
|------|------|------|
| `/version` 命令 | `src/commands/version.ts` | 运行时版本显示 |
| `package.json` version | 项目根目录 | npm 包版本 |
| `autoupdate` 版本检查 | 更新系统 | 检查并下载新版本 |

**重要提示**：`/version` 显示的是**当前内存中运行**的版本，而 autoupdate 可能已下载了更新的版本到磁盘，需要重启 CLI 才能使用。
