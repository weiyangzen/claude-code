# MCPServerApprovalDialog.tsx 研究文档

## 场景与职责

MCPServerApprovalDialog 是 Claude Code CLI 中用于 MCP (Model Context Protocol) 服务器审批的 UI 组件。当系统检测到项目目录下的 `.mcp.json` 文件中定义了新的 MCP 服务器时，会弹出此对话框请求用户确认是否启用该服务器。

该组件属于**安全审批流程**的一部分，确保用户在执行可能访问系统资源或执行代码的 MCP 服务器前明确授权。

### 触发场景
- 启动 Claude Code 时检测到 `.mcp.json` 中有新的服务器配置
- 服务器状态为 `pending`（待审批）时
- 用户之前未明确批准或拒绝该服务器

---

## 功能点目的

### 1. 服务器审批决策
提供三种用户选择：
- **"Use this and all future MCP servers in this project"** (`yes_all`): 启用当前服务器并自动批准该项目所有未来的 MCP 服务器
- **"Use this MCP server"** (`yes`): 仅启用当前服务器
- **"Continue without using this MCP server"** (`no`): 拒绝使用该服务器

### 2. 设置持久化
根据用户选择更新本地设置文件 (`.claude/settings.local.json`)：
- 批准时：将服务器名添加到 `enabledMcpjsonServers` 数组
- 批准全部时：设置 `enableAllProjectMcpServers: true`
- 拒绝时：将服务器名添加到 `disabledMcpjsonServers` 数组

### 3. 分析追踪
记录用户选择到分析系统 (`tengu_mcp_dialog_choice`)，用于理解用户行为模式。

---

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
interface Props {
  serverName: string;    // 待审批的服务器名称
  onDone(): void;        // 完成回调，关闭对话框
}

// 用户选择值类型
type ChoiceValue = 'yes_all' | 'yes' | 'no';
```

### 核心流程

1. **初始化渲染**
   ```
   Dialog 组件 (title="New MCP server found in .mcp.json: ${serverName}")
   ├── MCPServerDialogCopy (安全提示文本)
   └── Select 组件 (三个选项)
   ```

2. **用户选择处理** (`onChange` 函数)
   ```typescript
   switch (value) {
     case 'yes':
     case 'yes_all':
       // 1. 获取当前设置
       // 2. 检查服务器是否已在启用列表
       // 3. 更新 enabledMcpjsonServers
       // 4. 如选择 yes_all，设置 enableAllProjectMcpServers
       // 5. 调用 onDone()
       
     case 'no':
       // 1. 获取当前设置
       // 2. 检查服务器是否已在禁用列表
       // 3. 更新 disabledMcpjsonServers
       // 4. 调用 onDone()
   }
   ```

3. **设置更新机制**
   - 使用 `updateSettingsForSource("localSettings", {...})` 写入本地设置
   - 数组去重通过展开运算符和 `includes` 检查实现

### React Compiler 优化

代码使用 React Compiler (`_c` 函数) 进行自动记忆化：
- 13 个缓存槽位用于优化重渲染
- 依赖追踪确保 `onChange` 回调只在 `onDone` 或 `serverName` 变化时重新创建

---

## 关键代码路径与文件引用

### 本文件
- `/src/components/MCPServerApprovalDialog.tsx` (115 行)

### 直接依赖
| 文件 | 用途 |
|------|------|
| `src/components/MCPServerDialogCopy.tsx` | 安全提示文本组件 |
| `src/components/CustomSelect/index.js` | Select 下拉选择组件 |
| `src/components/design-system/Dialog.js` | 对话框容器组件 |
| `src/utils/settings/settings.js` | 设置读写 (`getSettings_DEPRECATED`, `updateSettingsForSource`) |
| `src/services/analytics/index.js` | 分析事件记录 (`logEvent`) |

### 调用方
| 文件 | 调用场景 |
|------|----------|
| `src/services/mcpServerApproval.tsx` | 主入口，处理待审批服务器列表 |

### 相关配置类型
```typescript
// src/utils/settings/types.ts 中的相关设置
interface SettingsJson {
  enabledMcpjsonServers?: string[];
  disabledMcpjsonServers?: string[];
  enableAllProjectMcpServers?: boolean;
}
```

---

## 依赖与外部交互

### 设置系统交互
```typescript
// 读取设置
const currentSettings = getSettings_DEPRECATED() || {};

// 写入设置
updateSettingsForSource("localSettings", {
  enabledMcpjsonServers: [...enabledServers, serverName]
});
```

### 分析系统交互
```typescript
logEvent("tengu_mcp_dialog_choice", {
  choice: value as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
});
```

### UI 组件层次
```
MCPServerApprovalDialog
├── Dialog (design-system)
│   ├── Title: "New MCP server found in .mcp.json: ${serverName}"
│   ├── MCPServerDialogCopy (安全提示)
│   └── Select (CustomSelect)
│       ├── Option: "Use this and all future MCP servers in this project"
│       ├── Option: "Use this MCP server"
│       └── Option: "Continue without using this MCP server"
```

---

## 风险、边界与改进建议

### 潜在风险

1. **设置竞态条件**
   - 问题：`getSettings_DEPRECATED()` 和 `updateSettingsForSource()` 之间可能存在竞态
   - 影响：如果用户在多个会话中同时审批，可能丢失设置更新
   - 缓解：设置更新使用文件锁和合并策略

2. **服务器名称规范化不一致**
   - 问题：审批时存储的是原始服务器名，但状态检查可能使用规范化名称
   - 相关代码：`src/services/mcp/utils.ts` 中的 `normalizeNameForMCP`
   - 建议：确保存储和检查时使用一致的名称格式

3. **分析数据敏感性**
   - 当前实现使用 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型标记
   - 确保服务器名称不包含敏感路径信息

### 边界情况

1. **空设置文件**
   - 处理：`getSettings_DEPRECATED() || {}` 提供默认值
   
2. **重复审批**
   - 处理：`includes` 检查防止重复添加

3. **快速连续调用**
   - `onDone` 回调后立即关闭对话框，防止重复操作

### 改进建议

1. **批量审批优化**
   - 当前：单服务器审批对话框
   - 建议：当多个服务器待审批时，考虑使用 `MCPServerMultiselectDialog` 统一处理

2. **审批历史记录**
   - 当前：仅记录启用/禁用列表
   - 建议：添加时间戳和审批上下文（如项目路径）

3. **撤销机制**
   - 当前：无直接撤销 UI
   - 建议：添加"撤销此选择"的快捷操作

4. **设置迁移**
   - `getSettings_DEPRECATED` 标记为废弃
   - 建议：迁移到新的设置 API (`getInitialSettings`)

5. **测试覆盖**
   - 建议添加单元测试：
     - 三种选择的设置更新逻辑
     - 重复审批的幂等性
     - 设置文件读写错误处理
