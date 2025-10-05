
# 第五章：Uniform Buffer 和数据传递

## 5.1 Uniform Buffer 概念理解

### 5.1.1 什么是 Uniform Buffer

**Uniform Buffer** 是 WebGPU 中用于在 CPU 和 GPU 之间传递数据的重要机制。它是一种特殊的缓冲区，允许我们将数据从 JavaScript 传递到着色器程序中。

**核心特点：**
- **一致性（Uniform）**：对于同一绘制调用中的所有顶点/片段，数据保持一致
- **只读访问**：着色器只能读取 Uniform 数据，不能修改
- **高效传输**：专为小量、频繁更新的数据设计

在我们的示例中，Uniform Buffer 用于传递：
- 当前时间（用于动画效果）
- 鼠标位置（用于交互效果）

### 5.1.2 Uniform Buffer vs Vertex Buffer

| 特性 | Uniform Buffer | Vertex Buffer |
|------|----------------|---------------|
| **数据用途** | 全局参数（时间、鼠标位置等） | 顶点特定数据（位置、颜色等） |
| **访问方式** | 所有顶点/片段共享相同值 | 每个顶点有不同的值 |
| **更新频率** | 通常每帧更新 | 通常创建时设置，很少更新 |
| **数据量** | 通常较小（几十到几百字节） | 可能很大（数千个顶点） |
| **典型用途** | 变换矩阵、时间、光照参数 | 顶点位置、法线、UV 坐标 |

### 5.1.3 CPU 到 GPU 的数据传递流程

```
CPU（JavaScript）                    GPU（着色器）
     │                                   │
     ├─ 1. 创建数据 ────────────────────►│
     │   Float32Array                    │
     │                                   │
     ├─ 2. writeBuffer ─────────────────►│ Uniform Buffer
     │   复制到 GPU 内存                  │
     │                                   │
     │                              ┌────┴────┐
     │                              │ 顶点着色器 │
     │                              │ 片段着色器 │
     │                              └─────────┘
     │                                   │
     └─ 每帧重复 ◄──────────────────────┘
```

---

## 5.2 Uniform 结构体设计

### 5.2.1 WGSL 中的 Uniform 定义

在 [`index.html`](study/3.wgsl_mini_demo/index.html:90-95) 中，我们定义了 Uniform 结构体：

```wgsl
// 定义 Uniform 结构体，用于从 CPU 传递数据到 GPU
struct Uniforms {
    time: f32,        // 时间（秒）
    mouseX: f32,      // 鼠标 X 坐标（归一化到 0-1）
    mouseY: f32,      // 鼠标 Y 坐标（归一化到 0-1）
    padding: f32,     // 填充字节，确保 16 字节对齐
}
```

### 5.2.2 16 字节对齐要求详解

WebGPU 要求 uniform buffer 必须满足 **16 字节对齐**。这是 GPU 硬件的要求，用于优化内存访问性能。

**内存布局图示：**

```
┌─────────────────────────────────────────────────┐
│ Uniform Buffer (16 字节)                        │
├─────────┬─────────┬─────────┬─────────┬─────────┤
│ time    │ mouseX  │ mouseY  │ padding │         │
│ (4 字节) │ (4 字节) │ (4 字节) │ (4 字节) │         │
│ f32     │ f32     │ f32     │ f32     │         │
├─────────┴─────────┴─────────┴─────────┴─────────┤
│ 0       4         8         12        16        │
│ ◄──────────── 16 字节对齐 ────────────►         │
└─────────────────────────────────────────────────┘
```

**关键规则：**

1. **基本对齐**：`f32` 类型占用 4 字节
2. **结构体对齐**：结构体大小必须是 16 字节的倍数
3. **填充字段**：当数据不足 16 字节倍数时，需要添加 padding

**示例对比：**

```wgsl
// ❌ 错误：只有 12 字节（3 × 4）
struct BadUniforms {
    time: f32,
    mouseX: f32,
    mouseY: f32,
}  // 12 字节，不是 16 的倍数！

// ✅ 正确：16 字节（4 × 4）
struct GoodUniforms {
    time: f32,
    mouseX: f32,
    mouseY: f32,
    padding: f32,  // 添加填充达到 16 字节
}
```

### 5.2.3 数据类型选择

WGSL 支持多种数据类型：

| WGSL 类型 | 大小 | JavaScript 对应 | 用途 |
|-----------|------|-----------------|------|
| `f32` | 4 字节 | `Float32Array` | 浮点数（最常用） |
| `i32` | 4 字节 | `Int32Array` | 整数 |
| `u32` | 4 字节 | `Uint32Array` | 无符号整数 |
| `vec2f` | 8 字节 | 2 个 float | 2D 向量 |
| `vec3f` | 12 字节 | 3 个 float | 3D 向量（需要填充到 16） |
| `vec4f` | 16 字节 | 4 个 float | 4D 向量 |
| `mat4x4f` | 64 字节 | 16 个 float | 4×4 矩阵 |

