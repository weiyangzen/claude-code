# sandboxTypes.ts 深度研究文档

## 文件元数据
- **路径**: `src/entrypoints/sandboxTypes.ts`
- **大小**: 5,735 bytes
- **类型**: TypeScript 类型定义文件

---

## 一、场景与职责

### 1.1 核心定位
`sandboxTypes.ts` 是 **Claude Code Agent SDK 的沙箱配置类型单一事实来源**，承担以下关键职责：

1. **沙箱配置类型定义**: 定义沙箱网络、文件系统和整体设置的 Zod schema
2. **跨模块共享**: 被 SDK 和设置验证系统共同导入
3. **运行时验证**: 提供可用于运行时验证的 Zod schema
4. **文档生成**: schema 中的 `.describe()` 为设置文档提供来源

### 1.2 使用场景

| 场景 | 说明 |
|------|------|
| **SDK 类型导出** | 通过 `sdk/coreTypes.ts` 暴露给 SDK 消费者 |
| **设置验证** | `utils/settings/types.ts` 导入用于设置 schema |
| **沙箱配置** | 沙箱管理器使用这些类型配置隔离环境 |
| **企业策略** | 托管设置使用这些类型定义沙箱策略 |

### 1.3 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    SDK 消费者                               │
│         (import { SandboxSettings } from ...)               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              src/entrypoints/sdk/coreTypes.ts               │
│              (re-exports from sandboxTypes.ts)              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              src/entrypoints/sandboxTypes.ts                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • SandboxNetworkConfigSchema                       │   │
│  │  • SandboxFilesystemConfigSchema                    │   │
│  │  • SandboxSettingsSchema                            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              设置验证系统                                   │
│              (utils/settings/types.ts)                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、功能点目的

### 2.1 沙箱网络配置

```typescript
export const SandboxNetworkConfigSchema = lazySchema(() =>
  z
    .object({
      allowedDomains: z.array(z.string()).optional(),
      allowManagedDomainsOnly: z.boolean().optional()
        .describe('When true (and set in managed settings), only allowedDomains and WebFetch(domain:...) allow rules from managed settings are respected...'),
      allowUnixSockets: z.array(z.string()).optional()
        .describe('macOS only: Unix socket paths to allow. Ignored on Linux...'),
      allowAllUnixSockets: z.boolean().optional()
        .describe('If true, allow all Unix sockets...'),
      allowLocalBinding: z.boolean().optional(),
      httpProxyPort: z.number().optional(),
      socksProxyPort: z.number().optional(),
    })
    .optional(),
)
```

**配置项说明**:

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `allowedDomains` | `string[]` | 允许的域名列表 |
| `allowManagedDomainsOnly` | `boolean` | 仅使用托管设置中的域名 |
| `allowUnixSockets` | `string[]` | 允许的 Unix socket 路径（macOS  only） |
| `allowAllUnixSockets` | `boolean` | 允许所有 Unix sockets |
| `allowLocalBinding` | `boolean` | 允许本地绑定 |
| `httpProxyPort` | `number` | HTTP 代理端口 |
| `socksProxyPort` | `number` | SOCKS 代理端口 |

### 2.2 沙箱文件系统配置

```typescript
export const SandboxFilesystemConfigSchema = lazySchema(() =>
  z
    .object({
      allowWrite: z.array(z.string()).optional()
        .describe('Additional paths to allow writing within the sandbox...'),
      denyWrite: z.array(z.string()).optional()
        .describe('Additional paths to deny writing within the sandbox...'),
      denyRead: z.array(z.string()).optional()
        .describe('Additional paths to deny reading within the sandbox...'),
      allowRead: z.array(z.string()).optional()
        .describe('Paths to re-allow reading within denyRead regions...'),
      allowManagedReadPathsOnly: z.boolean().optional()
        .describe('When true (set in managed settings), only allowRead paths from policySettings are used...'),
    })
    .optional(),
)
```

**配置项说明**:

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `allowWrite` | `string[]` | 额外允许写入的路径 |
| `denyWrite` | `string[]` | 额外拒绝写入的路径 |
| `denyRead` | `string[]` | 额外拒绝读取的路径 |
| `allowRead` | `string[]` | 在 denyRead 区域内重新允许读取的路径 |
| `allowManagedReadPathsOnly` | `boolean` | 仅使用托管设置中的读取路径 |

### 2.3 沙箱设置

