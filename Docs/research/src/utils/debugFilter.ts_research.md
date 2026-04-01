# debugFilter.ts 深度研究文档

## 场景与职责

`debugFilter.ts` 是 Claude Code 调试日志系统的 **过滤引擎**，负责根据用户指定的模式过滤调试日志输出。它支持包含模式和排除模式，允许用户精确控制哪些调试信息应该显示。

### 核心职责

1. **过滤器解析**：将命令行过滤字符串解析为结构化的过滤器配置
2. **类别提取**：从日志消息中提取类别信息
3. **过滤决策**：根据过滤器和类别决定是否显示消息
4. **模式匹配**：支持多种消息格式和类别定义

### 使用场景

- **命令行过滤**：`--debug=api,hooks` 只显示 api 和 hooks 类别的日志
- **排除模式**：`--debug=!1p,!file` 排除 1p 事件和文件操作日志
- **MCP 调试**：`--debug=mcp,servername` 调试特定 MCP 服务器
- **开发调试**：开发者在调试特定功能时过滤无关日志

---

## 功能点目的

### 1. 过滤器解析 (`parseDebugFilter`)

**目的**：将过滤字符串（如 `"api,hooks"` 或 `"!1p,!file"`）解析为结构化配置。

**支持的模式**：
- **包含模式**：`api,hooks,mcp` → 只显示这些类别的日志
- **排除模式**：`!1p,!file` → 排除这些类别的日志
- **混合模式**：当前视为错误，返回 null（显示所有）

**返回结构**：
```typescript
type DebugFilter = {
  include: string[]    // 包含的类别（小写）
  exclude: string[]    // 排除的类别（小写）
  isExclusive: boolean // true = 排除模式，false = 包含模式
}
```

### 2. 类别提取 (`extractDebugCategories`)

**目的**：从日志消息中提取一个或多个类别标签。

**支持的格式**：

| 格式 | 示例 | 提取类别 |
|------|------|----------|
| 前缀模式 | `api: message` | `api` |
| 方括号模式 | `[ScheduledTasks] message` | `scheduledtasks` |
| MCP 模式 | `MCP server "name": message` | `mcp`, `name` |
| 1P 事件 | `[ANT-ONLY] 1P event: tengu_timer` | `ant-only`, `1p` |
| 二级类别 | `AutoUpdaterWrapper: Installation type: development` | `autoupdaterwrapper`, `type` |

### 3. 过滤决策 (`shouldShowDebugCategories` / `shouldShowDebugMessage`)

**目的**：根据过滤器和类别决定是否显示消息。

**决策逻辑**：

**包含模式**：
```
无类别消息 → false（必须匹配一个类别）
有类别消息 → 任一类别在 include 列表中 → true
```

**排除模式**：
```
无类别消息 → false（安全默认，排除未知）
有类别消息 → 所有类别都不在 exclude 列表中 → true
```

---

## 具体技术实现

### 核心数据结构

```typescript
export type DebugFilter = {
  include: string[]
  exclude: string[]
  isExclusive: boolean
}
```

### 过滤器解析算法

```
parseDebugFilter(filterString)
    ↓
[空字符串] → null
    ↓
按逗号分割 → filters[]
    ↓
[空 filters] → null
    ↓
检查是否有排除模式（以 ! 开头）
检查是否有包含模式（不以 ! 开头）
    ↓
[同时存在] → null（错误，显示所有）
    ↓
清理 filters（移除 ! 前缀，转小写）
    ↓
返回 { include, exclude, isExclusive }
```

### 类别提取算法

```
extractDebugCategories(message)
    ↓
初始化 categories = []
    ↓
[MCP 模式匹配] → 添加 'mcp' 和服务器名
[前缀模式匹配] → 添加前缀
[方括号模式匹配] → 添加方括号内容
[1P 事件检测] → 添加 '1p'
[二级类别匹配] → 添加二级类别
    ↓
去重 → 返回 categories
```

### 正则表达式模式

```typescript
// MCP 服务器模式
/^MCP server ["']([^"']+)["']/

// 前缀模式（仅在非 MCP 时）
/^([^:[]+):/

// 方括号模式
/^\[([^\]]+)]/

// 二级类别模式
/:\s*([^:]+?)(?:\s+(?:type|mode|status|event)?:/
```

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `parseDebugFilter` | 16-53 | 解析过滤字符串 |
| `extractDebugCategories` | 65-108 | 从消息提取类别 |
| `shouldShowDebugCategories` | 116-139 | 根据类别决定是否显示 |
| `shouldShowDebugMessage` | 145-157 | 完整过滤检查（提取+决策） |

### 导出类型

| 类型 | 行号 | 用途 |
|------|------|------|
| `DebugFilter` | 3-7 | 过滤器配置类型 |

### 依赖文件

```
debugFilter.ts
├── 被调用方（上游）
│   └── src/utils/debug.ts                    # 调试日志系统
├── 被依赖模块（下游）
│   └── lodash-es/memoize.js                  # 缓存解析结果
└── 无其他内部依赖
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `lodash-es/memoize.js` | 缓存 `parseDebugFilter` 结果 |

### 内部模块依赖

无内部模块依赖。

### 使用模式

```typescript
// 在 debug.ts 中使用
const filter = getDebugFilter()  // 调用 parseDebugFilter 的 memoized 版本
if (!shouldShowDebugMessage(message, filter)) {
  return  // 不显示此消息
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **混合模式限制**
   - 当前不支持同时包含和排除（如 `api,!verbose`）
   - 这种组合被视为错误，回退到显示所有

2. **类别提取准确性**
   - 基于正则的模式匹配可能误匹配或漏匹配
   - 消息格式变更可能导致类别提取失败

3. **大小写敏感**
   - 类别统一转小写处理
   - 如果消息中的类别大小写不一致，可能匹配失败

4. **性能考虑**
   - 每条消息都需要执行多次正则匹配
   - 高频日志场景下可能成为瓶颈

### 边界情况

| 场景 | 处理 |
|------|------|
| 空过滤器 | 显示所有消息 |
| 混合包含/排除 | 显示所有消息（当前限制） |
| 消息无类别 | 包含模式：false；排除模式：false |
| 类别重复 | Set 去重 |
| 二级类别过长 | 忽略（长度 >= 30 或包含空格） |

### 改进建议

1. **混合模式支持**
   - 实现 `include + exclude` 组合逻辑
   - 例如：`api,!verbose` 显示 api 类别但排除 verbose 级别

2. **通配符支持**
   - 添加 `*` 通配符支持（如 `mcp.*` 匹配所有 MCP 服务器）
   - 添加正则表达式支持

3. **性能优化**
   - 缓存类别提取结果
   - 使用更高效的模式匹配算法
   - 考虑编译正则表达式

4. **类别标准化**
   - 定义标准的类别列表
   - 添加类别验证和自动修正

5. **调试支持**
   - 添加 `--debug-filter-test` 命令测试过滤器
   - 显示消息被过滤的原因

6. **文档完善**
   - 添加过滤器语法文档
   - 提供常用过滤器示例

### 相关 Issue/PR 参考

- 本模块是调试系统的基础设施，支持精细化的日志控制
- 与 `debug.ts` 紧密耦合，共同构成完整的调试日志系统
