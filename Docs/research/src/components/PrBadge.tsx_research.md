# PrBadge.tsx 研究文档

## 场景与职责

`PrBadge.tsx` 是 Claude Code 中用于显示 Pull Request 状态徽章的轻量级 UI 组件。该组件在以下场景使用：

1. **状态栏显示**: 在底部状态栏显示当前分支关联的 PR 状态
2. **PR 列表**: 在 PR 相关命令的输出中显示 PR 信息
3. **Git 集成**: 与 `gh pr view` 命令结果结合，提供视觉化的 PR 状态

组件的核心职责：
- 根据 PR 审核状态显示不同颜色
- 提供可点击的链接到 PR 页面
- 支持粗体/普通文本样式

## 功能点目的

### 1. 状态颜色映射
组件将 PR 审核状态映射到终端颜色：

| 状态 | 颜色 | 含义 |
|------|------|------|
| approved | success (绿色) | PR 已获批准 |
| changes_requested | error (红色) | 需要修改 |
| pending | warning (黄色) | 等待审核 |
| merged | merged (紫色) | 已合并 |
| 未指定 | undefined (默认) | 无特殊状态 |

### 2. 视觉层次设计
组件渲染结构：
```
<Text>
  <Text dimColor={!bold}>PR</Text>{" "}
  <Link url={url} fallback={label}>
    <Text color={statusColor} dimColor={!statusColor && !bold} underline bold={bold}>
      #{number}
    </Text>
  </Link>
</Text>
```

设计特点：
- "PR" 前缀在非粗体模式下显示为暗淡色
- PR 编号带下划线表示可点击
- 颜色编码快速传达状态信息

### 3. 链接回退机制
使用 Ink 的 `Link` 组件，支持终端超链接（OSC 8）：
- 如果终端支持超链接，点击直接打开 PR 页面
- 如果不支持，`fallback` 属性显示普通文本标签

## 具体技术实现

### 关键流程

#### 渲染流程
```
1. 接收 Props (number, url, reviewState, bold)
2. 调用 getPrStatusColor(reviewState) 获取颜色
3. 构建 label（非下划线版本，用于 fallback）
4. 构建带下划线的链接文本
5. 渲染 Link 组件包裹的 Text
```

### 数据结构

#### Props 定义
```typescript
type Props = {
  number: number;                    // PR 编号
  url: string;                       // PR 页面 URL
  reviewState?: PrReviewState;       // 审核状态（可选）
  bold?: boolean;                    // 是否粗体显示
};
```

#### PrReviewState 类型
```typescript
export type PrReviewState = 
  | 'approved' 
  | 'pending' 
  | 'changes_requested' 
  | 'draft' 
  | 'merged' 
  | 'closed';
```

### 颜色映射函数
```typescript
function getPrStatusColor(state?: PrReviewState): 'success' | 'error' | 'warning' | 'merged' | undefined {
  switch (state) {
    case 'approved': return 'success';
    case 'changes_requested': return 'error';
    case 'pending': return 'warning';
    case 'merged': return 'merged';
    default: return undefined;
  }
}
```

### React Compiler 优化
组件使用 React Compiler 运行时进行自动记忆化：
```typescript
const $ = _c(21);  // 21 个记忆化槽位
```

每个渲染决策都基于依赖比较，避免不必要的重新渲染。

## 关键代码路径与文件引用

### 本文件关键代码
| 行号 | 功能 |
|------|------|
| 5-10 | Props 类型定义 |
| 11-82 | PrBadge 主组件 |
| 20-26 | statusColor 记忆化计算 |
| 28-39 | label（fallback 文本）构建 |
| 41-71 | Link 组件渲染 |
| 83-96 | getPrStatusColor 函数 |

### 依赖文件引用

| 导入路径 | 用途 |
|----------|------|
| `../ink.js` | Link, Text 组件 |
| `../utils/ghPrStatus.js` | PrReviewState 类型 |

### 依赖的依赖

