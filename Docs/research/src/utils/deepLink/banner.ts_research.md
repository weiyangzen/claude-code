# banner.ts 研究文档

> 文件路径：`src/utils/deepLink/banner.ts`  
> 研究时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 场景与职责

`banner.ts` 负责构建并展示 **Deep Link 来源安全警告横幅**。当用户通过外部 `claude-cli://` 链接打开 Claude Code 会话时（尤其是 Linux 的 `xdg-open` 或浏览器已设置“始终允许”的场景），操作系统层面往往不会弹出二次确认。该模块在应用层补上一道**来源 provenance 信号**，让用户明确意识到：当前会话并非自己手动启动，而是来自外部来源；同时提示用户仔细审查预填充的 prompt（可能包含同形异义字符或隐藏指令），并注意当前加载的是哪个工作目录（进而对应哪份 `CLAUDE.md`）。

简言之，它是**安全互锁（security interstitial）的 CLI 等价物**，与 claude.ai 网页端对外部来源 prefill 的处理理念一致。

---

## 功能点目的

| 功能 | 目的 |
|------|------|
| `buildDeepLinkBanner` | 根据解析后的 deep link 信息生成多行警告文本，作为系统消息（`warning` 级别）插入会话首条消息。 |
| `readLastFetchTime` | 读取当前工作目录对应 Git 仓库的 `.git/FETCH_HEAD` 修改时间，用于判断 `CLAUDE.md` 是否可能已过时。 |
| `tildify` | 将绝对路径中的用户主目录缩写为 `~`，提升横幅可读性。 |

### 横幅内容规则
1. **始终显示工作目录**（`tildify` 后），让用户知道加载了哪份 `CLAUDE.md`。
2. 若 deep link 带 `?repo=` 且通过 MRU 映射解析到本地克隆，则显示仓库 slug 及最后 fetch 时间；若超过 7 天未 fetch，追加 `CLAUDE.md may be stale` 警告。
3. 若带 `?q=` 预填充 prompt：
   - 长度 ≤ 1000 字符：提示 "review carefully before pressing Enter"
   - 长度 > 1000 字符：提示 "scroll to review the entire prompt before pressing Enter"（防止恶意内容被藏在屏幕外）。

---

## 具体技术实现

### 关键流程

#### 1. `buildDeepLinkBanner(info: DeepLinkBannerInfo): string`
- 输入结构：
  ```ts
  type DeepLinkBannerInfo = {
    cwd: string
    prefillLength?: number
    repo?: string
    lastFetch?: Date
  }
  ```
- 按顺序拼接 1~3 行文本后返回，调用方（`main.tsx`）将其包装为 `createSystemMessage(..., 'warning')`。

#### 2. `readLastFetchTime(cwd: string): Promise<Date | undefined>`
- 通过 `getGitDir(cwd)` 获取 `.git` 目录路径（支持 worktree/submodule）。
- 再通过 `getCommonDir(gitDir)` 获取共享 git 目录（常规仓库返回 `null`）。
- 并发读取：
  - `<gitDir>/FETCH_HEAD`
  - `<commonDir>/FETCH_HEAD`（若存在）
- 返回两者中**更新**的时间。 rationale：worktree 的本地 FETCH_HEAD 不会因为主仓库 fetch 而被更新，因此需要检查 common dir，避免误报 "never fetched"。

#### 3. `mtimeOrUndefined(p: string): Promise<Date | undefined>`
- 简单的 `fs/promises.stat` 包装，ENOENT 时返回 `undefined`。

### 数据结构

| 常量/类型 | 值/说明 |
|-----------|---------|
| `STALE_FETCH_WARN_MS` | `7 * 24 * 60 * 60 * 1000`（7 天） |
| `LONG_PREFILL_THRESHOLD` | `1000`（字符数阈值） |

---

## 关键代码路径与文件引用

### 本文件导出
- `buildDeepLinkBanner(info)` → `src/main.tsx` ~L3787
- `readLastFetchTime(cwd)` → `src/utils/deepLink/protocolHandler.ts` ~L59

### 调用方详情

**`src/main.tsx`**
- 在 `run()` 的默认 action 中，当 `options.deepLinkOrigin` 为真时：
  ```ts
  deepLinkBanner = createSystemMessage(buildDeepLinkBanner({
    cwd: getCwd(),
    prefillLength: options.prefill?.length,
    repo: options.deepLinkRepo,
    lastFetch: options.deepLinkLastFetch !== undefined ? new Date(options.deepLinkLastFetch) : undefined
  }), 'warning');
  ```
- 同时触发 analytics：`logEvent('tengu_deep_link_opened', { has_prefill, has_repo })`。
- 若不带 `deepLinkOrigin` 但仍有 `prefill`，则降级为更简单的提示语。

### 被调用方详情

| 导入来源 | 符号 | 用途 |
|----------|------|------|
| `fs/promises` | `stat` | 读取 FETCH_HEAD mtime |
| `os` | `homedir` | `tildify` 路径缩短 |
| `path` | `join`, `sep` | 路径拼接 |
| `../format.js` | `formatNumber`, `formatRelativeTimeAgo` | 数字与相对时间格式化 |
| `../git/gitFilesystem.js` | `getCommonDir` | 获取共享 git 目录 |
| `../git.js` | `getGitDir` | 解析 `.git` 目录（含 worktree） |

---

## 依赖与外部交互

- **无外部网络请求**：纯本地文件系统读取。
- **Git 状态依赖**：依赖仓库存在 `.git/FETCH_HEAD`；对于从未 fetch 过的新克隆仓库，返回 `undefined`。
- **Worktree 兼容**：通过 `getCommonDir` 双路读取，确保主仓库 fetch 时间能被 worktree 感知。

---

## 风险、边界与改进建议

### 风险
1. **FETCH_HEAD 不存在即视为 "never"**：若用户使用的是 `git clone --mirror` 或某些特殊 CI 环境，FETCH_HEAD 可能不存在，导致 stale 警告误报或缺失。
2. **时间精度问题**：`stat.mtime` 受本地系统时钟影响，不能反映上游实际最新提交时间，只能作为“本地多久没同步”的代理指标。
3. **长 prompt 阈值固定**：`LONG_PREFILL_THRESHOLD = 1000` 基于 80 列终端约 12~15 行的估算，未考虑用户实际终端高度；在更高分辨率终端上可能过于保守或保守不足。

### 边界
- `tildify` 刻意不使用 `getDisplayPath()`，因为当前目录若是 cwd，`getDisplayPath` 的相对路径分支会将其折叠为空字符串，从而丢失目录信息。
- 仅处理**已解析**的 `cwd`、`repo`、`lastFetch`，URI 解析与路径解析逻辑全部下沉到 `protocolHandler.ts`。

### 改进建议
1. **动态终端高度感知**：未来可通过 `process.stdout.rows` 在启动时计算更贴合当前终端的阈值。
2. **上游时间对比**：若网络允许，可考虑对比本地 `FETCH_HEAD` 与远程 `HEAD` 的日期，给出更准确的 stale 判断。
3. **测试覆盖**：当前仓库未见针对 `banner.ts` 的单元测试，建议补充：
   - 各字段组合下的横幅文本快照测试；
   - `readLastFetchTime` 对常规仓库/worktree/无 git 目录的边界测试。
