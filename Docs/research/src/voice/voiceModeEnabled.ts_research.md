# 研究文档：src/voice/voiceModeEnabled.ts

> 研究范围：代码、脚本、配置、测试及必要实现上下文。
> 研究对象：`src/voice/voiceModeEnabled.ts` 及其调用方、被调用方、相关配置与运行时依赖。

---

## 一、场景与职责

`src/voice/voiceModeEnabled.ts` 是 Claude Code **语音模式（Voice Mode）的中央门禁文件**。它不承担任何录音、STT（Speech-to-Text）或 UI 渲染的具体实现，而是作为**纯同步的判断层**，决定语音功能在“当前运行时刻、当前用户身份、当前编译产物”下是否应当暴露或可用。

### 1.1 业务场景

- **编译期特性开关（DCE）**：外部构建（非 ant 构建）通过 `bun:bundle` 的 `feature('VOICE_MODE')` 在打包阶段做死代码消除（Dead Code Elimination），语音相关模块可能根本不会进入产物。
- **运行时紧急熔断（Kill-switch）**：已上线的 ant 构建通过 GrowthBook 远程配置 `tengu_amber_quartz_disabled` 实现紧急关闭，无需发版即可全量或灰度关闭语音。
- **身份与授权门禁**：语音模式依赖 Anthropic OAuth（`claude.ai` 账号），因为后端 `voice_stream` WebSocket 端点不接受 API Key、Bedrock、Vertex 或 Foundry 身份。该文件负责统一校验 OAuth token 的存在性。
- **命令/UI 可见性分层**：
  - 命令注册层（`src/commands/voice/index.ts`）用 `isVoiceGrowthBookEnabled()` 决定 `/voice` 命令是否注册；
  - 命令执行层（`src/commands/voice/voice.ts`）和配置工具（`ConfigTool`）用 `isVoiceModeEnabled()` 做完整校验；
  - React 渲染层（`useVoiceEnabled`、`VoiceModeNotice`）用 `hasVoiceAuth()` + GB 检查做轻量判断。

### 1.2 职责边界

| 职责 | 说明 |
|------|------|
| 不做录音实现 | 录音逻辑在 `src/services/voice.ts`（native/SoX/arecord） |
| 不做 STT 协议实现 | WebSocket STT 在 `src/services/voiceStreamSTT.ts` |
| 不做设置持久化 | `voiceEnabled` 的读写由 `src/utils/settings/settings.ts` 与 `ConfigTool` 负责 |
| 只做“能不能用”的判断 | 输出布尔值，供调用方做分支/隐藏/报错 |

---

## 二、功能点目的

文件导出三个函数，形成**两层门禁 + 一个 React 优化接口**的体系：

### 2.1 `isVoiceGrowthBookEnabled(): boolean`

**目的**：GrowthBook kill-switch 检查。决定语音模式是否“应当可见”。

- 先检查编译期开关 `feature('VOICE_MODE')`；若外部构建已剔除，直接返回 `false`。
- 再读取 GrowthBook flag `tengu_amber_quartz_disabled`：
  - 该 flag 为 **负向语义**（`true` = 禁用语音）。
  - 默认 fallback 为 `false`，意味着如果磁盘缓存缺失或失效，新安装用户会默认可见语音功能，无需等待 GrowthBook 初始化完成。
- 使用 **positive ternary pattern**（`feature(...) ? !getFeatureValue(...) : false`），这是项目 feature-gating 规范要求的代码模式，用于确保外部构建时字符串字面量能被正确消除，避免泄露内部 flag 名称。

### 2.2 `hasVoiceAuth(): boolean`

**目的**：OAuth 身份校验。判断当前用户是否具备调用 Anthropic `voice_stream` 端点的身份凭证。

- 先调用 `isAnthropicAuthEnabled()` 确认当前认证 provider 是 Anthropic OAuth（而非 API Key / Bedrock / Vertex / Foundry / apiKeyHelper）。
- 再调用 `getClaudeAIOAuthTokens()` 确认 `accessToken` 真实存在。
- 注释明确指出：`isAnthropicAuthEnabled()` 只检查 provider 配置，不检查 token 是否存在；缺少 `hasVoiceAuth()` 的二次校验会导致语音 UI 渲染出来，但后续 `connectVoiceStream` 静默失败。

### 2.3 `isVoiceModeEnabled(): boolean`

**目的**：完整运行时门禁。用于**命令执行时刻**或**配置写入时刻**的严格校验。

