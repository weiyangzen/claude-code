# VoiceModeNotice.tsx 深度研究文档

> 文件路径：`src/components/LogoV2/VoiceModeNotice.tsx`  
> 文件大小：7,668 bytes（编译后产物，含 source map）  
> 研究范围：源码、直接依赖、调用方、配置持久化、相关通知组件模式

---

## 一、场景与职责

`VoiceModeNotice` 是 Claude Code 欢迎页（`LogoV2`）顶部通知栏的一员，负责在**语音功能可用但用户尚未开启**时，展示一条轻量提示：

> "Voice mode is now available · /voice to enable"

它的核心职责是：
1. **功能发现引导**：让已具备语音使用条件（Anthropic OAuth + GrowthBook 未 kill）但还没在设置里打开 `voiceEnabled` 的用户，知道 `/voice` 命令的存在。
2. **防骚扰控制**：通过全局配置计数器 `voiceNoticeSeenCount` 限制最多展示 3 次。
3. **优先级避让**：当 `Opus1mMergeNotice` 也需要展示时，本通知主动退让（避免通知堆叠）。

---

## 二、功能点目的

| 功能点 | 目的 |
|--------|------|
| `feature("VOICE_MODE")` 外层守卫 | 遵循项目的 positive-ternary 死代码消除（DCE）规范，确保外部构建不会把提示字符串打包进产物。 |
| `MAX_SHOW_COUNT = 3` | 控制展示频次，防止每次启动都刷存在感。 |
| `useState(_temp)` 一次性求值 | 在 mount 时把“是否展示”定死，避免用户进入滚动历史后，通知因状态变化而消失/闪烁。 |
| `useEffect` 写回 `saveGlobalConfig` | 只有真正渲染了才递增计数，实现“看过才算数”。 |
| `AnimatedAsterisk` 前缀 | 用彩色旋转星号吸引注意力，与 `Opus1mMergeNotice` 保持统一的视觉语言。 |

---

## 三、具体技术实现

### 3.1 组件分层结构

```
VoiceModeNotice (exported)
└── feature("VOICE_MODE") ? <VoiceModeNoticeInner /> : null
    └── show ? <Box paddingLeft={2}><AnimatedAsterisk /><Text dimColor>...<Text></Box> : null
```

外层 `VoiceModeNotice` 只做 feature gate 分支；内层 `VoiceModeNoticeInner` 处理所有业务逻辑。这种拆分是为了让 `feature(...)` 的字符串常量被 DCE 完全剔除（见 `docs/feature-gating.md`）。

### 3.2 展示资格判定（mount 时一次性捕获）

```tsx
function _temp() {
  return (
    isVoiceModeEnabled() &&
    getInitialSettings().voiceEnabled !== true &&
    (getGlobalConfig().voiceNoticeSeenCount ?? 0) < MAX_SHOW_COUNT &&
    !shouldShowOpus1mMergeNotice()
  )
}
```

四个条件必须**同时满足**：
1. `isVoiceModeEnabled()` — 用户有 Anthropic OAuth token 且 GrowthBook 未关闭语音。
2. `getInitialSettings().voiceEnabled !== true` — 用户本地设置里没开语音。
3. 全局计数器 `< 3`。
4. `!shouldShowOpus1mMergeNotice()` — Opus 1M 合并通知不展示时，才让位给语音通知。

> 注意：这里用的是 `getInitialSettings()` 而非 `useSettings()`， intentionally 避免订阅设置变化导致重渲染。

### 3.3 计数器持久化流程

```tsx
useEffect(() => {
  if (!show) return
  const newCount = (getGlobalConfig().voiceNoticeSeenCount ?? 0) + 1
  saveGlobalConfig(prev => {
    if ((prev.voiceNoticeSeenCount ?? 0) >= newCount) return prev
    return { ...prev, voiceNoticeSeenCount: newCount }
  })
}, [show])
```

- 采用函数式更新，防止并发读写覆盖。
- 内部再做一次 `>= newCount` 的短路检查，避免 React StrictMode 双调或异常重渲染导致多计数。

### 3.4 关键常量与数据结构

| 名称 | 值/类型 | 说明 |
|------|---------|------|
| `MAX_SHOW_COUNT` | `3` | 硬编码展示上限 |
| `voiceNoticeSeenCount` | `number \| undefined` | 存储在 `GlobalConfig` 中的计数器 |
| `voiceEnabled` | `boolean \| undefined` | `SettingsJson` 中的用户偏好 |

---

## 四、关键代码路径与文件引用

### 4.1 本文件内部路径

| 行号区间 | 内容 |
|----------|------|
| `1-10` | 导入：React、Ink、`config`、`settings`、`voiceModeEnabled`、`AnimatedAsterisk`、`Opus1mMergeNotice` |
| `11` | `MAX_SHOW_COUNT` 常量 |
| `12-22` | 导出组件 `VoiceModeNotice`，做 feature gate |
| `23-64` | `VoiceModeNoticeInner`，含 `useState` + `useEffect` + 条件渲染 |
| `65-67` | `_temp` 辅助函数（展示资格判定） |

### 4.2 上游调用方

