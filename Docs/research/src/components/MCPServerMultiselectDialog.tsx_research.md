# MCPServerMultiselectDialog.tsx 研究文档

## 场景与职责

MCPServerMultiselectDialog 是 Claude Code CLI 中用于**批量审批 MCP 服务器**的交互式对话框。当项目 `.mcp.json` 文件中定义了多个新的 MCP 服务器时，此组件会显示一个多选界面，允许用户一次性选择要启用的服务器。

### 核心职责
1. **批量审批**：同时展示多个待审批的 MCP 服务器
2. **灵活选择**：用户可以选择启用全部、部分或拒绝全部
3. **设置持久化**：根据选择更新启用/禁用列表
4. **分析追踪**：记录批量审批的统计信息

### 触发场景
- 启动时检测到多个 `pending` 状态的 MCP 服务器
- 服务器数量大于 1 时替代 `MCPServerApprovalDialog`

---

## 功能点目的

### 1. 批量选择界面
- 使用 `SelectMulti` 组件展示所有待审批服务器
- 默认全部选中（`defaultValue={serverNames}`）
- 支持 Space 键切换选择，Enter 键确认

### 2. 批量处理逻辑
根据用户选择将服务器分为两组：
- **Approved Servers**: 用户选中的服务器 → 添加到 `enabledMcpjsonServers`
- **Rejected Servers**: 用户未选中的服务器 → 添加到 `disabledMcpjsonServers`

### 3. Esc 键拒绝全部
- 按 Esc 键时，将所有服务器标记为禁用
- 提供快速拒绝全部的功能

### 4. 分析统计
记录批量审批的统计信息：
```typescript
logEvent("tengu_mcp_multidialog_choice", {
  approved: approvedServers.length,
  rejected: rejectedServers.length
});
```

---

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
interface Props {
  serverNames: string[];  // 待审批的服务器名称列表
  onDone(): void;         // 完成回调
}

// 服务器分组结果
const [approvedServers, rejectedServers] = partition(
  serverNames,
  server => selectedServers.includes(server)
);
```

### 核心流程

1. **提交处理** (`onSubmit`)
   ```typescript
   function onSubmit(selectedServers: string[]) {
     // 1. 获取当前设置
     const currentSettings = getSettings_DEPRECATED() || {};
     const enabledServers = currentSettings.enabledMcpjsonServers || [];
     const disabledServers = currentSettings.disabledMcpjsonServers || [];
     
     // 2. 使用 lodash partition 分组服务器
     const [approvedServers, rejectedServers] = partition(
       serverNames,
       server => selectedServers.includes(server)
     );
     
     // 3. 记录分析事件
     logEvent("tengu_mcp_multidialog_choice", {
       approved: approvedServers.length,
       rejected: rejectedServers.length
     });
     
     // 4. 更新启用列表
     if (approvedServers.length > 0) {
       const newEnabledServers = [...new Set([...enabledServers, ...approvedServers])];
       updateSettingsForSource("localSettings", {
         enabledMcpjsonServers: newEnabledServers
       });
     }
     
     // 5. 更新禁用列表
     if (rejectedServers.length > 0) {
       const newDisabledServers = [...new Set([...disabledServers, ...rejectedServers])];
       updateSettingsForSource("localSettings", {
         disabledMcpjsonServers: newDisabledServers
       });
     }
     
     // 6. 完成
     onDone();
   }
   ```

2. **Esc 拒绝全部处理** (`handleEscRejectAll`)
   ```typescript
   const handleEscRejectAll = () => {
     const currentSettings = getSettings_DEPRECATED() || {};
     const disabledServers = currentSettings.disabledMcpjsonServers || [];
     // 将所有服务器添加到禁用列表
     const newDisabledServers = [...new Set([...disabledServers, ...serverNames])];
     updateSettingsForSource("localSettings", {
       disabledMcpjsonServers: newDisabledServers
     });
     onDone();
   };
   ```

3. **选项生成**
   ```typescript
   const options = serverNames.map(server => ({
     label: server,
     value: server
   }));
   ```

### UI 渲染结构

```
Dialog (title="${count} new MCP servers found in .mcp.json")
├── Subtitle: "Select any you wish to enable."
├── MCPServerDialogCopy (安全提示)
├── SelectMulti
│   ├── options: 服务器列表
│   ├── defaultValue: 全部选中
│   ├── onSubmit: onSubmit
│   └── onCancel: handleEscRejectAll
└── Keyboard Hints (Box)
    ├── Space: select
    ├── Enter: confirm
    └── Esc: reject all
