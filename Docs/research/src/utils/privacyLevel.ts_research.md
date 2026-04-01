# privacyLevel.ts 研究文档

## 场景与职责

`privacyLevel.ts` 是 Claude Code 中负责**隐私级别管理**的模块。它根据环境变量控制非必要网络流量和遥测的开关，满足用户对隐私和数据收集的不同需求。

**核心职责：**
1. **隐私级别定义**：定义三个隐私级别（default、no-telemetry、essential-traffic）
2. **级别解析**：根据环境变量确定当前隐私级别
3. **流量控制**：提供便捷函数检查是否应发送特定类型的流量
4. **原因追溯**：获取限制流量的环境变量名称（用于用户提示）

**业务场景：**
- 用户希望禁用所有遥测
- 企业环境需要限制外部网络连接
- 调试时需要了解为何某些功能不可用

---

## 功能点目的

### 1. 隐私级别定义

三个按限制程度排序的级别：

| 级别 | 说明 | 影响 |
|------|------|------|
| `default` | 默认级别 | 所有功能启用 |
| `no-telemetry` | 禁用遥测 | 分析/遥测禁用（Datadog、1P 事件、反馈调查） |
| `essential-traffic` | 仅必要流量 | 所有非必要网络流量禁用（遥测 + 自动更新、grove、发布说明、模型能力等） |

### 2. 级别解析 (`getPrivacyLevel`)

根据环境变量确定隐私级别：

```typescript
export function getPrivacyLevel(): PrivacyLevel
```

**优先级（从高到低）：**
1. `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` → `essential-traffic`
2. `DISABLE_TELEMETRY` → `no-telemetry`
3. 默认 → `default`

### 3. 流量控制检查

**仅必要流量 (`isEssentialTrafficOnly`)**
```typescript
export function isEssentialTrafficOnly(): boolean
```
- 等同于 `getPrivacyLevel() === 'essential-traffic'`
- 替代旧的 `process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 检查

**遥测禁用 (`isTelemetryDisabled`)**
```typescript
export function isTelemetryDisabled(): boolean
```
- `no-telemetry` 和 `essential-traffic` 级别都返回 `true`
- 用于判断是否应发送分析事件

### 4. 原因追溯 (`getEssentialTrafficOnlyReason`)

获取导致 `essential-traffic` 限制的环境变量名称：

```typescript
export function getEssentialTrafficOnlyReason(): string | null
```

**用途：**
- 在用户界面中显示 "unset X to re-enable" 提示
- 帮助用户理解限制来源

---

## 具体技术实现

### 数据结构

```typescript
type PrivacyLevel = 'default' | 'no-telemetry' | 'essential-traffic'
```

### 关键流程

**隐私级别解析流程：**
1. 检查 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
2. 如果存在，返回 `'essential-traffic'`
3. 检查 `DISABLE_TELEMETRY`
4. 如果存在，返回 `'no-telemetry'`
5. 默认返回 `'default'`

**遥测禁用检查流程：**
1. 获取当前隐私级别
2. 检查是否不等于 `'default'`
3. 返回布尔值

### 关键代码路径

| 函数 | 路径 | 说明 |
|------|------|------|
| `getPrivacyLevel` | L20-28 | 隐私级别解析 |
| `isEssentialTrafficOnly` | L34-36 | 仅必要流量检查 |
| `isTelemetryDisabled` | L42-44 | 遥测禁用检查 |
| `getEssentialTrafficOnlyReason` | L50-55 | 原因追溯 |

### 代码实现细节

**完整实现：**
```typescript
export function getPrivacyLevel(): PrivacyLevel {
  if (process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC) {
    return 'essential-traffic'
  }
  if (process.env.DISABLE_TELEMETRY) {
    return 'no-telemetry'
  }
  return 'default'
}

export function isEssentialTrafficOnly(): boolean {
  return getPrivacyLevel() === 'essential-traffic'
}

export function isTelemetryDisabled(): boolean {
  return getPrivacyLevel() !== 'default'
}