**最佳实践：**
- 优先使用 `vec4f` 代替多个 `f32`，自动满足对齐
- 对于 3D 向量，可以使用 `vec4f`，第四个分量用于填充

### 5.2.4 优化的结构体设计示例

```wgsl
// 方案 1：使用 padding
struct Uniforms {
    time: f32,
    mouseX: f32,
    mouseY: f32,
    padding: f32,
}  // 16 字节

// 方案 2：使用 vec4f（推荐）
struct Uniforms {
    timeAndMouse: vec4f,  // x=time, y=mouseX, z=mouseY, w=unused
}  // 16 字节，更简洁

// 方案 3：多个 vec4f
struct ComplexUniforms {
    timeAndMouse: vec4f,     // 时间和鼠标
    colorTint: vec4f,        // 颜色调整
    transformData: vec4f,    // 变换数据
}  // 48 字节
```

---

## 5.3 Buffer 创建和配置

### 5.3.1 createBuffer API 详解

在 [`index.html`](study/3.wgsl_mini_demo/index.html:179-183) 中，我们创建了 Uniform Buffer：

```javascript
const uniformBuffer = device.createBuffer({
    label: 'Uniform Buffer',
    size: 16,  // 4 * 4 bytes
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});
```

**参数详解：**

#### `label`（可选）
- **类型**：`string`
- **作用**：调试标识符
- **最佳实践**：始终提供有意义的标签

#### `size`（必需）
- **类型**：`number`（字节数）
- **计算方式**：根据结构体大小确定
- **我们的例子**：4 个 `f32` = 4 × 4 = 16 字节

**size 计算示例：**

```javascript
// 示例 1：单个 vec4f
const size1 = 16;  // 1 × vec4f = 16 字节

// 示例 2：时间 + 鼠标 + 填充
const size2 = 4 * 4;  // 4 个 f32 = 16 字节

// 示例 3：复杂场景
// struct ComplexUniforms {
//     mvpMatrix: mat4x4f,   // 64 字节
//     lightPos: vec4f,      // 16 字节
//     color: vec4f,         // 16 字节
// }
const size3 = 64 + 16 + 16;  // = 96 字节
```

#### `usage`（必需）
- **类型**：`GPUBufferUsageFlags`（位标志）
- **作用**：指定 Buffer 的用途

**常用的 usage 标志：**

| 标志 | 值 | 含义 | 何时使用 |
|------|---|------|----------|
| `UNIFORM` | 0x40 | 作为 uniform buffer | 传递常量数据到着色器 |
| `COPY_DST` | 0x08 | 可以作为复制目标 | 需要用 `writeBuffer` 更新 |
| `COPY_SRC` | 0x04 | 可以作为复制源 | 需要读取 buffer 内容 |
| `VERTEX` | 0x20 | 作为 vertex buffer | 存储顶点数据 |
| `INDEX` | 0x10 | 作为 index buffer | 存储索引数据 |
| `STORAGE` | 0x80 | 作为 storage buffer | 读写大量数据 |

**位运算组合：**

```javascript
// 使用 | 运算符组合多个标志
usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
//     └─────── 可以绑定为 uniform ────────┘     └─── 可以写入数据 ───┘

// 等价于
usage: 0x40 | 0x08  // = 0x48 = 72
```

**为什么需要 COPY_DST？**

```javascript
// ❌ 错误：只有 UNIFORM，无法更新
const badBuffer = device.createBuffer({
    size: 16,
    usage: GPUBufferUsage.UNIFORM,  // 缺少 COPY_DST
});
// 后续调用 writeBuffer 会失败！

// ✅ 正确：同时具备 UNIFORM 和 COPY_DST
const goodBuffer = device.createBuffer({
    size: 16,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});
device.queue.writeBuffer(goodBuffer, 0, data);  // 成功！
```

### 5.3.2 Buffer 创建的完整示例

```javascript
// 基础示例：时间和鼠标位置
const uniformBuffer = device.createBuffer({
    label: 'Time and Mouse Uniforms',
    size: 16,  // vec4f
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});

// 复杂示例：包含变换矩阵
const transformBuffer = device.createBuffer({
    label: 'Transform Uniforms',
    size: 64 + 16,  // mat4x4f + vec4f = 80 字节
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});

// 只读示例：初始化后不更新
const constantBuffer = device.createBuffer({
    label: 'Constants',
    size: 16,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
    mappedAtCreation: true,  // 创建时映射
});
// 在映射时写入数据
new Float32Array(constantBuffer.getMappedRange()).set([1, 2, 3, 4]);
constantBuffer.unmap();
```

---

