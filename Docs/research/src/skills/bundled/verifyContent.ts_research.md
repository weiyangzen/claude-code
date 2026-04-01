# verifyContent.ts 深度研究文档

## 场景与职责

`verifyContent.ts` 是 `/verify` 技能的**内容资产文件**，负责将验证技能所需的 Markdown 文档在构建时通过 Bun 的 text loader 内联为 TypeScript 字符串常量。

### 核心职责
1. **构建时文档内联**：利用 Bun 的 `*.md` text loader 将 Markdown 文件编译为 JS 字符串
2. **内容导出**：导出 `SKILL_MD`（主文档）和 `SKILL_FILES`（参考文件映射）
3. **零运行时依赖**：除 Bun 构建机制外，无其他运行时依赖

### 适用场景
- 构建 Claude Code CLI 二进制时
- `verify.ts` 运行时导入内容
- 模型需要读取验证示例时

---

## 功能点目的

### 1. 主文档导出
```typescript
export const SKILL_MD: string = skillMd
```
**目的**：提供 `/verify` 技能的主 SKILL.md 内容，包含技能的核心指令和 frontmatter。

### 2. 参考文件映射
```typescript
export const SKILL_FILES: Record<string, string> = {
  'examples/cli.md': cliMd,
  'examples/server.md': serverMd,
}
```
**目的**：提供额外的参考文档，模型可以按需读取这些文件获取特定场景（CLI/Server）的验证指南。

### 3. 构建时内联
```typescript
import cliMd from './verify/examples/cli.md'
import serverMd from './verify/examples/server.md'
import skillMd from './verify/SKILL.md'
```
**目的**：利用 Bun 的 text loader，在构建时将 `.md` 文件内容作为字符串内联，避免运行时文件系统读取。

---

## 具体技术实现

### 关键流程

#### 1. 构建时处理流程
```
构建阶段 (Bun):
  1. 遇到 import './verify/SKILL.md'
  2. text loader 读取文件内容为字符串
  3. 将字符串内联到编译输出中

运行时:
  1. skillMd 已经是字符串常量
  2. 直接导出供 verify.ts 使用
```

#### 2. 模块导出结构
```typescript
// 主文档 - 包含 frontmatter 和技能主体内容
export const SKILL_MD: string = skillMd

// 参考文件映射 - 相对路径 -> 内容
export const SKILL_FILES: Record<string, string> = {
  'examples/cli.md': cliMd,     // CLI 应用验证指南
  'examples/server.md': serverMd, // 服务器应用验证指南
}
```

### 数据结构

#### 导出常量
| 常量名 | 类型 | 描述 |
|--------|------|------|
| `SKILL_MD` | `string` | 主 SKILL.md 文件的完整内容 |
| `SKILL_FILES` | `Record<string, string>` | 参考文件路径到内容的映射 |

#### 内部导入
| 导入名 | 来源路径 | 描述 |
|--------|----------|------|
| `cliMd` | `./verify/examples/cli.md` | CLI 验证示例文档 |
| `serverMd` | `./verify/examples/server.md` | Server 验证示例文档 |
| `skillMd` | `./verify/SKILL.md` | 主技能文档 |

### 关键代码路径

| 功能 | 文件路径 |
|------|----------|
| 内容资产定义 | `src/skills/bundled/verifyContent.ts:1-13` |
| 使用者（verify.ts） | `src/skills/bundled/verify.ts:3` |
| 文件提取逻辑 | `src/skills/bundledSkills.ts:59-73` |

---

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `cli.md` | `./verify/examples/cli.md` | CLI 验证示例内容 |
| `server.md` | `./verify/examples/server.md` | Server 验证示例内容 |
| `SKILL.md` | `./verify/SKILL.md` | 主技能文档内容 |

### 构建工具依赖

| 工具 | 配置 | 用途 |
|------|------|------|
| Bun | `loader: { '.md': 'text' }` | 将 `.md` 文件作为字符串内联 |

### 外部交互
- **构建时**：Bun 读取 `src/skills/bundled/verify/` 目录下的 `.md` 文件
- **运行时**：无文件系统或网络交互，所有内容已内联

---

## 风险、边界与改进建议

### 风险点

1. **源文件缺失风险**
   - 构建时如果 `src/skills/bundled/verify/` 目录或 `.md` 文件不存在，构建会失败
   - **现状**：源代码仓库中该目录可能不存在（仅存在于构建环境）
   - **错误示例**：
     ```
     error: Could not resolve: "./verify/SKILL.md"
     ```

2. **硬编码的文件路径**
   ```typescript
   'examples/cli.md': cliMd,
   'examples/server.md': serverMd,
   ```
   - 新增示例需要修改代码并重新构建
   - 文件路径变更需要同步修改导入语句

3. **无内容验证**
   - 不验证导入的内容是否为空
   - 不验证 frontmatter 格式是否正确
   - 这些问题只能在运行时（verify.ts 解析时）发现

4. **内存占用**
   - 所有内容在构建时内联，会增加二进制文件大小
   - 对于大型文档，可能导致内存占用增加

### 边界条件

1. **空文件处理**
   - 如果 `.md` 文件为空，导入的字符串也为空
   - `verify.ts` 中的 `parseFrontmatter` 会返回空 frontmatter 和空 content

2. **文件编码**
   - Bun text loader 默认使用 UTF-8 编码
   - 非 UTF-8 编码的文件可能导致内容乱码

3. **文件大小限制**
   - Bun 对 text loader 导入的文件大小无明确限制
   - 但过大的文件会增加编译时间和二进制大小

### 改进建议

1. **动态文件加载**
   ```typescript
   // 建议：使用 glob 动态加载示例文件
   const exampleFiles = import.meta.glob('./verify/examples/*.md', { as: 'string' })
   
   export const SKILL_FILES: Record<string, string> = Object.fromEntries(
     Object.entries(exampleFiles).map(([path, content]) => [
       path.replace('./verify/', ''),
       content
     ])
   )
   ```

2. **构建时内容验证**
   ```typescript
   // 建议：添加构建时验证
   if (!skillMd || skillMd.trim().length === 0) {
     throw new Error('verifyContent: SKILL.md is empty')
   }
   
   if (!skillMd.startsWith('---')) {
     console.warn('verifyContent: SKILL.md may be missing frontmatter')
   }
   ```

3. **类型安全增强**
   ```typescript
   // 建议：定义明确的文件路径类型
   type VerifyExampleFile = 'examples/cli.md' | 'examples/server.md'
   
   export const SKILL_FILES: Record<VerifyExampleFile, string> = {
     'examples/cli.md': cliMd,
     'examples/server.md': serverMd,
   }
   ```

4. **内容拆分策略**
   - 对于大型技能文档，考虑拆分为多个主题文件
   - 使用 `files` 机制按需加载，而非全部内联到主提示

5. **文档同步检查**
   - 添加 CI 检查，确保 `verify/` 目录下的文档与代码逻辑保持同步
   - 验证 frontmatter 中的必需字段（description、allowed-tools 等）

6. **回退机制**
   ```typescript
   // 建议：添加运行时回退
   export const SKILL_MD: string = skillMd || '## Verify Skill\n\nContent unavailable.'
   ```
