# Onboarding.tsx 研究文档

## 场景与职责

`Onboarding.tsx` 是 Claude Code 的用户引导流程核心组件，负责新用户首次使用时的多步骤配置向导。该组件在以下场景触发：

1. **首次启动**：用户第一次运行 `claude` 命令时，检测到 `hasCompletedOnboarding` 为 false
2. **版本重置**：当 `MIN_VERSION_REQUIRING_ONBOARDING_RESET` 要求时，强制重新引导
3. **OAuth 重新认证**：需要重新登录时触发部分流程

组件的主要职责包括：
- 引导用户完成终端环境检查（preflight checks）
- 主题选择（ThemePicker 集成）
- API Key 审批（环境变量中检测到 ANTHROPIC_API_KEY 时）
- OAuth 登录流程（ConsoleOAuthFlow 集成）
- 安全提示展示
- 终端设置优化（根据终端类型推荐设置）

## 功能点目的

### 1. 分步骤引导架构
组件采用动态步骤数组设计，根据用户环境条件决定显示哪些步骤：

```typescript
type StepId = 'preflight' | 'theme' | 'oauth' | 'api-key' | 'security' | 'terminal-setup';
```

步骤动态构建逻辑：
- **preflight**: 仅在 OAuth 启用时显示，检查系统环境
- **theme**: 始终显示，让用户选择界面主题
- **api-key**: 条件显示，当检测到 `ANTHROPIC_API_KEY` 环境变量且不在 homespace 环境时
- **oauth**: 条件显示，当 OAuth 启用且用户未跳过（通过 API Key 审批可跳过）
- **security**: 始终显示，展示安全提示
- **terminal-setup**: 条件显示，当 `shouldOfferTerminalSetup()` 返回 true

### 2. API Key 审批机制
当检测到环境变量中的 API Key 时，组件会截断显示（只显示最后20位），让用户选择是否使用：

```typescript
const apiKeyNeedingApproval = useMemo(() => {
  if (!process.env.ANTHROPIC_API_KEY || isRunningOnHomespace()) {
    return '';
  }
  const customApiKeyTruncated = normalizeApiKeyForConfig(process.env.ANTHROPIC_API_KEY);
  if (getCustomApiKeyStatus(customApiKeyTruncated) === 'new') {
    return customApiKeyTruncated;
  }
}, []);
```

### 3. 终端设置自动化
根据检测到的终端类型（Apple_Terminal 或其他），提供一键优化：
- **Apple Terminal**: Option+Enter 换行、视觉提示音
- **其他终端**: Shift+Enter 换行

### 4. 可跳过步骤机制
`SkippableStep` 子组件允许条件跳过某些步骤（如用户已批准 API Key 时跳过 OAuth）

## 具体技术实现

### 关键流程

#### 步骤导航流程
```
1. 初始化 currentStepIndex = 0
2. 根据环境条件构建 steps 数组
3. 渲染当前步骤组件
4. 用户完成当前步骤 → goToNextStep()
5. 记录分析事件 tengu_onboarding_step
6. 如果是最后一步 → 调用 onDone() 完成引导
```

#### OAuth 条件判断
```typescript
const [oauthEnabled] = useState(() => isAnthropicAuthEnabled());
```
`isAnthropicAuthEnabled()` 逻辑（位于 `src/utils/auth.ts`）：
- `--bare` 模式禁用 OAuth
- SSH 远程环境检查 `ANTHROPIC_UNIX_SOCKET`
- 第三方服务（Bedrock/Vertex/Foundry）禁用 OAuth
- 外部 API Key 或 Auth Token 存在时禁用

### 数据结构

#### OnboardingStep 接口
```typescript
interface OnboardingStep {
  id: StepId;
  component: React.ReactNode;
}
```

#### Props 定义
```typescript
type Props = {
  onDone(): void;  // 引导完成回调
};
```

### 关键状态管理
- `currentStepIndex`: 当前步骤索引
- `skipOAuth`: 是否跳过 OAuth（当用户批准 API Key 时设置）
- `theme`: 当前主题设置（来自 useTheme hook）

## 关键代码路径与文件引用

### 本文件关键代码
| 行号 | 功能 |
|------|------|
| 22-26 | StepId 和 OnboardingStep 类型定义 |
| 30-32 | Props 定义 |
| 42-53 | goToNextStep 步骤导航逻辑 |
| 98-177 | 动态步骤数组构建 |
| 182-203 | 键盘快捷键绑定 |
| 214-243 | SkippableStep 子组件 |

### 依赖文件引用