export function getEssentialTrafficOnlyReason(): string | null {
  if (process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC) {
    return 'CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC'
  }
  return null
}
```

---

## 依赖与外部交互

### 导入依赖

```typescript
// 无外部导入
```

### 外部调用方

| 调用方文件 | 用途 |
|-----------|------|
| `src/bridge/trustedDevice.ts` | 受信任设备检测 |
| `src/services/policyLimits/index.ts` | 策略限制 |
| `src/services/claudeAiLimits.ts` | Claude AI 限制 |
| `src/services/api/bootstrap.ts` | API 引导 |
| `src/services/api/metricsOptOut.ts` | 指标退出 |
| `src/services/api/referral.ts` | 推荐功能 |
| `src/services/api/grove.ts` | Grove 服务 |
| `src/services/api/overageCreditGrant.ts` | 超额信用授予 |
| `src/services/analytics/config.ts` | 分析配置 |
| `src/components/Feedback.tsx` | 反馈组件 |
| `src/commands/feedback/index.ts` | 反馈命令 |
| `src/utils/fastMode.ts` | Fast Mode（跳过非必要网络请求） |
| `src/utils/log.ts` | 日志（遥测控制） |
| `src/utils/config.ts` | 配置（遥测控制） |
| `src/utils/model/modelCapabilities.ts` | 模型能力（缓存控制） |
| `src/utils/releaseNotes.ts` | 发布说明（网络请求控制） |

### 环境变量

| 变量名 | 说明 |
|--------|------|
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 禁用所有非必要流量（最高优先级） |
| `DISABLE_TELEMETRY` | 禁用遥测（标准变量名，被许多工具识别） |

---

## 风险、边界与改进建议

### 潜在风险

1. **环境变量冲突**
   - `DISABLE_TELEMETRY` 是通用变量名
   - 可能被其他工具设置，意外影响 Claude Code

2. **级别定义不清晰**
   - `no-telemetry` 和 `essential-traffic` 的区别可能让用户困惑
   - 文档需要清晰解释各级别的具体影响

3. **功能降级不明显**
   - 某些功能被禁用时，用户可能不知道原因
   - 需要更好的用户提示

4. **动态变更不支持**
   - 环境变量在运行时修改不会立即生效
   - 需要重启进程才能应用新设置

### 边界情况

1. **空字符串值**
   - `DISABLE_TELEMETRY=""` 会被视为设置，启用 `no-telemetry`
   - 与某些 shell 的 `unset` 行为不同

2. **多个变量设置**
   - `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 优先
   - 即使同时设置了 `DISABLE_TELEMETRY`，也是 `essential-traffic`

3. **大小写敏感**
   - 环境变量名大小写敏感
   - `disable_telemetry` 不会被识别

### 改进建议

1. **值解析增强**
   ```typescript
   import { isEnvTruthy } from './envUtils.js'
   
   export function getPrivacyLevel(): PrivacyLevel {
     if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC)) {
       return 'essential-traffic'
     }
     if (isEnvTruthy(process.env.DISABLE_TELEMETRY)) {
       return 'no-telemetry'
     }
     return 'default'
   }
   ```

2. **配置热重载**
   ```typescript
   let cachedLevel: PrivacyLevel | undefined
   
   export function getPrivacyLevel(): PrivacyLevel {
     if (cachedLevel === undefined) {
       cachedLevel = calculatePrivacyLevel()
     }
     return cachedLevel
   }
   
   export function reloadPrivacyLevel(): void {
     cachedLevel = undefined
   }
   ```

3. **详细影响说明**
   ```typescript
   export function getPrivacyLevelImpact(level: PrivacyLevel): string[] {
     const impacts: Record<PrivacyLevel, string[]> = {
       default: ['All features enabled'],
       'no-telemetry': [
         'Analytics disabled',
         'Feedback surveys disabled',
         'Usage metrics not collected'
       ],
       'essential-traffic': [
         'All telemetry disabled',
         'Auto-updates disabled',
         'Release notes not fetched',
         'Model capabilities cached only'
       ]
     }
     return impacts[level]
   }
   ```

4. **命令行开关**
   ```typescript
   // 支持 --no-telemetry 命令行参数
   export function getPrivacyLevel(): PrivacyLevel {
     if (process.argv.includes('--no-telemetry')) {
       return 'no-telemetry'
     }
     // ... 现有逻辑 ...
   }
   ```

5. **用户确认**
   ```typescript
   // 在设置隐私级别时提示用户
   export async function confirmPrivacyLevel(level: PrivacyLevel): Promise<boolean> {
     if (level === 'essential-traffic') {
       const confirmed = await askUser({
         message: 'This will disable all non-essential features including auto-updates. Continue?'
       })
       return confirmed
     }
     return true
   }
   ```

6. **测试覆盖**
   - 添加单元测试，覆盖各种环境变量组合
   - 测试边界情况（空字符串、大小写等）
   - 测试函数返回值