- 逻辑为 `hasVoiceAuth() && isVoiceGrowthBookEnabled()`。
- 注释区分了使用场景：
  - 命令时间路径（`/voice` 命令、`ConfigTool`、`VoiceModeNotice`）允许同步读 keychain；
  - React 渲染路径应使用 `useVoiceEnabled()`（对 auth 半区做 memoization，避免重复 spawn `security`）。

---

## 三、具体技术实现

### 3.1 关键流程

#### 3.1.1 语音命令注册与执行的分层校验流程

```
src/commands.ts (build-time DCE)
  └── feature('VOICE_MODE') ? require('./commands/voice/index.ts') : null

src/commands/voice/index.ts
  ├── isEnabled: () => isVoiceGrowthBookEnabled()      // 决定命令是否注册
  └── isHidden: !isVoiceModeEnabled()                   // 决定命令是否对用户可见

src/commands/voice/voice.ts (call)
  └── if (!isVoiceModeEnabled()) { return error text }  // 执行时二次校验
      └── 若通过，再检查 recording / voiceStream / mic permission
```

#### 3.1.2 React 渲染路径的优化校验流程

```
src/hooks/useVoiceEnabled.ts
  ├── userIntent = useAppState(s => s.settings.voiceEnabled === true)
  ├── authVersion = useAppState(s => s.authVersion)
  ├── authed = useMemo(hasVoiceAuth, [authVersion])    // memoize 昂贵 auth 检查
  └── return userIntent && authed && isVoiceGrowthBookEnabled()
```

- `authVersion` 在 `/login` 时 bump，背景 token refresh 不 bump，因此 memoized auth 结果在常规刷新周期内保持稳定。
- GB 检查**不放入 `useMemo`**，因为 GB 是廉价的缓存 Map 查找，且需要让 mid-session kill-switch 翻转在下次 render 立即生效。

#### 3.1.3 ConfigTool 中的语音设置门控流程

```
src/tools/ConfigTool/ConfigTool.ts
  ├── GET/SET 'voiceEnabled' 时：
  │   ├── 若 feature('VOICE_MODE') 且 setting === 'voiceEnabled'：
  │   │   └── 动态 import('../../voice/voiceModeEnabled.js')
  │   │       └── !isVoiceGrowthBookEnabled() → 返回 "Unknown setting"（kill-switch 开启时隐藏）
  │   └── SET voiceEnabled=true 时：
  │       └── 动态 import → !isVoiceModeEnabled() → 返回 auth/availability 错误
  │           └── 再通过 isVoiceStreamAvailable() / checkRecordingAvailability() 等做预检
```

- 使用动态 `import()` 是为了在 kill-switch 关闭时避免将 `voiceModeEnabled.ts` 及其依赖静态引入到 ConfigTool 的启动路径中（进一步减少外部构建体积和启动开销）。

### 3.2 数据结构

#### 3.2.1 GrowthBook 缓存读取

`getFeatureValue_CACHED_MAY_BE_STALE<T>(feature: string, defaultValue: T): T`（`src/services/analytics/growthbook.ts`）

- **优先级**：环境变量覆盖（`CLAUDE_INTERNAL_FC_OVERRIDES`）> 本地配置覆盖（`growthBookOverrides`）> 内存缓存（`remoteEvalFeatureValues` Map）> 磁盘缓存（`~/.claude.json` 的 `cachedGrowthBookFeatures`）> 默认值。
- **非阻塞**：纯同步读取，适用于启动关键路径和 React render loop。
- **staleness 是设计特性**：磁盘缓存由 `syncRemoteEvalToDisk()` 在每次成功的 GB init/refresh 时更新，因此值可能滞后于服务器最新状态，但绝不会阻塞。

#### 3.2.2 OAuth Token 结构

`getClaudeAIOAuthTokens()` 返回 `OAuthTokens | null`（`src/utils/auth.ts`）：

```ts
// src/services/oauth/types.ts（被 auth.ts 消费）
type OAuthTokens = {
  accessToken: string
  refreshToken: string
  expiresAt: number
  scopes: string[]
}
```

- `getClaudeAIOAuthTokens` 内部通过 `getSecureStorage()` 读取 macOS Keychain（`security` CLI）或平台等价物。
- 该函数被 `memoize` 包裹，首次调用同步 spawn `security`（~20-50ms），后续为缓存命中；缓存会在 token refresh（约每小时一次）时被清除。

### 3.3 协议与命令

#### 3.3.1 `security` CLI 调用（macOS）

