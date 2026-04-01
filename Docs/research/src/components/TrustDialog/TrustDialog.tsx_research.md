# TrustDialog.tsx 深度研究文档

## 场景与职责

`TrustDialog` 是 Claude Code CLI 的**工作区信任边界**核心组件，负责在用户首次进入工作目录时显示安全确认对话框。这是防止潜在恶意代码执行的第一道防线，确保用户了解并同意当前工作目录的安全风险。

### 核心职责

1. **工作区信任确认**：询问用户是否信任当前文件夹（自己创建的代码、知名开源项目、团队工作）
2. **安全风险披露**：告知用户 Claude Code 将能够读取、编辑和执行该目录下的文件
3. **安全功能检测**：扫描并报告可能带来安全风险的功能配置（hooks、bash权限、MCP服务器等）
4. **信任状态持久化**：将用户决策保存到项目配置，支持父子目录信任继承

### 触发场景

- 用户首次进入某个工作目录启动 Claude Code
- 当前目录的信任状态未被记录
- 非 CI/CD 环境（交互式会话）
- 非 `CLAUBBIT` 测试环境

---

## 功能点目的

### 1. 信任状态检查与快速跳过

```typescript
const hasTrustDialogAccepted = checkHasTrustDialogAccepted();
if (hasTrustDialogAccepted) {
  setTimeout(onDone);
  return null;
}
```

- 调用 `checkHasTrustDialogAccepted()` 检查当前目录或父目录是否已接受信任
- 若已信任，立即通过 `setTimeout` 异步调用 `onDone` 回调，不渲染对话框

### 2. 安全功能扫描与遥测

组件会扫描以下可能带来安全风险的功能配置：

| 检测项 | 检测函数 | 来源配置 |
|--------|----------|----------|
| MCP 服务器 | `getMcpConfigsByScope("project")` | `.mcp.json` |
| Hooks | `getHooksSources()` | `.claude/settings.json` |
| Bash 执行权限 | `getBashPermissionSources()` | `.claude/settings.json` 权限规则 |
| API Key Helper | `getApiKeyHelperSources()` | `.claude/settings.json` |
| AWS 命令 | `getAwsCommandsSources()` | `.claude/settings.json` |
| GCP 命令 | `getGcpCommandsSources()` | `.claude/settings.json` |
| OTel Headers Helper | `getOtelHeadersHelperSources()` | `.claude/settings.json` |
| 危险环境变量 | `getDangerousEnvVarsSources()` | `.claude/settings.json` |
| Slash 命令 Bash | 检查 `commands` 数组 | 已弃用的 `commands/` 目录 |
| Skills Bash | 检查 `commands` 数组 | Skills/Plugin 加载的命令 |

遥测事件：
- `tengu_trust_dialog_shown`：对话框显示时上报，包含各项安全功能的检测结果
- `tengu_trust_dialog_accept`：用户接受信任时上报

### 3. 用户决策处理

用户有两个选项：
- **"Yes, I trust this folder"** (`enable_all`)：接受信任
- **"No, exit"** (`exit`)：退出程序

信任接受后的处理：
```typescript
const onChange = (value: 'enable_all' | 'exit') => {
  if (value === "exit") {
    gracefulShutdownSync(1);  // 退出码 1
    return;
  }
  
  // 记录遥测
  logEvent("tengu_trust_dialog_accept", {...});
  
  // 信任状态持久化
  if (isHomeDir) {
    setSessionTrustAccepted(true);  // 主目录：仅会话级信任
  } else {
    saveCurrentProjectConfig({ hasTrustDialogAccepted: true });  // 其他目录：持久化到配置
  }
  
  onDone();
};
```

### 4. 特殊处理：主目录场景

当用户在主目录（`homedir() === getCwd()`）运行 Claude Code 时：
- 信任状态**不持久化**到磁盘（避免污染全局配置）
- 使用**会话级信任** (`setSessionTrustAccepted`)，仅在当前进程有效
- 遥测中标记 `isHomeDir: true`

---

## 具体技术实现

### 关键流程

