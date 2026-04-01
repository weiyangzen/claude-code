# gates.ts 研究文档

## 场景与职责

本文件实现 **Computer Use (Chicago) 功能的动态配置门控系统**，控制功能的启用/禁用以及各种子功能的开关。这是 GrowthBook 功能标志与本地业务逻辑的结合点。

核心职责：
1. **主功能开关**：基于订阅类型和配置的启用决策
2. **子功能门控**：鼠标动画、隐藏前动作、剪贴板保护等
3. **坐标模式**：像素 vs 归一化坐标
4. **Ant 用户特殊处理**：防止内部开发者意外启用

## 功能点目的

### 1. 主功能开关 (`getChicagoEnabled`)

启用条件：
- 订阅要求：Max/Pro 用户，或 Ant 用户（内部）
- GrowthBook 标志：`tengu_malort_pedway`
- Ant 用户额外检查：
  - 如果 shell 继承了 monorepo 开发配置（`MONOREPO_ROOT_DIR`）
  - 且未设置 `ALLOW_ANT_COMPUTER_USE_MCP=1`
  - 则禁用（防止内部开发者意外启用）

### 2. 子功能门控 (`getChicagoSubGates`)

从配置中提取子功能标志：
- `pixelValidation` - 像素验证 JPEG 解码+裁剪
- `clipboardPasteMultiline` - 多行剪贴板粘贴
- `mouseAnimation` - 鼠标动画移动
- `hideBeforeAction` - 动作前隐藏其他应用
- `autoTargetDisplay` - 自动目标显示器
- `clipboardGuard` - 剪贴板保护

### 3. 坐标模式 (`getChicagoCoordinateMode`)

- 首次读取时冻结（`frozenCoordinateMode`）
- 确保 `setup.ts` 和 `executor.ts` 使用相同值
- 支持中途 GrowthBook 切换时的一致性

## 具体技术实现

### 配置类型

```typescript
type ChicagoConfig = CuSubGates & {
  enabled: boolean
  coordinateMode: CoordinateMode  // 'pixels' | 'normalized'
}

const DEFAULTS: ChicagoConfig = {
  enabled: false,
  pixelValidation: false,
  clipboardPasteMultiline: true,
  mouseAnimation: true,
  hideBeforeAction: true,
  autoTargetDisplay: true,
  clipboardGuard: true,
  coordinateMode: 'pixels',
}
```

### 配置读取

```typescript
function readConfig(): ChicagoConfig {
  return {
    ...DEFAULTS,
    ...getDynamicConfig_CACHED_MAY_BE_STALE<Partial<ChicagoConfig>>(
      'tengu_malort_pedway',
      DEFAULTS,
    ),
  }
}
```

**设计说明**：
- 使用展开操作符合并默认值和 GrowthBook 配置
- 部分 JSON（如 `{"enabled": true}`）继承其余默认值
- 泛型是类型断言，非验证器

### 订阅检查

```typescript
function hasRequiredSubscription(): boolean {
  if (process.env.USER_TYPE === 'ant') return true
  const tier = getSubscriptionType()
  return tier === 'max' || tier === 'pro'
}
```

**Ant 绕过**：
- 允许内部开发者无论订阅等级都能使用
- 符合 CLAUDE.md:281 要求（USER_TYPE !== 'ant' 分支无 antfooding）

### Ant 用户保护

```typescript
if (
  process.env.USER_TYPE === 'ant' &&
  process.env.MONOREPO_ROOT_DIR &&
  !isEnvTruthy(process.env.ALLOW_ANT_COMPUTER_USE_MCP)
) {
  return false
}
```

**目的**：
- `MONOREPO_ROOT_DIR` 由 `config/local/zsh/zshrc` 导出
- `laptop-setup.sh` 将其写入 `~/.zshrc`
- 存在此变量表示"有 monorepo 访问权限"
- 防止内部开发者在工作会话中意外启用 CU

## 关键代码路径与文件引用

### 本文件导出
- `getChicagoEnabled()` - 主功能开关检查
- `getChicagoSubGates()` - 子功能门控
- `getChicagoCoordinateMode()` - 坐标模式

### 调用方
- `src/utils/computerUse/hostAdapter.ts:55` - `isDisabled` 回调
- `src/utils/computerUse/hostAdapter.ts:56` - `getSubGates` 回调
- `src/utils/computerUse/mcpServer.ts:64` - `getChicagoCoordinateMode`
- `src/utils/computerUse/setup.ts:29` - `getChicagoCoordinateMode`
- `src/utils/computerUse/wrapper.tsx:27` - `getChicagoCoordinateMode`
- `src/utils/computerUse/wrapper.tsx:235` - `getChicagoCoordinateMode`
- `src/utils/computerUse/executor.ts:44` - `getChicagoSubGates`（通过 hostAdapter）

### 依赖文件
- `src/services/analytics/growthbook.ts` - `getDynamicConfig_CACHED_MAY_BE_STALE`
- `src/utils/auth.ts` - `getSubscriptionType`
- `src/utils/envUtils.ts` - `isEnvTruthy`

## 依赖与外部交互

### GrowthBook 集成
- Feature Flag: `tengu_malort_pedway`
- 返回部分 `ChicagoConfig` 对象
- 缓存可能陈旧，但功能开关容忍短暂不一致

### 环境变量
- `USER_TYPE` - 用户类型（'ant' 为内部）
- `MONOREPO_ROOT_DIR` - monorepo 根目录路径
- `ALLOW_ANT_COMPUTER_USE_MCP` - 强制启用覆盖

### 订阅系统
- 通过 `getSubscriptionType()` 获取订阅等级
- Max/Pro 用户可访问 CU 功能

## 风险、边界与改进建议

### 已知风险

1. **缓存陈旧**：
   - `getDynamicConfig_CACHED_MAY_BE_STALE` 可能返回旧值
   - 功能开关容忍此风险
   - 坐标模式在首次读取后冻结

2. **环境变量依赖**：
   - `MONOREPO_ROOT_DIR` 检测可能误报/漏报
   - 某些内部设置可能不设置此变量

3. **订阅检查延迟**：
   - `getSubscriptionType()` 可能涉及网络请求
   - 在配置读取路径中需要注意性能

### 边界情况

1. **部分配置**：
   - GrowthBook 只返回部分字段
   - 默认值确保完整配置对象

2. **快速切换**：
   - 坐标模式在首次读取后冻结
   - 会话期间 GrowthBook 切换不会立即生效

3. **Ant 用户覆盖**：
   - `ALLOW_ANT_COMPUTER_USE_MCP=1` 强制启用
   - 即使 `MONOREPO_ROOT_DIR` 存在也生效

### 改进建议

1. **配置验证**：
   - 添加运行时配置验证（如 zod）
   - 在开发环境检测无效配置

2. **可观测性**：
   - 记录配置决策（为什么启用/禁用）
   - 监控 GrowthBook 配置加载失败率

3. **灰度发布**：
   - 考虑添加用户 ID 哈希灰度
   - 支持按用户群渐进发布

4. **文档化**：
   - 在内部 wiki 记录 Ant 用户覆盖流程
   - 添加故障排除指南

5. **测试覆盖**：
   - 测试各种订阅类型的行为
   - 测试环境变量组合
   - 验证默认值正确性

6. **性能优化**：
   - 考虑缓存订阅检查结果
   - 避免重复读取 GrowthBook 配置
