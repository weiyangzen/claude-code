# SandboxPermissionRequest.tsx 研究文档

## 场景与职责

`SandboxPermissionRequest.tsx` 是 Claude Code CLI 沙箱系统的**网络权限请求组件**。当沙箱内的工具尝试连接到未在允许列表中的网络主机时，该组件会向用户显示一个交互式对话框，请求用户批准或拒绝该网络连接。

该组件是沙箱安全模型的关键组成部分，确保：
- 用户明确知晓哪些外部网络请求正在发生
- 用户可以选择性地持久化允许规则（"不再询问"）
- 在托管域限制模式下（managed domains only），限制用户的选择范围

## 功能点目的

### 1. 网络权限请求 UI
提供一个清晰的界面展示：
- 目标主机信息（host）
- 连接请求上下文
- 用户响应选项

### 2. 托管域策略支持
当 `shouldAllowManagedSandboxDomainsOnly()` 返回 true 时：
- 仅允许一次性授权（不允许"不再询问"）
- 确保策略管理的域列表不被用户覆盖

### 3. 用户响应处理
支持三种用户响应：
- **Yes**: 允许本次连接
- **Yes, don't ask again**: 允许并持久化到设置
- **No**: 拒绝连接并提供反馈选项

## 具体技术实现

### 核心数据结构

```typescript
// 组件 Props
export type SandboxPermissionRequestProps = {
  hostPattern: NetworkHostPattern;  // 来自 @anthropic-ai/sandbox-runtime
  onUserResponse: (response: {
    allow: boolean;
    persistToSettings: boolean;
  }) => void;
};

// 内部使用的选项值
// "yes" | "yes-dont-ask-again" | "no"
```

### 关键流程

1. **选项生成流程**:
   ```
   shouldAllowManagedSandboxDomainsOnly() 
     → 决定是否包含 "yes-dont-ask-again" 选项
     → 构建 Select 组件的 options 数组
   ```

2. **用户选择处理** (`onSelect`):
   ```typescript
   switch (value) {
     case "yes":
       onUserResponse({ allow: true, persistToSettings: false });
       break;
     case "yes-dont-ask-again":
       onUserResponse({ allow: true, persistToSettings: true });
       break;
     case "no":
       onUserResponse({ allow: false, persistToSettings: false });
       break;
   }
   ```

3. **渲染结构**:
   ```
   PermissionDialog (title: "Network request outside of sandbox")
     ├── Box (host 信息)
     ├── Box (提示文本: "Do you want to allow this connection?")
     └── Select (选项列表)
   ```

### 关键代码路径

```typescript
// 沙箱适配器
import { 
  type NetworkHostPattern, 
  shouldAllowManagedSandboxDomainsOnly 
} from 'src/utils/sandbox/sandbox-adapter.js';

// 分析服务（虽然导入但未在组件中使用）
import { 
  type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS, 
  logEvent 
} from '../../services/analytics/index.js';

// UI 组件
import { Select } from '../CustomSelect/select.js';
import { PermissionDialog } from './PermissionDialog.js';
```

### React Compiler 优化

组件使用 React Compiler 的缓存机制：
- 缓存 `onSelect` 回调函数
- 缓存 `managedDomainsOnly` 计算结果
- 缓存选项数组的构建
- 缓存所有 JSX 元素

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `NetworkHostPattern` | `src/utils/sandbox/sandbox-adapter.js` | 主机模式类型 |
| `shouldAllowManagedSandboxDomainsOnly` | `src/utils/sandbox/sandbox-adapter.js` | 检查托管域策略 |
| `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | `../../services/analytics/index.js` | 分析元数据类型（未使用） |
| `logEvent` | `../../services/analytics/index.js` | 分析日志（未使用） |
| `Select` | `../CustomSelect/select.js` | 选择组件 |
| `PermissionDialog` | `./PermissionDialog.js` | 权限对话框容器 |

### 被调用方

该组件通过 `SandboxManager` 的 `initialize` 函数作为回调被调用：

```typescript
// sandbox-adapter.ts
export async function initialize(sandboxAskCallback?: SandboxAskCallback): Promise<void> {
  // ...
  const wrappedCallback: SandboxAskCallback | undefined = sandboxAskCallback
    ? async (hostPattern: NetworkHostPattern) => {
        if (shouldAllowManagedSandboxDomainsOnly()) {
          return false;  // 直接拒绝，不显示 UI
        }
        return sandboxAskCallback(hostPattern);  // 调用传入的回调
      }
    : undefined;
  // ...
}
```

实际调用链：
```
SandboxManager.initialize()
  → BaseSandboxManager.initialize(runtimeConfig, wrappedCallback)
    → 当网络请求被阻止时调用 wrappedCallback
      → 实际显示 SandboxPermissionRequest 组件
```

### 数据流

```
沙箱运行时 (@anthropic-ai/sandbox-runtime)
  ↓ 检测到未授权的网络请求
BaseSandboxManager
  ↓ 调用 SandboxAskCallback
Claude CLI 权限系统
  ↓ 渲染 SandboxPermissionRequest 组件
用户交互
  ↓ onUserResponse 回调
沙箱运行时继续/拒绝请求
```

## 风险、边界与改进建议

### 当前风险

1. **未使用的分析导入**：
   - 导入了 `logEvent` 和 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 但未使用
   - 可能是遗留代码或 TODO 项

2. **硬编码的选项标签**：
   - 所有 UI 文本都是硬编码的英文
   - 不支持国际化

3. **托管域策略的 UX 问题**：
   - 当 `shouldAllowManagedSandboxDomainsOnly()` 为 true 时，"不再询问"选项被隐藏
   - 用户可能不理解为什么某些情况下没有这个选项

### 边界情况

1. **空主机名**：如果 `hostPattern.host` 为空字符串，组件仍会渲染但显示空主机名

2. **快速连续请求**：如果多个网络请求同时被阻止，每个请求都会触发一个独立的权限对话框

3. **ESC 取消**：用户可以通过 ESC 键取消选择，触发 `onCancel` 回调（拒绝连接）

### 改进建议

1. **添加分析日志**：
   ```typescript
   // 在 onSelect 中添加
   logEvent('tengu_sandbox_network_permission_response', {
     host: hostPattern.host,
     allowed: allow,
     persisted: persistToSettings,
   });
   ```

2. **国际化支持**：
   - 将所有硬编码字符串提取到翻译文件
   - 支持动态语言切换

3. **主机信息增强**：
   - 显示更多主机信息（如 IP 地址、端口、协议）
   - 显示请求的上下文（哪个工具发起的请求）

4. **批量请求处理**：
   - 当多个请求同时发生时，考虑批量显示
   - 避免对话框叠加

5. **托管域策略提示**：
   - 当"不再询问"选项被禁用时，显示提示说明原因
   - 例如："Managed domain policy is active. Contact your administrator to add domains."

6. **超时处理**：
   - 添加超时机制，避免无限期等待用户响应
   - 超时时默认拒绝连接

7. **历史记录**：
   - 显示用户之前对该域的决策历史
   - 帮助用户做出更明智的决定
