# submitTranscriptShare.ts 研究文档

## 场景与职责

`submitTranscriptShare.ts` 是 Claude Code CLI 反馈调查系统的核心服务端交互模块，负责将用户同意的会话转录数据安全地上传到 Anthropic 服务器。该模块在用户同意分享转录后执行，是整个反馈流程的数据收集和传输层。

该模块的主要职责：
1. **收集会话数据**: 聚合主会话转录、子代理转录和原始 JSONL 转录
2. **敏感信息脱敏**: 在传输前对数据进行敏感信息检测和脱敏处理
3. **安全传输**: 使用 OAuth 认证将数据上传到 Anthropic API
4. **错误处理**: 优雅处理网络错误、认证错误和文件系统错误

## 功能点目的

### 1. 全面的数据收集
收集多维度数据以帮助 Anthropic 改进 Claude Code：
- **标准化转录**: 通过 `normalizeMessagesForAPI` 处理的消息记录
- **子代理转录**: 从子代理会话中提取的转录数据
- **原始 JSONL**: 完整的原始会话记录（带大小限制）
- **元数据**: 触发原因、版本、平台信息

### 2. 隐私保护机制
- **敏感信息脱敏**: 使用 `redactSensitiveInfo` 移除 API 密钥、令牌等敏感数据
- **文件大小限制**: 防止大文件导致内存溢出（`MAX_TRANSCRIPT_READ_BYTES`）
- **可选分享**: 仅在用户明确同意后执行

### 3. 多种触发场景支持
支持多种触发转录分享的场景：
- `bad_feedback_survey`: 用户给出差评后
- `good_feedback_survey`: 用户给出好评后
- `frustration`: 检测到用户沮丧情绪
- `memory_survey`: 内存功能相关调查

### 4. 健壮的错误处理
- 文件读取错误（文件不存在或权限问题）不中断流程
- 网络错误记录调试日志但不抛出
- 认证失败返回明确的错误状态

## 具体技术实现

### 关键数据结构

```typescript
// 分享结果类型
type TranscriptShareResult = {
  success: boolean;
  transcriptId?: string;
};

// 触发类型
export type TranscriptShareTrigger =
  | 'bad_feedback_survey'
  | 'good_feedback_survey'
  | 'frustration'
  | 'memory_survey';

// API 请求体结构
{
  content: string;           // JSON 序列化并脱敏后的数据
  appearance_id: string;     // 调查展示唯一标识
}

// 内部数据对象
{
  trigger: TranscriptShareTrigger;
  version: string;           // MACRO.VERSION
  platform: string;          // process.platform
  transcript: Message[];     // 标准化后的消息
  subagentTranscripts?: { [agentId: string]: Message[] };
  rawTranscriptJsonl?: string;
}
```

### 关键流程

#### 1. 数据收集流程
```
submitTranscriptShare(messages, trigger, appearanceId)
  ├── normalizeMessagesForAPI(messages) → 标准化主转录
  ├── extractAgentIdsFromMessages(messages) → 提取子代理 ID
  ├── loadSubagentTranscripts(agentIds) → 加载子代理转录
  ├── 读取原始 JSONL（带大小检查）
  │     ├── stat(transcriptPath) → 检查文件大小
  │     └── 如果 size <= MAX_TRANSCRIPT_READ_BYTES → readFile
  └── 组装数据对象
```

#### 2. 数据传输流程
```
组装数据对象
  ├── jsonStringify(data) → JSON 序列化
  ├── redactSensitiveInfo(content) → 敏感信息脱敏
  ├── checkAndRefreshOAuthTokenIfNeeded() → 刷新令牌
  ├── getAuthHeaders() → 获取认证头
  └── axios.post(API_ENDPOINT, { content, appearance_id }, { timeout: 30000 })
        ├── 成功 (200/201) → 返回 { success: true, transcriptId }
        └── 失败 → 记录错误日志，返回 { success: false }
```

### API 端点

```typescript
const API_ENDPOINT = 'https://api.anthropic.com/api/claude_code_shared_session_transcripts';
```

### 关键实现细节

