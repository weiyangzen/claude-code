# 研究文档：src/commands/thinkback-play/thinkback-play.ts

## 场景与职责

该文件是 `thinkback-play` 命令的**实际实现模块**，负责在命令被触发后执行具体的动画播放逻辑。其核心职责包括：

1. **插件安装状态校验**：从全局插件安装元数据中查找 `thinkback` 插件是否已安装。
2. **市场来源解析**：根据当前用户类型（`ant` 内部员工 vs 外部用户）决定从哪个 marketplace 获取插件。
3. **技能目录定位**：在插件安装目录下定位 `skills/thinkback` 子目录。
4. **动画播放委托**：调用 `../thinkback/thinkback.tsx` 中导出的 `playAnimation(skillDir)` 函数，实际执行动画渲染。
5. **错误降级**：在插件未安装或路径缺失时，返回友好的文本错误提示，引导用户先运行 `/think-back`。

## 功能点目的

- **分离播放与生成流程**：`/think-back` 命令负责完整的 JSX 交互式流程（安装、生成、菜单），而 `thinkback-play` 只负责"播放已生成的动画"，使 skill 在生成完成后能通过工具调用直接触发播放，无需重新走一遍完整交互流程。
- **插件化架构适配**：动画数据存储在 thinkback 插件的 skill 目录中，本模块需要正确解析插件安装路径才能找到动画文件。
- **双市场支持**：内部构建使用 `claude-code-marketplace`，外部构建使用 `claude-plugins-official`，通过环境变量 `USER_TYPE` 区分。

## 具体技术实现

### 关键常量与辅助函数

```typescript
const INTERNAL_MARKETPLACE_NAME = 'claude-code-marketplace'
const SKILL_NAME = 'thinkback'

function getPluginId(): string {
  const marketplaceName =
    process.env.USER_TYPE === 'ant'
      ? INTERNAL_MARKETPLACE_NAME
      : OFFICIAL_MARKETPLACE_NAME
  return `thinkback@${marketplaceName}`
}
```

- `getPluginId()` 动态构造插件标识符，格式为 `thinkback@{marketplace}`，与 `installed_plugins.json` 中的键名保持一致。

### 核心执行流程（`call()` 函数）

```typescript
export async function call(): Promise<LocalCommandResult> {
  const v2Data = loadInstalledPluginsV2()
  const pluginId = getPluginId()
  const installations = v2Data.plugins[pluginId]

  if (!installations || installations.length === 0) {
    return {
      type: 'text' as const,
      value: 'Thinkback plugin not installed. Run /think-back first to install it.',
    }
  }

  const firstInstall = installations[0]
  if (!firstInstall?.installPath) {
    return {
      type: 'text' as const,
      value: 'Thinkback plugin installation path not found.',
    }
  }

  const skillDir = join(firstInstall.installPath, 'skills', SKILL_NAME)
  const result = await playAnimation(skillDir)
  return { type: 'text' as const, value: result.message }
}
```

#### 流程说明

1. **加载插件元数据**：调用 `loadInstalledPluginsV2()` 读取 `~/.claude/plugins/installed_plugins.json`（V2 格式）。
2. **查找插件记录**：用 `getPluginId()` 生成的键从 `v2Data.plugins` 映射中取值。
3. **空记录检查**：若未安装，返回文本错误，提示用户运行 `/think-back`。
4. **安装路径检查**：取第一个安装条目（`installations[0]`），验证 `installPath` 存在。
5. **构造 skill 目录**：`join(installPath, 'skills', 'thinkback')`。
6. **调用播放函数**：`await playAnimation(skillDir)`，将播放结果的消息文本包装为 `LocalCommandResult` 返回。

### 数据结构

- **`InstalledPluginsFileV2`**（来自 `installedPluginsManager.ts`）：
  ```typescript
  {
    version: 2,
    plugins: {
      [pluginId: string]: PluginInstallationEntry[]
    }
  }
  ```