## 5.4 Bind Group Layout 和资源绑定

### 5.4.1 WebGPU 资源绑定模型

WebGPU 使用 **Bind Group** 系统来管理着色器资源：

```
资源绑定层次结构：
┌─────────────────────────────────────────┐
│ Pipeline Layout                         │
│  ├─ Bind Group Layout 0 (@group(0))    │
│  │   ├─ Binding 0 (@binding(0))        │
│  │   ├─ Binding 1 (@binding(1))        │
│  │   └─ ...                             │
│  ├─ Bind Group Layout 1 (@group(1))    │
│  └─ ...                                 │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ Bind Group (实际资源)                    │
│  ├─ Binding 0 → uniformBuffer          │
│  ├─ Binding 1 → texture                │
│  └─ ...                                 │
└─────────────────────────────────────────┘
```

### 5.4.2 创建 Bind Group Layout

在 [`index.html`](study/3.wgsl_mini_demo/index.html:186-193) 中：

```javascript
const bindGroupLayout = device.createBindGroupLayout({
    label: 'Bind Group Layout',
    entries: [{
        binding: 0,                           // 对应 @binding(0)
        visibility: GPUShaderStage.FRAGMENT,  // 仅片段着色器可见
        buffer: { type: 'uniform' }           // uniform buffer 类型
    }]
});
```

**参数详解：**

#### `binding`
- **类型**：`number`
- **作用**：绑定点编号，必须与 WGSL 中的 `@binding(N)` 匹配
- **规则**：从 0 开始，连续编号

#### `visibility`
- **类型**：`GPUShaderStageFlags`
- **作用**：指定哪些着色器阶段可以访问此资源

**visibility 选项：**

| 标志 | 值 | 含义 |
|------|---|------|
| `VERTEX` | 0x1 | 顶点着色器可见 |
| `FRAGMENT` | 0x2 | 片段着色器可见 |
| `COMPUTE` | 0x4 | 计算着色器可见 |

**组合使用：**

```javascript
// 仅顶点着色器
visibility: GPUShaderStage.VERTEX

// 仅片段着色器（我们的例子）
visibility: GPUShaderStage.FRAGMENT

// 顶点和片段着色器都可见
visibility: GPUShaderStage.VERTEX | GPUShaderStage.FRAGMENT

// 所有阶段
visibility: GPUShaderStage.VERTEX | GPUShaderStage.FRAGMENT | GPUShaderStage.COMPUTE
```

**为什么选择 FRAGMENT？**

在我们的示例中，uniform 数据（时间、鼠标位置）只在 [`fragmentMain`](study/3.wgsl_mini_demo/index.html:137-163) 中使用：

```wgsl
@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
    let t = uniforms.time;        // 使用 time
    let g = cos(uv.y * 3.14159 + uniforms.mouseY * 6.28);  // 使用 mouseY
    let b = sin(uniforms.mouseX * 6.28 + t * 0.5);         // 使用 mouseX
    // ...
}
```

如果顶点着色器也需要访问，应该使用：
```javascript
visibility: GPUShaderStage.VERTEX | GPUShaderStage.FRAGMENT
```

#### `buffer`
- **类型**：`GPUBufferBindingLayout`
- **作用**：描述 buffer 的属性

**buffer 类型选项：**

```javascript
// uniform buffer（我们的例子）
buffer: { type: 'uniform' }

// read-only storage buffer
buffer: { type: 'read-only-storage' }

// storage buffer（可读写）
buffer: { type: 'storage' }

// 带最小绑定大小
buffer: {
    type: 'uniform',
    minBindingSize: 16  // 最小 16 字节
}
```

### 5.4.3 创建 Pipeline Layout

```javascript
const pipelineLayout = device.createPipelineLayout({
    label: 'Pipeline Layout',
    bindGroupLayouts: [bindGroupLayout]  // 数组：可以有多个 group
});
```

**多个 Bind Group 的示例：**

```javascript
// 创建多个 bind group layout
const uniformLayout = device.createBindGroupLayout({...});    // @group(0)
const textureLayout = device.createBindGroupLayout({...});    // @group(1)
const storageLayout = device.createBindGroupLayout({...});    // @group(2)

const pipelineLayout = device.createPipelineLayout({
    bindGroupLayouts: [
        uniformLayout,   // @group(0)
        textureLayout,   // @group(1)
        storageLayout,   // @group(2)
    ]
});
```

对应的 WGSL 代码：
```wgsl
@group(0) @binding(0) var<uniform> uniforms: Uniforms;
@group(1) @binding(0) var myTexture: texture_2d<f32>;
@group(1) @binding(1) var mySampler: sampler;
@group(2) @binding(0) var<storage, read_write> data: array<f32>;
```

### 5.4.4 创建 Bind Group