在 `src/utils/auth.ts` 的调用链中，`getClaudeAIOAuthTokens` 最终会触发：

```bash
security find-generic-password -s <service> -a <account> -w
```

- 这是 `hasVoiceAuth()` 的“昂贵”来源，也是 `useVoiceEnabled.ts` 必须 memoize 的原因。
- 在 CI/Linux/Windows 上可能走不同的 secure storage 后端（如 `keytar` 替代或文件描述符传递），但 `voiceModeEnabled.ts` 本身不感知平台差异。

#### 3.3.2 GrowthBook Remote Eval 协议

- `initializeGrowthBook()` 使用 `@growthbook/growthbook` SDK，`remoteEval: true`。
- `processRemoteEvalPayload()` 处理服务器返回的畸形响应（API 当前用 `"value"` 而非 SDK 期望的 `"defaultValue"`）。
- 成功后将完整 feature map 写入 `remoteEvalFeatureValues`（内存）并同步刷盘到 `~/.claude.json`。

---

## 四、关键代码路径与文件引用

### 4.1 直接调用方（Import 引用）

| 文件 | 引用符号 | 用途 |
|------|----------|------|
| `src/commands/voice/index.ts` | `isVoiceGrowthBookEnabled`, `isVoiceModeEnabled` | 命令注册可用性与可见性 |
| `src/commands/voice/voice.ts` | `isVoiceModeEnabled` | `/voice` 命令执行前校验 |
| `src/hooks/useVoiceEnabled.ts` | `hasVoiceAuth`, `isVoiceGrowthBookEnabled` | React 渲染路径的语音启用状态 |
| `src/components/LogoV2/VoiceModeNotice.tsx` | `isVoiceModeEnabled` | 顶部通知条是否展示“语音模式已可用” |
| `src/tools/ConfigTool/ConfigTool.ts` | `isVoiceGrowthBookEnabled`, `isVoiceModeEnabled`（动态 import） | 配置工具读写 `voiceEnabled` 的门控与预检 |
| `src/tools/ConfigTool/prompt.ts` | `isVoiceGrowthBookEnabled` | 生成模型 prompt 时，kill-switch 开启则隐藏 `voiceEnabled` 配置项 |

### 4.2 直接依赖方（被调用）

| 文件 | 引用符号 | 用途 |
|------|----------|------|
| `bun:bundle` | `feature('VOICE_MODE')` | 编译期死代码消除 |
| `src/services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE` | 运行时 GB kill-switch 读取 |
| `src/utils/auth.ts` | `getClaudeAIOAuthTokens`, `isAnthropicAuthEnabled` | OAuth 身份校验 |

### 4.3 间接关联的关键路径

| 文件 | 关联说明 |
|------|----------|
| `src/services/voiceStreamSTT.ts` | `isVoiceStreamAvailable()` 与 `connectVoiceStream()` 实现；被 `/voice` 命令和 `useVoice.ts` 在通过 `voiceModeEnabled` 门控后调用 |
| `src/services/voice.ts` | 录音实现（native/SoX/arecord）；被 `/voice` 命令在门控后做预检 |
| `src/hooks/useVoice.ts` | 核心语音交互 hook，消费 `useVoiceEnabled()` 的输出决定是否激活录音 |
| `src/context/voice.tsx` | 语音状态上下文（`VoiceState`），由 `useVoice.ts` 更新状态 |
| `src/utils/settings/changeDetector.ts` | `settingsChangeDetector.notifyChange('userSettings')`；在 `ConfigTool` 或 `/voice` 成功修改 `voiceEnabled` 后触发，使 `useVoiceEnabled` 重新读取最新设置 |
| `src/utils/settings/types.ts` | `SettingsSchema` 中通过 `feature('VOICE_MODE')` 条件扩展定义 `voiceEnabled: boolean` |
| `src/tools/ConfigTool/supportedSettings.ts` | `SUPPORTED_SETTINGS` 注册 `voiceEnabled`，同样被 `feature('VOICE_MODE')` 门控 |
| `src/state/AppState.tsx` | 提供 `authVersion` 状态，供 `useVoiceEnabled.ts` 作为 memoization key |

---

## 五、依赖与外部交互

### 5.1 编译期依赖

- **`bun:bundle` 的 `feature()` 宏**：这是构建系统的注入点。`feature('VOICE_MODE')` 在 ant 构建中为 `true`，在外部构建中为 `false`。该宏的返回值决定了：
  - `src/commands.ts` 是否 `require('./commands/voice/index.ts')`；
  - `src/utils/settings/types.ts` 是否在 schema 中暴露 `voiceEnabled`；
  - `src/components/LogoV2/VoiceModeNotice.tsx` 是否渲染内部组件。