```typescript
export const SandboxSettingsSchema = lazySchema(() =>
  z
    .object({
      enabled: z.boolean().optional(),
      failIfUnavailable: z.boolean().optional()
        .describe('Exit with an error at startup if sandbox.enabled is true but the sandbox cannot start...'),
      // enabledPlatforms: 通过 .passthrough() 读取的未文档化设置
      autoAllowBashIfSandboxed: z.boolean().optional(),
      allowUnsandboxedCommands: z.boolean().optional()
        .describe('Allow commands to run outside the sandbox via the dangerouslyDisableSandbox parameter...'),
      network: SandboxNetworkConfigSchema(),
      filesystem: SandboxFilesystemConfigSchema(),
      ignoreViolations: z.record(z.string(), z.array(z.string())).optional(),
      enableWeakerNestedSandbox: z.boolean().optional(),
      enableWeakerNetworkIsolation: z.boolean().optional()
        .describe('macOS only: Allow access to com.apple.trustd.agent in the sandbox...'),
      excludedCommands: z.array(z.string()).optional(),
      ripgrep: z.object({
        command: z.string(),
        args: z.array(z.string()).optional(),
      }).optional()
        .describe('Custom ripgrep configuration for bundled ripgrep support'),
    })
    .passthrough(),  // 允许未文档化的 enabledPlatforms
)
```

**配置项说明**:

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `enabled` | `boolean` | 是否启用沙箱 |
| `failIfUnavailable` | `boolean` | 沙箱不可用时是否退出 |
| `enabledPlatforms` | `string[]` | 允许沙箱的平台（未文档化） |
| `autoAllowBashIfSandboxed` | `boolean` | 沙箱化时自动允许 Bash |
| `allowUnsandboxedCommands` | `boolean` | 允许通过参数禁用沙箱 |
| `network` | `SandboxNetworkConfig` | 网络配置 |
| `filesystem` | `SandboxFilesystemConfig` | 文件系统配置 |
| `ignoreViolations` | `Record<string, string[]>` | 忽略特定违规 |
| `enableWeakerNestedSandbox` | `boolean` | 启用较弱的嵌套沙箱 |
| `enableWeakerNetworkIsolation` | `boolean` | 启用较弱的网络隔离（macOS） |
| `excludedCommands` | `string[]` | 排除的命令 |
| `ripgrep` | `{command, args?}` | 自定义 ripgrep 配置 |

---

## 三、具体技术实现

### 3.1 延迟 Schema 创建

```typescript
import { lazySchema } from '../utils/lazySchema.js'

export const SandboxNetworkConfigSchema = lazySchema(() =>
  z.object({...})
)
```

**技术要点**:
- 使用 `lazySchema` 包装器延迟 schema 创建
- 避免在导入时立即创建 schema，减少启动开销
- 允许循环引用和复杂依赖

### 3.2 类型推断

```typescript
// 从 schema 推断类型
export type SandboxSettings = z.infer<ReturnType<typeof SandboxSettingsSchema>>
export type SandboxNetworkConfig = NonNullable<
  z.infer<ReturnType<typeof SandboxNetworkConfigSchema>>
>
export type SandboxFilesystemConfig = NonNullable<
  z.infer<ReturnType<typeof SandboxFilesystemConfigSchema>>
>
export type SandboxIgnoreViolations = NonNullable<
  SandboxSettings['ignoreViolations']
>
```

**技术要点**:
- 使用 `z.infer` 从 Zod schema 推断 TypeScript 类型
- `ReturnType` 处理延迟 schema 函数
- `NonNullable` 确保可选 schema 的非空类型

### 3.3 Passthrough 模式

```typescript
export const SandboxSettingsSchema = lazySchema(() =>
  z
    .object({...})
    .passthrough(),
)
```

**用途**:
- 允许 schema 中未定义的属性通过验证
- 用于 `enabledPlatforms` 等未文档化设置
- 支持前向兼容（新属性不会导致验证失败）

### 3.4 平台特定配置

**macOS 特定**:
- `allowUnixSockets`: Unix socket 路径过滤
- `enableWeakerNetworkIsolation`: 允许访问 `com.apple.trustd.agent`

**Linux 限制**:
- `allowUnixSockets`: 被忽略（seccomp 无法按路径过滤）

---

## 四、关键代码路径与文件引用

### 4.1 导入的模块

| 模块路径 | 用途 |
|----------|------|
| `zod/v4` | Zod schema 库 |
| `../utils/lazySchema.js` | 延迟 schema 创建工具 |

### 4.2 被依赖关系

```
sandboxTypes.ts
├── 被导入
│   ├── src/entrypoints/sdk/coreTypes.ts (re-exports)
│   ├── src/utils/settings/types.ts (SettingsSchema 中的 sandbox 字段)
│   └── ... (其他使用沙箱类型的模块)
└── 使用方
    ├── SDK 消费者 (通过 coreTypes.ts)
    ├── 设置验证 (settings/types.ts)
    └── 沙箱管理器 (sandbox-adapter.ts)
```

### 4.3 核心类型使用

| 类型 | 使用位置 | 用途 |
|------|----------|------|
| `SandboxSettings` | `utils/settings/types.ts` | 设置 schema 中的 `sandbox` 字段 |
| `SandboxNetworkConfig` | `utils/settings/types.ts` | 网络配置子类型 |
| `SandboxFilesystemConfig` | `utils/settings/types.ts` | 文件系统配置子类型 |
| `SandboxIgnoreViolations` | 沙箱实现 | 违规忽略配置 |

