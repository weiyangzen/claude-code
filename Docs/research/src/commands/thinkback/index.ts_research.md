# 研究文档：src/commands/thinkback/index.ts

## 场景与职责

该文件是 `/think-back` 命令的入口定义模块，负责注册 Claude Code 的"2025 Year in Review"（年度回顾）功能命令。这是一个面向用户的互动式功能，通过插件系统动态加载 thinkback 技能来生成和播放个性化的 ASCII 动画，展示用户过去一年使用 Claude Code 的编码旅程。

该命令属于本地 JSX 类型命令，采用懒加载模式，只有在用户实际调用时才加载完整的实现代码（thinkback.tsx），从而优化启动性能。

## 功能点目的

1. **命令注册**：定义命令元数据（名称、描述、类型、启用条件）
2. **功能门控**：通过 GrowthBook 特性开关 `tengu_thinkback` 控制命令是否可用
3. **懒加载优化**：使用动态导入延迟加载重型实现代码
4. **用户类型适配**：支持内部（ant）和外部用户不同的插件市场配置

## 具体技术实现

### 关键数据结构

```typescript
const thinkback = {
  type: 'local-jsx',           // 命令类型：本地 JSX 渲染
  name: 'think-back',          // 用户可见的命令名称
  description: 'Your 2025 Claude Code Year in Review',
  isEnabled: () => checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_thinkback'),
  load: () => import('./thinkback.js'),  // 懒加载实现
} satisfies Command
```

### 关键流程

1. **命令发现阶段**：
   - 在 `src/commands.ts` 中被导入并加入命令列表
   - 通过 `isEnabled()` 检查特性开关状态
   - 只有特性开关开启时命令才对用户可见

2. **命令执行阶段**：
   - 用户输入 `/think-back`
   - 系统调用 `load()` 动态导入 `thinkback.js`
   - 执行 `thinkback.tsx` 中导出的 `call()` 函数

### 特性门控机制

使用 `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` 函数检查特性开关：
- 优先检查环境变量覆盖（`CLAUDE_INTERNAL_FC_OVERRIDES`）
- 其次检查 GrowthBook 内存缓存
- 最后回退到 Statsig 缓存（迁移期兼容）
- 默认值为 `false`，即特性关闭

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `../../commands.js` | 导入 `Command` 类型定义 |
| `../../services/analytics/growthbook.js` | 特性门控检查函数 |
| `./thinkback.js` | 懒加载的实际实现 |

### 调用关系

```
用户输入 /think-back
    ↓
src/commands.ts 中的命令路由
    ↓
index.ts 中定义的 thinkback 命令
    ↓
load() → import('./thinkback.js')
    ↓
thinkback.tsx 中的 ThinkbackFlow 组件
```

### 相关命令

- `/thinkback-play` (`src/commands/thinkback-play/`)：隐藏命令，由 thinkback 技能在生成完成后调用，直接播放动画

## 依赖与外部交互

### 外部服务

1. **GrowthBook**：特性开关服务
   - 用于控制功能灰度发布
   - 支持按用户属性（userType、organization 等）定向开启

2. **插件市场**：
   - 内部用户：`anthropics/claude-code-marketplace`
   - 外部用户：`anthropics/claude-plugins-official`

### 运行时依赖

- 需要 `thinkback` 插件已安装并启用
- 插件从 GitHub 仓库动态克隆/更新
- 依赖 Node.js 子进程执行动画播放器

## 风险、边界与改进建议

### 已知风险

1. **特性开关依赖**：
   - 如果 GrowthBook 服务不可用或缓存过期，命令可能意外隐藏或显示
   - `CACHED_MAY_BE_STALE` 后缀明确表示可能返回过时值

2. **插件安装失败**：
   - 首次使用需要下载并安装插件，可能因网络问题失败
   - Git 克隆操作可能因 SSH 配置、超时等原因失败

3. **平台兼容性**：
   - 动画播放使用 `execa` 调用 Node.js 子进程
   - 浏览器打开 HTML 文件依赖平台特定命令（macOS: `open`, Windows: `start`, Linux: `xdg-open`）

### 边界情况

1. **重复调用**：
   - 命令支持重复调用，已生成动画的用户可选择播放、编辑、修复或重新生成

2. **并发处理**：
   - 安装/更新插件时有加载状态提示
   - 动画播放期间占用终端（alternate screen 模式）

3. **错误处理**：
   - 插件未安装时自动触发安装流程
   - 安装失败时提供明确的错误信息和重试指引

### 改进建议

1. **性能优化**：
   - 考虑预加载插件清单以减少首次使用时的等待时间
   - 动画数据文件（year_in_review.js）可能较大，考虑流式加载

2. **用户体验**：
   - 添加取消安装/生成的能力
   - 提供更详细的进度反馈（如插件下载百分比）

3. **可观测性**：
   - 添加专门的遥测事件追踪功能使用漏斗（发现 → 安装 → 生成 → 播放）
   - 监控插件安装失败率和原因分布

4. **代码健壮性**：
   - `isEnabled` 函数目前只检查特性开关，可考虑添加更多前置条件检查（如 Node.js 版本、终端类型）
   - 考虑添加命令可用性的缓存，避免频繁查询 GrowthBook