### 5.2 运行时外部服务

- **GrowthBook**：
  - 客户端初始化在 `src/services/analytics/growthbook.ts`。
  - `tengu_amber_quartz_disabled` 是远程 feature flag，通过 `api.anthropic.com` 的 remote eval 接口获取。
  - 首次启动时若网络未就绪，依赖磁盘缓存（`~/.claude.json`）。

- **macOS Keychain / Secure Storage**：
  - `getClaudeAIOAuthTokens()` 读取本地持久化的 OAuth token。
  - 在 macOS 上通过 `security` CLI 子进程读取；在 Windows/Linux 上通过 `src/utils/secureStorage/` 下的抽象层实现。

### 5.3 配置与状态交互

- **`~/.claude.json`**：存储 `cachedGrowthBookFeatures`，是 GB 缓存的落地点。
- **`settings.json` / `~/.claude/settings.json`**：存储用户显式设置的 `voiceEnabled`（布尔值）。
- **`AppState.authVersion`**：React 状态树中的整数，在 `/login` 成功时递增，用于触发 `useVoiceEnabled` 中 `hasVoiceAuth` 的重新计算。

---

## 六、风险、边界与改进建议

### 6.1 已知风险

#### R1: `security` CLI 同步阻塞（macOS 渲染路径）

- `hasVoiceAuth()` 在 macOS 上首次调用会同步 spawn `security`（~20-50ms）。
- 虽然 `useVoiceEnabled.ts` 已用 `useMemo(..., [authVersion])` 缓解，但任何**绕过 `useVoiceEnabled` 直接在 render 中调用 `hasVoiceAuth()` 或 `isVoiceModeEnabled()` 的新代码**都会重新引入阻塞风险。
- **缓解**：代码注释已明确区分“命令时间路径用 `isVoiceModeEnabled`” vs “React render 路径用 `useVoiceEnabled`”。

#### R2: Stale GrowthBook 缓存导致 kill-switch 延迟生效

- `getFeatureValue_CACHED_MAY_BE_STALE` 读取的是内存/磁盘缓存，不是实时服务器值。
- 如果 GrowthBook 初始化失败且磁盘缓存为旧值，kill-switch 可能在进程生命周期的前几分钟内不生效。
- **业务接受度**：该 flag 用于“紧急关闭”，延迟窗口在可接受范围内；且命令执行路径（`/voice`、`ConfigTool`）允许同步 keychain 读取，却不阻塞等待 GB 网络，这是性能与实时性的权衡。

#### R3: 外部构建与 ant 构建的行为分裂

- `feature('VOICE_MODE')` 是编译期常量，外部构建中语音代码被 DCE 完全移除。
- 这意味着：
  - 外部用户即使拥有 Claude.ai OAuth 账号，也无法通过任何配置开启语音；
  - 相关测试必须区分构建类型，不能假设语音功能一定存在。
- **边界**：这是产品策略层面的限制，非代码缺陷。

#### R4: `isAnthropicAuthEnabled()` 与 `hasVoiceAuth()` 的语义间隙

- `isAnthropicAuthEnabled()` 在 `ANTHROPIC_UNIX_SOCKET`（`claude ssh` 远程模式）场景下，仅检查 `CLAUDE_CODE_OAUTH_TOKEN` 占位符是否存在，不验证 token 有效性。
- 如果远程端的 OAuth token 实际已过期但环境变量仍在，`hasVoiceAuth()` 会返回 `true`，导致用户能看到语音 UI，但 `voiceStreamSTT.ts` 中的 WebSocket 连接会在握手后失败。
- **严重程度**：中低。最终会在 `connectVoiceStream` 或 `isVoiceStreamAvailable` 处暴露错误。

### 6.2 边界条件

| 边界 | 行为 |
|------|------|
| 无网络 + 无磁盘缓存 | `isVoiceGrowthBookEnabled()` 返回 `true`（默认值 `false` 表示不禁用），语音默认可见 |
| 有网络 + GB flag = `true`（禁用） | 语音命令隐藏、设置项不可见、已有通知不展示 |
| 用户未登录（无 OAuth token） | `hasVoiceAuth()` → `false`，`isVoiceModeEnabled()` → `false`；`/voice` 返回 "Please run /login" |
| 用户使用 API Key（非 OAuth） | `isAnthropicAuthEnabled()` → `false`，语音完全不可用 |
| token 存在但已过期 | `hasVoiceAuth()` 仍返回 `true`（只检查存在性）；实际 STT 连接时由 `voiceStreamSTT.ts` 的 `checkAndRefreshOAuthTokenIfNeeded()` 处理刷新或报错 |
| `/login` 后 authVersion bump | `useVoiceEnabled` 重新计算 `hasVoiceAuth`，UI 从禁用变为可用（无需刷新页面） |