在 [`index.html`](study/3.wgsl_mini_demo/index.html:202-209) 中：

```javascript
const bindGroup = device.createBindGroup({
    label: 'Bind Group',
    layout: bindGroupLayout,
    entries: [{
        binding: 0,
        resource: { buffer: uniformBuffer }
    }]
});
```

**Bind Group vs Bind Group Layout：**

| 概念 | 作用 | 类比 |
|------|------|------|
| Bind Group Layout | 定义资源的**类型**和**布局** | 蓝图/接口 |
| Bind Group | 绑定**实际的资源** | 实例/实现 |

```javascript
// Layout：定义"有一个 uniform buffer"
const layout = device.createBindGroupLayout({
    entries: [{ binding: 0, visibility: ..., buffer: { type: 'uniform' } }]
});

// Bind Group：指定"具体是哪个 buffer"
const bindGroup = device.createBindGroup({
    layout: layout,
    entries: [{ binding: 0, resource: { buffer: uniformBuffer } }]
});
```

### 5.4.5 完整的绑定示例

```javascript
// ========== 步骤 1：创建 buffer ==========
const uniformBuffer = device.createBuffer({
    size: 16,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});

// ========== 步骤 2：创建 bind group layout ==========
const bindGroupLayout = device.createBindGroupLayout({
    entries: [{
        binding: 0,
        visibility: GPUShaderStage.FRAGMENT,
        buffer: { type: 'uniform' }
    }]
});

// ========== 步骤 3：创建 pipeline layout ==========
const pipelineLayout = device.createPipelineLayout({
    bindGroupLayouts: [bindGroupLayout]
});

// ========== 步骤 4：创建 bind group ==========
const bindGroup = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [{
        binding: 0,
        resource: { buffer: uniformBuffer }
    }]
});

// ========== 步骤 5：在渲染时使用 ==========
renderPass.setBindGroup(0, bindGroup);  // 设置 @group(0)
```

---

## 5.5 数据更新机制

### 5.5.1 writeBuffer 详解

在 [`index.html`](study/3.wgsl_mini_demo/index.html:263-269) 的渲染循环中：

```javascript
function frame(timestamp) {
    const time = timestamp / 1000.0;

    // 更新 uniform buffer 数据
    const uniformData = new Float32Array([
        time,    // time
        mouseX,  // mouseX
        mouseY,  // mouseY
        0.0      // padding
    ]);
    device.queue.writeBuffer(uniformBuffer, 0, uniformData);

    // ... 渲染代码
}
```

**writeBuffer API：**

```javascript
device.queue.writeBuffer(
    buffer,        // 目标 buffer
    bufferOffset,  // buffer 中的偏移量（字节）
    data,          // 源数据（TypedArray 或 ArrayBuffer）
    dataOffset,    // 可选：data 中的偏移量
    size           // 可选：复制的大小
);
```

**参数示例：**

```javascript
const data = new Float32Array([1, 2, 3, 4, 5, 6, 7, 8]);

// 示例 1：写入全部数据
device.queue.writeBuffer(buffer, 0, data);

// 示例 2：只写入前 4 个元素（16 字节）
device.queue.writeBuffer(buffer, 0, data, 0, 4);

// 示例 3：从 buffer 的第 16 字节开始写入
device.queue.writeBuffer(buffer, 16, data);

// 示例 4：写入 data 的第 2-5 个元素
device.queue.writeBuffer(buffer, 0, data, 2, 4);
```

### 5.5.2 Float32Array 的使用

**为什么使用 Float32Array？**

1. **类型匹配**：WGSL 的 `f32` 对应 JavaScript 的 32 位浮点数
2. **内存效率**：直接映射到 GPU 内存格式
3. **性能优化**：避免类型转换

**创建方式：**

```javascript
// 方式 1：从数组创建
const data1 = new Float32Array([1.0, 2.0, 3.0, 4.0]);

// 方式 2：指定长度，初始化为 0
const data2 = new Float32Array(4);
data2[0] = 1.0;
data2[1] = 2.0;
data2[2] = 3.0;
data2[3] = 4.0;

// 方式 3：从 ArrayBuffer 创建
const buffer = new ArrayBuffer(16);  // 16 字节
const data3 = new Float32Array(buffer);
```

**其他 TypedArray 类型：**

| JavaScript 类型 | WGSL 类型 | 字节/元素 | 用途 |
|-----------------|-----------|-----------|------|
| `Float32Array` | `f32` | 4 | 浮点数（最常用） |
| `Int32Array` | `i32` | 4 | 有符号整数 |
| `Uint32Array` | `u32` | 4 | 无符号整数 |
| `Int16Array` | `i16` | 2 | 短整数 |
| `Uint16Array` | `u16` | 2 | 无符号短整数 |
| `Uint8Array` | - | 1 | 字节数据 |

