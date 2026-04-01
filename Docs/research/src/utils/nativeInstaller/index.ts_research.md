# 研究文档：src/utils/nativeInstaller/index.ts

## 场景与职责

`index.ts` 是 `src/utils/nativeInstaller/` 目录的 **barrel 文件（公共 API 入口）**。它的职责非常明确：控制该模块对外暴露的接口面，确保外部调用者只依赖经过筛选的公共函数和类型，避免直接引用内部实现文件（如 `download.ts`、`pidLock.ts`）。

这是项目中典型的"显式公共 API"模式——即便目录内有更多导出，barrel 文件只 re-export 实际被外部使用的子集。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| Re-export `installLatest` | 供自动更新器（`NativeAutoUpdater.tsx`）和 CLI `update.ts` 调用，执行原生安装/更新 |
| Re-export `checkInstall` | 供 `setup.ts`、`useInstallMessages.tsx` 等调用，检查原生安装状态并返回用户提示消息 |
| Re-export `cleanupOldVersions` | 供 `backgroundHousekeeping.ts`、安装成功后调用，清理过期版本二进制 |
| Re-export `cleanupNpmInstallations` | 供 `commands/install.tsx` 调用，在迁移到原生安装时清理旧 npm 安装 |
| Re-export `cleanupShellAliases` | 供安装流程调用，清理 shell 配置中遗留的 claude alias |
| Re-export `lockCurrentVersion` | 供启动流程调用，锁定当前运行版本防止被清理 |
| Re-export `removeInstalledSymlink` | 供 `update.ts`、安装切换逻辑调用，移除 `~/.local/bin/claude` 符号链接 |
| Re-export `SetupMessage` 类型 | 供 UI 层消费安装检查返回的消息结构 |

## 具体技术实现

文件本身无独立逻辑，仅使用 TypeScript 的命名重新导出语法：

```ts
export {
  checkInstall,
  cleanupNpmInstallations,
  cleanupOldVersions,
  cleanupShellAliases,
  installLatest,
  lockCurrentVersion,
  removeInstalledSymlink,
  type SetupMessage,
} from './installer.js'
```

注意：
- 所有导出均来自 `./installer.js`，说明 `installer.ts` 是该目录的**唯一实现核心**。
- `download.ts` 和 `pidLock.ts` 中的函数**未被 barrel 导出**，属于内部实现细节。外部如需下载或锁逻辑，应通过 `installLatest` 等高层 API 间接使用。

## 关键代码路径与文件引用

| 路径 | 角色 |
|------|------|
| `src/utils/nativeInstaller/index.ts` | 本文件，公共 API 门面 |
| `src/utils/nativeInstaller/installer.ts` | 实际实现源，提供所有被导出的函数和类型 |
| `src/components/NativeAutoUpdater.tsx` | 调用 `installLatest` |
| `src/cli/update.ts` | 调用 `installLatestNative`（从 index.ts import 并重命名）和 `removeInstalledSymlink` |
| `src/utils/backgroundHousekeeping.ts` | 调用 `cleanupOldVersions` |
| `src/commands/install.tsx` | 调用 `cleanupNpmInstallations`、`cleanupShellAliases`、`checkInstall` |
| `src/setup.ts` | 调用 `checkInstall` |
| `src/hooks/notifs/useInstallMessages.tsx` | 调用 `checkInstall` |

## 依赖与外部交互

- **无运行时依赖**：该文件不 import 任何第三方包或 Node.js 内置模块。
- **编译期依赖**：TypeScript 编译器需要能解析 `./installer.js` 的对应 `.ts` 源文件（项目使用 `.js` 扩展名导入约定）。

## 风险、边界与改进建议

### 风险

1. **barrel 文件与 tree-shaking 的权衡**：虽然该文件很小，但如果未来引入大量类型重新导出，某些打包器可能无法完美消除未使用的导出，导致少量体积冗余。
2. **公共 API 与内部实现耦合**：所有导出都来自单一 `installer.ts`。随着 `installer.ts` 膨胀（目前已超过 1700 行），barrel 文件没有起到"按功能拆分模块"的隔离作用，只是起到了"命名空间收敛"的作用。

### 边界

- **不导出 `download.ts` 和 `pidLock.ts`**：这是设计上的刻意选择，但这也意味着外部模块无法在不破坏封装的情况下使用底层功能（例如自定义下载逻辑）。
- **无独立测试价值**：该文件逻辑为空，通常不需要单独测试；其正确性由 `installer.ts` 的测试覆盖。

### 改进建议

1. **考虑按功能拆分 `installer.ts`**：当前 `installer.ts` 已非常庞大（安装、下载、清理、锁、shell 别名、npm 卸载等）。可以将其拆分为：
   - `install.ts`（核心安装/更新）
   - `cleanup.ts`（版本清理、npm 清理、alias 清理）
   - `symlink.ts`（符号链接管理）
   然后 `index.ts` 从多个子模块 re-export，提升可维护性。
2. **添加 `@deprecated` 或注释说明**：如果某些导出仅供特定调用方使用，可以在 `index.ts` 中通过 JSDoc 标注，提醒开发者不要随意在新代码中使用。
3. **考虑显式 `__internals` 导出用于测试**：如果测试需要访问 `download.ts` 的测试导出（如 `_downloadAndVerifyBinaryForTesting`），可以评估是否值得在 barrel 中增加一个 `__internals` 命名空间，避免测试直接 import 内部路径。
