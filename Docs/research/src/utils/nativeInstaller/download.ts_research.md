# 研究文档：src/utils/nativeInstaller/download.ts

## 场景与职责

`download.ts` 是 Claude Code 原生安装器（Native Installer）的**二进制下载模块**，负责从远端源获取平台特定的原生二进制文件。它支撑整个原生安装流程中的"获取"环节：根据用户类型（`ant` 内部用户 vs `external` 外部用户）和版本通道（`stable`/`latest`），选择正确的下载源、解析最新版本号、下载并校验二进制完整性。

该模块是 `installer.ts` 中 `performVersionUpdate` 的下游依赖，仅在确认需要安装/重装某版本时被调用。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `getLatestVersion*` | 解析用户指定的通道或版本字符串，获取实际可用的最新版本号 |
| `downloadVersionFromArtifactory` | 为内部用户通过 Artifactory NPM 包机制下载平台特定包，利用 `npm ci` 做完整性校验 |
| `downloadVersionFromBinaryRepo` | 为外部用户从 GCS 二进制仓库下载原生二进制，基于 `manifest.json` 做 SHA-256 校验 |
| `downloadAndVerifyBinary` | 通用二进制下载逻辑，带**停滞超时检测**（stall detection）和重试机制 |
| `downloadVersion` | 统一入口，根据环境路由到 Artifactory 或 GCS，并支持测试版本特殊通道 |

## 具体技术实现

### 1. 版本源路由 (`getLatestVersion`)

```ts
// 伪代码逻辑
if 输入是 semver 格式:
    if 99.99.x 且非测试模式 -> throw
    return 输入版本
if 通道不是 stable/latest -> throw
if USER_TYPE === 'ant':
    return getLatestVersionFromArtifactory(tag)
else:
    return getLatestVersionFromBinaryRepo(channel, GCS_BUCKET_URL)
```

- **Artifactory 源**：调用本地 `npm view ${MACRO.NATIVE_PACKAGE_URL}@${tag} version --registry ${ARTIFACTORY_REGISTRY_URL}`，30 秒超时。
- **GCS 源**：通过 `axios.get` 读取 `${GCS_BUCKET_URL}/${channel}` 的纯文本版本号，30 秒超时。
- 两个路径均埋点 `tengu_version_check_success` / `tengu_version_check_failure`。

### 2. Artifactory 下载 (`downloadVersionFromArtifactory`)

此路径不直接下载二进制，而是构造一个**临时隔离的 npm 项目**来利用 npm 的完整性校验：

1. 清理 `stagingPath`。
2. 通过 `npm view ${platformPackageName}@${version} dist.integrity` 获取平台包的 `integrity`（SRI hash）。
3. 在 `stagingPath` 下生成 `package.json` 和手工构造的 `package-lock.json`（lockfileVersion: 3），将平台包声明为 `optionalDependencies` 并写入 `integrity`。
4. 执行 `npm ci --prefer-online --registry ${ARTIFACTORY_REGISTRY_URL}`（60 秒超时），由 npm 负责下载并校验完整性。
5. 成功后二进制位于 `stagingPath/node_modules/@anthropic-ai/claude-cli-native-${platform}/cli`。

### 3. GCS 二进制下载 (`downloadVersionFromBinaryRepo`)

1. 清理 `stagingPath`。
2. `axios.get` 拉取 `${baseUrl}/${version}/manifest.json`，获取各平台信息。
3. 提取当前平台的 `checksum`（SHA-256）。
4. 调用 `downloadAndVerifyBinary` 下载 `${baseUrl}/${version}/${platform}/${binaryName}`。

### 4. 停滞检测与重试 (`downloadAndVerifyBinary`)

这是核心的网络鲁棒性逻辑：

- **Stall Timeout**：默认 60 秒（可通过 `CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING` 覆盖）。如果连续 60 秒没有收到任何数据块，则通过 `AbortController` 中止请求。
- **Total Timeout**：axios 层设置 5 分钟。
- **Retry**：仅在 **stall timeout** 触发时重试，最多 3 次；其他错误（HTTP 错误、校验和不匹配）直接抛出，不重试。
- **校验**：下载完成后用 Node.js `crypto.createHash('sha256')` 计算摘要，与预期比对；通过后 `writeFile` + `chmod 0o755`。

```ts
const response = await axios.get(binaryUrl, {
  timeout: 5 * 60000,
  responseType: 'arraybuffer',
  signal: controller.signal,
  onDownloadProgress: () => resetStallTimer(),
})
```