### 5.5.3 实时更新策略

**策略 1：每帧更新（我们的例子）**

```javascript
function frame(timestamp) {
    // 每帧都更新
    const uniformData = new Float32Array([time, mouseX, mouseY, 0.0]);
    device.queue.writeBuffer(uniformBuffer, 0, uniformData);

    // 渲染
    renderPass.draw(6);
    requestAnimationFrame(frame);
}
```

**优点**：简单直接，适合频繁变化的数据
**缺点**：每帧都有 CPU 到 GPU 的数据传输开销

**策略 2：按需更新**

```javascript
let isDirty = false;
let cachedData = new Float32Array(4);

function updateUniforms(time, mouseX, mouseY) {
    cachedData[0] = time;
    cachedData[1] = mouseX;
    cachedData[2] = mouseY;
    isDirty = true;
}

function frame() {
    if (isDirty) {
        device.queue.writeBuffer(uniformBuffer, 0, cachedData);
        isDirty = false;
    }
    // 渲染
}
```

**优点**：减少不必要的数据传输
**缺点**：需要额外的状态管理

**策略 3：部分更新**

```javascript
// 假设有一个大的 uniform buffer：
// struct Uniforms {
//     constantData: vec4f,   // 偏移 0，很少变化
//     dynamicData: vec4f,    // 偏移 16，频繁变化
// }

// 只更新动态部分
const dynamicData = new Float32Array([time, mouseX, mouseY, 0.0]);
device.queue.writeBuffer(uniformBuffer, 16, dynamicData);  // 偏移 16 字节
```

**优点**：最小化数据传输
**缺点**：需要仔细计算偏移量

### 5.5.4 性能优化建议

**1. 重用 TypedArray**

```javascript
// ❌ 不好：每帧创建新数组
function badFrame() {
    const data = new Float32Array([time, x, y, 0]);  // 每帧分配内存
    device.queue.writeBuffer(buffer, 0, data);
}

// ✅ 好：重用数组
const uniformData = new Float32Array(4);
function goodFrame() {
    uniformData[0] = time;
    uniformData[1] = x;
    uniformData[2] = y;
    device.queue.writeBuffer(buffer, 0, uniformData);
}
```

**2. 批量更新**

```javascript
// ❌ 不好：多次 writeBuffer 调用
device.queue.writeBuffer(buffer1, 0, data1);
device.queue.writeBuffer(buffer2, 0, data2);
device.queue.writeBuffer(buffer3, 0, data3);

// ✅ 好：合并到一个大 buffer
const combinedData = new Float32Array([...data1, ...data2, ...data3]);
device.queue.writeBuffer(combinedBuffer, 0, combinedData);
```

**3. 避免不必要的更新**

```javascript
let lastTime = 0;
const UPDATE_INTERVAL = 16;  // 约 60 FPS

function frame(timestamp) {
    if (timestamp - lastTime > UPDATE_INTERVAL) {
        // 只在需要时更新
        device.queue.writeBuffer(uniformBuffer, 0, uniformData);
        lastTime = timestamp;
    }
}
```

---

## 5.6 着色器中的使用

### 5.6.1 @group 和 @binding 装饰器

在 [`index.html`](study/3.wgsl_mini_demo/index.html:98) 中：

```wgsl
@group(0) @binding(0) var<uniform> uniforms: Uniforms;
```

**语法解析：**

```wgsl
@group(N)        // 对应 setBindGroup(N, ...)
@binding(M)      // 对应 entries[M]
var<storage_class> name: Type;
```

**示例：**

```wgsl
// 示例 1：uniform buffer
@group(0) @binding(0) var<uniform> myUniforms: Uniforms;

// 示例 2：texture 和 sampler
@group(1) @binding(0) var myTexture: texture_2d<f32>;
@group(1) @binding(1) var mySampler: sampler;

// 示例 3：storage buffer
@group(2) @binding(0) var<storage, read_write> data: array<f32>;
```

**对应的 JavaScript 绑定：**

```javascript
// WGSL: @group(0) @binding(0)
renderPass.setBindGroup(0, bindGroup0);

// WGSL: @group(1) @binding(0) 和 @binding(1)
renderPass.setBindGroup(1, bindGroup1);

// WGSL: @group(2) @binding(0)
renderPass.setBindGroup(2, bindGroup2);
```

### 5.6.2 Uniform 变量访问

在片段着色器中使用 uniform 数据：

```wgsl
@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
    // 直接访问 uniform 成员
    let t = uniforms.time;        // 读取时间
    let mx = uniforms.mouseX;     // 读取鼠标 X
    let my = uniforms.mouseY;     // 读取鼠标 Y

    // 使用 uniform 数据计算颜色
    let r = sin(uv.x * 3.14159 + t) * 0.5 + 0.5;
    let g = cos(uv.y * 3.14159 + my * 6.28) * 0.5 + 0.5;
    let b = sin(mx * 6.28 + t * 0.5) * 0.5 + 0.5;

    return vec4f(r, g, b, 1.0);
}
```

