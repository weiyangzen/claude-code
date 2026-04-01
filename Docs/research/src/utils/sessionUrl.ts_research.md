# sessionUrl.ts 深度研究

## 场景与职责

`sessionUrl.ts` 提供**会话恢复标识符的解析功能**，支持多种格式的会话标识：纯 UUID、会话 URL、JSONL 文件路径。

**核心职责：**
1. 解析用户输入的会话恢复标识符
2. 区分不同类型的标识（UUID、URL、本地文件）
3. 为每种类型生成统一的解析结果

**应用场景：**
- `/resume` 命令解析用户输入的会话 ID 或 URL
- CCR (Claude Code Remote) 会话连接
- 本地 JSONL 文件的直接加载

---

## 功能点目的

### 会话标识符解析
```typescript
export type ParsedSessionUrl = {
  sessionId: UUID
  ingressUrl: string | null
  isUrl: boolean
  jsonlFile: string | null
  isJsonlFile: boolean
}

export function parseSessionIdentifier(resumeIdentifier: string): ParsedSessionUrl | null
```

**解析优先级：**
1. **JSONL 文件路径** - 检查 `.jsonl` 后缀（优先于 URL 解析，避免 Windows 绝对路径被误判为 URL）
2. **纯 UUID** - 验证标准 UUID v4 格式
3. **URL** - 解析为会话入口 URL

**处理逻辑：**

| 输入类型 | 处理 | sessionId | ingressUrl | jsonlFile |
|---------|------|-----------|------------|-----------|
| `.jsonl` 文件 | 生成本地随机 UUID | `randomUUID()` | null | 原路径 |
| 纯 UUID | 直接使用 | 原值 | null | null |
| URL | 生成新 UUID | `randomUUID()` | 完整 URL | null |
| 无效输入 | 返回 null | - | - | - |

---

## 具体技术实现

### JSONL 文件检测
```typescript
// 优先检查，避免 Windows 路径 (C:\path\file.jsonl) 被解析为 URL (C: 作为协议)
if (resumeIdentifier.toLowerCase().endsWith('.jsonl')) {
  return {
    sessionId: randomUUID() as UUID,
    ingressUrl: null,
    isUrl: false,
    jsonlFile: resumeIdentifier,
    isJsonlFile: true,
  }
}
```

### UUID 验证
```typescript
// 复用 uuid.ts 的验证函数
if (validateUuid(resumeIdentifier)) {
  return {
    sessionId: resumeIdentifier as UUID,
    ingressUrl: null,
    isUrl: false,
    jsonlFile: null,
    isJsonlFile: false,
  }
}
```

### URL 解析
```typescript
try {
  const url = new URL(resumeIdentifier)
  return {
    sessionId: randomUUID() as UUID,
    ingressUrl: url.href,
    isUrl: true,
    jsonlFile: null,
    isJsonlFile: false,
  }
} catch {
  // Not a valid URL
}
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `ParsedSessionUrl` | 解析结果类型定义 |
| `parseSessionIdentifier` | 会话标识符解析函数 |

### 依赖模块
| 模块 | 用途 |
|------|------|
| `crypto` | `randomUUID`, `UUID` 类型 |
| `./uuid.js` | `validateUuid` |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/components/BridgeDialog.tsx` | Bridge 对话框 |
| `src/bridge/replBridge.ts` | REPL Bridge |
| `src/bridge/workSecret.ts` | Work secret 处理 |
| `src/bridge/remoteBridgeCore.ts` | 远程 Bridge 核心 |
| `src/components/tasks/RemoteSessionDetailDialog.tsx` | 远程会话详情 |
| `src/bridge/replBridgeTransport.ts` | REPL Bridge 传输 |
| `src/entrypoints/agentSdkTypes.ts` | SDK 类型定义 |
| `src/tools/AgentTool/AgentTool.tsx` | Agent 工具 |
| `src/cli/transports/ccrClient.ts` | CCR 客户端 |
| `src/cli/print.ts` | 打印处理 |
| `src/tools/AgentTool/UI.tsx` | Agent UI |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 远程 Agent 任务 |
| `src/commands/bridge/bridge.tsx` | Bridge 命令 |
| `src/commands/review/reviewRemote.ts` | 远程审查 |
| `src/utils/attribution.ts` | 归属统计 |
| `src/hooks/useReplBridge.tsx` | REPL Bridge Hook |

---

## 依赖与外部交互

### 外部依赖
| 模块 | 用途 |
|------|------|
| `crypto` | UUID 生成和类型 |

### 内部依赖
| 模块 | 用途 |
|------|------|
| `uuid.js` | UUID 格式验证 |

---

## 风险、边界与改进建议

### 已知风险

1. **Windows 路径误判**
   - Windows 绝对路径如 `C:\path\file.jsonl` 会被 `new URL()` 解析为协议 `C:`
   - 缓解：优先检查 `.jsonl` 后缀

2. **UUID 生成策略**
   - URL 和 JSONL 文件都生成随机 UUID，可能与现有会话冲突
   - 当前实现假设冲突概率可接受

3. **URL 格式宽松**
   - 任何有效的 URL 都被接受，不验证是否为有效的会话入口
   - 实际有效性由调用方验证

### 边界情况

| 场景 | 处理 |
|------|------|
| 空字符串 | 返回 null |
| 仅包含空白字符 | URL 解析失败，返回 null |
| 大小写混合的 `.jsonl` | `toLowerCase()` 处理 |
| 带查询参数的 URL | 保留完整 `url.href` |
| 相对路径 | 非 `.jsonl` 结尾，UUID 验证失败，URL 解析失败，返回 null |

### 改进建议

1. **增强验证**
   - 添加 URL 白名单验证（仅允许特定域名）
   - 验证 JSONL 文件存在性（可选，避免阻塞）

2. **冲突检测**
   - 检查生成的 UUID 是否已存在
   - 实现 UUID 命名空间避免冲突

3. **支持更多格式**
   - 支持短 ID 或别名
   - 支持会话名称模糊匹配

4. **错误信息**
   - 返回具体错误原因而非 null
   - 区分 "无效 UUID"、"无效 URL"、"文件不存在"