```
TrustDialog 渲染流程：
1. 检查 hasTrustDialogAccepted → 已信任则直接 onDone()
2. 并行检测各项安全功能（MCP、Hooks、Bash权限等）
3. 渲染 PermissionDialog 包裹的 UI
4. 用户选择后：
   - Exit → gracefulShutdownSync(1)
   - Enable All → 记录遥测 + 持久化信任 + onDone()
```

### 数据结构

#### Props 定义
```typescript
type Props = {
  onDone(): void;           // 完成回调
  commands?: Command[];     // 可选的命令列表（用于检测 skills/slash 命令中的 bash）
};
```

#### 安全检测辅助函数（来自 utils.ts）

```typescript
// 返回包含该功能的配置文件路径数组
export function getHooksSources(): string[]
export function getBashPermissionSources(): string[]
export function getApiKeyHelperSources(): string[]
export function getAwsCommandsSources(): string[]
export function getGcpCommandsSources(): string[]
export function getOtelHeadersHelperSources(): string[]
export function getDangerousEnvVarsSources(): string[]
```

### React Compiler 优化

代码使用 React Compiler（`"react/compiler-runtime"`）进行自动记忆化：

```typescript
const $ = _c(33);  // 创建缓存数组，33 个槽位

// 模式：检查缓存，命中则复用，否则计算并缓存
let t1;
if ($[0] === Symbol.for("react.memo_cache_sentinel")) {
  t1 = getMcpConfigsByScope("project");
  $[0] = t1;
} else {
  t1 = $[0];
}
```

### 键盘快捷键支持

```typescript
// Ctrl+C/D 退出支持
const exitState = useExitOnCtrlCDWithKeybindings(() => gracefulShutdownSync(1));

// confirm:no 快捷键绑定（拒绝信任）
useKeybinding("confirm:no", () => gracefulShutdownSync(0), { context: "Confirmation" });
```

---

## 关键代码路径与文件引用

### 核心文件

| 文件 | 作用 |
|------|------|
| `src/components/TrustDialog/TrustDialog.tsx` | 主组件实现 |
| `src/components/TrustDialog/utils.ts` | 安全功能检测辅助函数 |
| `src/utils/config.ts` | 信任状态持久化 (`checkHasTrustDialogAccepted`, `saveCurrentProjectConfig`) |
| `src/bootstrap/state.ts` | 会话级信任状态 (`setSessionTrustAccepted`, `getSessionTrustAccepted`) |

### 依赖的外部模块

```typescript
// 配置与状态
import { setSessionTrustAccepted } from '../../bootstrap/state.js';
import { checkHasTrustDialogAccepted, saveCurrentProjectConfig } from '../../utils/config.js';

// MCP 配置检测
import { getMcpConfigsByScope } from '../../services/mcp/config.js';

// 安全功能检测（来自同目录 utils.ts）
import { 
  getApiKeyHelperSources, 
  getAwsCommandsSources, 
  getBashPermissionSources,
  getDangerousEnvVarsSources, 
  getGcpCommandsSources, 
  getHooksSources, 
  getOtelHeadersHelperSources 
} from './utils.js';

// UI 组件
import { Select } from '../CustomSelect/index.js';
import { PermissionDialog } from '../permissions/PermissionDialog.js';

// 工具与钩子
import { useExitOnCtrlCDWithKeybindings } from '../../hooks/useExitOnCtrlCDWithKeybindings.js';
import { useKeybinding } from '../../keybindings/useKeybinding.js';
import { gracefulShutdownSync } from '../../utils/gracefulShutdown.js';
import { logEvent } from 'src/services/analytics/index.js';
```

### 调用方

主要调用方在 `src/interactiveHelpers.tsx` 的 `showSetupScreens` 函数：

```typescript
// 始终显示信任对话框（交互式会话）
if (!isEnvTruthy(process.env.CLAUBBIT)) {
  if (!checkHasTrustDialogAccepted()) {
    const { TrustDialog } = await import('./components/TrustDialog/TrustDialog.js');
    await showSetupDialog(root, done => <TrustDialog commands={commands} onDone={done} />);
  }
  
  // 标记信任已验证
  setSessionTrustAccepted(true);
  
  // 重置并重新初始化 GrowthBook（信任后可使用 auth headers）
  resetGrowthBook();
  void initializeGrowthBook();
  
  // 预取系统上下文
  void getSystemContext();
}
```

