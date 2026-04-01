# classifyForCollapse.ts 研究文档

## 场景与职责

classifyForCollapse.ts 是 MCP 工具的**UI 折叠分类模块**，负责判断 MCP 工具是否应该被归类为"搜索"或"读取"操作，从而在 UI 中进行折叠显示。

### 核心职责

1. **工具分类**：根据工具名称判断其属于搜索操作还是读取操作
2. **UI 折叠决策**：为 `isSearchOrReadCommand` 方法提供数据支持
3. **白名单管理**：维护已知 MCP 服务器的搜索和读取工具白名单

### 使用场景

- 当 MCP 工具执行完成后，UI 需要决定是否将其结果折叠显示
- 搜索类工具（如 `search_code`、`slack_search_public`）的结果通常可以折叠
- 读取类工具（如 `read_file`、`get_issue`）的结果通常可以折叠
- 写入/修改类工具（如 `send_message`、`create_issue`）的结果不应折叠

### 在系统中的位置

```
src/services/mcp/client.ts
└── fetchToolsForClient()
    └── isSearchOrReadCommand() { return classifyMcpToolForCollapse(client.name, tool.name) }
```

---

## 功能点目的

### 1. 工具分类函数 (`classifyMcpToolForCollapse`)

```typescript
export function classifyMcpToolForCollapse(
  _serverName: string,
  toolName: string,
): { isSearch: boolean; isRead: boolean }
```

**功能**：
- 接收服务器名称和工具名称
- 返回工具是否属于搜索或读取类别
- 未知工具默认不归入任何类别（保守策略）

**分类逻辑**：
1. 将工具名称标准化（camelCase/kebab-case → snake_case）
2. 检查是否在 `SEARCH_TOOLS` 集合中
3. 检查是否在 `READ_TOOLS` 集合中
4. 返回分类结果

### 2. 名称标准化 (`normalize`)

```typescript
function normalize(name: string): string {
  return name
    .replace(/([a-z])([A-Z])/g, '$1_$2')  // camelCase → snake_case
    .replace(/-/g, '_')                   // kebab-case → snake_case
    .toLowerCase()
}
```

**示例转换**：
- `searchJiraIssues` → `search_jira_issues`
- `search-issues` → `search_issues`
- `get_file` → `get_file`

### 3. 搜索工具白名单 (`SEARCH_TOOLS`)

包含来自 30+ 个 MCP 服务器的搜索类工具：

| 服务器 | 示例工具 |
|--------|----------|
| Slack | `slack_search_public`, `slack_search_channels` |
| GitHub | `search_code`, `search_repositories`, `search_issues` |
| Linear | `search_documentation` |
| Datadog | `search_logs`, `search_spans`, `search_monitors` |
| Sentry | `search_docs`, `search_events` |
| Notion | `search` |
| Gmail | `gmail_search_messages` |
| Google Drive | `google_drive_search` |
| Jira/Confluence | `search_jira_issues_using_jql`, `confluence_search` |
| Filesystem | `search_files` |
| Brave Search | `brave_web_search`, `brave_local_search` |
| PubMed | `search_articles`, `pubmed_search` |
| Firecrawl | `firecrawl_search` |
| Exa | `web_search_exa`, `people_search_exa` |
| Perplexity | `perplexity_search` |
| Tavily | `tavily_search` |
| MongoDB | `find`, `search_knowledge` |
| Elasticsearch | `esql` |
| AWS | `search_documentation`, `search_catalog` |

### 4. 读取工具白名单 (`READ_TOOLS`)

包含来自 40+ 个 MCP 服务器的读取类工具：

| 服务器 | 示例工具 |
|--------|----------|
| Slack | `slack_read_channel`, `slack_get_channel_history` |
| GitHub | `get_file_contents`, `list_commits`, `get_issue` |
| Linear | `get_document`, `list_my_issues` |
| Datadog | `query_metrics`, `get_dashboard` |
| Sentry | `get_issue_details`, `get_trace_details` |
| Notion | `fetch`, `get_users` |
| Gmail | `gmail_read_message`, `gmail_read_thread` |
| Google Drive | `google_drive_fetch`, `google_drive_export` |
| Jira/Confluence | `get_jira_issue`, `get_confluence_page` |
| Filesystem | `read_file`, `list_directory` |
| Git | `git_status`, `git_log`, `git_diff` |
| Grafana | `get_dashboard_by_uid`, `query_prometheus` |
| PagerDuty | `list_incidents`, `get_service` |
| Supabase | `list_organizations`, `get_logs` |
| Stripe | `list_customers`, `list_products` |
| BigQuery | `bigquery_query`, `bigquery_schema` |
| Firecrawl | `firecrawl_scrape`, `firecrawl_crawl` |
| Playwright | `browser_take_screenshot`, `browser_snapshot` |
| MongoDB | `list_databases`, `collection_schema` |
| Kubernetes | `kubectl_get`, `kubectl_logs`, `pods_list` |

---

## 具体技术实现

### 关键流程

#### 1. 分类流程

```typescript
export function classifyMcpToolForCollapse(
  _serverName: string,
  toolName: string,
): { isSearch: boolean; isRead: boolean } {
  const normalized = normalize(toolName)
  return {
    isSearch: SEARCH_TOOLS.has(normalized),
    isRead: READ_TOOLS.has(normalized),
  }
}
```

**步骤**：
1. 标准化工具名称
2. 在搜索集合中查找
3. 在读取集合中查找
4. 返回结果对象

#### 2. 名称标准化流程

```typescript
function normalize(name: string): string {
  return name
    .replace(/([a-z])([A-Z])/g, '$1_$2')  // 处理 camelCase
    .replace(/-/g, '_')                   // 处理 kebab-case
    .toLowerCase()
}
```