**访问规则：**
- ✅ 可以读取：`let value = uniforms.time;`
- ❌ 不可修改：`uniforms.time = 1.0;`（编译错误）
- ✅ 可以传递：`myFunction(uniforms.time);`

### 5.6.3 复杂数据结构访问

```wgsl
// 嵌套结构体
struct Material {
    color: vec4f,
    roughness: f32,
    metallic: f32,
    padding: vec2f,
}

struct Uniforms {
    mvpMatrix: mat4x4f,
    material: Material,
}

@group(0) @binding(0) var<uniform> uniforms: Uniforms;

@fragment
fn fragmentMain() -> @location(0) vec4f {
    // 访问嵌套成员
    let color = uniforms.material.color;
    let roughness = uniforms.material.roughness;

    // 访问矩阵
    let transformedPos = uniforms.mvpMatrix * vec4f(position, 1.0);

    return color;
}
```

**数组访问：**

```wgsl
struct Uniforms {
    lightPositions: array<vec4f, 4>,  // 4 个光源位置
}

@group(0) @binding(0) var<uniform> uniforms: Uniforms;

@fragment
fn fragmentMain() -> @location(0) vec4f {
    // 访问数组元素
    let light0 = uniforms.lightPositions[0];
    let light1 = uniforms.lightPositions[1];

    // 循环访问
    var totalLight = vec3f(0.0);
    for (var i = 0u; i < 4u; i++) {
        totalLight += uniforms.lightPositions[i].xyz;
    }

    return vec4f(totalLight, 1.0);
}
```

---

## 5.7 完整的数据传递流程

### 5.7.1 流程图

```
┌─────────────────────────────────────────────────────────┐
│ 1. JavaScript：定义数据                                  │
│    const time = timestamp / 1000.0;                     │
│    const uniformData = new Float32Array([time, x, y, 0]);│
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 2. CPU → GPU：传输数据                                   │
│    device.queue.writeBuffer(uniformBuffer, 0, data);    │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 3. GPU：存储在 Uniform Buffer                            │
│    ┌─────────┬─────────┬─────────┬─────────┐           │
│    │ time    │ mouseX  │ mouseY  │ padding │           │
│    └─────────┴─────────┴─────────┴─────────┘           │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 4. Bind Group：绑定到着色器                              │
│    renderPass.setBindGroup(0, bindGroup);               │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 5. WGSL：读取数据                                        │
│    @group(0) @binding(0) var<uniform> uniforms: ...;   │
│    let t = uniforms.time;                               │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 6. 着色器：使用数据计算                                   │
│    let color = sin(t) * 0.5 + 0.5;                      │
└─────────────────────────────────────────────────────────┘
```

### 5.7.2 完整代码示例

**JavaScript 端：**

```javascript
// ========== 1. 创建 Uniform Buffer ==========
const uniformBuffer = device.createBuffer({
    label: 'Uniforms',
    size: 16,  // 4 × f32
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});

// ========== 2. 创建 Bind Group Layout ==========
const bindGroupLayout = device.createBindGroupLayout({
    entries: [{
        binding: 0,
        visibility: GPUShaderStage.FRAGMENT,
        buffer: { type: 'uniform' }
    }]
});

// ========== 3. 创建 Bind Group ==========
const bindGroup = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [{
        binding: 0,
        resource: { buffer: uniformBuffer }
    }]
});

// ========== 4. 渲染循环中更新 ==========
function frame(timestamp) {
    const time = timestamp / 1000.0;

    // 准备数据
    const uniformData = new Float32Array([
        time, mouseX, mouseY, 0.0
    ]);

    // 传输到 GPU
    device.queue.writeBuffer(uniformBuffer, 0, uniformData);

    // 渲染
    const commandEncoder = device.createCommandEncoder();
    const renderPass = commandEncoder.beginRenderPass({...});

    renderPass.setPipeline(pipeline);
    renderPass.setBindGroup(0, bindGroup);  // 绑定 uniform
    renderPass.draw(6);
    renderPass.end();

    device.queue.submit([commandEncoder.finish()]);
    requestAnimationFrame(frame);
}
```

**WGSL 端：**

```wgsl
// ========== 1. 定义结构体 ==========
struct Uniforms {
    time: f32,
    mouseX: f32,
    mouseY: f32,
    padding: f32,
}

// ========== 2. 绑定 Uniform ==========
@group(0) @binding(0) var<uniform> uniforms: Uniforms;

// ========== 3. 在着色器中使用 ==========
@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
    let t = uniforms.time;
    let color = vec3f(
        sin(input.uv.x * 3.14 + t) * 0.5 + 0.5,
        cos(input.uv.y * 3.14 + uniforms.mouseY * 6.28) * 0.5 + 0.5,
        sin(uniforms.mouseX * 6.28 + t * 0.5) * 0.5 + 0.5
    );
    return vec4f(color, 1.0);
}
```

