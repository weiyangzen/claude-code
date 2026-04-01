# Research: src/utils/teleport/gitBundle.ts

## 场景与职责

`src/utils/teleport/gitBundle.ts` 实现 **Git Bundle 种子上传** 机制，使 Claude Code 能够在**没有 GitHub 连接**的情况下，将本地仓库的完整状态（包括未提交的 WIP）打包并上传到 Anthropic Files API，供远程 CCR 容器克隆。这是以下功能的关键 fallback：

- `--teleport` / `teleportToRemote`
- Remote Agent 任务
- Bridge 会话（当显式启用 `useBundle` 时）
- 其他需要把本地代码同步到远程容器的流程

该模块的核心价值在于：**仅依赖本地 `.git/` 目录，不需要 GitHub App 安装或远程 origin 可访问**。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `createAndUploadGitBundle` | 外部入口：定位 git 仓库 → 清理旧引用 → 检查空仓库 → 捕获 WIP → 创建 bundle → 上传 → 清理。 |
| `BundleUploadResult` | 联合类型：成功时返回 `fileId`、`bundleSizeBytes`、`scope`、`hasWip`；失败时返回 `error` 与 `failReason`。 |
| `BundleScope` | 标记 bundle 的粒度：`all`（全部历史）、`head`（仅当前分支）、`squashed`（单提交无历史）。 |
| `BundleFailReason` | 失败原因分类：`git_error`、`too_large`、`empty_repo`。 |

内部辅助：
- `_bundleWithFallback`：执行 `--all` → `HEAD` → `squashed` 的三级降级打包逻辑。

## 具体技术实现

### 完整流程（`createAndUploadGitBundle`）

1. **定位仓库**
   - `findGitRoot(opts?.cwd ?? getCwd())` — 找不到则返回 `success: false, error: 'Not in a git repository'`。

2. **清理陈旧引用**
   - 删除 `refs/seed/stash` 与 `refs/seed/root`，防止上一次崩溃运行遗留的引用污染新 bundle 或被用户仓库意外保留。

3. **空仓库检查**
   - 执行 `git for-each-ref --count=1 refs/`。
   - 若 stdout 为空，视为空仓库，返回 `failReason: 'empty_repo'`。
   - 注释说明：这比对 `HEAD` 检查更健壮，因为 orphan branch 可能在 `HEAD` 不存在时仍有提交。

4. **捕获 WIP（Work In Progress）**
   - `git stash create` — 生成一个悬空提交（dangling commit），**不修改工作区或 `refs/stash`**。
   - 若成功且 stdout 非空，则 `git update-ref refs/seed/stash <sha>` 使其可达。
   - **明确不捕获 untracked 文件**（`stash create` 的默认行为）。

5. **Bundle 创建（`_bundleWithFallback`）**
   - 尝试顺序：
     1. `git bundle create <path> --all [refs/seed/stash]` — 包含所有引用及 WIP。
     2. 若体积超过 `maxBytes`，覆盖重试 `HEAD [refs/seed/stash]` — 丢弃其他分支/标签历史。
     3. 若仍超过 `maxBytes`，执行 `git commit-tree HEAD^{tree}`（如有 WIP 则用 `refs/seed/stash^{tree}`）生成一个无父提交，更新-ref 到 `refs/seed/root`，再 bundle 该引用。
     4. 若仍然过大，返回 `too_large`。
   - 默认大小上限：`100 * 1024 * 1024`（100MB）。
   - 上限可通过 GrowthBook feature flag `tengu_ccr_bundle_max_bytes` 覆盖。

6. **上传**
   - 调用 `uploadFile(bundlePath, '_source_seed.bundle', config)`（来自 `src/services/api/filesApi.ts`）。
   - `config` 包含 `oauthToken`、`sessionId`、`baseUrl`。

7. **清理（`finally`）**
   - `unlink(bundlePath)` 删除本地临时文件。
   - 再次 `update-ref -d refs/seed/stash` 与 `refs/seed/root`，确保不污染用户仓库。

### Git 命令细节

| 步骤 | 命令 | 说明 |
|------|------|------|
| 清理旧引用 | `git update-ref -d refs/seed/stash` / `refs/seed/root` | 幂等删除。 |
| 空仓库检查 | `git for-each-ref --count=1 refs/` | 检查是否有任何引用存在。 |
| 捕获 WIP | `git stash create` → `git update-ref refs/seed/stash <sha>` | 仅捕获已跟踪文件的暂存/未暂存改动。 |
| Bundle --all | `git bundle create <path> --all [refs/seed/stash]` | 打包所有可达对象。 |
| Bundle HEAD | `git bundle create <path> HEAD [refs/seed/stash]` | 仅当前分支 + WIP。 |
| Squash | `git commit-tree <tree> -m seed` → `update-ref refs/seed/root <sha>` → `bundle create ... refs/seed/root` | 生成无历史单提交。 |

