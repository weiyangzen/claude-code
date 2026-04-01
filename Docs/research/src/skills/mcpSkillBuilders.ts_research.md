# mcpSkillBuilders.ts 研究文档

## 场景与职责

`mcpSkillBuilders.ts` 是 Claude Code CLI 中用于**解决循环依赖问题**的轻量级注册表模块。它为 MCP（Model Context Protocol）技能发现提供对 `loadSkillsDir.ts` 中核心函数的访问，同时避免在模块依赖图中形成循环。

### 核心职责

1. **依赖解耦**：作为 `loadSkillsDir.ts` 和 `mcpSkills.ts` 之间的间接层，打破循环依赖
2. **函数注册**：允许 `loadSkillsDir.ts` 注册技能构建函数
3. **函数提供**：允许 MCP 客户端获取并使用这些构建函数

### 循环依赖问题背景

在没有此模块的情况下，会出现以下循环依赖：

```
client.ts → mcpSkills.ts → loadSkillsDir.ts → ... → client.ts
```

**为什么不能使用动态导入？**

1. **变量动态导入失败**：`await import(variable)` 在 Bun 打包的二进制文件中运行时失败，因为模块路径在 `/bunfs/root/...` 下解析，而非原始源码树
2. **字面量动态导入引发循环**：字面量导入（`await import('./loadSkillsDir.js')`）虽然能在 Bun 中工作，但会被 dependency-cruiser 追踪，导致 `loadSkillsDir.ts` 的广泛依赖扇出成多个新的循环违规

### 在系统中的位置

```
src/
├── skills/
│   ├── mcpSkillBuilders.ts   # 本文件：MCP 技能构建器注册表
│   ├── loadSkillsDir.ts      # 注册构建函数的调用方
│   └── bundledSkills.ts      # 内置技能注册
├── services/
│   └── mcp/
│       ├── client.ts         # 使用构建函数的调用方
│       └── ...
└── types/
    └── command.ts            # Command 类型定义
```

## 功能点目的

### 1. MCPSkillBuilders 类型

定义 MCP 技能发现所需的两个核心函数：

```typescript
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand    // 创建技能命令对象
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields  // 解析前 matter
}
```

### 2. 注册与获取机制

**注册（由 loadSkillsDir.ts 调用）**：
```typescript
registerMCPSkillBuilders({
  createSkillCommand,
  parseSkillFrontmatterFields,
})
```

**获取（由 MCP 客户端调用）**：
```typescript
const builders = getMCPSkillBuilders()
const command = builders.createSkillCommand({...})
```

### 3. 模块初始化时序

**关键设计**：注册发生在 `loadSkillsDir.ts` 模块初始化时，通过静态导入在启动时立即执行：

```
commands.ts (static import) → loadSkillsDir.ts (module init) → registerMCPSkillBuilders()
                                                            ↓
                                                    mcpSkillBuilders.ts (存储)
                                                            ↓
                                    client.ts (later) → getMCPSkillBuilders()
```

这确保了在第一个 MCP 服务器连接之前，构建函数已经注册完成。

## 具体技术实现

### 代码结构

```typescript
// 仅导入类型，不导入运行时值
import type {
  createSkillCommand,
  parseSkillFrontmatterFields,
} from './loadSkillsDir.js'

// 类型定义
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

// 内部状态
let builders: MCPSkillBuilders | null = null

// 注册函数
export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

// 获取函数
export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) {
    throw new Error(
      'MCP skill builders not registered — loadSkillsDir.ts has not been evaluated yet'
    )
  }
  return builders
}
```

### 关键设计决策

#### 1. 类型导入而非值导入

```typescript
// ✅ 正确：仅导入类型
import type { createSkillCommand } from './loadSkillsDir.js'

// ❌ 错误：导入值会形成实际依赖
import { createSkillCommand } from './loadSkillsDir.js'
```

使用 `import type` 确保 TypeScript 编译后不会留下运行时导入，保持模块作为依赖图叶子节点。

#### 2. 写一次注册模式

```typescript
// loadSkillsDir.ts 模块初始化时注册
registerMCPSkillBuilders({
  createSkillCommand,
  parseSkillFrontmatterFields,
})
```

这是幂等的——即使模块被多次评估（如热重载场景），相同的函数引用会被重复设置。

#### 3. 运行时错误处理

```typescript
export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) {
    throw new Error('MCP skill builders not registered...')
  }
  return builders
}
```

如果 `getMCPSkillBuilders` 在注册前被调用，抛出明确的错误信息，帮助诊断启动顺序问题。

## 关键代码路径与文件引用

### 核心导出

| 导出 | 类型 | 用途 |
|------|------|------|
| `MCPSkillBuilders` | type | 构建器集合类型定义 |
| `registerMCPSkillBuilders` | function | 注册构建函数 |
| `getMCPSkillBuilders` | function | 获取构建函数 |

### 调用方

#### 1. 注册调用方：`src/skills/loadSkillsDir.ts`

```typescript
// 文件末尾，模块初始化时执行
registerMCPSkillBuilders({
  createSkillCommand,
  parseSkillFrontmatterFields,
})
```