#### 文件大小保护
```typescript
// 行 44-58
const { size } = await stat(transcriptPath);
if (size <= MAX_TRANSCRIPT_READ_BYTES) {
  rawTranscriptJsonl = await readFile(transcriptPath, 'utf-8');
} else {
  logForDebugging(`Skipping raw transcript read: file too large (${size} bytes)`, { level: 'warn' });
}
```

#### 敏感信息脱敏
```typescript
// 行 72
const content = redactSensitiveInfo(jsonStringify(data));
```

#### 认证流程
```typescript
// 行 74-79
await checkAndRefreshOAuthTokenIfNeeded();
const authResult = getAuthHeaders();
if (authResult.error) {
  return { success: false };
}
```

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `../Feedback.js` | `redactSensitiveInfo` 函数 |

### 外部依赖
| 文件 | 用途 |
|------|------|
| `axios` | HTTP 请求库 |
| `fs/promises` | 文件系统操作 |
| `../../types/message.js` | Message 类型定义 |
| `../../utils/auth.js` | OAuth 认证相关 |
| `../../utils/debug.js` | 调试日志 |
| `../../utils/errors.js` | 错误处理工具 |
| `../../utils/http.js` | HTTP 工具函数 |
| `../../utils/messages.js` | 消息标准化 |
| `../../utils/sessionStorage.js` | 会话存储工具 |
| `../../utils/slowOperations.js` | JSON 序列化 |

### 关键代码片段

#### 主函数入口
```typescript
// 行 29-112
export async function submitTranscriptShare(
  messages: Message[],
  trigger: TranscriptShareTrigger,
  appearanceId: string,
): Promise<TranscriptShareResult> {
  try {
    logForDebugging('Collecting transcript for sharing', { level: 'info' });
    
    // 数据收集
    const transcript = normalizeMessagesForAPI(messages);
    const agentIds = extractAgentIdsFromMessages(messages);
    const subagentTranscripts = await loadSubagentTranscripts(agentIds);
    
    // 原始转录读取（带大小保护）
    let rawTranscriptJsonl: string | undefined;
    try {
      const transcriptPath = getTranscriptPath();
      const { size } = await stat(transcriptPath);
      if (size <= MAX_TRANSCRIPT_READ_BYTES) {
        rawTranscriptJsonl = await readFile(transcriptPath, 'utf-8');
      }
    } catch {
      // File may not exist
    }
    
    // 组装并脱敏数据
    const data = { trigger, version: MACRO.VERSION, platform: process.platform, ... };
    const content = redactSensitiveInfo(jsonStringify(data));
    
    // 认证和传输
    await checkAndRefreshOAuthTokenIfNeeded();
    const authResult = getAuthHeaders();
    if (authResult.error) return { success: false };
    
    const response = await axios.post(API_ENDPOINT, { content, appearance_id: appearanceId }, {
      headers: { 'Content-Type': 'application/json', 'User-Agent': getUserAgent(), ...authResult.headers },
      timeout: 30000,
    });
    
    if (response.status === 200 || response.status === 201) {
      return { success: true, transcriptId: response.data?.transcript_id };
    }
    return { success: false };
  } catch (err) {
    logForDebugging(errorMessage(err), { level: 'error' });
    return { success: false };
  }
}
```

## 依赖与外部交互

### 输入依赖
1. **消息记录**: `messages` - 当前会话的完整消息数组
2. **触发类型**: `trigger` - 标识触发分享的原因
3. **展示标识**: `appearanceId` - 关联到特定的调查展示实例

### 输出交互
1. **API 调用**: POST 到 `https://api.anthropic.com/api/claude_code_shared_session_transcripts`
2. **调试日志**: 通过 `logForDebugging` 记录操作状态
3. **返回结果**: `TranscriptShareResult` 包含成功状态和转录 ID

