# src/utils/aws.ts 深入研究

## 场景与职责

`aws.ts` 是 Claude Code 与 AWS 生态交互的基础工具层，职责高度聚焦：
- **类型定义**：声明 AWS STS 临时凭证结构（`AwsCredentials`、`AwsStsOutput`）。
- **凭证有效性校验**：提供类型守卫 `isValidAwsStsOutput`，确保 `assume-role` / `get-session-token` 输出包含必需的 `AccessKeyId`、`SecretAccessKey`、`SessionToken`。
- **错误识别**：`isAwsCredentialsProviderError` 用于区分 AWS SDK 的凭证提供链错误。
- **身份探测**：`checkStsCallerIdentity` 调用 STS `GetCallerIdentity` 验证当前凭证是否有效。
- **缓存刷新**：`clearAwsIniCache` 强制刷新 `fromIni` 缓存，使 `~/.aws/credentials` 的变更立即生效。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `isValidAwsStsOutput` | 在解析 AWS CLI JSON 输出或 SDK 响应时做运行时类型校验，防止不完整凭证向下传递 |
| `isAwsCredentialsProviderError` | 在重试/降级逻辑中识别凭证类错误，避免对所有 APIError 一视同仁 |
| `checkStsCallerIdentity` | 登录/配置流程中快速验证 AWS 凭证是否可用 |
| `clearAwsIniCache` | 用户修改 `~/.aws/credentials` 后无需重启进程即可使新凭证生效 |

## 具体技术实现

### 动态导入策略
- `checkStsCallerIdentity` 与 `clearAwsIniCache` 均采用 `await import(...)` **动态加载** `@aws-sdk/client-sts` 与 `@aws-sdk/credential-providers`。
- 这样做的好处是：当用户不使用 AWS Bedrock 时，避免将庞大的 AWS SDK 打包进启动依赖图。

### 类型守卫细节
```ts
function isValidAwsStsOutput(obj: unknown): obj is AwsStsOutput {
  // 1. 必须是对象且非 null
  // 2. Credentials 字段必须是对象
  // 3. AccessKeyId / SecretAccessKey / SessionToken 必须是长度 > 0 的字符串
}
```
- 显式检查 `.length > 0`，排除空字符串凭证。

### 缓存刷新机制
```ts
const iniProvider = fromIni({ ignoreCache: true })
await iniProvider()
```
- `ignoreCache: true` 让 `fromIni` 重新读取磁盘文件。
- 调用 `await iniProvider()` 触发一次实际读取，从而更新 AWS SDK 内部全局文件缓存。

## 关键代码路径与文件引用

```
src/utils/auth.ts
  └── checkStsCallerIdentity, clearAwsIniCache, isValidAwsStsOutput
      [AWS 认证流程：assume-role、凭证刷新、Bedrock 初始化]

src/services/api/withRetry.ts
  └── isAwsCredentialsProviderError
      [API 重试层：识别凭证错误后触发凭证缓存清理]
```

### 依赖模块
- `@aws-sdk/client-sts` — `STSClient`, `GetCallerIdentityCommand`
- `@aws-sdk/credential-providers` — `fromIni`
- `src/utils/debug.js` — `logForDebugging`

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| AWS STS API | `@aws-sdk/client-sts` | 动态导入，验证 CallerIdentity |
| AWS 凭证文件 | `@aws-sdk/credential-providers/fromIni` | 动态导入，刷新 `~/.aws/credentials` 缓存 |
| 本地调试日志 | `logForDebugging` | 记录缓存清理动作与失败（静默忽略） |

## 风险、边界与改进建议

### 风险
1. **动态导入失败未显式处理**：`checkStsCallerIdentity` 与 `clearAwsIniCache` 未在函数内部包裹 `try/catch`（调用方负责捕获），若 AWS SDK 未安装会抛出未处理异常。
2. **`isAwsCredentialsProviderError` 类型断言过宽**：仅检查 `name === 'CredentialsProviderError'`，若未来 AWS SDK 更改错误类名或层次结构，可能漏判。
3. **`clearAwsIniCache` 静默吞错**：`catch (_error)` 不返回任何状态，调用方无法感知缓存刷新是否成功。

### 边界
- `isValidAwsStsOutput` 不校验 `Expiration` 字段的存在性或格式，仅确保核心凭证三要素存在。
- `clearAwsIniCache` 对无 AWS 凭证配置的用户也会打印“预期中的失败”日志，属于正常行为。

### 改进建议
1. **统一 AWS SDK 错误识别**：考虑引入 `@smithy/property-provider` 的 `CredentialsProviderError` 类型进行 `instanceof` 校验，而非仅比较 `name` 字符串。
2. **暴露缓存刷新结果**：让 `clearAwsIniCache` 返回 `boolean` 或 `Result` 类型，便于调用方决定是否重试。
3. **增加 Expiration 校验**：在 `isValidAwsStsOutput` 中可选检查 `Expiration` 是否为合法 ISO 日期，提前发现过期凭证。