## 关键代码路径与文件引用

- **本文件**：`src/utils/teleport/gitBundle.ts`（292 行）
- **直接调用方**：
  - `src/utils/teleport.tsx` — `teleportToRemote` 在 GitHub preflight 失败或 `CCR_FORCE_BUNDLE=1` / `useBundle` 时调用。
  - `src/bridge/createSession.ts` — 当显式传入 `useBundle` 时通过 teleport 路径间接使用。
  - `src/utils/ultraplan/ccrSession.ts` — ultraplan 相关远程会话。
  - `src/commands/teleport/index.js`、`src/commands/bughunter/index.js`、`src/commands/review/reviewRemote.ts` — 通过 `teleportToRemote` 或直接使用 bundle 逻辑。

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `fs/promises` (`stat`, `unlink`) | 检查 bundle 大小、清理临时文件。 |
| `src/services/analytics/index.ts` | `logEvent` 记录 `tengu_ccr_bundle_upload` 等事件。 |
| `src/services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE` 读取 `tengu_ccr_bundle_max_bytes` feature flag。 |
| `src/services/api/filesApi.ts` | `uploadFile` 上传到 Anthropic Public Files API；`FilesApiConfig` 类型。 |
| `src/utils/cwd.ts` | `getCwd` 获取当前工作目录。 |
| `src/utils/debug.ts` | `logForDebugging` 输出调试日志。 |
| `src/utils/execFileNoThrow.ts` | `execFileNoThrowWithCwd` 执行 git 命令（不抛异常，返回 code/stderr）。 |
| `src/utils/git.ts` | `findGitRoot` 定位仓库根目录；`gitExe` 获取 git 可执行路径。 |
| `src/utils/tempfile.ts` | `generateTempFilePath` 生成临时 bundle 文件路径。 |

外部 API：
- 通过 `uploadFile` 间接调用 `POST /v1/files`（Files API）。

## 风险、边界与改进建议

### 风险
1. **Untracked 文件丢失**：`git stash create` 不捕获 untracked 文件，且代码没有额外步骤（如 `git add -N` 或手动打包）来弥补。用户在远程容器中看不到新增但未跟踪的文件，可能导致行为不一致。
2. **进程中断导致引用残留**：虽然 `finally` 会清理 `refs/seed/stash` 和 `refs/seed/root`，但如果进程在 bundle 创建后、清理前被强制终止（SIGKILL），这些引用会留在用户仓库中，直到下一次调用 `createAndUploadGitBundle` 的清理逻辑才会被删除。
3. **Squashed 模式的后端依赖**：`squashed` 模式生成的是一个无父提交（parentless commit），远程容器必须能识别并正确处理 `refs/seed/root`。若后端对该引用的处理逻辑出现回归，用户会拿到一个只有单提交、没有分支关联的仓库。
4. **大小限制对 monorepo 不友好**：默认 100MB 上限对大型仓库（尤其是历史长或包含大文件的仓库）很容易触发。虽然存在 feature flag 覆盖，但普通用户无法自行调整。
5. **Analytics 类型强制转换**：多处使用 `as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 将字符串字面量强转为 analytics 类型，若 analytics schema 变更，编译期无法发现不匹配。

### 边界
- 仅支持本地存在 `.git` 的仓库；bare repo 或非 git 目录直接返回失败。
- `git stash create` 失败（如 exit 非 0）不会中断流程，而是记录 debug 日志后继续无 WIP 打包。
- `uploadFile` 内部有自己的重试逻辑（3 次指数退避），但非网络类错误（如 413 过大、403 禁止）不会重试。

### 改进建议
- **补充 untracked 文件捕获**：在 `stash create` 之后，增加对 untracked 文件的检测与打包（例如通过 `git ls-files --others --exclude-standard` 读取列表，作为附加元数据上传，或在 bundle 前临时 `git add -N` 使其进入索引树）。
- **增强清理鲁棒性**：在 `createAndUploadGitBundle` 入口处（已有）和 Node 进程 `exit` 事件中注册清理钩子，降低 SIGKILL 外其他终止信号的残留概率。
- **暴露用户可配置的大小上限**：除了 GrowthBook feature flag，可考虑读取环境变量（如 `CLAUDE_CODE_BUNDLE_MAX_MB`）作为临时逃生舱。
- **为 squashed 模式增加后端能力协商**：在创建会话前，通过 API 查询后端是否支持 `refs/seed/root` 解析，避免在旧版本后端上静默降级到不可用的单提交仓库。
- **减少类型断言**：将 `logEvent` 的 payload 字段提取为受约束的联合类型，替代 `as` 强制转换。