- **`src/components/LogoV2/LogoV2.tsx:26, 189, 299, 460`**  
  `LogoV2` 在多个布局分支（horizontal/compact）中直接渲染 `<VoiceModeNotice />`。

### 4.3 下游依赖

- **`src/voice/voiceModeEnabled.ts`**  
  提供 `isVoiceModeEnabled()`，内部串联 `hasVoiceAuth()`（OAuth token 校验）和 `isVoiceGrowthBookEnabled()`（GrowthBook kill-switch）。
- **`src/utils/config.ts`**  
  `getGlobalConfig()` / `saveGlobalConfig()` 读写磁盘 JSON 配置，字段 `voiceNoticeSeenCount` 定义在 `GlobalConfig` 接口中（约第 338 行）。
- **`src/utils/settings/settings.ts`**  
  `getInitialSettings()` 返回启动时解析的 settings 快照。
- **`src/components/LogoV2/AnimatedAsterisk.tsx`**  
  提供带 1.5s × 2  sweep 动画的星号组件；支持 `prefersReducedMotion`。
- **`src/components/LogoV2/Opus1mMergeNotice.tsx`**  
  提供 `shouldShowOpus1mMergeNotice()`，两者互斥展示。

---

## 五、依赖与外部交互

### 5.1 运行时依赖

| 模块 | 用途 |
|------|------|
| `bun:bundle` (`feature`) | 编译期/运行期特性开关 |
| `react` | Hooks (`useState`, `useEffect`) |
| `src/ink.js` (`Box`, `Text`) | 终端 UI 渲染 |
| `src/utils/config.js` | 全局配置持久化 |
| `src/utils/settings/settings.js` | 读取初始设置 |
| `src/voice/voiceModeEnabled.js` | 语音功能可用性判定 |
| `./AnimatedAsterisk.js` | 动画前缀图标 |
| `./Opus1mMergeNotice.js` | 互斥通知优先级判定 |

### 5.2 配置/状态交互图

```
VoiceModeNotice.tsx
    ├─读取─► GlobalConfig.voiceNoticeSeenCount
    ├─写入─► GlobalConfig.voiceNoticeSeenCount  (+1)
    ├─读取─► InitialSettings.voiceEnabled
    ├─调用─► isVoiceModeEnabled()
    │           ├─► hasVoiceAuth() ──► Keychain / OAuth tokens
    │           └─► isVoiceGrowthBookEnabled() ──► GrowthBook flag
    └─调用─► shouldShowOpus1mMergeNotice()
                └─► GlobalConfig.opus1mMergeNoticeSeenCount
```

---

## 六、风险、边界与改进建议

### 6.1 已知风险

1. **Mount 时快照导致状态漂移**  
   `show` 在 mount 时一次性求值。如果用户在本次会话期间登录了 Anthropic 账号或修改了设置，通知不会动态出现/消失。这是设计上的 trade-off（避免滚动历史区重渲染），但意味着通知对实时状态变化不敏感。

2. **与 Opus 通知的硬编码互斥**  
   `!shouldShowOpus1mMergeNotice()` 把两个通知的优先级写死。如果未来增加第三种通知，需要重写互斥逻辑，否则会出现通知堆叠或覆盖。

3. **计数器 race condition（低概率）**  
   `saveGlobalConfig` 虽然用了函数式更新，但 `getGlobalConfig()` 在 effect 开头读取的 `newCount` 与函数式更新里的 `prev` 可能跨越多次磁盘写。若用户极快速重启多个 Claude 进程，理论上可能丢计数。

4. **编译后产物体积**  
   文件为 React Compiler 编译产物，包含大量 `_c(...)` memo cache 代码和 base64 source map。源码本身很小，但编译产物使文件达到 7.6KB。

### 6.2 边界情况

- **无 OAuth token**：`isVoiceModeEnabled()` 返回 `false`，通知不展示。
- **已开启 `voiceEnabled`**：即使语音可用，也不会打扰已启用用户。
- **计数已达 3**：永久不再展示（除非手动改配置）。
- **Apple Terminal / 窄终端**：通知本身只是单行 `Box`，不受终端宽度影响；但父组件 `LogoV2` 在 compact 模式下可能把它和 Clawd 上下堆叠。

### 6.3 改进建议

| 优先级 | 建议 | 理由 |
|--------|------|------|
| 中 | 把 `MAX_SHOW_COUNT` 抽成 GrowthBook 远程配置 | 便于 A/B 测试展示频次对 `/voice` 命令使用率的影响。 |
| 低 | 统一通知优先级调度器 | 将 `VoiceModeNotice`、`Opus1mMergeNotice`、`ChannelsNotice` 的互斥逻辑收敛到一个 hook 或工具函数，避免硬编码 `!shouldShowXxx()`。 |
| 低 | 考虑在 `saveGlobalConfig` 回调里直接计算 `newCount` | 进一步消除 `getGlobalConfig()` 与函数式更新之间的时隙，彻底避免 race。 |
| 低 | 源码与编译产物分离 | 当前仓库直接提交编译后 `.tsx`（含 source map），不利于 diff review；可考虑构建流水线在 CI 中生成。 |

---

*文档生成时间：2026-04-01*  
*执行器：kimi (k2p5)*