### 5. 测试版本特殊通道

当 `feature('ALLOW_TEST_VERSIONS')` 为 true 且版本号匹配 `^99.99.` 时：
- 调用 `gcloud auth print-access-token` 获取 GCP 访问令牌。
- 使用 Bearer Token 从私有 sentinel bucket (`claude-code-ci-sentinel`) 下载。
- 该分支在发布构建中会被 DCE（Dead Code Elimination）完全剔除。

## 关键代码路径与文件引用

| 路径 | 角色 |
|------|------|
| `src/utils/nativeInstaller/download.ts` | 本文件，提供下载与版本查询 API |
| `src/utils/nativeInstaller/installer.ts` | 调用方：`performVersionUpdate` → `downloadVersion` |
| `src/utils/nativeInstaller/pidLock.ts` | 同目录，负责安装过程的并发锁（与 download 无直接 import，但协同工作） |
| `src/utils/fsOperations.ts` | `getFsImplementation()` 提供可替换的 fs 抽象 |
| `src/utils/execFileNoThrow.ts` | `execFileNoThrowWithCwd` 用于调用 npm/gcloud |
| `src/services/analytics/index.ts` | `logEvent` 埋点 |

## 依赖与外部交互

### 外部服务

- **Artifactory Registry**：`https://artifactory.infra.ant.dev/artifactory/api/npm/npm-all/`
- **GCS Public Bucket**：`https://storage.googleapis.com/claude-code-dist-86c565f3-f756-42ad-8dfa-d59b1c096819/claude-code-releases`
- **GCS CI Sentinel Bucket**：`claude-code-ci-sentinel`（仅限测试）

### 环境变量

| 变量 | 作用 |
|------|------|
| `USER_TYPE` | 决定使用 Artifactory (`ant`) 还是 GCS (`external`) |
| `CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING` | 覆盖默认 stall timeout |

### NPM 包

- `axios`：HTTP 客户端
- `bun:bundle` 的 `feature()`：编译期特性开关
- Node.js 内置 `crypto`：SHA-256 校验

## 风险、边界与改进建议

### 风险

1. **npm 作为下载工具的风险**：`downloadVersionFromArtifactory` 依赖本地 `npm` 可执行文件。如果用户环境的 npm 被篡改、版本过旧或配置异常（如 `.npmrc` 代理），可能导致下载失败或行为不可预期。虽然使用了 `--registry` 参数，但全局 npm 配置仍可能产生副作用。
2. **Stall Timeout 的误杀**：在网络极慢但仍有进度的场景下，60 秒 stall timeout 可能过于激进，导致大文件下载反复中断。
3. **GCS 的公网可达性**：外部用户完全依赖 GCS 的公网访问。在某些网络受限地区或企业防火墙后，GCS 可能被屏蔽，导致原生安装器对外部用户不可用。
4. **测试版本分支的安全隐患**：`ALLOW_TEST_VERSIONS` 分支引入了 `gcloud` 调用和私有 bucket URL。虽然 DCE 会在发布构建中移除，但如果构建系统配置错误，可能残留内部测试逻辑到生产构建。

### 边界

- **无流式写入**：`downloadAndVerifyBinary` 使用 `responseType: 'arraybuffer'`，将整个二进制加载到内存后再写入磁盘。对于数百 MB 的二进制，这会一次性占用大量堆内存（在 issue 历史中曾出现 `arrayBuffers climb to 91GB` 的极端情况）。
- **仅支持 `stable`/`latest` 通道**：自定义通道不被接受。
- **Windows 与 Unix 的二进制命名**：由 `installer.ts` 的 `getBinaryName` 决定，download.ts 本身不处理 `.exe` 扩展名逻辑。

### 改进建议

1. **引入流式下载**：将 `arraybuffer` 改为 `stream` 或 `pipe` 到文件系统，边下载边写，显著降低内存峰值。
2. **渐进式 stall timeout**：根据文件大小或历史下载速度动态调整 stall timeout，而非固定 60 秒。
3. **npm 下载的沙盒化**：考虑使用 `npm` 的 `--prefix` 和隔离环境变量进一步减少用户本地 npm 配置的干扰，或直接用 `curl`/`axios` 下载 tarball 并手动校验 integrity。
4. **多 CDN/镜像 fallback**：为 GCS 增加备用源（如 CloudFront、区域镜像），提升外部用户在全球不同网络环境下的可用性。
5. **更细粒度的重试策略**：当前仅对 stall timeout 重试；可对 5xx 网络错误也实施有限重试，并引入指数退避。
