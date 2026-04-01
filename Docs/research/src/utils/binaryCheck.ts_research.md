# src/utils/binaryCheck.ts 深入研究

## 场景与职责

`binaryCheck.ts` 提供一个轻量级的二进制/命令存在性检查工具。在 Claude Code 中，它主要用于：
- **LSP 推荐系统**：判断用户环境是否已安装某语言服务器（如 `gopls`、`rust-analyzer`），以决定是否需要推荐安装对应插件。
- **其他工具链探测**：在需要时检查外部 CLI 是否可用。

该模块通过封装 `which` 命令实现跨平台探测，并引入会话级缓存避免重复执行系统调用。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `isBinaryInstalled(command)` | 异步检查指定命令是否存在于系统 PATH 中 |
| `clearBinaryCache()` | 清除会话缓存，主要用于测试隔离 |

## 具体技术实现

### 缓存机制
- 使用 `Map<string, boolean>` 作为会话级内存缓存（`binaryCache`）。
- 对输入命令做 `trim()` 规范化后作为缓存键。
- 首次查询后结果常驻内存，直到进程结束或调用 `clearBinaryCache()`。

### 探测实现
```ts
let exists = false
if (await which(trimmedCommand).catch(() => null)) {
  exists = true
}
```
- 依赖 `src/utils/which.js` 的 `which` 函数（跨平台封装了 Unix `which` 与 Windows `where`）。
- 使用 `.catch(() => null)` 将异常静默转为 `null`，从而得到 `false`。

### 边缘情况处理
- 空字符串或纯空白字符命令：直接返回 `false` 并记录 debug 日志。
- 命令前后空格：统一 `trim()` 后再查询与缓存。

## 关键代码路径与文件引用

```
src/utils/plugins/lspRecommendation.ts
  └── isBinaryInstalled(command)
      [根据已安装的语言服务器决定是否推荐对应插件]
```

### 依赖模块
- `src/utils/debug.js` — `logForDebugging`
- `src/utils/which.js` — `which`

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| 操作系统 PATH | `which(command)` | 调用系统 `which`/`where` 探测可执行文件 |
| 调试日志 | `logForDebugging` | 记录缓存命中/未命中及探测结果 |

## 风险、边界与改进建议

### 风险
1. **缓存不感知 PATH 变化**：若用户在会话期间修改了 `PATH` 并安装了新二进制，缓存仍返回旧结果。
2. **仅检查存在性，不验证版本/可执行性**：`which` 成功不代表该二进制能正常启动或版本符合要求。
3. **无超时控制**：`which` 在极端情况下（如 NFS 挂载的 PATH 目录无响应）可能长时间阻塞。

### 边界
- 仅返回 `boolean`，不提供二进制绝对路径、版本号或错误详情。
- 缓存是进程级的，CLI 重启后自然失效。

### 改进建议
1. **可选的缓存 TTL**：为缓存增加短时间 TTL（如 5 分钟），在保持性能的同时对 PATH 变化有一定敏感度。
2. **扩展为能力探测**：增加 `checkBinaryVersion(command, minVersion?)` 变体，通过 `command --version` 进一步验证可用性。
3. **并发控制**：若调用方在短时间内批量检查多个二进制，当前实现已足够轻量；若未来检查数量激增，可考虑 `Promise.all` 批量并发。
4. **返回路径信息**：将 `which` 的返回路径暴露出来，供上层展示“在 /usr/local/bin/gopls 找到”等更丰富的提示。