```
PrBadge.tsx
└── ghPrStatus.ts
    ├── deriveReviewState()  // 从 GitHub API 响应派生状态
    └── fetchPrStatus()      // 获取 PR 状态
```

## 依赖与外部交互

### 外部依赖
1. **React Compiler Runtime**: `react/compiler-runtime` 自动记忆化
2. **Ink**: 终端 UI 组件库

### 数据来源
PR 状态通常来自 `ghPrStatus.ts` 中的 `fetchPrStatus()`：

```typescript
export async function fetchPrStatus(): Promise<PrStatus | null> {
  // 执行: gh pr view --json number,url,reviewDecision,isDraft,...
  const { stdout, code } = await execFileNoThrow('gh', [
    'pr', 'view',
    '--json', 'number,url,reviewDecision,isDraft,headRefName,state'
  ]);
  // 解析并返回 PrStatus
}
```

### 颜色主题集成
颜色名称（'success', 'error', 'warning', 'merged'）映射到当前主题的对应颜色值。主题系统定义在 `src/utils/theme.ts`。

## 风险、边界与改进建议

### 已知风险

1. **状态映射不完整**
   - `draft` 和 `closed` 状态没有专门的颜色映射
   - 当前返回 `undefined`，显示为默认颜色
   - 用户可能无法区分草稿 PR 和已关闭 PR

2. **颜色可访问性**
   - 依赖终端主题的颜色定义
   - 某些主题中颜色对比度可能不足

3. **URL 有效性**
   - 组件不验证 URL 格式
   - 无效 URL 可能导致 Link 组件行为异常

### 边界情况

1. **PR 编号为 0 或负数**
   - 组件接受 `number: number`，不验证范围
   - 显示 `#0` 或负数可能不符合预期

2. **空 URL**
   - 如果 `url` 为空字符串，Link 组件可能无法正常工作

3. **未知状态**
   - GitHub API 可能返回新的 reviewDecision 值
   - 当前会返回 `undefined`，显示为默认样式

4. **终端超链接支持**
   - 旧版终端不支持 OSC 8 超链接
   - `fallback` 属性确保在这些终端中仍可显示文本

### 改进建议

1. **完整状态支持**
   ```typescript
   function getPrStatusColor(state?: PrReviewState): Color | undefined {
     switch (state) {
       case 'approved': return 'success';
       case 'changes_requested': return 'error';
       case 'pending': return 'warning';
       case 'merged': return 'merged';
       case 'draft': return 'subtle';  // 新增：草稿状态
       case 'closed': return 'error';  // 新增：已关闭
       default: return undefined;
     }
   }
   ```

2. **添加 PR 标题显示**
   - 当前只显示编号，建议可选显示标题
   - 在状态栏空间有限时可能需要截断

3. **Props 验证**
   ```typescript
   // 添加运行时验证
   if (number <= 0) {
     console.warn(`Invalid PR number: ${number}`);
   }
   if (!url.startsWith('http')) {
     console.warn(`Invalid PR URL: ${url}`);
   }
   ```

4. **悬停提示**
   - 添加更多 PR 信息（标题、作者、创建时间）
   - 在支持的终端中显示为悬停提示

5. **批量 PR 显示**
   - 当前设计针对单个 PR
   - 如果需要显示多个 PR，考虑添加紧凑模式

6. **图标支持**
   - 在 PR 编号前添加状态图标（✓, ✗, ⏳ 等）
   - 增强视觉识别度

### 测试建议

1. **单元测试**
   - `getPrStatusColor` 的所有状态映射
   - Props 变化时的重新渲染行为

2. **视觉回归测试**
   - 不同状态下的输出快照
   - 粗体/非粗体模式对比

3. **集成测试**
   - 与 `fetchPrStatus` 的集成
   - 实际 GitHub PR 数据渲染

### 相关文件
- `src/utils/ghPrStatus.ts`: PR 状态获取逻辑
- `src/utils/theme.ts`: 颜色主题定义
- `src/ink.tsx`: Ink 组件封装