---

## 依赖与外部交互

### 依赖服务

| 服务 | 用途 |
|------|------|
| `getMcpConfigsByScope` | 检测项目级 MCP 服务器配置 |
| `getSettingsForSource` | 读取项目/本地设置 |
| `getPermissionRulesForSource` | 读取权限规则 |
| `logEvent` | 遥测上报 |
| `checkHasTrustDialogAccepted` | 检查信任状态（支持父子目录继承） |
| `saveCurrentProjectConfig` | 持久化信任状态到项目配置 |
| `setSessionTrustAccepted` | 设置会话级信任状态（主目录场景） |

### 配置依赖

信任状态存储位置：
- **常规目录**：`~/.claude.json` 中的 `projects[projectPath].hasTrustDialogAccepted`
- **主目录**：仅内存中的 `sessionTrustAccepted` 标志

父子目录继承逻辑（`src/utils/config.ts`）：
```typescript
function computeTrustDialogAccepted(): boolean {
  // 1. 检查会话级信任
  if (getSessionTrustAccepted()) return true;
  
  // 2. 检查项目路径的配置
  const projectPath = getProjectPathForConfig();
  if (config.projects?.[projectPath]?.hasTrustDialogAccepted) return true;
  
  // 3. 向上遍历父目录
  let currentPath = normalizePathForConfigKey(getCwd());
  while (true) {
    if (config.projects?.[currentPath]?.hasTrustDialogAccepted) return true;
    const parentPath = normalizePathForConfigKey(resolve(currentPath, '..'));
    if (parentPath === currentPath) break;
    currentPath = parentPath;
  }
  return false;
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **SessionEnd Hooks 执行风险**（已修复）
   - 历史问题：用户拒绝信任对话框后，SessionEnd hooks 仍可能执行
   - 修复：`shouldSkipHookDueToTrust()` 函数现在强制要求所有 hooks 都需要工作区信任

2. **主目录信任不持久化**
   - 设计如此，但用户可能期望在主目录的信任决策被记住
   - 每次新会话都需要重新确认

3. **遥测数据可能不完整**
   - 如果用户在显示对话框后立即关闭终端，accept 事件可能丢失
   - 但 `tengu_trust_dialog_shown` 事件已发送，可推断转化率

### 边界情况

| 场景 | 行为 |
|------|------|
| 非交互式会话 (CI/CD) | 不显示对话框，信任检查被跳过 |
| CLAUBBIT 环境 | 完全跳过信任对话框 |
| 父目录已信任 | 子目录自动继承信任状态 |
| 配置损坏 | `checkHasTrustDialogAccepted` 返回 false，显示对话框 |
| 快速连续切换目录 | 每次切换都重新检查，但缓存机制避免重复磁盘读取 |

### 改进建议

1. **增强安全功能展示**
   - 当前仅通过遥测上报，用户界面不显示检测到的具体风险
   - 建议在对话框中列出检测到的安全功能（如 "检测到 3 个 MCP 服务器、2 个 Hooks"）

2. **信任撤销机制**
   - 当前信任是单向的（false → true），不支持撤销
   - 建议添加 `/untrust` 命令或配置选项允许用户撤销信任

3. **更细粒度的信任控制**
   - 当前是"全有或全无"模式
   - 可考虑允许用户选择信任级别（如只读、允许编辑但不允许执行等）

4. **信任过期机制**
   - 长期未访问的项目信任状态可设置过期时间
   - 重新进入时再次确认

5. **团队协作场景优化**
   - 支持 `.claude/trust.json` 文件在团队间共享信任决策
   - 需配合签名验证防止篡改

### 测试建议

当前未发现针对 TrustDialog 的单元测试，建议添加：

1. **组件渲染测试**：验证对话框正确渲染选项和文本
2. **信任状态检查测试**：模拟已信任/未信任状态的行为
3. **安全功能检测测试**：验证各项检测函数正确识别配置
4. **主目录特殊处理测试**：验证主目录场景使用会话级信任
5. **键盘交互测试**：测试 Enter、Esc、Ctrl+C 的行为
