# index.ts 研究文档

## 场景与职责

`index.ts` 是 `src/skills/bundled/` 目录的**统一初始化入口**，负责在 CLI 启动时按固定顺序注册所有内置技能（bundled skills）。它通过条件编译与运行时特性开关（`bun:bundle` 的 `feature()`、环境变量、外部配置）决定哪些技能实际可用，起到技能注册调度中心的作用。

## 功能点目的

1. **集中注册所有内置技能**：将散落在各文件中的 `registerXxxSkill()` 函数统一调用。
2. **特性门控（Feature Gating）**：基于构建时和运行时标志，有条件地注册实验性/受限技能（如 `dream`、`hunter`、`loop`、`schedule`、`claude-api`、`runSkillGenerator`）。
3. **外部依赖条件注册**：如 `claude-in-chrome` 技能仅在检测到 Chrome 扩展已安装且满足其他启用条件时才注册。
4. **提供扩展指南**：文件顶部注释明确说明新增内置技能的三个步骤，降低贡献门槛。

## 具体技术实现

### 关键流程

```ts
export function initBundledSkills(): void {
  registerUpdateConfigSkill()
  registerKeybindingsSkill()
  registerVerifySkill()
  registerDebugSkill()
  registerLoremIpsumSkill()
  registerSkillifySkill()
  registerRememberSkill()
  registerSimplifySkill()
  registerBatchSkill()
  registerStuckSkill()

  if (feature('KAIROS') || feature('KAIROS_DREAM')) {
    const { registerDreamSkill } = require('./dream.js')
    registerDreamSkill()
  }
  if (feature('REVIEW_ARTIFACT')) {
    const { registerHunterSkill } = require('./hunter.js')
    registerHunterSkill()
  }
  if (feature('AGENT_TRIGGERS')) {
    const { registerLoopSkill } = require('./loop.js')
    registerLoopSkill()
  }
  if (feature('AGENT_TRIGGERS_REMOTE')) {
    const { registerScheduleRemoteAgentsSkill } = require('./scheduleRemoteAgents.js')
    registerScheduleRemoteAgentsSkill()
  }
  if (feature('BUILDING_CLAUDE_APPS')) {
    const { registerClaudeApiSkill } = require('./claudeApi.js')
    registerClaudeApiSkill()
  }
  if (shouldAutoEnableClaudeInChrome()) {
    registerClaudeInChromeSkill()
  }
  if (feature('RUN_SKILL_GENERATOR')) {
    const { registerRunSkillGeneratorSkill } = require('./runSkillGenerator.js')
    registerRunSkillGeneratorSkill()
  }
}
```

### 注册顺序

1. `update-config`（配置更新）
2. `keybindings-help`（快捷键帮助）
3. `verify`（验证）
4. `debug`（调试）
5. `lorem-ipsum`（占位文本生成）
6. `skillify`（会话转技能）
7. `remember`（记忆整理）
8. `simplify`（代码简化审查）
9. `batch`（批量并行任务）
10. `stuck`（诊断卡死会话）
11. 条件技能：`dream`、`hunter`、`loop`、`schedule`、`claude-api`、`claude-in-chrome`、`run-skill-generator`

### 动态 require 模式

对于所有条件技能，使用 `require()` 而非顶层 `import`，以便在构建时通过 `feature()` 进行死代码消除（dead code elimination），减少二进制体积。

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/index.ts`
- 被调用方：`src/skills/bundledSkills.ts`（`registerBundledSkill`）
- 启动调用链：由 CLI 启动流程（如 `src/init.ts` 或类似入口）在初始化阶段调用 `initBundledSkills()`
- 特性标志来源：`bun:bundle` 的 `feature()` 函数
- Chrome 启用判断：`src/utils/claudeInChrome/setup.ts`（`shouldAutoEnableClaudeInChrome`）

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `feature('KAIROS')` 等 | Bun 构建时特性标志，控制条件编译 |
| `shouldAutoEnableClaudeInChrome()` | 运行时判断 Chrome 扩展是否就绪 |
| 各 `registerXxxSkill` | 实际执行技能注册逻辑 |

- **无网络调用**：纯本地注册逻辑。
- **启动时执行**：`initBundledSkills` 在 CLI 启动时同步执行，任何注册异常会直接影响启动流程。

## 风险、边界与改进建议

1. **边界：条件技能的懒加载与类型安全**：使用 `require()` 虽然实现了死代码消除，但丢失了 TypeScript 的静态类型检查（返回值类型为 `any`）。若被 require 的文件导出名称拼写错误，只能在运行时暴露。
2. **风险：注册顺序隐式语义**：`keybindings-help` 在 `update-config` 之后注册，但某些下游逻辑可能假设特定顺序。当前顺序无显式文档说明其重要性。
3. **风险：`shouldAutoEnableClaudeInChrome` 的缓存值**：该函数内部有模块级缓存 `shouldAutoEnable`，在启动时计算一次。若用户在会话期间安装/卸载 Chrome 扩展，缓存不会刷新，技能可见性不会动态变化。
4. **改进建议**：
   - 为条件技能的 `require()` 引入类型断言或生成一个类型安全的条件加载辅助函数，减少运行时拼写错误风险。
   - 考虑将注册顺序的重要性以注释形式文档化（例如为什么 `update-config` 必须在最前面）。
   - 为 `claude-in-chrome` 增加运行时重新检测机制（如通过文件 watcher 或定时轮询），而非一次性缓存。
