# crossProjectResume.ts 深度研究文档

## 场景与职责

`crossProjectResume.ts` 是 Claude Code 会话恢复功能的 **跨项目恢复检测模块**，负责判断要恢复的会话是否来自不同的项目目录，并生成相应的恢复策略。

### 核心职责

1. **跨项目检测**：判断目标会话是否来自与当前不同的项目目录
2. **工作树检测**：检测目标会话是否在同一 Git 仓库的不同工作树中
3. **命令生成**：为跨项目恢复生成 `cd && claude --resume` 命令
4. **权限控制**：通过 `USER_TYPE` 环境变量控制工作树检测功能的 rollout

### 使用场景

- **会话选择器**：`ResumeConversation.tsx` 中用户选择其他项目的会话时
- **命令行恢复**：`resume.tsx` 处理 `--resume` 参数时
- **Teleport 功能**：跨设备恢复会话时
- **多项目工作流**：用户在不同项目间切换时恢复之前的对话

---

## 功能点目的

### 1. 跨项目恢复检测 (`checkCrossProjectResume`)

**目的**：判断会话是否需要跨项目恢复，并确定恢复策略。

**返回类型**：
```typescript
type CrossProjectResumeResult =
  | { isCrossProject: false }                                    // 同项目，直接恢复
  | { isCrossProject: true; isSameRepoWorktree: true; projectPath: string }   // 同仓库工作树，直接恢复
  | { isCrossProject: true; isSameRepoWorktree: false; command: string; projectPath: string }  // 不同项目，需要 cd
```

**检测逻辑**：
1. 如果 `showAllProjects` 为 false 或无 `projectPath` 或路径相同 → 同项目
2. 如果 `USER_TYPE !== 'ant'` → 跳过工作树检测，直接生成 cd 命令
3. 检查 `projectPath` 是否匹配任一已知工作树路径
4. 如果是工作树 → 同仓库恢复
5. 如果不是工作树 → 生成 cd 命令

### 2. 工作树检测

**目的**：识别同一 Git 仓库的不同工作树，允许无缝恢复。

**实现细节**：
- 遍历 `worktreePaths` 数组
- 检查精确匹配或路径前缀匹配（`startsWith(wt + sep)`）
- 仅对 `USER_TYPE === 'ant'` 用户启用（功能 rollout 控制）

### 3. 恢复命令生成

**目的**：为跨项目恢复生成可执行的 shell 命令。

**命令格式**：
```bash
cd <project-path> && claude --resume <session-id>
```

**安全处理**：
- 使用 `quote()` 函数对路径进行 shell 转义
- 防止路径中的特殊字符导致命令注入

---

## 具体技术实现

### 核心数据结构

```typescript
export type CrossProjectResumeResult =
  | {
      isCrossProject: false
    }
  | {
      isCrossProject: true
      isSameRepoWorktree: true
      projectPath: string
    }
  | {
      isCrossProject: true
      isSameRepoWorktree: false
      command: string
      projectPath: string
    }
```

### 函数签名

```typescript
export function checkCrossProjectResume(
  log: LogOption,                    // 要恢复的会话日志
  showAllProjects: boolean,          // 是否显示所有项目的会话
  worktreePaths: string[],           // 当前仓库的所有工作树路径
): CrossProjectResumeResult
```

### 检测算法

```
checkCrossProjectResume(log, showAllProjects, worktreePaths)
    ↓
获取 currentCwd = getOriginalCwd()
    ↓
[showAllProjects=false] 或 [无 projectPath] 或 [projectPath === currentCwd]
    ↓
返回 { isCrossProject: false }
    ↓
[USER_TYPE !== 'ant']
    ↓
生成 cd 命令 → 返回 { isCrossProject: true, isSameRepoWorktree: false, command, projectPath }
    ↓
检查 worktreePaths 是否包含 log.projectPath
    ↓
[是工作树] → 返回 { isCrossProject: true, isSameRepoWorktree: true, projectPath }
[不是工作树] → 生成 cd 命令 → 返回 { isCrossProject: true, isSameRepoWorktree: false, command, projectPath }
```