### 依赖关系图
```
submitTranscriptShare.ts
  ├── axios (外部依赖)
  ├── fs/promises (Node.js 内置)
  ├── ../Feedback.js (redactSensitiveInfo)
  ├── ../../utils/auth.js (checkAndRefreshOAuthTokenIfNeeded, getAuthHeaders)
  ├── ../../utils/sessionStorage.js (extractAgentIdsFromMessages, getTranscriptPath, loadSubagentTranscripts, MAX_TRANSCRIPT_READ_BYTES)
  ├── ../../utils/messages.js (normalizeMessagesForAPI)
  ├── ../../utils/slowOperations.js (jsonStringify)
  └── ../../utils/debug.js (logForDebugging)
```

## 风险、边界与改进建议

### 已知风险

1. **同步文件大小检查**
   - 行 47: `stat(transcriptPath)` 是同步阻塞操作
   - 大文件系统或慢存储可能导致延迟

2. **内存使用风险**
   - 行 72: `jsonStringify(data)` 可能产生大字符串
   - 虽然原始转录有大小限制，但标准化后的 `transcript` 可能很大

3. **错误信息过于简略**
   - 行 77-79: 认证失败仅返回 `{ success: false }`，无具体原因
   - 行 105-111: 所有错误统一处理，可能掩盖不同类型的失败

4. **硬编码 API 端点**
   - 行 87-88: 端点 URL 硬编码
   - 环境切换或端点变更需要代码修改

### 边界情况

1. **文件不存在**: 原始转录文件可能不存在，被 try-catch 捕获并忽略
2. **文件过大**: 超过 `MAX_TRANSCRIPT_READ_BYTES` 时跳过读取，记录警告
3. **网络超时**: 30 秒超时后请求失败
4. **认证失效**: OAuth 令牌刷新失败导致分享失败
5. **空子代理转录**: `subagentTranscripts` 为空对象时不包含在请求中

### 改进建议

1. **流式处理大文件**
   ```typescript
   // 建议：使用流式读取处理大文件
   import { createReadStream } from 'fs';
   import { createGzip } from 'zlib';
   
   // 压缩后分块上传
   const stream = createReadStream(transcriptPath).pipe(createGzip());
   ```

2. **增强错误分类**
   ```typescript
   // 建议：区分不同类型的错误
   if (axios.isAxiosError(err)) {
     if (err.response?.status === 401) return { success: false, error: 'auth_failed' };
     if (err.code === 'ECONNABORTED') return { success: false, error: 'timeout' };
   }
   return { success: false, error: 'unknown' };
   ```

3. **配置化 API 端点**
   ```typescript
   // 建议：从环境变量或配置获取
   const API_ENDPOINT = process.env.CLAUDE_TRANSCRIPT_API || 
     'https://api.anthropic.com/api/claude_code_shared_session_transcripts';
   ```

4. **添加重试机制**
   ```typescript
   // 建议：对网络错误添加指数退避重试
   for (let attempt = 1; attempt <= 3; attempt++) {
     try {
       return await attemptUpload();
     } catch (err) {
       if (attempt === 3 || !isRetryableError(err)) throw err;
       await sleep(1000 * attempt);
     }
   }
   ```

5. **进度反馈**
   ```typescript
   // 建议：对于大文件上传，提供进度回调
   const response = await axios.post(url, data, {
     onUploadProgress: (progress) => {
       logForDebugging(`Upload progress: ${(progress.loaded / progress.total * 100).toFixed(1)}%`);
     }
   });
   ```

6. **数据压缩**
   ```typescript
   // 建议：上传前压缩数据以减少带宽
   import { gzip } from 'zlib';
   import { promisify } from 'util';
   
   const gzipAsync = promisify(gzip);
   const compressed = await gzipAsync(Buffer.from(content));
   // 添加 Content-Encoding: gzip 头
   ```

### 测试建议

1. **单元测试**:
   - 验证各种触发类型的正确处理
   - 验证文件大小限制行为
   - 验证敏感信息脱敏调用
   - 验证错误处理和返回格式

2. **集成测试**:
   - 模拟 API 成功/失败响应
   - 测试认证失败场景
   - 测试网络超时处理

3. **边界测试**:
   - 空消息数组
   - 超大文件（超过限制）
   - 不存在的转录文件
   - 特殊字符和 Unicode 内容
