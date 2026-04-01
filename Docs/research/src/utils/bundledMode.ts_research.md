# src/utils/bundledMode.ts 深入研究

## 场景与职责

`bundledMode.ts` 负责**运行时环境探测**，判断当前 Claude Code 是：
- 通过 `bun` 命令直接运行的 JS 文件
- Bun 编译后的独立可执行文件（standalone executable）

这一区分对包体积优化、资源加载路径（如 embedded files）、更新策略、以及特定平台行为（如 ripgrep 内置/嵌入选择）至关重要。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `isRunningWithBun()` | 检测当前 JavaScript 运行时是否为 Bun |
| `isInBundledMode()` | 检测当前是否运行在 Bun 编译后的独立可执行文件中 |

## 具体技术实现

### Bun 运行时检测
```ts
export function isRunningWithBun(): boolean {
  return process.versions.bun !== undefined
}
```
- 利用 Node.js/Bun 兼容的 `process.versions` 对象。Bun 会在 `process.versions` 上设置 `bun` 字段（如 `"1.1.x"`），而 Node.js 不存在该字段。
- 参考 Bun 官方文档的推荐做法：https://bun.com/guides/util/detect-bun

### 编译模式（Standalone）检测
```ts
export function isInBundledMode(): boolean {
  return (
    typeof Bun !== 'undefined' &&
    Array.isArray(Bun.embeddedFiles) &&
    Bun.embeddedFiles.length > 0
  )
}
```
- `Bun` 全局对象仅在 Bun 运行时存在。
- `Bun.embeddedFiles` 是 Bun 编译二进制时用于打包静态资源的 API；当编译为独立可执行文件时，该数组包含嵌入的文件列表。
- 直接运行 JS 文件时，`Bun.embeddedFiles` 通常为空数组或不存在。

## 关键代码路径与文件引用

```
src/main.tsx
  └── isInBundledMode, isRunningWithBun
      [启动时根据模式选择不同的初始化路径或资源加载策略]

src/utils/env.ts
  └── isRunningWithBun
      [环境判断，影响 npm/bun 包管理器选择]

src/utils/doctorDiagnostic.ts
  └── isInBundledMode
      [doctor 命令展示安装类型：native / package-manager / npm-global 等]

src/utils/ripgrep.ts
  └── isInBundledMode
      [决定使用系统 ripgrep、内置二进制还是 embedded 资源]

src/utils/fastMode.ts
  └── isInBundledMode
      [fast mode 相关路径判断]

src/utils/computerUse/setup.ts
  └── isInBundledMode
      [计算机使用功能的资源路径选择]

src/utils/swarm/spawnUtils.ts
  └── isInBundledMode
      [swarm 子进程启动时的路径处理]

src/utils/claudeInChrome/setup.ts
  └── isInBundledMode
      [Chrome 扩展相关资源加载]

src/bridge/bridgeMain.ts
  └── isInBundledMode
      [桥接进程的资源路径]

src/tools/FileReadTool/imageProcessor.ts
  └── isInBundledMode
      [图像处理依赖的加载方式]

src/tools/shared/spawnMultiAgent.ts
  └── isInBundledMode
      [多 agent 启动路径]

src/hooks/notifs/useNpmDeprecationNotification.tsx
  └── isInBundledMode
      [仅在非 bundled 模式时显示 npm 弃用通知]

src/keybindings/defaultBindings.ts
  └── isRunningWithBun
      [键绑定加载方式]
```

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| Bun 运行时 | `process.versions.bun` / `Bun.embeddedFiles` | 直接读取 Bun 提供的全局 API |
| 调用方模块 | 同步布尔返回值 | 影响文件路径、资源加载、更新策略、诊断输出 |

## 风险、边界与改进建议

### 风险
1. **`Bun.embeddedFiles` 非官方稳定 API**：虽然 Bun 文档有提及，但 `embeddedFiles` 的语义未来可能变化（如开发模式下也可能非空），导致 `isInBundledMode` 误判。
2. **Node.js 兼容性**：若未来迁移到 Node.js 原生编译器（如 `sea` / `pkg`），`isRunningWithBun` 与 `isInBundledMode` 均会失效，需要大规模重构。
3. **无缓存**：两个函数均为纯计算，虽然开销极低，但在热路径（如 `ripgrep.ts` 的每次调用）中可能被反复执行。

### 边界
- `isRunningWithBun` 在 Node.js 环境下返回 `false`，在 Deno 环境下行为未定义（`process.versions` 可能不存在 `bun`，大概率也返回 `false`）。
- `isInBundledMode` 在直接 `bun run` 时返回 `false`；在 `bun build --compile` 产物中返回 `true`。
- 两个函数之间无包含关系：可以直接 `isRunningWithBun() === false` 但 `isInBundledMode() === true` 吗？理论上不可能（因为 `isInBundledMode` 先检查 `typeof Bun !== 'undefined'`），但逻辑上两者是独立概念。

### 改进建议
1. **增加缓存**：将结果缓存到模块级常量，避免热路径重复计算：
   ```ts
   const RUNNING_WITH_BUN = process.versions.bun !== undefined
   const IN_BUNDLED_MODE = /* ... */
   ```
2. **抽象运行时接口**：若未来可能支持 Node.js SEA，可引入 `getRuntimeMode(): 'node' | 'bun' | 'bun-bundled'` 统一封装，减少调用方对 Bun 专有 API 的依赖。
3. **防御性检查**：对 `Bun.embeddedFiles` 增加更严格的校验（如检查是否包含预期的特定嵌入文件），降低 API 语义变化带来的误判风险。
4. **文档化编译流程依赖**：在 `AGENTS.md` 或构建脚本中明确记录 `isInBundledMode` 的判定逻辑与 `bun build --compile` 的关联，防止新成员修改构建流程时意外破坏模式检测。