```

---

## 关键代码路径与文件引用

### 本文件
- `/src/components/MCPServerMultiselectDialog.tsx` (133 行)

### 直接依赖
| 文件 | 用途 |
|------|------|
| `lodash-es/partition.js` | 数组分组工具 |
| `src/components/MCPServerDialogCopy.tsx` | 安全提示文本 |
| `src/components/CustomSelect/SelectMulti.js` | 多选组件 |
| `src/components/design-system/Dialog.js` | 对话框容器 |
| `src/components/design-system/Byline.js` | 键盘提示布局 |
| `src/components/design-system/KeyboardShortcutHint.js` | 快捷键提示 |
| `src/components/ConfigurableShortcutHint.js` | 可配置快捷键提示 |
| `src/utils/settings/settings.js` | 设置读写 |
| `src/services/analytics/index.js` | 分析事件 |
| `src/ink.js` | Ink 组件 (`Box`, `Text`) |

### 调用方
| 文件 | 调用场景 |
|------|----------|
| `src/services/mcpServerApproval.tsx` | 当 `pendingServers.length > 1` 时 |

### 调用代码
```typescript
// src/services/mcpServerApproval.tsx
if (pendingServers.length === 1) {
  root.render(<MCPServerApprovalDialog ... />);
} else {
  root.render(<MCPServerMultiselectDialog 
    serverNames={pendingServers} 
    onDone={done} 
  />);
}
```

---

## 依赖与外部交互

### Lodash Partition
```typescript
import partition from 'lodash-es/partition.js';

// 将服务器分为两组：选中的和未选中的
const [approvedServers, rejectedServers] = partition(
  serverNames,
  server => selectedServers.includes(server)
);
```

### 设置系统交互
```typescript
// 读取当前设置
const currentSettings = getSettings_DEPRECATED() || {};

// 合并并去重
const newEnabledServers = [...new Set([...enabledServers, ...approvedServers])];

// 写入设置
updateSettingsForSource("localSettings", {
  enabledMcpjsonServers: newEnabledServers
});
```

### 分析系统交互
```typescript
logEvent("tengu_mcp_multidialog_choice", {
  approved: approvedServers.length,
  rejected: rejectedServers.length
});
```

---

## 风险、边界与改进建议

### 潜在风险

1. **设置更新竞态**
   - 问题：批量更新启用和禁用列表是两个独立调用
   - 风险：中间状态不一致（虽然实际影响小）
   - 建议：考虑合并为单个设置更新

2. **大列表性能**
   - 问题：如果 `.mcp.json` 包含大量服务器（如 50+）
   - 风险：`SelectMulti` 渲染性能下降
   - 当前缓解：`visibleOptionCount` 限制可见数量

3. **用户误操作**
   - 问题：Esc 键直接拒绝全部，无二次确认
   - 风险：用户可能误按 Esc 导致全部拒绝
   - 建议：添加确认提示或撤销机制

4. **名称规范化不一致**
   - 问题：存储原始名称，但状态检查可能使用规范化名称
   - 相关：`normalizeNameForMCP` 在 `src/services/mcp/utils.ts`

### 边界情况

1. **空列表**
   - 处理：`serverNames` 为空数组时，对话框显示 "0 new MCP servers"
   - 实际：调用方应提前检查，不会渲染空列表

2. **全选与全不选**
   - 全选：所有服务器添加到启用列表
   - 全不选：所有服务器添加到禁用列表

3. **重复提交**
   - `onDone()` 后立即关闭对话框，防止重复操作

### 改进建议

1. **确认对话框**
   - 当用户拒绝大量服务器时（如 >5），显示确认提示

2. **服务器详情展示**
   - 显示每个服务器的类型（stdio/sse/http）和描述
   - 帮助用户做出更明智的选择

3. **搜索/过滤**
   - 当服务器数量多时，添加搜索功能

4. **批量操作优化**
   - 添加"全选"/"全不选"快捷键
   - 支持按类型筛选

5. **设置事务性**
   ```typescript
   // 建议：合并为单次更新
   updateSettingsForSource("localSettings", {
     enabledMcpjsonServers: newEnabled,
     disabledMcpjsonServers: newDisabled
   });
   ```

6. **测试覆盖**
   ```typescript
   // 建议添加的测试
   describe('MCPServerMultiselectDialog', () => {
     it('partitions servers correctly on submit', () => {});
     it('handles empty selection (reject all)', () => {});
     it('deduplicates server names in settings', () => {});
     it('logs analytics with correct counts', () => {});
   });
   ```

7. **使用新设置 API**
   - 迁移 `getSettings_DEPRECATED` 到 `getInitialSettings`

### 代码质量

**优点**：
- 使用 `partition` 简化分组逻辑
- 使用 `Set` 去重，代码简洁
- React Compiler 优化渲染性能

**可改进**：
- `onSubmit` 和 `handleEscRejectAll` 有重复的设置更新逻辑，可提取公共函数
- 硬编码的分析事件名称应提取为常量
