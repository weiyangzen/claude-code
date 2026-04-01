# pathValidation.ts 深度研究文档

## 场景与职责

`pathValidation.ts` 是 Claude Code 权限系统的核心路径验证模块，负责在文件系统操作前执行安全检查和路径验证。该模块处于工具调用链的关键路径上，确保所有文件读写操作都经过严格的权限审查。

**核心职责：**
1. **路径解析与规范化**：处理 tilde 展开、glob 模式、相对/绝对路径转换
2. **安全检查**：识别危险路径模式、阻止路径遍历攻击、检测可疑 Windows 路径
3. **权限决策**：根据操作类型（读/写/创建）和权限上下文决定是否允许访问
4. **沙箱集成**：与 SandboxManager 协作，支持沙箱写白名单验证

## 功能点目的

### 1. 路径格式化与展示

**`formatDirectoryList`** - 目录列表格式化
- 限制最多显示 5 个目录，超出时显示 "and X more"
- 用于错误消息和提示信息中展示允许的目录列表

### 2. Glob 模式处理

**`getGlobBaseDirectory`** - 提取 glob 模式的基础目录
- 识别 glob 元字符（`*?[]{}`）并提取其前的目录部分
- 示例：`/path/to/*.txt` → `/path/to`
- 用于验证 glob 展开的基础目录权限

### 3. 路径展开与规范化