---

## 五、依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 |
|------|------|
| `zod/v4` | Schema 定义和验证 |

### 5.2 内部依赖

| 依赖路径 | 说明 |
|----------|------|
| `../utils/lazySchema.js` | 延迟 schema 创建工具 |

### 5.3 类型导出链

```
sandboxTypes.ts
    │
    ├──(re-exports)──► sdk/coreTypes.ts
    │                    │
    │                    └──(re-exports)──► agentSdkTypes.ts
    │                                         │
    │                                         └──(exports)──► SDK 消费者
    │
    └──(imports)──► utils/settings/types.ts
                       │
                       └──(uses)──► SettingsSchema
```

---

## 六、风险、边界与改进建议

### 6.1 当前风险

| 风险点 | 严重程度 | 说明 |
|--------|----------|------|
| **Passthrough 风险** | 中 | `.passthrough()` 允许任意属性，可能隐藏拼写错误 |
| **平台差异** | 中 | macOS 和 Linux 的配置行为不同，可能导致混淆 |
| **未文档化设置** | 低 | `enabledPlatforms` 未文档化，依赖 `.passthrough()` |
| **安全降级选项** | 中 | `enableWeakerNetworkIsolation` 降低安全性 |

### 6.2 边界条件

1. **平台限制**
   - `allowUnixSockets` 在 Linux 上被忽略
   - `enableWeakerNetworkIsolation` 仅在 macOS 上有效

2. **设置优先级**
   - `allowManagedDomainsOnly`: 仅当在托管设置中设置时生效
   - `allowManagedReadPathsOnly`: 仅当在托管设置中设置时生效

3. **安全边界**
   - `allowUnsandboxedCommands`: 默认 `true`，允许通过 `dangerouslyDisableSandbox` 禁用沙箱
   - `enableWeakerNetworkIsolation`: 打开潜在的数据泄露通道

4. **Schema 验证**
   - `.passthrough()` 允许未知属性
   - 拼写错误的属性名不会导致验证错误

### 6.3 改进建议

1. **平台检测增强**
   ```typescript
   // 建议：添加平台特定的 schema 验证
   .refine((data) => {
     if (process.platform !== 'darwin' && data.allowUnixSockets) {
       console.warn('allowUnixSockets is ignored on Linux')
     }
     return true
   })
   ```

2. **文档化未文档化设置**
   ```typescript
   // 建议：将 enabledPlatforms 添加到 schema 中
   enabledPlatforms: z.array(z.enum(['macos', 'linux', 'win32'])).optional()
     .describe('Restrict sandboxing to specific platforms...'),
   ```

3. **安全警告增强**
   ```typescript
   // 建议：为安全降级选项添加更详细的警告
   enableWeakerNetworkIsolation: z.boolean().optional()
     .describe('⚠️ SECURITY WARNING: Allow access to com.apple.trustd.agent...'),
   ```

4. **严格模式选项**
   ```typescript
   // 建议：提供严格模式 schema（无 passthrough）
   export const StrictSandboxSettingsSchema = lazySchema(() =>
     z.object({...}) // 无 .passthrough()
   )
   ```

5. **配置验证增强**
   ```typescript
   // 建议：添加交叉字段验证
   .refine((data) => {
     if (data.allowAllUnixSockets && data.allowUnixSockets?.length > 0) {
       return false // 互斥选项
     }
     return true
   }, { message: 'allowAllUnixSockets and allowUnixSockets are mutually exclusive' })
   ```

### 6.4 相关配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `enabled` | `undefined` | 沙箱未明确启用 |
| `failIfUnavailable` | `false` | 沙箱失败时显示警告而非退出 |
| `allowUnsandboxedCommands` | `true` | 允许通过参数禁用沙箱 |
| `enableWeakerNetworkIsolation` | `false` | 默认禁用弱网络隔离 |

---

## 七、总结

`sandboxTypes.ts` 是 Claude Code 沙箱系统的**类型基石**，设计哲学是：

1. **单一事实来源**: 所有沙箱相关类型集中定义
2. **延迟加载**: 使用 `lazySchema` 减少启动开销
3. **灵活验证**: `.passthrough()` 支持前向兼容和未文档化设置
4. **平台感知**: 明确区分 macOS 和 Linux 的配置差异
5. **企业就绪**: 支持托管设置策略（`allowManaged*` 选项）

文件的关键成功因素是**类型安全与灵活性的平衡**——既提供严格的 TypeScript 类型，又通过 Zod schema 允许运行时验证和灵活配置。

使用注意事项：
- 注意平台特定的配置行为差异
- 谨慎使用安全降级选项
- 依赖 `.passthrough()` 的未文档化设置可能在将来版本中被正式纳入 schema
