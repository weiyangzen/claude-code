# bundledSkills.ts 研究文档

## 场景与职责

`bundledSkills.ts` 是 Claude Code CLI 中**内置技能（Bundled Skills）**的核心注册与管理模块。内置技能是指随 CLI 二进制文件一起打包、对所有用户可用的技能，与从文件系统动态加载的用户自定义技能（位于 `~/.claude/skills/` 或项目 `.claude/skills/`）相对。

### 核心职责

1. **技能注册机制**：提供 `registerBundledSkill()` API，供 CLI 内部模块注册内置技能
2. **技能存储管理**：维护内存中的技能注册表，支持获取和清理操作
3. **文件提取系统**：支持将打包在二进制中的技能参考文件（如模板、配置）安全地提取到临时目录
4. **安全文件写入**：实现防御性的文件写入机制，防止符号链接攻击和目录遍历

### 在系统中的位置

```
src/
├── skills/
│   ├── bundledSkills.ts      # 本文件：内置技能注册核心
│   ├── bundled/              # 内置技能实现目录
│   │   ├── index.ts          # 初始化所有内置技能
│   │   ├── debug.ts          # /debug 技能
│   │   ├── skillify.ts       # /skillify 技能
│   │   └── ...               # 其他内置技能
│   ├── loadSkillsDir.ts      # 文件系统技能加载器
│   └── mcpSkillBuilders.ts   # MCP 技能构建器注册
└── commands.ts               # 命令总线，整合所有技能源
```

## 功能点目的

### 1. BundledSkillDefinition 类型定义

定义了内置技能的完整配置结构：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | 技能名称（命令名） |
| `description` | `string` | 技能描述 |
| `aliases` | `string[]?` | 别名 |
| `whenToUse` | `string?` | 使用场景说明 |
| `argumentHint` | `string?` | 参数提示 |
| `allowedTools` | `string[]?` | 允许使用的工具列表 |
| `model` | `string?` | 指定模型 |
| `disableModelInvocation` | `boolean?` | 禁用模型自动调用 |
| `userInvocable` | `boolean?` | 用户是否可直接调用 |
| `isEnabled` | `() => boolean?` | 动态启用检查 |
| `hooks` | `HooksSettings?` | Hook 配置 |
| `context` | `'inline' \| 'fork'?` | 执行上下文 |
| `agent` | `string?` | Agent 类型 |
| `files` | `Record<string, string>?` | 需要提取的参考文件 |
| `getPromptForCommand` | `function` | 生成提示内容的异步函数 |

### 2. registerBundledSkill 函数

**目的**：将内置技能注册到内部注册表，使其对模型可用。

**关键逻辑**：
- 如果技能包含 `files` 字段，会包装 `getPromptForCommand` 以支持文件提取
- 使用闭包级 Promise memoization 确保并发调用时只执行一次提取
- 将 `BundledSkillDefinition` 转换为统一的 `Command` 类型

### 3. 文件提取系统

**目的**：允许内置技能附带参考文件（如代码模板、配置文件），在首次调用时提取到磁盘。

**安全机制**：
- 提取目录包含**每进程随机 nonce**（16字节十六进制），防止预创建攻击
- 使用 `O_NOFOLLOW | O_EXCL` 标志创建文件，防止符号链接攻击
- 目录权限 `0o700`，文件权限 `0o600`，确保仅所有者可访问
- 路径遍历防护：`resolveSkillFilePath` 函数验证相对路径不包含 `..`

### 4. 安全文件写入 (safeWriteFile)

**平台差异处理**：
- Windows：使用字符串标志 `'wx'`（排他创建）
- Unix/Linux/macOS：使用数值标志 `O_WRONLY | O_CREAT | O_EXCL | O_NOFOLLOW`

**设计原则**：
- 不使用 `unlink()` + 重试策略，因为 `unlink()` 会跟随中间符号链接
- 失败时返回 `null` 而非抛出，确保技能在无文件情况下仍可工作

## 具体技术实现

### 关键流程

#### 技能注册流程

```typescript
// 1. 定义技能
registerBundledSkill({
  name: 'debug',
  description: 'Debug your current Claude Code session',
  files: {  // 可选：附带参考文件
    'template.json': '{"key": "value"}',
  },
  async getPromptForCommand(args, context) {
    return [{ type: 'text', text: '...' }]
  },
})

// 2. 内部转换为 Command 对象
const command: Command = {
  type: 'prompt',
  name: definition.name,
  source: 'bundled',
  loadedFrom: 'bundled',
  // ... 其他字段映射
}

// 3. 推入注册表
bundledSkills.push(command)
```

#### 文件提取流程

```typescript
// 首次调用时触发
extractionPromise ??= extractBundledSkillFiles(definition.name, files)
const extractedDir = await extractionPromise

// 提取步骤：
// 1. 获取提取目录：/tmp/claude-{uid}/bundled-skills/{VERSION}/{nonce}/{skillName}/
// 2. 按父目录分组文件，批量创建目录
// 3. 使用 safeWriteFile 写入每个文件
// 4. 在提示内容前添加 "Base directory for this skill: {dir}\n\n"
```

### 数据结构

```typescript
// 内部注册表
const bundledSkills: Command[] = []

// 文件分组结构（用于批量写入）
const byParent = new Map<string, [string, string][]>()
// key: 父目录路径
// value: [文件路径, 内容] 元组数组
```

