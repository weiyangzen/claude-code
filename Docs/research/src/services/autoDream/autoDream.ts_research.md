# autoDream.ts 深度研究文档

## 1. 场景与职责

### 1.1 模块定位
`autoDream.ts` 是 Claude Code 的**后台记忆整合（Memory Consolidation）**核心模块，负责在后台自动触发"梦境"（Dream）子代理，对历史会话进行回顾、整理和记忆归档。

### 1.2 业务场景
- **长期会话管理**：当用户与 Claude Code 进行多次会话后，系统自动整理这些会话中的关键信息
- **记忆持久化**：将分散在多个会话中的信息整合成结构化的记忆文件
- **智能触发**：基于时间和会话数量阈值自动触发，避免频繁干扰用户

### 1.3 核心职责
1. **门控检查**：多重条件判断是否应该触发记忆整合
2. **子代理调度**：通过 forked agent 模式启动独立的记忆整理任务
3. **任务生命周期管理**：注册、跟踪、完成或失败处理
4. **进度监控**：实时收集子代理的执行进度和文件操作

---

## 2. 功能点目的

### 2.1 三级门控机制（Gate Order）
```
时间门 → 扫描节流 → 会话门 → 锁门
```

| 门控 | 目的 | 默认阈值 |
|------|------|----------|
| Time Gate | 确保距离上次整合有足够间隔 | 24小时 (minHours) |
| Session Gate | 确保有足够的新会话值得整合 | 5个会话 (minSessions) |
| Lock Gate | 防止多个进程同时执行整合 | PID-based 文件锁 |
| Scan Throttle | 避免频繁扫描文件系统 | 10分钟 |

### 2.2 配置管理
- **GrowthBook 远程配置**：通过 `tengu_onyx_plover` feature flag 动态调整阈值
- **本地设置覆盖**：用户可通过 `settings.json` 中的 `autoDreamEnabled` 显式控制
- **防御性验证**：对远程配置进行类型和范围校验，防止错误配置导致异常

### 2.3 子代理执行
- **Forked Agent 模式**：利用 `runForkedAgent` 创建与主循环共享 prompt cache 的子代理
- **工具约束**：限制子代理只能使用只读 Bash 命令（`ls`, `find`, `grep`, `cat` 等）
- **内存目录隔离**：子代理只能读写自动记忆目录内的文件

### 2.4 任务状态机
```
registerDreamTask → addDreamTurn (progress) → completeDreamTask/failDreamTask
```

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
// 配置结构
interface AutoDreamConfig {
  minHours: number;      // 默认 24
  minSessions: number;   // 默认 5
}

