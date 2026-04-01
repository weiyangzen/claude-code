# imageProcessor.ts 研究文档

## 场景与职责

imageProcessor.ts 是 FileReadTool 的图片处理模块抽象层，负责提供统一的图片处理 API，同时支持两种底层实现：

1. **原生图片处理模块** (`image-processor-napi`) - 用于打包/发布模式
2. **Sharp 库** (`sharp`) - 用于开发模式或原生模块不可用时

该模块在以下场景发挥关键作用：
- 读取图片文件时需要调整尺寸或压缩
- 生成新图片（如截图、图表）
- 处理用户粘贴的图片内容
- PDF 页面提取后的图片处理

## 功能点目的

### 1. 图片处理抽象

提供统一的 `SharpFunction` 接口，隐藏底层实现差异：
- 获取图片元数据（宽度、高度、格式）
- 调整图片尺寸（保持宽高比）
- 格式转换（JPEG、PNG、WebP）
- 压缩质量调整
- 输出为 Buffer

### 2. 双模式支持

**打包模式** (`isInBundledMode()`):
- 优先尝试加载 `image-processor-napi` 原生模块
- 性能更好，无需 Node.js 依赖
- 如果失败，回退到 sharp

**开发模式**:
- 直接使用 sharp 库
- 开发体验更好，调试更方便

### 3. 图片生成功能

提供 `getImageCreator` 函数用于从零生成图片：
- 创建纯色背景图片
- 指定尺寸和通道数（RGB/RGBA）
- 注意：原生模块不支持创建，始终使用 sharp

## 具体技术实现

### 核心数据结构

```typescript
// Sharp 实例接口（子集）
interface SharpInstance {
  metadata(): Promise<{ width: number; height: number; format: string }>
  resize(width: number, height: number, options?: { 
    fit?: string 
    withoutEnlargement?: boolean 
  }): SharpInstance
  jpeg(options?: { quality?: number }): SharpInstance
  png(options?: { 
    compressionLevel?: number 
    palette?: boolean 
    colors?: number 
  }): SharpInstance
  webp(options?: { quality?: number }): SharpInstance
  toBuffer(): Promise<Buffer>
}

// 图片处理函数类型
interface SharpFunction {
  (input: Buffer): SharpInstance
}

// 图片创建选项
interface SharpCreatorOptions {
  create: {
    width: number
    height: number
    channels: 3 | 4
    background: { r: number; g: number; b: number }
  }
}

// 图片创建函数类型
interface SharpCreator {
  (options: SharpCreatorOptions): SharpInstance
}
```

### 关键流程

#### 1. 获取图片处理器 (`getImageProcessor`)

```
getImageProcessor():
    1. 检查缓存（imageProcessorModule）
    2. 如果是打包模式:
        a. 尝试导入 'image-processor-napi'
        b. 如果成功，缓存并返回
        c. 如果失败，打印警告，继续到步骤 3
    3. 导入 'sharp' 库
    4. 解包默认导出（处理 ESM/CJS 差异）
    5. 缓存并返回
```

#### 2. 获取图片创建器 (`getImageCreator`)

```
getImageCreator():
    1. 检查缓存（imageCreatorModule）
    2. 导入 'sharp' 库（原生模块不支持创建）
    3. 解包默认导出
    4. 缓存并返回
```

### 关键代码路径

| 功能 | 代码路径 | 行号 |
|------|----------|------|
| 获取图片处理器 | `getImageProcessor()` | 37-67 |
| 获取图片创建器 | `getImageCreator()` | 74-85 |
| 模块解包 | `unwrapDefault<T>(mod)` | 90-93 |
| 缓存检查 | `imageProcessorModule` / `imageCreatorModule` | 34-35 |

### 依赖与外部交互

#### 直接依赖模块

| 模块 | 用途 |
|------|------|
| `buffer` | `Buffer` 类型定义 |
| `../../utils/bundledMode.js` | `isInBundledMode()` 检测打包模式 |