- **`PluginInstallationEntry`**：包含 `scope`, `installPath`, `version`, `installedAt`, `lastUpdated`, `gitCommitSha`, `projectPath` 等字段。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/commands/thinkback-play/thinkback-play.ts` | 本文件：动画播放命令的实现 |
| `src/commands/thinkback-play/index.ts` | 命令定义入口，负责懒加载本模块 |
| `src/commands/thinkback/thinkback.tsx` | 提供 `playAnimation(skillDir)`，实际执行 alternate screen 切换、子进程播放、浏览器打开 HTML |
| `src/utils/plugins/installedPluginsManager.ts` | 提供 `loadInstalledPluginsV2()` 和插件安装元数据类型 |
| `src/utils/plugins/officialMarketplace.ts` | 提供 `OFFICIAL_MARKETPLACE_NAME` 常量 |
| `src/types/command.ts` | `LocalCommandResult` 类型定义 |

## 依赖与外部交互

### 直接依赖

- **`path.join`**：Node.js 内置模块，用于跨平台路径拼接。
- **`../../commands.js`**（`src/commands.ts` / `src/types/command.ts`）：`LocalCommandResult` 类型。
- **`../../utils/plugins/installedPluginsManager.js`**：`loadInstalledPluginsV2()`。
- **`../../utils/plugins/officialMarketplace.js`**：`OFFICIAL_MARKETPLACE_NAME`。
- **`../thinkback/thinkback.js`**（实际为 `.tsx`）：`playAnimation()`。

### 外部交互

- **文件系统**：读取 `~/.claude/plugins/installed_plugins.json`。
- **子进程**：通过 `playAnimation()` 间接调用 `execa('node', [playerPath], { stdio: 'inherit' })` 播放动画。
- **终端状态**：通过 `playAnimation()` 间接操作 Ink 实例的 `enterAlternateScreen()` / `exitAlternateScreen()`。
- **浏览器**：动画播放完成后，`playAnimation()` 会尝试用系统默认浏览器打开 `year_in_review.html`。

## 风险、边界与改进建议

### 风险与边界

1. **多安装条目歧义**：代码直接取 `installations[0]`，未考虑同一插件存在多个 scope 安装（如 `user` + `project`）的情况。如果用户在不同 scope 安装了不同版本的 thinkback，可能播放的不是期望的版本。
2. **路径硬编码假设**：假设插件目录结构固定为 `{installPath}/skills/thinkback`。如果插件未来重构了内部目录结构，此代码会失效。
3. **无动画数据预检**：本模块本身不检查 `year_in_review.js` 或 `year_in_review.html` 是否存在，这些检查被下放到 `playAnimation()` 中。虽然最终能返回正确错误，但调用链较长，调试时不够直观。
4. **`playAnimation` 的副作用**：`playAnimation` 会接管整个终端（alternate screen），如果在某些终端模拟器或远程会话中调用，可能出现渲染异常或无法退出 alternate screen 的情况。
5. **浏览器打开是"fire and forget"**：`execFileNoThrow(openCmd, [htmlPath])` 被 `void` 忽略，若用户环境没有 `open`/`xdg-open`/`start` 命令，失败完全静默。
6. **非交互式模式不支持**：`index.ts` 中声明了 `supportsNonInteractive: false`，但本模块的 `call()` 签名并不接收上下文来判断当前是否处于非交互模式，依赖上层过滤。

### 改进建议

1. **优先选择当前项目相关的安装**：应使用 `isInstallationRelevantToCurrentProject()`（来自 `installedPluginsManager.ts`）过滤安装条目，而不是简单取 `[0]`，以确保在多 scope 场景下行为正确。
2. **增加目录结构版本校验**：可在插件 manifest 中声明 skill 目录相对路径，本模块从 manifest 读取而非硬编码 `skills/thinkback`。
3. **前置文件存在性检查**：在调用 `playAnimation()` 前，可提前检查 `year_in_review.js` 是否存在，若不存在则直接返回 "Run /think-back first to generate your animation"，避免进入子进程后才报错。
4. **浏览器打开结果反馈**：可考虑将 `execFileNoThrow` 的返回结果（或至少错误日志）纳入 `LocalCommandResult`，让用户知道 HTML 是否成功打开。
5. **增加测试覆盖**：当前未找到针对 `thinkback-play` 的单元测试。建议增加测试覆盖以下场景：
   - 插件未安装时的错误返回
   - 多 scope 安装时的路径选择
   - `playAnimation` 成功/失败时的结果包装
6. **考虑提取公共的 marketplace 解析逻辑**：`getPluginId()` 和 `getMarketplaceName()` 的逻辑在 `thinkback.tsx` 中几乎重复，可提取到共享模块以减少维护成本。
