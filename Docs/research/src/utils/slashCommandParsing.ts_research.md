# slashCommandParsing.ts 深度研究

## 场景与职责

`slashCommandParsing.ts` 提供**斜杠命令解析的集中式工具**，将用户输入的命令字符串解析为结构化的命令名称、参数和 MCP 标志。

**核心职责：**
1. 解析斜杠命令输入
2. 提取命令名称和参数
3. 识别 MCP（Model Context Protocol）命令

**应用场景：**
- 技能目录加载
- 安全审查命令
- Agent 工具加载
- 插件命令处理
- 用户输入处理

---

## 功能点目的

### 1. 解析结果类型
```typescript
export type ParsedSlashCommand = {
  commandName: string
  args: string
  isMcp: boolean
}
```

### 2. 命令解析
```typescript
export function parseSlashCommand(input: string): ParsedSlashCommand | null
```

**解析规则：**
1. 去除首尾空白
2. 检查以 `/` 开头
3. 按空格分割
4. 第一个词为命令名
5. 检测 MCP 标志（第二词为 `(MCP)`）
6. 剩余部分为参数

**示例：**
| 输入 | commandName | args | isMcp |
|------|-------------|------|-------|
| `/search foo bar` | `search` | `foo bar` | false |
| `/mcp:tool (MCP) arg1 arg2` | `mcp:tool (MCP)` | `arg1 arg2` | true |
| `not-a-command` | null | - | - |

---

## 具体技术实现

### 解析算法
```typescript
export function parseSlashCommand(input: string): ParsedSlashCommand | null {
  const trimmedInput = input.trim()

  // 检查斜杠前缀
  if (!trimmedInput.startsWith('/')) {
    return null
  }

  // 去除斜杠并分割
  const withoutSlash = trimmedInput.slice(1)
  const words = withoutSlash.split(' ')

  if (!words[0]) {
    return null
  }

  let commandName = words[0]
  let isMcp = false
  let argsStartIndex = 1

  // 检测 MCP 标志
  if (words.length > 1 && words[1] === '(MCP)') {
    commandName = commandName + ' (MCP)'
    isMcp = true
    argsStartIndex = 2
  }

  // 提取参数
  const args = words.slice(argsStartIndex).join(' ')

  return { commandName, args, isMcp }
}
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `ParsedSlashCommand` | 解析结果类型 |
| `parseSlashCommand` | 解析函数 |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/skills/loadSkillsDir.ts` | 技能加载 |
| `src/commands/security-review.ts` | 安全审查 |
| `src/tools/AgentTool/loadAgentsDir.ts` | Agent 加载 |
| `src/utils/markdownConfigLoader.ts` | Markdown 配置 |
| `src/utils/plugins/loadPluginCommands.ts` | 插件命令 |
| `src/utils/plugins/loadPluginAgents.ts` | 插件 Agent |
| `src/utils/processUserInput/processUserInput.ts` | 用户输入处理 |
| `src/utils/processUserInput/processSlashCommand.tsx` | 斜杠命令处理 |

---

## 依赖与外部交互

### 外部依赖
- 无外部依赖

### 内部依赖
- 无内部依赖

---

## 风险、边界与改进建议

### 已知风险

1. **简单分割**
   - 使用空格分割，不支持引号包裹的参数
   - 参数中的多个连续空格被压缩为单个空格

2. **MCP 检测硬编码**
   - `(MCP)` 标志为硬编码字符串
   - 格式变更需要修改代码

3. **无验证**
   - 不验证命令名有效性
   - 不检查参数格式

### 边界情况

| 场景 | 处理 |
|------|------|
| 仅 `/` | 返回 null（words[0] 为空） |
| `/command` | args 为空字符串 |
| 多个连续空格 | `split(' ')` 保留空字符串，`join(' ')` 压缩 |
| `(MCP)` 不在第二位 | 视为普通参数 |
| 前导/尾随空白 | `trim()` 处理 |

### 改进建议

1. **参数解析增强**
   - 支持引号包裹（`arg="with spaces"`）
   - 支持反斜杠转义
   - 实现类 shell 的解析

2. **验证层**
   - 添加命令名白名单验证
   - 参数类型检查
   - 必需参数验证

3. **错误信息**
   - 返回具体错误而非 null
   - 提供解析失败原因

4. **扩展性**
   - 支持更多标志类型
   - 插件可注册自定义解析规则

5. **性能**
   - 添加缓存避免重复解析相同命令
   - 使用正则优化分割逻辑