### 安全路径解析

```typescript
function resolveSkillFilePath(baseDir: string, relPath: string): string {
  const normalized = normalize(relPath)
  // 拒绝绝对路径
  if (isAbsolute(normalized)) throw new Error(...)
  // 拒绝路径遍历（检查 Unix 和 Windows 分隔符）
  if (normalized.split(pathSep).includes('..') || 
      normalized.split('/').includes('..')) throw new Error(...)
  return join(baseDir, normalized)
}
```

## 关键代码路径与文件引用

### 核心导出

| 导出 | 类型 | 用途 |
|------|------|------|
| `registerBundledSkill` | function | 注册内置技能 |
| `getBundledSkills` | function | 获取所有已注册技能 |
| `clearBundledSkills` | function | 清空注册表（测试用） |
| `getBundledSkillExtractDir` | function | 获取技能提取目录 |
| `BundledSkillDefinition` | type | 技能定义类型 |

### 调用方（谁在使用）

1. **`src/skills/bundled/index.ts`**
   - `initBundledSkills()` 函数调用各技能的注册函数
   - 在 CLI 启动时执行

2. **各内置技能文件**
   - `src/skills/bundled/debug.ts`：`registerDebugSkill()` → `registerBundledSkill()`
   - `src/skills/bundled/skillify.ts`：`registerSkillifySkill()` → `registerBundledSkill()`
   - `src/skills/bundled/verify.ts`、`stuck.ts`、`simplify.ts` 等

3. **`src/commands.ts`**
   - `getSkills()` 函数调用 `getBundledSkills()` 获取所有内置技能
   - 内置技能与文件系统技能、插件技能合并

4. **`src/utils/permissions/filesystem.ts`**
   - `getBundledSkillsRoot()` 提供提取根目录
   - 安全权限检查包含 bundled skills 目录

### 被调用方（依赖）

| 依赖 | 路径 | 用途 |
|------|------|------|
| `getBundledSkillsRoot` | `../utils/permissions/filesystem.js` | 获取提取根目录 |
| `ContentBlockParam` | `@anthropic-ai/sdk` | 提示内容类型 |
| `ToolUseContext` | `../Tool.js` | 工具使用上下文 |
| `Command` | `../types/command.js` | 统一命令类型 |
| `HooksSettings` | `../utils/settings/types.js` | Hook 配置类型 |

## 依赖与外部交互

### 文件系统交互

- **读取**：无直接读取操作
- **写入**：
  - 临时目录：`/tmp/claude-{uid}/bundled-skills/{VERSION}/{nonce}/{skillName}/`
  - 使用 `fs/promises` 的 `mkdir` 和 `open`
  - 创建模式：`0o700`（目录）、`0o600`（文件）

### 与权限系统的交互

`getBundledSkillsRoot()` 生成的路径被权限系统识别为安全路径：
- 允许模型读取提取的文件
- 允许在技能目录内执行操作

### 启动时序

```
1. CLI 启动
   ↓
2. 加载 src/skills/bundled/index.ts
   ↓
3. initBundledSkills() 调用各 register*Skill()
   ↓
4. registerBundledSkill() 将技能加入内存注册表
   ↓
5. 用户调用 /{skill} 命令
   ↓
6. 如有 files，首次调用触发文件提取
   ↓
7. 执行 getPromptForCommand 生成提示
```

## 风险、边界与改进建议

### 安全风险与防护

| 风险 | 防护措施 | 代码位置 |
|------|----------|----------|
| 预创建目录攻击 | 每进程随机 nonce | `getBundledSkillsRoot()` |
| 符号链接攻击 | `O_NOFOLLOW` 标志 | `SAFE_WRITE_FLAGS` |
| 文件覆盖攻击 | `O_EXCL` 排他创建 | `SAFE_WRITE_FLAGS` |
| 路径遍历 | `resolveSkillFilePath` 验证 | 第196-206行 |
| 权限泄露 | `0o700/0o600` 模式 | `mkdir` 和 `open` 调用 |

### 边界情况

1. **并发提取**：Promise memoization 确保并发调用只执行一次提取
2. **提取失败**：返回 `null`，技能继续工作（只是没有 base directory 前缀）
3. **Windows 兼容**：使用字符串标志 `'wx'` 而非数值标志
4. **空 files 对象**：跳过提取逻辑，直接返回原始提示

### 已知限制

1. **文件大小**：所有 `files` 内容在编译时打包进二进制，大文件会增加二进制体积
2. **提取时机**：仅在首次调用时提取，无法预提取
3. **清理机制**：依赖系统 `/tmp` 目录的自动清理，无显式清理逻辑

### 改进建议

1. **预提取选项**：考虑在启动时预提取常用技能的文件，减少首次调用延迟
2. **缓存验证**：添加文件哈希验证，确保提取的文件未被篡改
3. **清理策略**：添加进程退出时的清理逻辑，或设置最大保留时间
4. **监控指标**：添加提取耗时和成功率的监控，用于性能优化

### 测试要点

- 并发调用同一技能时只提取一次
- 符号链接攻击防护有效性
- 路径遍历攻击防护
- Windows 平台文件写入
- 提取失败时的降级行为