**`expandTilde`** - Tilde 展开
- 将 `~` 和 `~/` 展开为用户主目录
- **安全注意**：不支持 `~user`、`~+`、`~-` 等变体（安全原因）
- 平台适配：支持 Windows 的 `~\` 格式

### 4. 沙箱写白名单检查

**`isPathInSandboxWriteAllowlist`** - 沙箱写权限验证
- 当沙箱启用时，检查路径是否在允许写入的目录列表中
- 支持 `denyWithinAllow` 列表（允许目录内的特定禁止路径）
- 使用 `memoize` 缓存配置路径的解析结果以优化性能

### 5. 核心权限检查

**`isPathAllowed`** - 路径权限决策核心函数
- 检查顺序（关键安全逻辑）：
  1. **Deny 规则优先** - 显式拒绝规则最先检查
  2. **内部可编辑路径** - 计划文件、scratchpad 等（写操作）
  3. **安全检查** - Windows 模式、Claude 配置文件、危险文件
  4. **工作目录检查** - acceptEdits 模式下允许工作目录写入
  5. **内部可读路径** - Session memory、项目目录等（读操作）
  6. **沙箱白名单** - 工作目录外的沙箱允许路径（写操作）
  7. **Allow 规则** - 显式允许规则

### 6. Glob 模式验证

**`validateGlobPattern`** - 完整的 glob 模式验证流程
- 处理路径遍历检测
- 解析并验证基础目录权限
- 返回包含 `resolvedPath` 的完整结果

### 7. 危险删除路径检测

**`isDangerousRemovalPath`** - 防止危险删除操作
- 阻止通配符删除（`*`, `/*`）
- 阻止根目录、主目录删除
- 阻止根目录直接子项删除（/usr, /tmp 等）
- Windows 支持：阻止驱动器根目录和直接子项

### 8. 主验证入口

**`validatePath`** - 完整的路径验证流程
- **安全层 1**：UNC 路径检测（防止凭证泄露）
- **安全层 2**：Tilde 变体检测（防止 TOCTOU 攻击）
- **安全层 3**：Shell 展开语法检测（`$`, `%`, `=`）
- **安全层 4**：Glob 模式限制（写操作禁止 glob）
- 路径解析与权限决策

## 具体技术实现

### 关键数据结构

```typescript
// 文件操作类型
type FileOperationType = 'read' | 'write' | 'create'

// 路径检查结果
type PathCheckResult = {
  allowed: boolean
  decisionReason?: PermissionDecisionReason
}

// 带解析路径的完整结果
type ResolvedPathCheckResult = PathCheckResult & {
  resolvedPath: string
}
```

### 关键流程

#### 路径验证流程（validatePath）

```
输入: path, cwd, context, operationType
  ↓
1. 去除引号 + tilde 展开
  ↓
2. 安全检查层:
   - UNC 路径? → 拒绝
   - Tilde 变体? → 拒绝
   - Shell 展开语法? → 拒绝
  ↓
3. Glob 模式处理:
   - 写操作 + glob? → 拒绝
   - 读操作 + glob? → validateGlobPattern
  ↓
4. 路径解析 (safeResolvePath)
  ↓
5. 权限决策 (isPathAllowed)
  ↓
返回: ResolvedPathCheckResult
```

#### 权限决策流程（isPathAllowed）

```
输入: resolvedPath, context, operationType, precomputedPathsToCheck?
  ↓
1. 检查 deny 规则 → 匹配则拒绝
  ↓
2. 写操作? → 检查内部可编辑路径 → 允许则通过
  ↓
3. 写操作? → 安全检查 → 不安全则拒绝
  ↓
4. 检查工作目录 → 在目录内?
   - 读操作? → 允许
   - acceptEdits 模式? → 允许
  ↓
5. 读操作? → 检查内部可读路径 → 允许则通过
  ↓
6. 写操作 + 不在工作目录? → 检查沙箱白名单 → 允许则通过
  ↓
7. 检查 allow 规则 → 匹配则允许
  ↓
8. 默认拒绝
```

### 性能优化

1. **Memoization**: `getResolvedSandboxConfigPath` 使用 lodash memoize 缓存配置路径解析
2. **预计算路径**: `precomputedPathsToCheck` 参数避免重复的路径解析系统调用
3. **批量检查**: 路径检查使用 `getPathsForPermissionCheck` 同时检查原始路径和解析后的符号链接路径

## 关键代码路径与文件引用

### 核心依赖

| 导入路径 | 用途 |
|---------|------|
| `../fsOperations.js` | `getFsImplementation`, `safeResolvePath`, `getPathsForPermissionCheck` |
| `./filesystem.js` | `checkEditableInternalPath`, `checkReadableInternalPath`, `pathInAllowedWorkingPath`, `matchingRuleForInput` |
| `../sandbox/sandbox-adapter.js` | `SandboxManager` |
| `../shell/readOnlyCommandValidation.js` | `containsVulnerableUncPath` |
| `../path.js` | `containsPathTraversal` |

### 被调用方

- **BashTool**: 验证命令中的文件重定向路径
- **FileEditTool**: 验证编辑目标路径
- **FileReadTool**: 验证读取源路径
- **GlobTool**: 验证 glob 模式的基础目录

### 测试文件

- `src/utils/permissions/__tests__/pathValidation.test.ts`（如果存在）

## 依赖与外部交互

### 运行时依赖

1. **Node.js 内置模块**:
   - `os.homedir()` - 用户主目录获取
   - `path` - 路径解析与操作

2. **外部库**:
   - `lodash-es/memoize` - 函数结果缓存

3. **项目内部模块**:
   - `fsOperations.js` - 文件系统抽象层
   - `filesystem.js` - 权限规则匹配与内部路径检查
   - `sandbox-adapter.js` - 沙箱配置管理

### 配置交互

- **沙箱配置**: 通过 `SandboxManager.getFsWriteConfig()` 获取允许/禁止路径
- **权限上下文**: `ToolPermissionContext` 包含所有权限规则和模式设置

## 风险、边界与改进建议

### 已知风险

1. **TOCTOU 攻击**:
   - 路径在验证和执行之间可能被修改
   - 缓解：使用 `getPathsForPermissionCheck` 检查所有路径表示形式

2. **符号链接绕过**:
   - 攻击者可能通过符号链接指向受保护文件
   - 缓解：同时检查原始路径和 `realpath` 解析后的路径

3. **Windows 路径规范化绕过**:
   - 8.3 短名称、ADS、长路径前缀等
   - 缓解：`hasSuspiciousWindowsPathPattern` 在 `filesystem.ts` 中检测

4. **大小写敏感绕过**:
   - macOS/Windows 文件系统大小写不敏感
   - 缓解：`normalizeCaseForComparison` 在 `filesystem.ts` 中使用

### 边界条件

1. **路径长度限制**: 依赖底层文件系统限制
2. **非 UTF-8 路径**: 依赖 Node.js 路径处理
3. **网络文件系统**: UNC 路径被阻止，但已挂载的网络路径难以检测

### 改进建议

1. **缓存优化**:
   - 考虑为 `isPathAllowed` 添加 LRU 缓存，避免重复检查相同路径
   - 注意：缓存键需包含 context 的哈希值

2. **错误信息增强**:
   - 当前拒绝原因可能过于笼统
   - 建议添加更具体的拒绝原因分类（如 "路径包含 shell 展开语法" vs "路径在拒绝列表中"）

3. **性能监控**:
   - 添加路径验证耗时指标，监控复杂权限规则的性能影响

4. **安全增强**:
   - 考虑添加路径规范化后的二次验证
   - 对敏感操作添加审计日志

5. **代码简化**:
   - `isPathAllowed` 函数较长，考虑拆分为更小的专用函数
   - 检查顺序逻辑复杂，考虑使用策略模式重构
