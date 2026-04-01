# 研究文档：src/utils/git/gitignore.ts

## 场景与职责

`gitignore.ts` 负责与 git 忽略规则相关的两类操作：
1. **查询路径是否被 git 忽略**：通过调用 `git check-ignore` 子进程，准确判断某个路径是否被仓库的 `.gitignore`、`.git/info/exclude` 或全局 gitignore 忽略。
2. **向全局 gitignore 添加规则**：在需要时（如本地设置文件生成）将文件模式写入用户主目录下的 `~/.config/git/ignore`，并避免重复添加。

该模块的核心价值在于**利用 git 原生的忽略解析能力**，而不是在 Node.js 中重新实现复杂的 gitignore 匹配逻辑（嵌套规则、优先级、全局规则等）。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `isPathGitignored(filePath, cwd)` | 调用 `git check-ignore` 判断路径是否被忽略；在仓库外时返回 `false`（fail open） |
| `getGlobalGitignorePath()` | 返回全局 gitignore 的固定路径：`~/.config/git/ignore` |
| `addFileGlobRuleToGitignore(filename, cwd?)` | 将 `**/{filename}` 规则写入全局 gitignore（若尚未被任何 gitignore 忽略） |

## 具体技术实现

### 1. isPathGitignored

**实现：**
```typescript
const { code } = await execFileNoThrowWithCwd(
  'git',
  ['check-ignore', filePath],
  { preserveOutputOnError: false, cwd }
)
return code === 0
```

**git check-ignore 的退出码语义：**
- `0`：路径被忽略
- `1`：路径未被忽略
- `128`：不在 git 仓库中（或其他错误）

**设计选择：**
- 使用 `execFileNoThrowWithCwd`（非抛出版），因此 `128` 不会抛异常，而是返回 `code === 0` → `false`
- 这种 "fail open" 策略确保：即使不在 git 仓库中，调用方也不会因为误判而被阻塞

**适用场景：**
- 动态技能发现时（`src/skills/loadSkillsDir.ts`），判断 `.claude/skills` 所在目录是否被 gitignore，防止加载 `node_modules` 等被忽略目录下的技能

### 2. addFileGlobRuleToGitignore

**流程：**
```
1. 检查 cwd 是否在 git 仓库中（dirIsInGitRepo）
   └─ 否 → 直接返回
2. 构造规则：`**/{filename}`
3. 构造测试路径：
   - 若 filename 以 / 结尾（目录模式），测试路径为 `{filename}sample-file.txt`
   - 否则测试路径就是 filename
4. 调用 isPathGitignored(testPath, cwd)
   └─ 若已被忽略 → 直接返回（无需重复添加）
5. 获取全局 gitignore 路径：~/.config/git/ignore
6. 确保 ~/.config/git 目录存在（mkdir recursive）
7. 读取现有内容：
   - 若文件已包含该规则 → 返回
   - 否则追加 `\n{rule}\n`
   - 若文件不存在（ENOENT）→ 新建文件并写入 `{rule}\n`
8. 捕获并记录其他异常
```

**设计细节：**
- 使用全局 gitignore（`~/.config/git/ignore`）而非仓库级 `.gitignore`，因为该函数主要用于写入用户本地生成的文件（如 `settings.local.json`），不应污染项目的版本控制文件。
- 规则前缀为 `**/`，表示在任何层级匹配该文件名。
- 目录模式的测试路径转换：因为 `git check-ignore` 对目录模式的处理可能与文件不同，通过附加一个虚拟文件名来确保检测的准确性。

## 关键代码路径与文件引用

### 内部调用关系

```
isPathGitignored
  └── execFileNoThrowWithCwd('git', ['check-ignore', filePath], { cwd })

addFileGlobRuleToGitignore
  ├── dirIsInGitRepo(cwd)   [from src/utils/git.js]
  ├── isPathGitignored(testPath, cwd)
  ├── getGlobalGitignorePath()
  ├── mkdir(configGitDir)
  ├── readFile(globalGitignorePath)
  ├── appendFile(globalGitignorePath, ...)
  └── writeFile(globalGitignorePath, ...)  [ENOENT fallback]
```

### 外部调用方

