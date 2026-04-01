# agentMemory.ts 深度研究文档

## 场景与职责

`agentMemory.ts` 是 Claude Code 中负责**代理持久化内存**管理的核心模块。它实现了代理在不同作用域下的长期记忆存储，使代理能够跨会话保留学习成果、偏好设置和项目知识。

该模块解决的核心问题：
1. **记忆持久化**：代理生成的记忆如何在不同会话间保持
2. **作用域隔离**：区分用户级、项目级和本地级的记忆范围
3. **远程存储支持**：支持通过环境变量将记忆存储到远程目录
4. **安全性**：防止路径遍历攻击，确保记忆文件的安全访问

## 功能点目的

### 1. 记忆作用域（AgentMemoryScope）
定义三种记忆作用域：
- **user**：`~/.claude/agent-memory/` - 用户级，跨所有项目共享
- **project**：`.claude/agent-memory/` - 项目级，通过版本控制与团队共享
- **local**：`.claude/agent-memory-local/` - 本地级，仅当前机器，不加入版本控制

### 2. 记忆目录管理
- `getAgentMemoryDir()`：根据代理类型和作用域获取记忆目录路径
- `getLocalAgentMemoryDir()`：处理本地作用域的特殊逻辑（支持远程存储）
- `isAgentMemoryPath()`：安全检查，判断路径是否在记忆目录内

### 3. 记忆提示构建
- `loadAgentMemoryPrompt()`：加载代理记忆并构建系统提示
- 根据作用域添加不同的使用指导说明
- 自动创建记忆目录（fire-and-forget 模式）

### 4. 路径安全处理
- `sanitizeAgentTypeForPath()`：清理代理类型名称，将冒号替换为破折号（Windows 兼容）
- 路径规范化防止目录遍历攻击

## 具体技术实现

### 关键数据类型

```typescript
// 记忆作用域类型
export type AgentMemoryScope = 'user' | 'project' | 'local'
```

### 核心函数实现

**1. 记忆目录获取**

```typescript
export function getAgentMemoryDir(
  agentType: string,
  scope: AgentMemoryScope,
): string {
  const dirName = sanitizeAgentTypeForPath(agentType)
  switch (scope) {
    case 'project':
      return join(getCwd(), '.claude', 'agent-memory', dirName) + sep
    case 'local':
      return getLocalAgentMemoryDir(dirName)
    case 'user':
      return join(getMemoryBaseDir(), 'agent-memory', dirName) + sep
  }
}
```

**2. 本地记忆目录（支持远程存储）**

```typescript
function getLocalAgentMemoryDir(dirName: string): string {
  if (process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) {
    return (
      join(
        process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR,
        'projects',
        sanitizePath(
          findCanonicalGitRoot(getProjectRoot()) ?? getProjectRoot(),
        ),
        'agent-memory-local',
        dirName,
      ) + sep
    )
  }
  return join(getCwd(), '.claude', 'agent-memory-local', dirName) + sep
}
```

**3. 路径安全检查**

```typescript
export function isAgentMemoryPath(absolutePath: string): boolean {
  // SECURITY: Normalize to prevent path traversal bypasses via .. segments
  const normalizedPath = normalize(absolutePath)
  const memoryBase = getMemoryBaseDir()

  // User scope
  if (normalizedPath.startsWith(join(memoryBase, 'agent-memory') + sep)) {
    return true
  }

  // Project scope
  if (normalizedPath.startsWith(join(getCwd(), '.claude', 'agent-memory') + sep)) {
    return true
  }

  // Local scope (支持远程存储)
  if (process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) {
    if (
      normalizedPath.includes(sep + 'agent-memory-local' + sep) &&
      normalizedPath.startsWith(
        join(process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR, 'projects') + sep,
      )
    ) {
      return true
    }
  } else if (
    normalizedPath.startsWith(
      join(getCwd(), '.claude', 'agent-memory-local') + sep,
    )
  ) {
    return true
  }

  return false
}
```

**4. 记忆提示加载**