---

## 5.8 实践练习

### 5.8.1 练习 1：添加新的 Uniform 参数

**任务**：在现有示例基础上添加一个 `scale` 参数，用于缩放图形。

**步骤：**

1. **修改 WGSL 结构体**：
```wgsl
struct Uniforms {
    time: f32,
    mouseX: f32,
    mouseY: f32,
    scale: f32,  // 新增：缩放参数
}
```

2. **修改 Buffer 大小**（无需改变，仍然是 16 字节）

3. **更新 JavaScript 数据**：
```javascript
const uniformData = new Float32Array([
    time,
    mouseX,
    mouseY,
    1.0 + Math.sin(time) * 0.5  // 动态缩放
]);
```

4. **在着色器中使用**：
```wgsl
@vertex
fn vertexMain(@builtin(vertex_index) vertexIndex: u32) -> VertexOutput {
    var pos = positions[vertexIndex];
    pos = pos * uniforms.scale;  // 应用缩放
    output.position = vec4f(pos, 0.0, 1.0);
    return output;
}
```

### 5.8.2 练习 2：多 Uniform Buffer

**任务**：创建两个独立的 Uniform Buffer，一个用于时间，一个用于鼠标。

**步骤：**

1. **创建两个 Buffer**：
```javascript
const timeBuffer = device.createBuffer({
    size: 16,  // vec4f (time, deltaTime, frameCount, padding)
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});

const mouseBuffer = device.createBuffer({
    size: 16,  // vec4f (x, y, clickX, clickY)
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});
```

2. **修改 Bind Group Layout**：
```javascript
const bindGroupLayout = device.createBindGroupLayout({
    entries: [
        {
            binding: 0,
            visibility: GPUShaderStage.VERTEX | GPUShaderStage.FRAGMENT,
            buffer: { type: 'uniform' }
        },
        {
            binding: 1,
            visibility: GPUShaderStage.FRAGMENT,
            buffer: { type: 'uniform' }
        }
    ]
});
```

3. **WGSL 定义**：
```wgsl
struct TimeUniforms {
    time: f32,
    deltaTime: f32,
    frameCount: f32,
    padding: f32,
}

struct MouseUniforms {
    position: vec2f,
    clickPosition: vec2f,
}

@group(0) @binding(0) var<uniform> timeData: TimeUniforms;
@group(0) @binding(1) var<uniform> mouseData: MouseUniforms;
```

### 5.8.3 练习 3：调试数据传递

**常见问题和解决方案：**

**问题 1：数据没有更新**

```javascript
// ❌ 错误：忘记 COPY_DST 标志
const buffer = device.createBuffer({
    size: 16,
    usage: GPUBufferUsage.UNIFORM,  // 缺少 COPY_DST
});

// ✅ 正确
const buffer = device.createBuffer({
    size: 16,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});
```

**问题 2：对齐错误**

```javascript
// ❌ 错误：12 字节，不符合 16 字节对齐
const data = new Float32Array([time, x, y]);  // 12 字节

// ✅ 正确：添加 padding
const data = new Float32Array([time, x, y, 0.0]);  // 16 字节
```

**问题 3：binding 不匹配**

```wgsl
// WGSL 中
@group(0) @binding(1) var<uniform> uniforms: Uniforms;
```

```javascript
// JavaScript 中
const bindGroup = device.createBindGroup({
    entries: [{
        binding: 0,  // ❌ 错误：应该是 1
        resource: { buffer: uniformBuffer }
    }]
});
```

### 5.8.4 调试技巧

**1. 使用 console.log 验证数据**

```javascript
const uniformData = new Float32Array([time, mouseX, mouseY, 0.0]);
console.log('Uniform data:', uniformData);  // 检查数据值
device.queue.writeBuffer(uniformBuffer, 0, uniformData);
```

**2. 在着色器中输出调试颜色**

```wgsl
@fragment
fn fragmentMain() -> @location(0) vec4f {
    // 将 uniform 值映射到颜色，便于观察
    return vec4f(
        uniforms.time * 0.1,      // R 通道显示时间
        uniforms.mouseX,          // G 通道显示鼠标 X
        uniforms.mouseY,          // B 通道显示鼠标 Y
        1.0
    );
}
```

**3. 验证 Buffer 大小**

```javascript
console.log('Buffer size:', uniformBuffer.size);  // 应该是 16
console.log('Data byte length:', uniformData.byteLength);  // 也应该是 16
```

---