| 调用方文件 | 调用符号 | 用途 |
|-----------|---------|------|
| `src/skills/loadSkillsDir.ts` | `isPathGitignored` | 动态技能发现时，跳过 gitignored 目录下的 `.claude/skills`（如 `node_modules/pkg/.claude/skills`） |
| `src/utils/settings/settings.ts` | `addFileGlobRuleToGitignore` | 更新 `settings.local.json` 后，自动将其加入全局 gitignore，避免用户意外提交本地敏感配置 |

### 直接读写的文件

- `~/.config/git/ignore`（全局 gitignore）
- 通过 `git check-ignore` 间接读取仓库内的各级 `.gitignore`、`.git/info/exclude`、全局 gitignore

## 依赖与外部交互

### 导入依赖

```typescript
import { appendFile, mkdir, readFile, writeFile } from 'fs/promises'
import { homedir } from 'os'
import { dirname, join } from 'path'
import { getCwd } from '../cwd.js'
import { getErrnoCode } from '../errors.js'
import { execFileNoThrowWithCwd } from '../execFileNoThrow.js'
import { dirIsInGitRepo } from '../git.js'
import { logError } from '../log.js'
```

### 外部命令

- **`git check-ignore`**：唯一依赖的 git 子进程命令。这是有意为之，因为 gitignore 规则的优先级和嵌套解析非常复杂，自行实现容易出错。

## 风险、边界与改进建议

### 已知限制与风险

1. **依赖 git 可执行文件**：
   - `isPathGitignored` 必须能调用 `git`。在极少数没有安装 git 的环境中（虽然 Claude Code 本身重度依赖 git），该函数会返回 `false`。
   - 由于使用 `execFileNoThrowWithCwd`，失败时不会抛异常，调用方需理解这是 "fail open" 语义。

2. **全局 gitignore 路径硬编码**：
   - `getGlobalGitignorePath()` 固定返回 `~/.config/git/ignore`。这是 XDG 规范下的默认路径，但用户可能通过 `core.excludesFile` 配置了其他全局 gitignore 路径。
   - 当前实现**不读取** `git config core.excludesFile`，因此：
     - `addFileGlobRuleToGitignore` 写入的位置可能与用户预期的全局 gitignore 不一致
     - `isPathGitignored` 本身不受影响（因为 `git check-ignore` 会正确读取用户的 `core.excludesFile`）

3. **并发写入风险**：
   - `addFileGlobRuleToGitignore` 的 "读取 → 检查包含 → 追加/写入" 不是原子操作。若同一进程或外部进程并发修改 `~/.config/git/ignore`，可能出现重复条目或内容截断。
   - 实际场景：该函数在会话中通常只被调用一两次（写 `settings.local.json`），并发概率极低。

4. **目录模式检测的边界情况**：
   - 对于以 `/` 结尾的 `filename`，测试路径使用 `{filename}sample-file.txt`。若 `.gitignore` 中恰好有只匹配 `sample-file.txt` 的规则，可能导致误判。但 `sample-file.txt` 是一个足够独特的虚拟文件名，实际误触概率可忽略。

### 改进建议

1. **支持 `core.excludesFile`**：
   - 可通过 `git config --global core.excludesFile` 或读取 `~/.gitconfig` 来获取用户自定义的全局 gitignore 路径，使 `getGlobalGitignorePath()` 和 `addFileGlobRuleToGitignore` 更贴合用户配置。
   - 若实现，建议缓存该路径结果，避免每次调用都启动 `git config` 子进程。

2. **原子写入**：
   - 对 `~/.config/git/ignore` 的修改可采用 "写临时文件 → rename 覆盖" 的方式，避免并发截断。不过考虑到该文件的用户可编辑性，追加模式通常更安全（不会丢失用户手动添加的其他规则）。
   - 若采用原子写入，需先完整读取、去重、再写回，确保不破坏现有内容。

3. **增加单元测试**：
   - 模拟 `git check-ignore` 的三种退出码（0/1/128）
   - 测试 `addFileGlobRuleToGitignore` 的幂等性（多次调用不重复添加）
   - 测试目录模式（`filename/`）的 `isPathGitignored` 前置检查逻辑
   - 测试全局 gitignore 文件从缺失到创建的路径

4. **考虑纯 JS 实现的 fallback**：
   - 若未来需要在无法调用 git 的环境中判断 gitignore，可引入 `ignore` 库（已在 `src/utils/worktree.ts` 中使用）做本地近似判断。但这会失去对 `core.excludesFile` 和 `.git/info/exclude` 的自动支持，需谨慎权衡。
