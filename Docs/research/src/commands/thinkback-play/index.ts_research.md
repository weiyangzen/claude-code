# 研究文档：src/commands/thinkback-play/index.ts

## 场景与职责

该文件是 `thinkback-play` 命令的**入口定义文件**，负责将 `thinkback-play` 注册到 Claude Code 的全局命令体系中。它是一个 `local` 类型的内置命令，主要职责包括：

1. **命令注册**：导出一个符合 `Command` 接口的对象，使 `src/commands.ts` 能够将其纳入命令列表。
2. **特性开关控制**：通过 GrowthBook feature gate `tengu_thinkback` 动态控制该命令是否对用户可见/可用。
3. **懒加载委托**：通过 `load` 回调延迟加载实际实现模块 `./thinkback-play.js`，避免启动时加载不必要的依赖。
4. **隐藏命令声明**：标记 `isHidden: true`，使其不出现在帮助文档或自动补全中，仅作为内部机制被调用。

从注释可知，该命令是 "Hidden command that just plays the animation"，且 "Called by the thinkback skill after generation is complete"，说明它是 thinkback 插件生成动画后的**播放触发入口**。

## 功能点目的

- **动画播放触发**：在 thinkback skill 完成年度回顾动画生成后，提供一个独立的命令来播放该动画。
- **与主命令解耦**：`/think-back`（`thinkback` 命令）是一个 `local-jsx` 命令，负责安装插件、生成动画、展示交互菜单；而 `thinkback-play` 是一个轻量级的 `local` 命令，专门负责"纯播放"逻辑，无需渲染 JSX UI。
- **Feature Gate 保护**：通过 `checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_thinkback')` 确保只有在 thinkback 功能对用户开放时，该命令才可用。

## 具体技术实现

### 关键数据结构

```typescript
const thinkbackPlay = {
  type: 'local',
  name: 'thinkback-play',
  description: 'Play the thinkback animation',
  isEnabled: () =>
    checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_thinkback'),
  isHidden: true,
  supportsNonInteractive: false,
  load: () => import('./thinkback-play.js'),
} satisfies Command
```

- `type: 'local'`：表示这是一个本地执行的命令，返回 `LocalCommandResult`，不渲染 React 组件。
- `isHidden: true`：隐藏命令，用户无法通过 `/thinkback-play` 主动发现（但可以通过代码或 skill 调用）。
- `supportsNonInteractive: false`：不支持非交互式会话（如 CI/headless 模式）。
- `satisfies Command`：TypeScript 类型约束，确保对象结构符合 `Command` 联合类型。

### 命令加载流程

1. `src/commands.ts` 第 125 行 `import thinkbackPlay from './commands/thinkback-play/index.js'`
2. 第 330 行将其加入 `COMMANDS` 数组
3. 当用户或模型触发 `/thinkback-play` 时，`commands.ts` 中的命令分发器调用 `thinkbackPlay.load()`
4. `load()` 动态导入 `./thinkback-play.js`，返回包含 `call` 函数的模块
5. 执行 `call()` 完成动画播放

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/commands/thinkback-play/index.ts` | 本文件：命令定义与注册入口 |
| `src/commands/thinkback-play/thinkback-play.ts` | 实际实现：查找插件、调用播放 |
| `src/commands.ts:125` | 导入并注册 `thinkbackPlay` |
| `src/commands.ts:330` | 将 `thinkbackPlay` 加入内置命令列表 |
| `src/types/command.ts` | `Command`、`LocalCommandResult` 类型定义 |
| `src/services/analytics/growthbook.ts` | `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` 实现 |

## 依赖与外部交互

### 直接依赖

- **`../../commands.js`**（实际为 `src/commands.ts` / `src/types/command.ts`）：引入 `Command` 类型。
- **`../../services/analytics/growthbook.js`**（实际为 `.ts`）：引入 `checkStatsigFeatureGate_CACHED_MAY_BE_STALE`，用于非阻塞地读取 feature gate 缓存值。

### 间接依赖（通过懒加载）

- `thinkback-play.ts` 依赖插件系统（`installedPluginsManager.ts`、`officialMarketplace.ts`）和 `thinkback.tsx` 中的 `playAnimation`。

### 外部交互

- **GrowthBook**：通过 `tengu_thinkback` gate 控制命令可用性。
- **命令分发器**：`src/commands.ts` 中的 `getCommands()` / `findCommand()` 负责解析和调度。

## 风险、边界与改进建议

### 风险与边界

1. **Feature Gate 缓存可能过期**：`isEnabled` 使用的是 `_CACHED_MAY_BE_STALE` 后缀的函数，读取的是本地磁盘缓存。如果用户刚刚被加入/移出实验组，缓存可能有几分钟到几小时的延迟，导致命令可用性与服务端状态不一致。
2. **隐藏命令仍可被手动调用**：虽然 `isHidden` 使其不出现在补全中，但知道命令名的用户仍可直接输入 `/thinkback-play`。由于它不支持非交互式模式，在 headless 环境中调用会失败或被过滤。
3. **懒加载失败无本地处理**：`load()` 只是简单的 `import()`，如果 `./thinkback-play.js` 模块加载失败（如文件损坏），异常会向上抛给命令分发器。
4. **无参数校验**：该命令不接受任何参数，但 `local` 命令的 `call` 签名会收到 `args` 字符串，实现中未对多余参数做处理或提示。

### 改进建议

1. **考虑统一 gate 检查**：如果 thinkback 功能需要更精确的实时控制，可在 `thinkback-play.ts` 的 `call()` 内部增加二次 gate 检查，而非仅依赖注册时的 `isEnabled`。
2. **增加非交互式支持评估**：动画播放依赖终端 alternate screen 和 `execa` 子进程，确实不适合纯 headless 环境。如果未来需要支持，可考虑在 non-interactive 模式下返回静态文本或 HTML 路径。
3. **错误信息本地化**：当前错误消息是硬编码英文，若产品有多语言计划，可抽离到 i18n 资源中。
4. **文档化调用契约**：注释中提到 "Called by the thinkback skill after generation is complete"，建议将这一调用契约写入更正式的 skill 协议文档或类型约束中，防止其他 skill 误用。