## 5.9 性能优化和最佳实践

### 5.9.1 Uniform Buffer 大小限制

不同硬件有不同的 uniform buffer 大小限制：

```javascript
// 查询设备限制
const limits = device.limits;
console.log('Max uniform buffer size:', limits.maxUniformBufferBindingSize);
// 通常是 64KB (65536 字节)

console.log('Max uniform buffers per stage:', limits.maxUniformBuffersPerShaderStage);
// 通常是 12
```

**建议：**
- 单个 uniform buffer 保持在 256-512 字节以内
- 超过 1KB 的数据考虑使用 storage buffer

### 5.9.2 更新频率优化

**按更新频率分组：**

```javascript
// 方案 1：混合在一起（不推荐）
struct Uniforms {
    mvpMatrix: mat4x4f,    // 每次相机移动更新
    lightColor: vec4f,     // 很少更新
    time: f32,             // 每帧更新
}

// 方案 2：分开 buffer（推荐）
// Buffer 1：每帧更新
struct PerFrameUniforms {
    time: f32,
    deltaTime: f32,
    frameCount: f32,
    padding: f32,
}

// Buffer 2：相机移动时更新
struct PerCameraUniforms {
    viewMatrix: mat4x4f,
    projMatrix: mat4x4f,
}

// Buffer 3：很少更新
struct SceneUniforms {
    lightColor: vec4f,
    ambientColor: vec4f,
}
```

### 5.9.3 内存对齐最佳实践

**使用 vec4f 简化对齐：**

```wgsl
// ❌ 复杂：需要手动计算 padding
struct BadUniforms {
    a: f32,        // 0-3
    b: f32,        // 4-7
    c: f32,        // 8-11
    padding1: f32, // 12-15
    d: vec3f,      // 16-27
    padding2: f32, // 28-31
}

// ✅ 简单：vec4f 自动对齐
struct GoodUniforms {
    abc: vec4f,    // a, b, c, padding
    d: vec4f,      // d.x, d.y, d.z, padding
}
```

### 5.9.4 数据更新策略总结

| 策略 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| **每帧更新** | 动画参数 | 简单直接 | CPU 开销大 |
| **按需更新** | 交互参数 | 减少传输 | 需要状态管理 |
| **部分更新** | 大 buffer | 最小传输 | 计算复杂 |
| **双缓冲** | 复杂场景 | 避免同步 | 内存占用 |

---

## 5.10 总结

### 5.10.1 关键要点

1. **Uniform Buffer 是 CPU 到 GPU 数据传递的核心机制**
   - 用于传递全局参数（时间、变换矩阵等）
   - 所有顶点/片段共享相同的值

2. **16 字节对齐是硬性要求**
   - 使用 padding 字段确保对齐
   - 优先使用 vec4f 简化对齐

3. **完整的绑定流程**
   ```
   创建 Buffer → 创建 Layout → 创建 Bind Group → 设置到渲染通道
   ```

4. **性能优化关键**
   - 重用 TypedArray，避免频繁分配
   - 按更新频率分组 buffer
   - 只更新必要的数据

### 5.10.2 下一步

在下一章中，我们将学习：
- **纹理和采样器**：如何加载和使用图片
- **纹理坐标**：UV 映射原理
- **采样技术**：过滤和包裹模式

Uniform Buffer 为纹理采样提供了参数控制的基础！

---

## 附录：快速参考

### API 速查表

```javascript
// 创建 Uniform Buffer
device.createBuffer({
    size: 16,  // 字节数
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});

// 更新数据
const data = new Float32Array([1, 2, 3, 4]);
device.queue.writeBuffer(buffer, 0, data);

// Bind Group Layout
device.createBindGroupLayout({
    entries: [{
        binding: 0,
        visibility: GPUShaderStage.FRAGMENT,
        buffer: { type: 'uniform' }
    }]
});

// Bind Group
device.createBindGroup({
    layout: layout,
    entries: [{
        binding: 0,
        resource: { buffer: uniformBuffer }
    }]
});
```

### WGSL 语法速查

```wgsl
// 定义结构体
struct Uniforms {
    data: vec4f,
}

// 绑定 uniform
@group(0) @binding(0) var<uniform> uniforms: Uniforms;

// 使用
let value = uniforms.data.x;
```

### 常见错误速查

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 数据不更新 | 缺少 COPY_DST | 添加 `| GPUBufferUsage.COPY_DST` |
| 对齐错误 | 不满足 16 字节 | 添加 padding 字段 |
| binding 不匹配 | JS 和 WGSL 不一致 | 检查 @binding(N) 值 |
| visibility 错误 | 着色器类型不匹配 | 检查 VERTEX/FRAGMENT 标志 |

---

**本章完成！** 你已经掌握了 WebGPU 中最重要的数据传递机制。🎉