### 路径匹配逻辑

```typescript
const isSameRepo = worktreePaths.some(
  wt => log.projectPath === wt || log.projectPath!.startsWith(wt + sep)
)
```

**注意**：使用 `sep`（路径分隔符）确保精确匹配，避免 `/foo` 错误匹配 `/foobar`

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `checkCrossProjectResume` | 30-75 | 跨项目恢复检测 |

### 导出类型

| 类型 | 行号 | 用途 |
|------|------|------|
| `CrossProjectResumeResult` | 7-22 | 检测结果类型 |

### 依赖文件

```
crossProjectResume.ts
├── 被调用方（上游）
│   ├── src/screens/ResumeConversation.tsx   # 恢复界面
│   └── src/commands/resume/resume.tsx       # 恢复命令
├── 被依赖模块（下游）
│   ├── src/bootstrap/state.js               # getOriginalCwd
│   ├── src/types/logs.js                    # LogOption 类型
│   ├── src/utils/bash/shellQuote.js         # quote
│   └── src/utils/sessionStorage.js          # getSessionIdFromLog
└── Node.js 内置
    └── path                                 # sep
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `path` | 路径分隔符 `sep` |

### 内部模块依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `src/bootstrap/state.js` | `getOriginalCwd` | 获取当前工作目录 |
| `src/types/logs.js` | `LogOption` | 日志类型定义 |
| `src/utils/bash/shellQuote.js` | `quote` | shell 命令转义 |
| `src/utils/sessionStorage.js` | `getSessionIdFromLog` | 从日志提取会话 ID |

### 环境变量

| 变量 | 用途 |
|------|------|
| `USER_TYPE` | 功能 rollout 控制，仅 `'ant'` 用户启用工作树检测 |

---

## 风险、边界与改进建议

### 已知风险

1. **路径匹配准确性**
   - 前缀匹配可能导致误判（如 `/foo` 和 `/foo-bar`）
   - 当前通过 `sep` 后缀缓解，但非完美解决方案

2. **符号链接**
   - 如果项目路径包含符号链接，可能导致路径不匹配
   - 未对路径进行规范化处理

3. **大小写敏感**
   - Windows 路径不区分大小写，但当前实现区分
   - 可能导致 Windows 上工作树检测失败

4. **功能 Rollout**
   - 工作树检测仅对 ant 用户启用
   - 普通用户总是看到 cd 命令，体验不一致

### 边界情况

| 场景 | 处理 |
|------|------|
| log.projectPath 为 undefined | 视为同项目 |
| worktreePaths 为空数组 | 非 ant 用户行为（生成 cd 命令） |
| projectPath 等于 currentCwd | 同项目 |
| projectPath 是 worktreePaths 子目录 | 同仓库工作树 |
| 路径包含特殊字符 | `quote()` 函数转义 |

### 改进建议

1. **路径处理**
   - 使用 `realpath` 规范化路径，处理符号链接
   - Windows 环境下进行大小写不敏感比较
   - 考虑使用 `path.relative` 进行更精确的路径关系判断

2. **功能 Rollout**
   - 完成 ant 用户测试后，向所有用户开放工作树检测
   - 添加配置选项允许用户禁用工作树检测

3. **用户体验**
   - 在 UI 中显示工作树关系（如 "Same repository (worktree)"）
   - 提供一键复制 cd 命令的按钮
   - 考虑自动执行 cd 命令（如果安全）

4. **错误处理**
   - 添加路径无效时的错误处理
   - 提供更详细的恢复失败原因

5. **测试覆盖**
   - 添加 Windows 路径测试
   - 添加符号链接场景测试
   - 添加各种路径关系测试（父子、兄弟、无关）

### 相关 Issue/PR 参考

- 本模块是跨项目恢复功能的一部分，支持多项目工作流
- 工作树检测功能正在 ant 用户中 rollout