// 内部状态（闭包内）
let runner: ((context, appendSystemMessage?) => Promise<void>) | null = null;
let lastSessionScanAt = 0;  // 上次扫描时间戳
```

### 3.2 关键流程

#### 3.2.1 初始化流程 `initAutoDream()`
```typescript
export function initAutoDream(): void {
  // 创建闭包作用域的状态
  let lastSessionScanAt = 0;
  
  // 定义 runner 函数
  runner = async function runAutoDream(context, appendSystemMessage) {
    // 1. 获取配置
    const cfg = getConfig();
    const force = isForced();
    
    // 2. 门控检查
    if (!force && !isGateOpen()) return;
    
    // 3. 时间门检查
    const lastAt = await readLastConsolidatedAt();
    const hoursSince = (Date.now() - lastAt) / 3_600_000;
    if (!force && hoursSince < cfg.minHours) return;
    
    // 4. 扫描节流检查
    const sinceScanMs = Date.now() - lastSessionScanAt;
    if (!force && sinceScanMs < SESSION_SCAN_INTERVAL_MS) return;
    lastSessionScanAt = Date.now();
    
    // 5. 会话门检查
    let sessionIds = await listSessionsTouchedSince(lastAt);
    sessionIds = sessionIds.filter(id => id !== currentSession);
    if (!force && sessionIds.length < cfg.minSessions) return;
    
    // 6. 获取锁
    const priorMtime = await tryAcquireConsolidationLock();
    if (priorMtime === null) return;
    
    // 7. 执行任务
    await executeDreamTask(context, sessionIds, priorMtime, appendSystemMessage);
  };
}
```

#### 3.2.2 任务执行流程
```typescript
async function executeDreamTask(context, sessionIds, priorMtime, appendSystemMessage) {
  // 1. 注册任务
  const taskId = registerDreamTask(setAppState, {
    sessionsReviewing: sessionIds.length,
    priorMtime,
    abortController,
  });
  
  try {
    // 2. 构建提示词
    const prompt = buildConsolidationPrompt(memoryRoot, transcriptDir, extra);
    
    // 3. 运行 forked agent
    const result = await runForkedAgent({
      promptMessages: [createUserMessage({ content: prompt })],
      cacheSafeParams: createCacheSafeParams(context),
      canUseTool: createAutoMemCanUseTool(memoryRoot),
      querySource: 'auto_dream',
      forkLabel: 'auto_dream',
      skipTranscript: true,
      overrides: { abortController },
      onMessage: makeDreamProgressWatcher(taskId, setAppState),
    });
    
    // 4. 完成任务
    completeDreamTask(taskId, setAppState);
    
    // 5. 发送完成消息（如果文件被修改）
    if (dreamState.filesTouched.length > 0) {
      appendSystemMessage({
        ...createMemorySavedMessage(dreamState.filesTouched),
        verb: 'Improved',
      });
    }
  } catch (e) {
    // 6. 失败处理：回滚锁
    failDreamTask(taskId, setAppState);
    await rollbackConsolidationLock(priorMtime);
  }
}
```

#### 3.2.3 进度监控 `makeDreamProgressWatcher()`
```typescript
function makeDreamProgressWatcher(taskId, setAppState): (msg: Message) => void {
  return msg => {
    if (msg.type !== 'assistant') return;
    
    let text = '';
    let toolUseCount = 0;
    const touchedPaths: string[] = [];
    
    for (const block of msg.message.content) {
      if (block.type === 'text') {
        text += block.text;
      } else if (block.type === 'tool_use') {
        toolUseCount++;
        // 捕获 Edit/Write 工具的文件路径
        if (block.name === FILE_EDIT_TOOL_NAME || block.name === FILE_WRITE_TOOL_NAME) {
          const input = block.input as { file_path?: unknown };
          if (typeof input.file_path === 'string') {
            touchedPaths.push(input.file_path);
          }
        }
      }
    }
    
    // 更新任务状态
    addDreamTurn(taskId, { text: text.trim(), toolUseCount }, touchedPaths, setAppState);
  };
}
```

### 3.3 门控条件详解

#### 3.3.1 `isGateOpen()` - 主开关
```typescript
function isGateOpen(): boolean {
  if (getKairosActive()) return false;        // KAIROS 模式使用磁盘技能
  if (getIsRemoteMode()) return false;        // 远程模式禁用
  if (!isAutoMemoryEnabled()) return false;   // 自动记忆总开关
  return isAutoDreamEnabled();                // autoDream 专用开关
}
```

#### 3.3.2 配置获取 `getConfig()`
```typescript
function getConfig(): AutoDreamConfig {
  const raw = getFeatureValue_CACHED_MAY_BE_STALE<Partial<AutoDreamConfig> | null>(
    'tengu_onyx_plover',
    null,
  );
  return {
    minHours: typeof raw?.minHours === 'number' && Number.isFinite(raw.minHours) && raw.minHours > 0
      ? raw.minHours
      : DEFAULTS.minHours,
    minSessions: typeof raw?.minSessions === 'number' && Number.isFinite(raw.minSessions) && raw.minSessions > 0
      ? raw.minSessions
      : DEFAULTS.minSessions,
  };
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 入口点
| 文件 | 函数 | 说明 |
|------|------|------|
| `src/utils/backgroundHousekeeping.ts` | `initAutoDream()` | 启动时初始化 |
| `src/query/stopHooks.ts` | `executeAutoDream()` | 每轮对话后触发 |

### 4.2 核心调用链
```
handleStopHooks (stopHooks.ts:155)
  └── executeAutoDream (autoDream.ts:319)
        └── runner (闭包内)
              ├── getConfig()
              ├── isGateOpen()
              ├── readLastConsolidatedAt() → consolidationLock.ts
              ├── listSessionsTouchedSince() → consolidationLock.ts
              ├── tryAcquireConsolidationLock() → consolidationLock.ts
              ├── registerDreamTask() → DreamTask.ts
              ├── buildConsolidationPrompt() → consolidationPrompt.ts
              ├── runForkedAgent() → forkedAgent.ts
              │     └── makeDreamProgressWatcher() (回调)
              ├── completeDreamTask() / failDreamTask() → DreamTask.ts
              └── rollbackConsolidationLock() → consolidationLock.ts
```

### 4.3 关键依赖文件
| 文件 | 用途 |
|------|------|
| `src/services/autoDream/config.ts` | 启用状态检查 `isAutoDreamEnabled()` |
| `src/services/autoDream/consolidationLock.ts` | 锁管理和时间戳读取 |
| `src/services/autoDream/consolidationPrompt.ts` | 提示词构建 |
| `src/tasks/DreamTask/DreamTask.ts` | 任务注册和状态管理 |
| `src/utils/forkedAgent.ts` | Forked agent 执行框架 |
| `src/services/extractMemories/extractMemories.ts` | `createAutoMemCanUseTool()` 共享 |
| `src/memdir/paths.ts` | `getAutoMemPath()`, `isAutoMemoryEnabled()` |

---

## 5. 依赖与外部交互

### 5.1 上游依赖（调用本模块）
```typescript
// backgroundHousekeeping.ts
import { initAutoDream } from '../services/autoDream/autoDream.js';
startBackgroundHousekeeping() {
  initAutoDream();
}

// stopHooks.ts
import { executeAutoDream } from '../services/autoDream/autoDream.js';
if (!toolUseContext.agentId) {
  void executeAutoDream(stopHookContext, toolUseContext.appendSystemMessage);
}
```

### 5.2 下游依赖（本模块调用）
| 模块 | 函数/类型 | 用途 |
|------|-----------|------|
| `growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE()` | 远程配置获取 |
| `analytics/index.ts` | `logEvent()` | 埋点上报 |
| `forkedAgent.ts` | `runForkedAgent()`, `createCacheSafeParams()` | 子代理执行 |
| `messages.ts` | `createUserMessage()`, `createMemorySavedMessage()` | 消息构建 |
| `bootstrap/state.ts` | `getOriginalCwd()`, `getKairosActive()`, `getIsRemoteMode()`, `getSessionId()` | 状态获取 |
| `sessionStorage.ts` | `getProjectDir()` | 项目目录获取 |

### 5.3 配置依赖
| 配置项 | 来源 | 说明 |
|--------|------|------|
| `autoDreamEnabled` | `settings.json` | 用户显式设置 |
| `tengu_onyx_plover` | GrowthBook | 远程动态配置 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 环境变量 | 禁用自动记忆 |
| `CLAUDE_CODE_SIMPLE` | 环境变量 | bare 模式禁用 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 锁竞争与死锁
- **风险**：多个进程同时尝试获取锁可能导致竞争条件
- **缓解**：使用 PID 检查 + 1小时过期时间的乐观锁策略
- **代码位置**：`consolidationLock.ts:46-84`

#### 6.1.2 扫描性能问题
- **风险**：会话目录文件过多时，`listSessionsTouchedSince` 可能变慢
- **缓解**：10分钟扫描节流 + 基于 mtime 的快速过滤
- **潜在问题**：大量文件时仍可能影响性能

#### 6.1.3 子代理失败恢复
- **风险**：forked agent 执行失败时，需要正确回滚锁时间戳
- **处理**：`rollbackConsolidationLock()` 恢复 `priorMtime`
- **边界**：如果进程崩溃，锁可能残留，需等待 1 小时过期

### 6.2 边界条件

| 场景 | 行为 |
|------|------|
| 首次运行（无锁文件） | `readLastConsolidatedAt()` 返回 0，时间门通过 |
| 当前会话是唯一会话 | 过滤后 `sessionIds.length = 0`，会话门阻止触发 |
| 用户手动触发 /dream | `recordConsolidation()` 更新时间戳，重置计时 |
| 进程持有锁时崩溃 | 下次启动检测到过期 PID，重新获取锁 |
| KAIROS 模式 | `isGateOpen()` 返回 false，完全禁用 |

### 6.3 改进建议

#### 6.3.1 可观测性增强
```typescript
// 建议：增加更详细的调试日志和指标
logEvent('tengu_auto_dream_gate_decision', {
  gate: 'time|session|lock',
  passed: boolean,
  value: number,
  threshold: number,
});
```

#### 6.3.2 扫描优化
- 考虑使用文件系统监听（如 `fs.watch`）代替轮询扫描
- 或者维护一个会话变更的内存索引

#### 6.3.3 配置热更新
- 当前配置在每次 `runner` 调用时重新获取，已支持动态调整
- 建议增加配置变更的日志，便于排查问题

#### 6.3.4 测试覆盖
- 当前未发现专门的 `autoDream.test.ts`
- 建议增加单元测试覆盖门控逻辑和错误处理路径

### 6.4 相关 Issue 模式
- 锁文件残留导致长时间无法触发 → 检查 `HOLDER_STALE_MS` 是否合理
- 会话计数不准确 → 检查 `listSessionsTouchedSince` 的过滤逻辑
- 配置不生效 → 检查 GrowthBook 缓存和本地设置优先级