#### 外部库（动态导入）

| 库 | 场景 | 说明 |
|----|------|------|
| `image-processor-napi` | 打包模式优先 | 原生 Node-API 模块，性能更好 |
| `sharp` | 开发模式 / 回退 | 纯 Node.js 图片处理库 |

### 模块导入处理

```typescript
// 处理 ESM/CJS 模块差异
type MaybeDefault<T> = T | { default: T }

function unwrapDefault<T extends (...args: never[]) => unknown>(
  mod: MaybeDefault<T>,
): T {
  return typeof mod === 'function' ? mod : mod.default
}
```

## 风险、边界与改进建议

### 已知风险

1. **原生模块加载失败**
   - 风险：`image-processor-napi` 可能因平台不兼容或安装问题加载失败
   - 缓解：有 sharp 回退机制，但会打印警告
   - 注意：首次加载失败后才回退，有一定延迟

2. **API 兼容性**
   - 风险：原生模块和 sharp 的 API 可能有细微差异
   - 当前：使用 `SharpInstance` 类型定义子集，确保两者兼容
   - 潜在问题：高级功能（如图层、特效）可能不兼容

3. **内存管理**
   - 风险：sharp 的流式处理可能占用大量内存
   - 缓解：调用方负责管理 buffer 生命周期
   - 注意：大图片处理可能导致 OOM

4. **缓存策略**
   - 当前：模块级别缓存，进程生命周期内有效
   - 风险：如果底层库需要重新初始化（如配置变更），无法刷新

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| 原生模块和 sharp 都不可用 | 抛出模块加载错误，由调用方处理 |
| 输入 buffer 为空 | 由 sharp 实例方法处理，通常抛出错误 |
| 不支持的图片格式 | 由底层库处理，返回相应错误 |
| 内存不足 | 抛出 ENOMEM 错误，由调用方处理 |

### 改进建议

1. **配置化回退**
   - 当前：原生模块失败自动回退到 sharp
   - 建议：增加环境变量控制是否允许回退（`CLaude_CODE_FORCE_NATIVE_IMAGE_PROCESSOR`）

2. **健康检查**
   - 建议：增加 `checkImageProcessorHealth()` 函数，预检模块可用性
   - 用途：启动时检测，提前发现问题

3. **性能指标**
   - 建议：增加处理耗时统计，用于性能监控
   - 示例：`logEvent('tengu_image_process', { duration_ms, backend: 'native' \| 'sharp' })`

4. **格式支持检测**
   - 建议：增加 `getSupportedFormats()` 函数
   - 用途：根据底层库能力动态调整支持格式

5. **并发控制**
   - 风险：大量并发图片处理可能耗尽内存
   - 建议：增加信号量或连接池限制并发数

6. **渐进式加载**
   - 建议：支持流式处理大图片，而非一次性加载到内存
   - 依赖：sharp 支持流式 API，但当前抽象层未暴露

### 与 FileReadTool 的协作

imageProcessor.ts 主要被以下模块使用：

1. **`src/utils/imageResizer.js`**
   - 调用 `getImageProcessor()` 获取处理器
   - 用于 `maybeResizeAndDownsampleImageBuffer`
   - 用于 `compressImageBuffer`

2. **`src/tools/FileReadTool/FileReadTool.ts`**
   - 通过 imageResizer.js 间接使用
   - 处理读取的图片文件

### 测试注意事项

- 需要安装 sharp 作为 devDependency
- 原生模块测试需要在打包环境中进行
- 建议 mock 底层库以测试抽象层逻辑
- 需要测试 ESM/CJS 两种模块导入场景

### 相关文件

| 文件 | 关系 |
|------|------|
| `src/utils/imageResizer.js` | 主要调用方 |
| `src/utils/bundledMode.js` | 模式检测 |
| `src/tools/FileReadTool/FileReadTool.ts` | 间接使用 |