**调用时机**：
- 在 `commands.ts` 静态导入 `loadSkillsDir.ts` 时
- CLI 启动早期，在任何 MCP 连接之前

#### 2. 获取调用方：`src/services/mcp/client.ts`

```typescript
// 在 feature-gated 代码路径中
const fetchMcpSkillsForClient = feature('MCP_SKILLS')
  ? (require('../../skills/mcpSkills.js')).fetchMcpSkillsForClient
  : null
```

**注意**：实际使用在 `mcpSkills.ts`（如果存在），`client.ts` 通过条件 require 访问。

#### 3. 其他潜在调用方

- `src/services/mcp/useManageMCPConnections.ts`
- `src/utils/attachments.ts`
- `src/tools/SkillTool/SkillTool.ts`

### 被调用方（依赖）

| 依赖 | 路径 | 用途 |
|------|------|------|
| `createSkillCommand` (type) | `./loadSkillsDir.js` | 类型定义来源 |
| `parseSkillFrontmatterFields` (type) | `./loadSkillsDir.js` | 类型定义来源 |

## 依赖与外部交互

### 模块依赖图

```
                    ┌─────────────────────┐
                    │  mcpSkillBuilders   │
                    │     (本模块)        │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │ type import       │ value import      │ value import
           ▼                   ▼                   ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ loadSkillsDir   │    │  mcpSkills.ts   │    │  client.ts      │
│   (注册方)       │    │  (使用方，可选)  │    │  (使用方)        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 启动时序

```
1. CLI 启动
   ↓
2. 加载 commands.ts
   ↓
3. 静态导入 loadSkillsDir.ts
   ↓
4. loadSkillsDir.ts 模块初始化
   ↓
5. 执行 registerMCPSkillBuilders() → 存储函数引用
   ↓
6. 后续：MCP 服务器连接
   ↓
7. 调用 getMCPSkillBuilders() → 返回存储的函数
```

### 与 MCP 技能的交互

MCP 技能发现流程：

```typescript
// 1. MCP 客户端获取工具/提示列表
const tools = await client.listTools()
const prompts = await client.listPrompts()

// 2. 对于可作为技能的提示，使用构建器创建 Command
const builders = getMCPSkillBuilders()
const command = builders.createSkillCommand({
  skillName: prompt.name,
  description: prompt.description || '',
  markdownContent: prompt.content || '',
  // ... 其他字段
  source: 'mcp',
  loadedFrom: 'mcp',
})

// 3. 将 Command 添加到可用命令列表
```

## 风险、边界与改进建议

### 风险与防护

| 风险 | 可能性 | 影响 | 防护措施 |
|------|--------|------|----------|
| 注册前调用 | 低 | 高 | 明确的错误信息 |
| 多次注册 | 中 | 低 | 简单赋值，后注册者胜 |
| 循环依赖复发 | 低 | 高 | 仅类型导入，dep-cruiser 检查 |
| 类型不匹配 | 低 | 中 | TypeScript 编译时检查 |

### 边界情况

1. **模块热重载**：如果 `loadSkillsDir.ts` 被热重载，会重新注册，这是预期行为
2. **测试隔离**：测试应清理注册状态，或接受已注册的状态
3. **功能开关**：`MCP_SKILLS` 功能关闭时，此模块仍会被加载但不会被使用

### 已知限制

1. **单一注册点**：只支持一组构建函数，无法扩展
2. **无注销机制**：一旦注册，无法移除（测试场景可能需要）
3. **同步 API**：不支持异步构建函数注册

### 改进建议

#### 1. 扩展为通用注册表

```typescript
// 支持多个命名空间
const registries = new Map<string, unknown>()

export function register<T>(namespace: string, value: T): void
export function get<T>(namespace: string): T
```

#### 2. 添加注销支持

```typescript
export function unregisterMCPSkillBuilders(): void {
  builders = null
}
```

#### 3. 验证注册内容

```typescript
export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  if (typeof b.createSkillCommand !== 'function') {
    throw new TypeError('createSkillCommand must be a function')
  }
  if (typeof b.parseSkillFrontmatterFields !== 'function') {
    throw new TypeError('parseSkillFrontmatterFields must be a function')
  }
  builders = b
}
```

#### 4. 调试支持

```typescript
export function getMCPSkillBuildersDebugInfo() {
  return {
    isRegistered: builders !== null,
    hasCreateSkillCommand: builders?.createSkillCommand !== undefined,
    hasParseSkillFrontmatterFields: builders?.parseSkillFrontmatterFields !== undefined,
  }
}
```

### 测试要点

- 注册后能否正确获取
- 获取前调用是否抛出正确错误
- 多次注册行为（后注册者胜）
- 类型安全性（TypeScript 编译）
- 模块加载顺序（模拟不同导入顺序）

### 相关代码审查要点

1. **新增依赖检查**：确保此模块不引入新的运行时依赖
2. **类型导入验证**：确保所有从 `loadSkillsDir.js` 的导入都是 `import type`
3. **启动顺序**：验证 `commands.ts` 确实在 `client.ts` 之前导入 `loadSkillsDir.ts`
