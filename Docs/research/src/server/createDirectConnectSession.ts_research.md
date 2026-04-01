# createDirectConnectSession.ts 深度研究

## 场景与职责

本模块是 Claude Code CLI 的**直连服务器会话创建器**，负责在 `claude connect` 或 `claude open cc://` 命令执行时，与远程 Claude 服务器建立初始会话连接。它是 Direct Connect 功能的客户端入口点，将本地 CLI 实例连接到自托管或远程的 Claude 服务器实例。

**核心场景：**
1. 用户执行 `claude connect <server-url>` 连接到自己的 Claude 服务器
2. 用户通过 `cc://` 或 `cc+unix://` URL 协议直接启动连接
3. Headless 模式 (`-p`) 下通过 `claude open cc://` 进行非交互式直连

## 功能点目的

### 1. 会话创建 (createDirectConnectSession)
向直连服务器的 `/sessions` 端点发送 HTTP POST 请求，创建新会话并获取连接配置。

**输入参数：**
- `serverUrl`: 服务器基础 URL
- `authToken?`: 可选的 Bearer Token 认证
- `cwd`: 工作目录（传递给服务器）
- `dangerouslySkipPermissions?`: 是否跳过权限检查（危险标志）

**输出结果：**
- `config`: `DirectConnectConfig` 包含 `serverUrl`, `sessionId`, `wsUrl`, `authToken`
- `workDir?`: 服务器返回的推荐工作目录

### 2. 错误处理 (DirectConnectError)
定义专门的错误类，区分以下失败场景：
- 网络连接失败（fetch 抛出异常）
- HTTP 错误响应（非 2xx 状态码）
- 响应格式无效（Zod 验证失败）

## 具体技术实现

### 关键流程

```
┌─────────────────────────────────────────────────────────────┐
│  createDirectConnectSession                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. 构建请求头                                          │  │
│  │    - Content-Type: application/json                    │  │
│  │    - Authorization: Bearer <authToken> (if provided)   │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ 2. 发送 POST 请求到 ${serverUrl}/sessions              │  │
│  │    Body: { cwd, dangerously_skip_permissions? }        │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ 3. 错误处理                                            │  │
│  │    - fetch 异常 → DirectConnectError(网络错误)         │  │
│  │    - !resp.ok → DirectConnectError(HTTP错误)           │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ 4. 响应验证                                            │  │
│  │    - 使用 connectResponseSchema (Zod) 验证             │  │
│  │    - 验证失败 → DirectConnectError(格式错误)           │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ 5. 返回 DirectConnectConfig                            │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 数据结构

**请求体格式：**
```typescript
{
  cwd: string
  dangerously_skip_permissions?: boolean  // 蛇形命名，服务器协议要求
}
```

**响应体格式（Zod 验证）：**
```typescript
{
  session_id: string   // 服务器分配的唯一会话ID
  ws_url: string       // WebSocket 连接地址
  work_dir?: string    // 可选的工作目录
}
```

### 协议细节

- **传输协议**: HTTP POST + WebSocket 升级
- **认证方式**: Bearer Token（可选）
- **内容编码**: JSON
- **命名约定**: 请求体使用蛇形命名（`dangerously_skip_permissions`）以匹配服务器协议

## 关键代码路径与文件引用

### 本文件关键代码

| 行号 | 代码 | 说明 |
|------|------|------|
| 26-39 | `createDirectConnectSession` 函数定义 | 主入口，异步创建会话 |
| 40-45 | 请求头构建 | 条件添加 Authorization 头 |
| 48-58 | fetch POST 请求 | 发送会话创建请求 |
| 59-63 | 网络错误处理 | 捕获 fetch 异常 |
| 65-69 | HTTP 错误处理 | 检查 resp.ok |
| 71-76 | 响应验证 | Zod schema 验证 |
| 79-87 | 返回配置 | 组装 DirectConnectConfig |

### 依赖文件

| 文件路径 | 导入内容 | 用途 |
|----------|----------|------|
| `../utils/errors.js` | `errorMessage` | 错误消息提取 |
| `../utils/slowOperations.js` | `jsonStringify` | JSON 序列化（带性能监控）|
| `./directConnectManager.js` | `DirectConnectConfig` (type) | 配置类型定义 |
| `./types.js` | `connectResponseSchema` | Zod 响应验证 schema |

### 调用方

| 文件路径 | 调用方式 | 场景 |
|----------|----------|------|
| `src/main.tsx` | `import { createDirectConnectSession }` | 处理 `cc://` URL 和 `--print` 模式 |
| `src/main.tsx` | 用于初始化直连会话 | REPL 启动时传入 `directConnectConfig` |

## 依赖与外部交互

### 运行时依赖

1. **全局 fetch**: 使用原生 fetch API（Bun/Node 18+）
2. **Zod 验证**: 通过 `connectResponseSchema` 验证响应

### 服务器端交互

```
┌──────────────┐      POST /sessions      ┌──────────────────┐
│   CLI 客户端  │ ───────────────────────→ │  Claude 直连服务器 │
│  (本模块)     │  { cwd, authToken }      │                  │
│              │ ←─────────────────────── │  创建会话并返回    │
│              │  { session_id, ws_url }   │  WebSocket 地址   │
└──────────────┘                          └──────────────────┘
```

### 与 directConnectManager 的关系

- `createDirectConnectSession.ts` 负责**初始 HTTP 握手**，获取会话凭证
- `directConnectManager.ts` 负责**WebSocket 连接管理**和消息流转
- 两者通过 `DirectConnectConfig` 类型衔接

## 风险、边界与改进建议

### 已知风险

1. **认证令牌暴露**
   - 风险：`authToken` 通过 URL 或命令行传递时可能泄露到 shell 历史
   - 缓解：建议通过环境变量或配置文件传递敏感凭证

2. **权限跳过标志**
   - 风险：`dangerouslySkipPermissions` 会完全禁用工具权限检查
   - 缓解：明确命名为 "dangerously"，需要显式启用

3. **网络超时**
   - 风险：当前实现无显式超时控制，依赖平台默认行为
   - 影响：在网络不稳定时可能挂起

### 边界情况

| 场景 | 行为 |
|------|------|
| 服务器返回 401 | 抛出 DirectConnectError，提示认证失败 |
| 服务器返回 5xx | 抛出 DirectConnectError，包含状态码和状态文本 |
| 响应 JSON 无效 | Zod 验证失败，抛出格式错误 |
| 响应缺少字段 | Zod 验证失败，明确提示缺失字段 |
| fetch 抛出异常 | 捕获并包装为 DirectConnectError |

### 改进建议

1. **添加超时控制**
   ```typescript
   // 建议添加 AbortController 控制超时
   const controller = new AbortController()
   const timeout = setTimeout(() => controller.abort(), 30000)
   resp = await fetch(url, { ...options, signal: controller.signal })
   ```

2. **重试机制**
   - 对瞬态网络错误（ECONNREFUSED、ETIMEDOUT）实现指数退避重试

3. **更详细的错误分类**
   - 区分 DNS 解析失败、连接拒绝、TLS 错误等不同网络错误类型

4. **响应日志**
   - 在 debug 模式下记录请求/响应详情（注意脱敏 authToken）

5. **Schema 版本协商**
   - 未来协议演进时，考虑添加 API 版本协商机制

### 测试建议

- 单元测试：mock fetch 验证各种 HTTP 状态码处理
- 集成测试：使用本地测试服务器验证完整流程
- 边界测试：验证网络超时、无效 JSON、缺失字段等场景