```typescript
export function loadAgentMemoryPrompt(
  agentType: string,
  scope: AgentMemoryScope,
): string {
  // 根据作用域生成不同的使用说明
  let scopeNote: string
  switch (scope) {
    case 'user':
      scopeNote = '- Since this memory is user-scope, keep learnings general since they apply across all projects'
      break
    case 'project':
      scopeNote = '- Since this memory is project-scope and shared with your team via version control, tailor your memories to this project'
      break
    case 'local':
      scopeNote = '- Since this memory is local-scope (not checked into version control), tailor your memories to this project and machine'
      break
  }

  const memoryDir = getAgentMemoryDir(agentType, scope)

  // Fire-and-forget: 异步创建目录，不阻塞同步调用
  void ensureMemoryDirExists(memoryDir)

  const coworkExtraGuidelines = process.env.CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES
  return buildMemoryPrompt({
    displayName: 'Persistent Agent Memory',
    memoryDir,
    extraGuidelines: coworkExtraGuidelines && coworkExtraGuidelines.trim().length > 0
      ? [scopeNote, coworkExtraGuidelines]
      : [scopeNote],
  })
}
```

## 依赖与外部交互

### 依赖模块

| 模块路径 | 用途 |
|---------|------|
| `path` (Node.js) | 路径操作 (`join`, `normalize`, `sep`) |
| `../../bootstrap/state.js` | 获取项目根目录 (`getProjectRoot`) |
| `../../memdir/memdir.js` | 构建记忆提示、确保目录存在 (`buildMemoryPrompt`, `ensureMemoryDirExists`) |
| `../../memdir/paths.js` | 获取记忆基础目录 (`getMemoryBaseDir`) |
| `../../utils/cwd.js` | 获取当前工作目录 (`getCwd`) |
| `../../utils/git.js` | 查找 Git 根目录 (`findCanonicalGitRoot`) |
| `../../utils/path.js` | 路径清理 (`sanitizePath`) |

### 被调用方

通过 Grep 搜索，该模块被以下文件引用：
- `src/tools/AgentTool/loadAgentsDir.ts` - 加载代理定义时注入记忆提示
- `src/components/memory/MemoryFileSelector.tsx` - 记忆文件选择器 UI
- `src/utils/memoryFileDetection.ts` - 记忆文件检测

### 环境变量

| 变量名 | 用途 |
|-------|------|
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | 远程记忆存储目录路径 |
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | 额外的记忆使用指导 |

## 风险、边界与改进建议

### 已知风险

1. **目录创建竞态条件**：`ensureMemoryDirExists` 是 fire-and-forget 模式，在极端并发场景下可能出现问题
2. **路径遍历风险**：虽然使用了 `normalize`，但仍需持续审计路径检查逻辑
3. **远程存储权限**：`CLAUDE_CODE_REMOTE_MEMORY_DIR` 的权限管理依赖外部环境
4. **代理类型名称冲突**：`sanitizeAgentTypeForPath` 将冒号替换为破折号，可能导致不同代理类型映射到同一路径

### 边界情况

1. **Windows 兼容性**：代理类型名称中的冒号在 Windows 上是非法字符，已处理
2. **Git 工作树**：使用 `findCanonicalGitRoot` 确保在 worktree 中也能正确定位
3. **空代理类型**：未对空字符串进行特殊处理，可能导致创建意外的目录
4. **作用域切换**：代理从一种作用域切换到另一种时，记忆不会自动迁移

### 改进建议

1. **记忆迁移工具**：提供命令或 API 用于在不同作用域间迁移代理记忆
2. **记忆加密**：对敏感的记忆内容提供加密存储选项
3. **记忆版本控制**：为记忆文件添加版本信息，支持回滚和冲突解决
4. **记忆大小限制**：添加记忆目录的大小监控和限制
5. **记忆同步**：支持多设备间的记忆同步机制
6. **记忆清理**：提供自动清理过期或无用记忆的机制

### 安全建议

1. 定期审计 `isAgentMemoryPath` 函数，确保新的路径遍历攻击向量被覆盖
2. 对 `CLAUDE_CODE_REMOTE_MEMORY_DIR` 指向的目录进行权限检查
3. 考虑对记忆文件内容进行沙箱化处理，防止恶意代码注入
4. 添加记忆访问日志，用于审计和异常检测