| 导入路径 | 用途 |
|----------|------|
| `src/services/analytics/index.js` | 分析事件记录 (logEvent) |
| `../commands/terminalSetup/terminalSetup.js` | 终端设置功能 |
| `../hooks/useExitOnCtrlCDWithKeybindings.js` | Ctrl+C/D 退出处理 |
| `../ink.js` | Ink UI 组件 (Box, Link, Newline, Text, useTheme) |
| `../keybindings/useKeybinding.js` | 键盘快捷键绑定 |
| `../utils/auth.js` | isAnthropicAuthEnabled 检查 |
| `../utils/authPortable.js` | normalizeApiKeyForConfig |
| `../utils/config.js` | getCustomApiKeyStatus |
| `../utils/env.js` | env 对象（检测终端类型） |
| `../utils/envUtils.js` | isRunningOnHomespace |
| `../utils/preflightChecks.js` | PreflightStep 组件 |
| `./ApproveApiKey.js` | API Key 审批组件 |
| `./ConsoleOAuthFlow.js` | OAuth 流程组件 |
| `./CustomSelect/select.js` | Select 组件 |
| `./LogoV2/WelcomeV2.js` | 欢迎界面 |
| `./PressEnterToContinue.js` | 按 Enter 继续提示 |
| `./ThemePicker.js` | 主题选择器 |
| `./ui/OrderedList.js` | 有序列表 UI |

## 依赖与外部交互

### 外部依赖
1. **React Compiler Runtime**: `react/compiler-runtime` 用于自动记忆化
2. **Ink**: 终端 UI 渲染库
3. **usehooks-ts**: 可能用于 useInterval（虽然本组件未直接使用）

### 内部模块交互
```
Onboarding.tsx
├── terminalSetup.ts (终端设置命令)
├── auth.ts (OAuth 启用检查)
├── config.ts (API Key 状态)
├── env.ts (终端类型检测)
├── PreflightStep (环境检查)
├── ApproveApiKey (API Key 审批)
├── ConsoleOAuthFlow (OAuth 登录)
├── ThemePicker (主题选择)
└── WelcomeV2 (欢迎界面)
```

### 分析事件
组件记录以下分析事件：
- `tengu_began_setup`: 引导开始时
- `tengu_onboarding_step`: 每步切换时（包含 stepId）

## 风险、边界与改进建议

### 已知风险

1. **步骤顺序依赖**
   - 步骤数组在渲染时动态构建，依赖多个状态变量
   - 如果依赖项变化（如 oauthEnabled 异步更新），可能导致步骤闪烁或跳转异常

2. **API Key 检测窗口**
   - `apiKeyNeedingApproval` 使用 `useMemo` 但依赖空数组，只在挂载时计算
   - 如果环境变量在引导过程中变化，不会重新检测

3. **终端设置错误处理**
   - `setupTerminal(theme).catch(() => {})` 静默吞掉错误
   - 用户无法得知终端设置是否成功

### 边界情况

1. **Homespace 环境**
   - 在内部 homespace 环境（`isRunningOnHomespace()`）中，忽略 `ANTHROPIC_API_KEY`
   - 这是为了强制使用 Console Key 而非个人 API Key

2. **Bare 模式**
   - `--bare` 标志禁用 OAuth，此时 `oauthEnabled` 为 false
   - 步骤数组会跳过 preflight、oauth 步骤

3. **Ctrl+C/D 处理**
   - 使用 `useExitOnCtrlCDWithKeybindings` 处理退出
   - 显示 "Press Ctrl+C again to exit" 提示

### 改进建议

1. **错误处理增强**
   ```typescript
   // 当前
   void setupTerminal(theme).catch(() => {}).finally(goToNextStep);
   
   // 建议：至少记录日志
   void setupTerminal(theme)
     .catch(err => logForDebugging(`Terminal setup failed: ${err}`))
     .finally(goToNextStep);
   ```

2. **步骤持久化**
   - 当前如果用户在引导过程中意外退出，需要从头开始
   - 建议：将当前步骤索引持久化到配置，支持断点续传

3. **键盘导航改进**
   - 当前仅支持 Enter 确认、Esc 跳过
   - 建议：支持数字键直接选择选项（如按 "1" 选择第一个主题）

4. **可访问性**
   - 当前没有为屏幕阅读器优化
   - 建议：添加 ARIA 标签（虽然终端环境有限）

5. **测试覆盖**
   - 步骤动态构建逻辑复杂，建议添加单元测试
   - 特别是 `apiKeyNeedingApproval` 和 `shouldOfferTerminalSetup` 的条件组合

### 相关 Issue/PR 参考
- 配置保存安全机制：GH #3117（`wouldLoseAuthState` 检查）
- API Key 助手安全：信任检查防止项目设置中的恶意配置