**示例**：
```
slackSearchPublic → slack_search_public
slack-search-public → slack_search_public
slack_search_public → slack_search_public
```

### 数据结构

#### 工具集合

```typescript
// prettier-ignore
const SEARCH_TOOLS = new Set([
  'slack_search_public',
  'search_code',
  'search_repositories',
  // ... 更多工具
])

// prettier-ignore
const READ_TOOLS = new Set([
  'slack_read_channel',
  'get_file_contents',
  'read_file',
  // ... 更多工具
])
```

#### 返回类型

```typescript
{
  isSearch: boolean,  // 是否为搜索操作
  isRead: boolean     // 是否为读取操作
}
```

---

## 关键代码路径与文件引用

### 直接依赖

```
classifyForCollapse.ts
└── 无外部依赖（纯数据文件）
```

### 被引用位置

```
src/services/mcp/client.ts
├── import { classifyMcpToolForCollapse } from '../../tools/MCPTool/classifyForCollapse.js'
└── 在 fetchToolsForClient 中的工具定义：
    isSearchOrReadCommand() {
      return classifyMcpToolForCollapse(client.name, tool.name)
    }
```

### 调用链

```
Tool.isSearchOrReadCommand()
└── classifyMcpToolForCollapse(serverName, toolName)
    ├── normalize(toolName)
    ├── SEARCH_TOOLS.has(normalized)
    └── READ_TOOLS.has(normalized)
```

---

## 依赖与外部交互

### 无运行时依赖

该模块是纯数据文件，不依赖任何外部模块。

### 被依赖模块

| 模块 | 用途 |
|------|------|
| `src/services/mcp/client.ts` | 在工具创建时调用分类函数 |

### 分类结果消费

分类结果被用于 `Tool.isSearchOrReadCommand()` 方法，影响：
- UI 是否折叠显示工具结果
- 消息列表的紧凑模式渲染
- 历史记录的显示方式

---

## 风险、边界与改进建议

### 当前风险

1. **白名单维护成本**：
   - 需要持续跟踪新 MCP 服务器的工具命名
   - 风险：新工具可能无法正确分类
   - 缓解：保守策略（未知工具不折叠）确保不会错误折叠重要操作

2. **命名冲突**：
   - 不同服务器可能有同名工具
   - 风险：一个服务器的读取工具可能是另一个服务器的写入工具
   - 缓解：当前按工具名称匹配，不考虑服务器前缀

3. **标准化局限性**：
   - 某些特殊命名可能无法正确标准化
   - 风险：`XMLHttpRequest` → `x_m_l_http_request`（可能不符合预期）

### 边界情况

1. **未知工具**：
   - 不在白名单中的工具返回 `{ isSearch: false, isRead: false }`
   - 结果：不会在 UI 中折叠

2. **同时匹配**：
   - 理论上工具可能同时匹配搜索和读取
   - 实际：白名单设计避免了重叠

3. **空名称**：
   - 空字符串标准化后仍为空
   - 结果：不匹配任何类别

4. **特殊字符**：
   - 包含数字、下划线以外的特殊字符
   - 处理：仅转换 camelCase 和 kebab-case，其他保持不变

### 改进建议

1. **动态配置**：
   - 支持从配置文件加载白名单
   - 允许用户自定义分类规则
   - 支持通配符匹配（如 `*search*`）

2. **服务器感知**：
   - 考虑服务器前缀进行分类
   - 支持服务器特定的白名单

3. **自动发现**：
   - 基于工具描述自动推断类别
   - 使用 LLM 分析工具用途

4. **注解支持**：
   - 支持 MCP 工具注解（如 `readOnlyHint`）
   - 优先使用服务器提供的元数据

5. **统计分析**：
   - 记录未分类工具的使用频率
   - 定期生成报告补充白名单

6. **测试覆盖**：
   - 添加单元测试验证标准化逻辑
   - 测试边界情况和特殊字符

---

## 附录：白名单统计

### 搜索工具统计

| 类别 | 数量 | 示例 |
|------|------|------|
| 代码搜索 | 5 | GitHub code/repo/issue/PR search |
| 日志搜索 | 7 | Datadog logs/spans, AWS logs |
| 文档搜索 | 6 | Notion, Confluence, AWS docs |
| 网络搜索 | 8 | Brave, Exa, Perplexity, Tavily |
| 数据库搜索 | 4 | MongoDB, Neo4j, Elasticsearch |
| 其他 | 20+ | Slack, Linear, Sentry, etc. |

**总计**：约 140+ 个搜索工具

### 读取工具统计

| 类别 | 数量 | 示例 |
|------|------|------|
| 文件读取 | 10 | Filesystem, Git show/diff |
| 问题/工单读取 | 15 | GitHub issues/PRs, Jira, Linear |
| 指标/日志读取 | 20 | Datadog, Grafana, AWS |
| 资源列表 | 25 | 各种 list/get/describe 操作 |
| 数据库查询 | 10 | SQL, MongoDB, BigQuery |
| 其他 | 30+ | Slack, Notion, Stripe, etc. |

**总计**：约 200+ 个读取工具

### 命名模式分析

**搜索工具常见前缀**：
- `search_` - 通用搜索
- `find_` - 查找特定项
- `lookup_` - 查找引用
- `query_` - 查询数据

**读取工具常见前缀**：
- `get_` - 获取单个资源
- `list_` - 列出多个资源
- `read_` - 读取内容
- `fetch_` - 获取数据
- `describe_` - 描述资源
- `download_` - 下载内容

**读取工具常见后缀**：
- `_history` - 历史记录
- `_logs` - 日志
- `_status` - 状态