### 6.3 改进建议

#### S1: 统一“可用性”接口，减少调用方误用

当前调用方需要在 `isVoiceGrowthBookEnabled`、`hasVoiceAuth`、`isVoiceModeEnabled` 之间做选择。建议：

- 保留现有三个导出函数（已有大量调用方），但新增一个**带上下文的校验函数**，例如：
  ```ts
  export type VoiceEnablementContext = 'command-registration' | 'command-execution' | 'react-render' | 'config-prompt'
  export function checkVoiceEnabled(context: VoiceEnablementContext): boolean
  ```
- 该函数内部根据 context 自动选择最优检查路径（如 render 路径自动走 memoized auth），降低未来开发者误用风险。

#### S2: 为 `hasVoiceAuth()` 增加 token 过期预检（可选）

- 当前 `hasVoiceAuth()` 仅检查 `accessToken` 字符串存在性。
- 可考虑引入轻量的过期时间预检（`expiresAt`），在 token 已过期时返回 `false`，并触发一次后台刷新。这样可以在 UI 层面更早地提示用户重新登录，而不是等到 WebSocket 连接失败后才暴露。
- **权衡**：刷新 token 是异步网络操作，若引入会打破 `hasVoiceAuth()` 的同步语义，需要仔细设计接口（如拆分为 `hasVoiceAuth()` 和 `ensureVoiceAuth()`）。

#### S3: 增强 kill-switch 的同步阻塞路径（低优先级）

- 对于安全相关的紧急关闭，可考虑在 `initializeGrowthBook()` 的 reinitializing promise 上提供一个“快速阻塞”版本（类似 `checkSecurityRestrictionGate`），在 `/voice` 命令执行时做 1-2 秒的短暂阻塞，确保 kill-switch 在命令触发时是最新的。
- **权衡**：会损害命令响应速度，需产品层面确认是否值得。

#### S4: 测试覆盖

- 当前未搜索到针对 `voiceModeEnabled.ts` 的单元测试（`*.test.*` 中无匹配）。
- 建议补充测试：
  1. `isVoiceGrowthBookEnabled` 在 `feature('VOICE_MODE')=false` 时返回 `false`；
  2. `hasVoiceAuth` 在 `isAnthropicAuthEnabled()=false` 时短路返回 `false`；
  3. `hasVoiceAuth` 在 token 存在/缺失时的行为；
  4. `isVoiceModeEnabled` 的正确组合逻辑（真值表覆盖）。
- 测试时需要 mock `bun:bundle` 的 `feature` 宏、`../services/analytics/growthbook.js` 和 `../utils/auth.js`。

---

## 七、附录：引用关系图（文字版）

```
src/voice/voiceModeEnabled.ts
  ├─ imports ──► bun:bundle (feature)
  ├─ imports ──► src/services/analytics/growthbook.ts (getFeatureValue_CACHED_MAY_BE_STALE)
  └─ imports ──► src/utils/auth.ts (getClaudeAIOAuthTokens, isAnthropicAuthEnabled)

  ▲ called by    src/commands/voice/index.ts        (command registration / visibility)
  ▲ called by    src/commands/voice/voice.ts        (/voice execution guard)
  ▲ called by    src/hooks/useVoiceEnabled.ts       (React render path, memoized auth)
  ▲ called by    src/components/LogoV2/VoiceModeNotice.tsx  (notice display guard)
  ▲ called by    src/tools/ConfigTool/ConfigTool.ts (dynamic import, config guard)
  ▲ called by    src/tools/ConfigTool/prompt.ts     (prompt generation filter)

  (after passing the gate)
     downstream ──► src/services/voice.ts            (recording)
     downstream ──► src/services/voiceStreamSTT.ts   (WebSocket STT)
     downstream ──► src/hooks/useVoice.ts            (React voice interaction)
     downstream ──► src/context/voice.tsx            (voice state context)
```

---

*文档生成时间：2026-04-01*
*研究执行器：kimi (model=k2p5)*
