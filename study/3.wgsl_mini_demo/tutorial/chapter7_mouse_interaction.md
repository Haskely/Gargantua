# 第七章：鼠标交互与实时响应

在前面的章节中，我们学习了WebGPU的基础概念、顶点着色器、片段着色器、Uniform Buffer和渲染循环。现在是时候让我们的WebGPU应用变得更加生动和具有交互性了。本章将深入探讨如何在WebGPU应用中实现鼠标交互，让用户的操作能够实时影响GPU渲染的视觉效果。

## 7.1 鼠标交互基础概念

### 7.1.1 Web中的事件驱动编程

在Web开发中，用户与页面的交互是通过**事件系统**来实现的。事件驱动编程是一种编程范式，程序的执行流程由事件（如鼠标移动、点击、键盘按键等）来决定。

**核心概念：**

1. **事件源（Event Source）**：触发事件的对象，如canvas元素
2. **事件类型（Event Type）**：事件的种类，如`'mousemove'`、`'click'`等
3. **事件监听器（Event Listener）**：响应事件的函数
4. **事件对象（Event Object）**：包含事件详细信息的对象

```javascript
// 事件驱动的基本模式
element.addEventListener('eventType', (eventObject) => {
    // 处理事件的代码
});
```

### 7.1.2 GPU渲染中的交互响应机制

在传统的CPU渲染中，交互响应相对简单：事件发生 → 处理数据 → 重新绘制。但在GPU渲染中，数据流动的路径更复杂：

```
用户操作（鼠标移动）
    ↓
JavaScript事件处理器
    ↓
更新JavaScript变量（mouseX, mouseY）
    ↓
渲染循环读取变量
    ↓
写入Uniform Buffer
    ↓
传递给GPU着色器
    ↓
着色器使用数据计算颜色
    ↓
渲染结果显示在屏幕
```

这个流程的关键在于**数据同步**：JavaScript层的数据变化需要及时传递到GPU，这就是我们在第五章学习的Uniform Buffer的作用。

### 7.1.3 鼠标事件的特点

**mousemove事件**是最常用的交互事件之一，它具有以下特点：

1. **高频触发**：鼠标移动时会连续触发，频率可达每秒数十甚至上百次
2. **实时性**：事件几乎立即触发，延迟极低
3. **坐标信息**：提供详细的位置信息（屏幕坐标、客户端坐标等）
4. **性能影响**：高频触发可能影响性能，需要优化

## 7.2 鼠标事件处理详解

让我们深入分析[`study/3.wgsl_mini_demo/index.html`](study/3.wgsl_mini_demo/index.html:251)中的鼠标事件处理代码（第246-255行）：

```javascript
// 鼠标位置状态（归一化到 0-1）
let mouseX = 0.5;
let mouseY = 0.5;

// 监听鼠标移动事件
canvas.addEventListener('mousemove', (e) => {
    const rect = canvas.getBoundingClientRect();
    mouseX = (e.clientX - rect.left) / rect.width;
    mouseY = (e.clientY - rect.top) / rect.height;
});
```

### 7.2.1 全局状态变量的声明

```javascript
let mouseX = 0.5;
let mouseY = 0.5;
```

**为什么初始化为0.5？**

- 0.5表示canvas的中心位置（归一化坐标系中）
- 在用户还未移动鼠标时，提供一个合理的默认值
- 确保着色器中的计算不会出现异常值

**变量作用域：**

这两个变量声明在[`render()`](study/3.wgsl_mini_demo/index.html:238)函数的作用域内，但在事件监听器和渲染循环函数中都可以访问。这是JavaScript闭包的典型应用。

### 7.2.2 addEventListener方法详解

```javascript
canvas.addEventListener('mousemove', (e) => { ... });
```

**[`addEventListener()`](study/3.wgsl_mini_demo/index.html:251)的三个参数：**

1. **事件类型**：`'mousemove'` - 鼠标移动事件
2. **事件处理函数**：箭头函数 `(e) => { ... }`
3. **选项**（可选）：如`{ passive: true }`等，这里使用默认值

**为什么绑定到canvas而不是document？**

```javascript
canvas.addEventListener('mousemove', ...)  // ✅ 推荐
// vs
document.addEventListener('mousemove', ...) // ❌ 不推荐
```

原因：
- **精确性**：只在canvas区域内响应，避免无关移动
- **性能**：减少事件处理次数
- **逻辑清晰**：交互范围与渲染范围一致

### 7.2.3 事件对象（e）的属性

事件处理函数的参数`e`是一个MouseEvent对象，包含丰富的信息：

```javascript
(e) => {
    // e.clientX - 相对于浏览器视口的X坐标
    // e.clientY - 相对于浏览器视口的Y坐标
    // e.screenX - 相对于屏幕的X坐标
    // e.screenY - 相对于屏幕的Y坐标
    // e.pageX   - 相对于整个文档的X坐标
    // e.pageY   - 相对于整个文档的Y坐标
    // e.offsetX - 相对于目标元素的X坐标
    // e.offsetY - 相对于目标元素的Y坐标
}
```

**为什么使用clientX/clientY？**

- 相对于浏览器视口，是最常用的坐标系统
- 配合[`getBoundingClientRect()`](study/3.wgsl_mini_demo/index.html:252)可以精确计算元素内的位置
- 适合处理滚动页面的场景

## 7.3 坐标系统转换

这是鼠标交互中最关键也最容易混淆的部分。我们需要将**浏览器坐标**转换为**归一化的canvas坐标**。

### 7.3.1 getBoundingClientRect()方法

```javascript
const rect = canvas.getBoundingClientRect();
```

这个方法返回一个DOMRect对象，包含元素的大小和位置信息：

```javascript
{
    left: 100,    // 元素左边缘相对于视口的距离
    top: 50,      // 元素上边缘相对于视口的距离
    right: 900,   // 元素右边缘相对于视口的距离
    bottom: 650,  // 元素下边缘相对于视口的距离
    width: 800,   // 元素宽度
    height: 600,  // 元素高度
    x: 100,       // 等同于left
    y: 50         // 等同于top
}
```

**为什么需要getBoundingClientRect()？**

因为canvas可能不在页面的原点，可能有CSS定位、边距、滚动等影响。rect对象告诉我们canvas在视口中的确切位置和大小。

### 7.3.2 坐标转换的数学原理

```javascript
mouseX = (e.clientX - rect.left) / rect.width;
mouseY = (e.clientY - rect.top) / rect.height;
```

**步骤分解：**

**第一步：计算相对于canvas的像素坐标**

```javascript
const relativeX = e.clientX - rect.left;
const relativeY = e.clientY - rect.top;
```

例如：
- 如果canvas的left = 100，鼠标clientX = 450
- 那么relativeX = 450 - 100 = 350（在canvas内部的第350个像素）

**第二步：归一化到0-1范围**

```javascript
mouseX = relativeX / rect.width;
mouseY = relativeY / rect.height;
```

例如：
- 如果canvas宽度800，relativeX = 400
- 那么mouseX = 400 / 800 = 0.5（canvas中心）

**完整示例：**

假设canvas的位置和大小如下：
- left: 100px, top: 50px
- width: 800px, height: 600px

当鼠标在canvas中心时：
```javascript
e.clientX = 100 + 400 = 500  // 视口X坐标
e.clientY = 50 + 300 = 350   // 视口Y坐标

mouseX = (500 - 100) / 800 = 400 / 800 = 0.5
mouseY = (350 - 50) / 600 = 300 / 600 = 0.5
```

当鼠标在canvas左上角时：
```javascript
e.clientX = 100
e.clientY = 50

mouseX = (100 - 100) / 800 = 0 / 800 = 0.0
mouseY = (50 - 50) / 600 = 0 / 600 = 0.0
```

当鼠标在canvas右下角时：
```javascript
e.clientX = 900
e.clientY = 650

mouseX = (900 - 100) / 800 = 800 / 800 = 1.0
mouseY = (650 - 50) / 600 = 600 / 600 = 1.0
```

### 7.3.3 坐标系统对比图

```
浏览器视口坐标系（clientX, clientY）:
(0,0) ─────────────────────► X
  │   ┌─────────────┐
  │   │  页面内容    │
  │   │  ┌────────┐ │
  │   │  │ Canvas │ │  ← canvas.getBoundingClientRect()
  │   │  │ (rect) │ │     告诉我们canvas在视口中的位置
  │   │  └────────┘ │
  │   └─────────────┘
  ▼
  Y

Canvas内部坐标系（相对坐标）:
Canvas左上角(rect.left, rect.top)
  (0,0) ──────────► relativeX
    │   ┌────────┐
    │   │        │
    │   │ Canvas │
    │   │        │
    │   └────────┘
    ▼
  relativeY

归一化坐标系（mouseX, mouseY）:
  (0,0) ────────► mouseX (1.0)
    │   ┌────────┐
    │   │        │
    │   │ Canvas │
    │   │        │
    │   └────────┘
    ▼
  mouseY (1.0)
```

### 7.3.4 坐标归一化的重要性

**为什么要归一化到0-1？**

1. **分辨率无关**：无论canvas是800x600还是1920x1080，鼠标在中心都是(0.5, 0.5)
2. **着色器友好**：GPU着色器通常使用0-1范围的坐标
3. **数学计算简便**：许多图形算法基于0-1范围设计
4. **易于理解**：0表示起点，1表示终点，0.5表示中心

## 7.4 实时数据更新机制

现在我们理解了如何捕获和转换鼠标坐标，接下来看这些数据如何流动到GPU。

### 7.4.1 数据流动路径

```javascript
// 第一步：事件监听器更新全局变量
canvas.addEventListener('mousemove', (e) => {
    mouseX = (e.clientX - rect.left) / rect.width;  // 更新全局变量
    mouseY = (e.clientY - rect.top) / rect.height;
});

// 第二步：渲染循环读取全局变量
function frame(timestamp) {
    // ... 其他代码 ...

    // 第三步：写入Uniform Buffer
    const uniformData = new Float32Array([
        time,    // time
        mouseX,  // 读取全局变量
        mouseY,  // 读取全局变量
        0.0      // padding
    ]);
    device.queue.writeBuffer(uniformBuffer, 0, uniformData);

    // ... 渲染代码 ...
}
```

### 7.4.2 异步更新的工作原理

**关键问题：事件监听器和渲染循环是如何协同工作的？**

```
时间轴：
─────────────────────────────────────────────►
  Event    Frame   Event   Event   Frame   Event
   ↓        ↓       ↓       ↓       ↓       ↓
 mouseX   读取    mouseX  mouseX  读取    mouseX
 = 0.3    0.3    = 0.4   = 0.45   0.45   = 0.5
 更新     并      更新    更新     并      更新
         传给GPU                  传给GPU
```

**工作机制：**

1. **事件监听器**：随时更新`mouseX`和`mouseY`变量
2. **渲染循环**：每帧读取最新的变量值
3. **自动同步**：不需要手动同步，JavaScript的单线程特性保证了数据一致性

### 7.4.3 与Uniform Buffer的结合

在[`frame()`](study/3.wgsl_mini_demo/index.html:258)函数（第258-305行）中，鼠标数据被写入Uniform Buffer：

```javascript
// 更新 uniform buffer 数据
const uniformData = new Float32Array([
    time,    // time: 当前时间（秒）
    mouseX,  // mouseX: 鼠标X坐标（0-1）
    mouseY,  // mouseY: 鼠标Y坐标（0-1）
    0.0      // padding: 对齐填充
]);
device.queue.writeBuffer(uniformBuffer, 0, uniformData);
```

**每一帧都会更新Uniform Buffer**，即使鼠标没有移动：

- 如果鼠标移动了：新的坐标值会被传递
- 如果鼠标没动：传递的是上一次的坐标值
- 这确保了GPU始终有有效的数据

### 7.4.4 内存布局对应关系

**JavaScript侧：**
```javascript
Float32Array([time, mouseX, mouseY, 0.0])
// 索引:      [0]    [1]     [2]     [3]
// 字节偏移:  0      4       8       12
```

**WGSL着色器侧：**
```wgsl
struct Uniforms {
    time: f32,      // 偏移 0, 4字节
    mouseX: f32,    // 偏移 4, 4字节
    mouseY: f32,    // 偏移 8, 4字节
    padding: f32,   // 偏移 12, 4字节
}
```

完美匹配！这就是为什么GPU能正确读取鼠标位置。

## 7.5 着色器中的交互响应

现在数据已经到达GPU，让我们看看着色器如何使用这些数据来创建交互效果。

### 7.5.1 鼠标位置影响颜色计算

在[片段着色器](study/3.wgsl_mini_demo/index.html:136)（第136-163行）中：

```wgsl
@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
    let uv = input.uv;
    let t = uniforms.time;

    // R 通道：基于 UV.x 和时间的正弦波
    let r = sin(uv.x * 3.14159 + t) * 0.5 + 0.5;

    // G 通道：基于 UV.y 和鼠标 Y 位置的余弦波
    let g = cos(uv.y * 3.14159 + uniforms.mouseY * 6.28) * 0.5 + 0.5;

    // B 通道：基于鼠标 X 位置和时间
    let b = sin(uniforms.mouseX * 6.28 + t * 0.5) * 0.5 + 0.5;

    // 波纹效果
    let dist = distance(uv, vec2f(uniforms.mouseX, uniforms.mouseY));
    let wave = sin(dist * 20.0 - t * 3.0) * 0.2 + 0.8;

    // 组合最终颜色
    let color = vec3f(r, g, b) * wave;
    return vec4f(color, 1.0);
}
```

### 7.5.2 G通道对鼠标Y位置的响应

```wgsl
let g = cos(uv.y * 3.14159 + uniforms.mouseY * 6.28) * 0.5 + 0.5;
```

**数学分析：**

1. **`uv.y * 3.14159`**：
   - `uv.y`范围是[0, 1]
   - 乘以π得到[0, π]
   - 创建从上到下的渐变基础

2. **`uniforms.mouseY * 6.28`**：
   - `uniforms.mouseY`范围是[0, 1]
   - 乘以2π（6.28）得到[0, 2π]
   - 这是**相位偏移**，鼠标Y位置控制波的起始位置

3. **`cos(...)`**：
   - 余弦函数，范围[-1, 1]
   - 创建周期性变化

4. **`* 0.5 + 0.5`**：
   - 将[-1, 1]映射到[0, 1]
   - 确保颜色值有效

**交互效果：**
- 鼠标在canvas顶部（mouseY=0）：相位偏移=0
- 鼠标在canvas中部（mouseY=0.5）：相位偏移=π，波形反转
- 鼠标在canvas底部（mouseY=1）：相位偏移=2π，完整周期
- **垂直移动鼠标，会看到绿色波纹上下滚动**

### 7.5.3 B通道对鼠标X位置的响应

```wgsl
let b = sin(uniforms.mouseX * 6.28 + t * 0.5) * 0.5 + 0.5;
```

**特点：**

1. **与UV无关**：整个画面的蓝色值相同
2. **鼠标X控制基础值**：`uniforms.mouseX * 6.28`
3. **时间调制**：`t * 0.5`使蓝色缓慢变化
4. **正弦波动**：创建周期性的蓝色闪烁

**交互效果：**
- 左右移动鼠标，整个画面的蓝色基调会改变
- 同时蓝色还会随时间缓慢脉动

### 7.5.4 波纹效果的鼠标跟随

```wgsl
let dist = distance(uv, vec2f(uniforms.mouseX, uniforms.mouseY));
let wave = sin(dist * 20.0 - t * 3.0) * 0.2 + 0.8;
```

这是最直观的交互效果！

**第一步：计算距离**
```wgsl
let dist = distance(uv, vec2f(uniforms.mouseX, uniforms.mouseY));
```

- `uv`：当前像素的位置（0-1范围）
- `vec2f(uniforms.mouseX, uniforms.mouseY)`：鼠标位置（0-1范围）
- `distance()`：计算两点间的欧几里得距离

**数学公式：**
```
dist = √[(uv.x - mouseX)² + (uv.y - mouseY)²]
```

例如：
- 鼠标在(0.5, 0.5)，像素在(0.5, 0.5)：dist = 0（中心）
- 鼠标在(0.5, 0.5)，像素在(1.0, 0.5)：dist = 0.5
- 鼠标在(0.5, 0.5)，像素在(1.0, 1.0)：dist ≈ 0.707

**第二步：创建波纹**
```wgsl
let wave = sin(dist * 20.0 - t * 3.0) * 0.2 + 0.8;
```

- `dist * 20.0`：距离越远，正弦波频率越高，创建同心圆
- `- t * 3.0`：时间负向偏移，使波纹向外扩散
- `* 0.2 + 0.8`：将[-1, 1]映射到[0.6, 1.0]，保持画面明亮

**第三步：应用波纹**
```wgsl
let color = vec3f(r, g, b) * wave;
```

- 将RGB颜色乘以波纹系数
- wave范围[0.6, 1.0]，所以颜色会在60%-100%之间波动
- 创建从鼠标位置向外扩散的明暗波纹

**视觉效果：**
```
鼠标位置
    ◉  ← 波纹中心
   ╱│╲
  ╱ │ ╲  ← 亮环
 │  │  │
╱   │   ╲ ← 暗环
│   │    │
    ↓     ← 向外扩散
```

移动鼠标，波纹中心会实时跟随！

## 7.6 交互效果的完整流程

让我们把所有部分连接起来，理解一次完整的交互循环：

```
1. 用户移动鼠标到canvas中心
   ↓
2. 浏览器触发mousemove事件
   事件对象：{ clientX: 500, clientY: 350, ... }
   ↓
3. 事件监听器执行
   rect = canvas.getBoundingClientRect()
   → { left: 100, top: 50, width: 800, height: 600 }

   mouseX = (500 - 100) / 800 = 0.5
   mouseY = (350 - 50) / 600 = 0.5
   ↓
4. 渲染循环下一帧执行（约16ms后）
   uniformData = [time, 0.5, 0.5, 0.0]
   ↓
5. 写入Uniform Buffer
   device.queue.writeBuffer(...)
   ↓
6. 顶点着色器执行
   （鼠标数据在这里未使用）
   ↓
7. 片段着色器为每个像素执行
   对于像素uv=(0.6, 0.4):

   dist = distance((0.6,0.4), (0.5,0.5))
        = √[(0.6-0.5)² + (0.4-0.5)²]
        = √[0.01 + 0.01]
        = √0.02 ≈ 0.141

   wave = sin(0.141 * 20.0 - t * 3.0) * 0.2 + 0.8
        = sin(2.82 - t * 3.0) * 0.2 + 0.8

   g = cos(0.4 * π + 0.5 * 2π) * 0.5 + 0.5
     = cos(1.256 + 3.14) * 0.5 + 0.5
     ≈ -0.5 * 0.5 + 0.5 = 0.25

   b = sin(0.5 * 2π + t * 0.5) * 0.5 + 0.5
     = sin(3.14 + t * 0.5) * 0.5 + 0.5

   最终颜色 = (r, 0.25, b) * wave
   ↓
8. 渲染到屏幕
   用户看到以鼠标为中心的波纹效果
```

## 7.7 性能考虑与优化

### 7.7.1 鼠标事件的频率

**问题：** mousemove事件触发频率非常高。

在典型场景下：
- 普通鼠标移动：约30-60次/秒
- 快速移动：可达100+次/秒
- 高刷新率显示器：可能更高

但是：
- 渲染循环频率：60次/秒（60 FPS）
- 或120次/秒（120Hz显示器）

**结论：** 事件频率 ≈ 渲染频率，通常不需要额外优化。

### 7.7.2 何时需要节流（Throttle）

如果你的事件处理器执行**复杂计算**，可能需要节流：

```javascript
let lastUpdateTime = 0;
const UPDATE_INTERVAL = 16; // 约60 FPS

canvas.addEventListener('mousemove', (e) => {
    const now = performance.now();

    // 节流：每16ms最多更新一次
    if (now - lastUpdateTime < UPDATE_INTERVAL) {
        return;
    }

    lastUpdateTime = now;

    const rect = canvas.getBoundingClientRect();
    mouseX = (e.clientX - rect.left) / rect.width;
    mouseY = (e.clientY - rect.top) / rect.height;
});
```

**何时使用：**
- ✅ 事件处理器中有复杂计算
- ✅ 需要发送网络请求
- ✅ 需要更新大量DOM元素
- ❌ 简单的变量赋值（如我们的例子）

### 7.7.3 防抖（Debounce）vs 节流（Throttle）

理解这两种优化技术的区别：

**节流（Throttle）**：
```javascript
// 每隔一段时间执行一次
let lastTime = 0;
canvas.addEventListener('mousemove', (e) => {
    const now = Date.now();
    if (now - lastTime >= 16) {  // 16ms ≈ 60 FPS
        lastTime = now;
        updateMousePosition(e);
    }
});
```

**防抖（Debounce）**：
```javascript
// 只在停止移动后执行
let debounceTimer;
canvas.addEventListener('mousemove', (e) => {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => {
        updateMousePosition(e);
    }, 100);  // 停止100ms后执行
});
```

**使用场景：**
- **节流**：适合连续交互（如我们的鼠标跟随效果）
- **防抖**：适合最终确认（如搜索输入框）

**我们的案例：** 不需要额外优化，因为：
1. 事件处理器只做简单计算
2. 渲染循环自然限制了更新频率
3. GPU并行处理能力强大

### 7.7.4 getBoundingClientRect的性能优化

`getBoundingClientRect()`会触发浏览器的**重排（reflow）**，在高频调用时可能影响性能。

**优化策略：缓存rect对象**

```javascript
// ❌ 不推荐：每次mousemove都调用
canvas.addEventListener('mousemove', (e) => {
    const rect = canvas.getBoundingClientRect();  // 可能触发重排
    mouseX = (e.clientX - rect.left) / rect.width;
    mouseY = (e.clientY - rect.top) / rect.height;
});

// ✅ 推荐：只在需要时更新
let cachedRect = canvas.getBoundingClientRect();

// 窗口大小改变时更新缓存
window.addEventListener('resize', () => {
    cachedRect = canvas.getBoundingClientRect();
});

// 使用缓存的rect
canvas.addEventListener('mousemove', (e) => {
    mouseX = (e.clientX - cachedRect.left) / cachedRect.width;
    mouseY = (e.clientY - cachedRect.top) / cachedRect.height;
});
```

**何时需要：**
- Canvas大小固定：可以缓存
- Canvas响应式布局：需要监听resize事件
- 高性能要求：使用ResizeObserver API

### 7.7.5 内存和GPU优化

**Uniform Buffer更新频率：**

我们当前的实现每帧都更新Uniform Buffer：
```javascript
function frame(timestamp) {
    // 每帧都执行
    device.queue.writeBuffer(uniformBuffer, 0, uniformData);
    // ...
}
```

**这样做是合理的吗？** 是的！

1. **数据量小**：只有16字节（4个float32）
2. **WebGPU优化**：现代GPU擅长处理小量频繁更新
3. **简化逻辑**：无需判断是否需要更新

**如果数据量很大（如大型矩阵数组），可以这样优化：**

```javascript
let lastMouseX = 0.5;
let lastMouseY = 0.5;

function frame(timestamp) {
    // 只在数据变化时更新
    if (mouseX !== lastMouseX || mouseY !== lastMouseY) {
        const uniformData = new Float32Array([
            timestamp / 1000.0,
            mouseX,
            mouseY,
            0.0
        ]);
        device.queue.writeBuffer(uniformBuffer, 0, uniformData);

        lastMouseX = mouseX;
        lastMouseY = mouseY;
    }
    // ...
}
```

## 7.8 扩展交互功能

### 7.8.1 添加鼠标点击事件

让我们扩展代码，添加点击时的波纹爆发效果：

```javascript
// 添加点击状态
let clickEffect = 0.0;  // 0-1，表示效果强度

canvas.addEventListener('click', (e) => {
    const rect = canvas.getBoundingClientRect();
    mouseX = (e.clientX - rect.left) / rect.width;
    mouseY = (e.clientY - rect.top) / rect.height;
    clickEffect = 1.0;  // 触发效果
});

// 在渲染循环中衰减效果
function frame(timestamp) {
    clickEffect *= 0.95;  // 每帧衰减5%
    if (clickEffect < 0.01) clickEffect = 0;

    const uniformData = new Float32Array([
        timestamp / 1000.0,
        mouseX,
        mouseY,
        clickEffect  // 使用第4个字段传递点击效果
    ]);
    // ...
}
```

**WGSL着色器中使用：**

```wgsl
struct Uniforms {
    time: f32,
    mouseX: f32,
    mouseY: f32,
    clickEffect: f32,  // 改名，不再是padding
}

@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
    // ... 原有代码 ...

    // 点击时的脉冲效果
    let dist = distance(uv, vec2f(uniforms.mouseX, uniforms.mouseY));
    let clickPulse = exp(-dist * 10.0) * uniforms.clickEffect;

    // 叠加到颜色
    let color = vec3f(r, g, b) * wave + vec3f(clickPulse);
    return vec4f(color, 1.0);
}
```

### 7.8.2 多点触控支持

在触摸设备上支持多点触控：

```javascript
let touches = new Map();  // 存储多个触摸点

canvas.addEventListener('touchmove', (e) => {
    e.preventDefault();
    const rect = canvas.getBoundingClientRect();

    // 处理所有触摸点
    for (let touch of e.touches) {
        const x = (touch.clientX - rect.left) / rect.width;
        const y = (touch.clientY - rect.top) / rect.height;
        touches.set(touch.identifier, { x, y });
    }

    // 使用第一个触摸点更新鼠标位置
    if (e.touches.length > 0) {
        const first = e.touches[0];
        mouseX = (first.clientX - rect.left) / rect.width;
        mouseY = (first.clientY - rect.top) / rect.height;
    }
});

canvas.addEventListener('touchend', (e) => {
    // 清除结束的触摸点
    for (let touch of e.changedTouches) {
        touches.delete(touch.identifier);
    }
});
```

**传递多点数据到GPU：**

```javascript
// 使用Storage Buffer而不是Uniform Buffer
const maxTouches = 10;
const touchBuffer = device.createBuffer({
    size: maxTouches * 8,  // 每个触摸点2个float32
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
});

// 更新触摸数据
function updateTouchData() {
    const touchData = new Float32Array(maxTouches * 2);
    let i = 0;
    for (let [id, pos] of touches) {
        if (i >= maxTouches) break;
        touchData[i * 2] = pos.x;
        touchData[i * 2 + 1] = pos.y;
        i++;
    }
    device.queue.writeBuffer(touchBuffer, 0, touchData);
}
```

### 7.8.3 键盘交互

结合键盘控制创建更复杂的交互：

```javascript
let interactionMode = 0;  // 0: 波纹, 1: 旋涡, 2: 粒子

window.addEventListener('keydown', (e) => {
    switch(e.key) {
        case '1':
            interactionMode = 0;
            break;
        case '2':
            interactionMode = 1;
            break;
        case '3':
            interactionMode = 2;
            break;
    }
});

// 传递到着色器
const uniformData = new Float32Array([
    timestamp / 1000.0,
    mouseX,
    mouseY,
    interactionMode  // 使用第4个字段传递模式
]);
```

**着色器中根据模式切换效果：**

```wgsl
@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
    let mode = i32(uniforms.clickEffect);  // 复用字段作为模式

    var color: vec3f;

    if (mode == 0) {
        // 波纹模式
        let dist = distance(uv, vec2f(uniforms.mouseX, uniforms.mouseY));
        let wave = sin(dist * 20.0 - t * 3.0) * 0.2 + 0.8;
        color = vec3f(r, g, b) * wave;
    } else if (mode == 1) {
        // 旋涡模式
        let delta = uv - vec2f(uniforms.mouseX, uniforms.mouseY);
        let angle = atan2(delta.y, delta.x);
        let spiral = sin(angle * 5.0 + t) * 0.5 + 0.5;
        color = vec3f(spiral, g, b);
    } else {
        // 粒子模式
        let grid = fract(uv * 20.0);
        let particle = step(0.9, grid.x) * step(0.9, grid.y);
        color = vec3f(particle) * vec3f(r, g, b);
    }

    return vec4f(color, 1.0);
}
```

### 7.8.4 鼠标拖拽效果

实现拖拽绘制轨迹：

```javascript
let isDrawing = false;
let trail = [];  // 存储轨迹点

canvas.addEventListener('mousedown', () => {
    isDrawing = true;
    trail = [];
});

canvas.addEventListener('mousemove', (e) => {
    const rect = canvas.getBoundingClientRect();
    const x = (e.clientX - rect.left) / rect.width;
    const y = (e.clientY - rect.top) / rect.height;

    if (isDrawing) {
        trail.push({ x, y, time: performance.now() });
        // 限制轨迹点数量
        if (trail.length > 100) {
            trail.shift();
        }
    }

    mouseX = x;
    mouseY = y;
});

canvas.addEventListener('mouseup', () => {
    isDrawing = false;
});

// 传递轨迹到GPU（使用Storage Buffer）
function updateTrail() {
    const now = performance.now();
    const trailData = new Float32Array(trail.length * 3);

    trail.forEach((point, i) => {
        trailData[i * 3] = point.x;
        trailData[i * 3 + 1] = point.y;
        trailData[i * 3 + 2] = (now - point.time) / 1000.0;  // 年龄
    });

    device.queue.writeBuffer(trailBuffer, 0, trailData);
}
```

## 7.9 实践练习

### 练习1：鼠标悬停高亮

**任务：** 当鼠标靠近某个区域时，该区域变亮。

**提示：**
```wgsl
let dist = distance(uv, vec2f(uniforms.mouseX, uniforms.mouseY));
let brightness = exp(-dist * 5.0);  // 距离越近越亮
let color = baseColor * (1.0 + brightness);
```

### 练习2：颜色拾取器

**任务：** 点击时改变整个画面的基础色调。

**提示：**
1. 点击时记录鼠标位置
2. 基于位置计算色相（Hue）
3. 使用HSV到RGB的转换

### 练习3：粒子跟随

**任务：** 创建跟随鼠标的粒子系统。

**提示：**
1. 在着色器中基于像素位置生成"粒子"
2. 粒子受鼠标位置吸引
3. 使用噪声函数添加随机性

### 练习4：交互式滤镜

**任务：** 鼠标X控制模糊程度，Y控制饱和度。

**提示：**
```wgsl
let blur = uniforms.mouseX * 0.1;
let saturation = uniforms.mouseY * 2.0;
// 应用到颜色计算
```

### 练习5：多手势识别

**任务：** 识别不同的手势（单击、双击、长按、滑动）。

**提示：**
```javascript
let clickCount = 0;
let lastClickTime = 0;

canvas.addEventListener('click', () => {
    const now = Date.now();
    if (now - lastClickTime < 300) {
        clickCount++;
    } else {
        clickCount = 1;
    }
    lastClickTime = now;

    if (clickCount === 2) {
        // 双击处理
    }
});
```

## 7.10 调试技巧

### 7.10.1 可视化鼠标坐标

在页面上显示实时坐标：

```javascript
const coordsDisplay = document.createElement('div');
coordsDisplay.style.cssText = `
    position: fixed;
    top: 10px;
    right: 10px;
    background: rgba(0,0,0,0.8);
    color: white;
    padding: 10px;
    font-family: monospace;
`;
document.body.appendChild(coordsDisplay);

canvas.addEventListener('mousemove', (e) => {
    // ... 坐标计算 ...

    coordsDisplay.textContent = `
        Screen: (${e.clientX}, ${e.clientY})
        Canvas: (${(mouseX * 800).toFixed(0)}, ${(mouseY * 600).toFixed(0)})
        Normalized: (${mouseX.toFixed(3)}, ${mouseY.toFixed(3)})
    `;
});
```

### 7.10.2 调试着色器中的鼠标数据

在片段着色器中直接可视化鼠标位置：

```wgsl
@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
    // 显示鼠标位置为红点
    let dist = distance(uv, vec2f(uniforms.mouseX, uniforms.mouseY));
    if (dist < 0.05) {
        return vec4f(1.0, 0.0, 0.0, 1.0);  // 红色圆点
    }

    // 显示鼠标坐标为颜色
    return vec4f(uniforms.mouseX, uniforms.mouseY, 0.0, 1.0);
}
```

### 7.10.3 性能监控

监控事件处理和渲染性能：

```javascript
let eventCount = 0;
let frameCount = 0;
let lastStatsTime = performance.now();

canvas.addEventListener('mousemove', () => {
    eventCount++;
    // ... 处理代码 ...
});

function frame(timestamp) {
    frameCount++;

    const now = performance.now();
    if (now - lastStatsTime >= 1000) {
        console.log(`
            Events/sec: ${eventCount}
            FPS: ${frameCount}
            Event/Frame ratio: ${(eventCount / frameCount).toFixed(2)}
        `);

        eventCount = 0;
        frameCount = 0;
        lastStatsTime = now;
    }

    // ... 渲染代码 ...
}
```

## 7.11 常见问题与解决方案

### 问题1：鼠标位置不准确

**症状：** 点击位置和效果位置不匹配。

**原因：**
- Canvas有CSS变换（scale, translate等）
- 页面滚动未考虑
- getBoundingClientRect缓存过期

**解决：**
```javascript
// 考虑CSS变换
const rect = canvas.getBoundingClientRect();
const scaleX = canvas.width / rect.width;
const scaleY = canvas.height / rect.height;

mouseX = ((e.clientX - rect.left) * scaleX) / canvas.width;
mouseY = ((e.clientY - rect.top) * scaleY) / canvas.height;
```

### 问题2：移动设备上无响应

**症状：** 触摸不工作。

**原因：** 没有处理touch事件。

**解决：**
```javascript
// 同时支持鼠标和触摸
function handlePointer(x, y) {
    const rect = canvas.getBoundingClientRect();
    mouseX = (x - rect.left) / rect.width;
    mouseY = (y - rect.top) / rect.height;
}

canvas.addEventListener('mousemove', (e) => {
    handlePointer(e.clientX, e.clientY);
});

canvas.addEventListener('touchmove', (e) => {
    e.preventDefault();
    if (e.touches.length > 0) {
        handlePointer(e.touches[0].clientX, e.touches[0].clientY);
    }
});
```

### 问题3：性能下降

**症状：** FPS降低，卡顿。

**原因：**
- 事件处理器中有复杂计算
- 频繁的DOM操作
- Uniform Buffer过大

**解决：**
```javascript
// 使用requestAnimationFrame同步
let pendingMouseUpdate = false;

canvas.addEventListener('mousemove', (e) => {
    if (!pendingMouseUpdate) {
        pendingMouseUpdate = true;
        requestAnimationFrame(() => {
            // 在下一帧更新
            const rect = canvas.getBoundingClientRect();
            mouseX = (e.clientX - rect.left) / rect.width;
            mouseY = (e.clientY - rect.top) / rect.height;
            pendingMouseUpdate = false;
        });
    }
});
```

### 问题4：内存泄漏

**症状：** 长时间运行后内存占用增加。

**原因：** 事件监听器未清理。

**解决：**
```javascript
// 正确的清理方式
function setupInteraction(canvas) {
    const handleMove = (e) => {
        // ... 处理代码 ...
    };

    canvas.addEventListener('mousemove', handleMove);

    // 返回清理函数
    return () => {
        canvas.removeEventListener('mousemove', handleMove);
    };
}

// 使用
const cleanup = setupInteraction(canvas);

// 需要时清理
// cleanup();
```

## 7.12 最佳实践总结

### 坐标转换
✅ **推荐：**
- 使用getBoundingClientRect()获取准确位置
- 归一化到0-1范围，保持分辨率无关
- 考虑CSS变换和缩放

❌ **避免：**
- 硬编码canvas位置
- 使用screenX/screenY（不可靠）
- 忽略页面滚动

### 事件处理
✅ **推荐：**
- 绑定到canvas而不是document
- 使用箭头函数保持this上下文
- 同时支持鼠标和触摸

❌ **避免：**
- 在事件处理器中执行重计算
- 忘记preventDefault（触摸设备）
- 创建内存泄漏

### 性能优化
✅ **推荐：**
- 缓存getBoundingClientRect结果
- 使用节流避免过度计算
- 监控性能指标

❌ **避免：**
- 过度优化简单代码
- 在不需要时使用复杂节流
- 忽略实际性能瓶颈

### 数据传递
✅ **推荐：**
- 保持Uniform Buffer对齐
- 使用合适的数据类型
- 文档化数据结构

❌ **避免：**
- 传递冗余数据
- 违反对齐规则
- 频繁更新大型缓冲区

## 7.13 本章小结

本章我们深入学习了WebGPU应用中的鼠标交互：

**核心概念：**
1. **事件驱动编程**：理解事件监听器的工作原理
2. **坐标系统转换**：掌握屏幕坐标到归一化坐标的转换
3. **实时数据流动**：从JavaScript到GPU的完整数据路径
4. **着色器交互**：如何在着色器中响应用户输入

**关键技术：**
- [`addEventListener()`](study/3.wgsl_mini_demo/index.html:251)注册事件监听器
- [`getBoundingClientRect()`](study/3.wgsl_mini_demo/index.html:252)获取元素位置
- 坐标归一化算法
- Uniform Buffer实时更新
- 着色器中的距离计算和波纹效果

**实践技能：**
- 实现鼠标跟随效果
- 创建交互式波纹
- 优化高频事件性能
- 调试交互问题

**下一章预告：**

在下一章中，我们将学习**纹理和采样**，了解如何在WebGPU中加载和使用图像纹理，结合鼠标交互创建更丰富的视觉效果。

---

**练习检查清单：**
- [ ] 实现基本的鼠标跟随效果
- [ ] 添加点击触发的脉冲效果
- [ ] 优化坐标转换性能
- [ ] 支持触摸设备
- [ ] 创建自定义交互模式
- [ ] 实现性能监控
- [ ] 解决常见问题

**延伸阅读：**
- [MDN: MouseEvent API](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent)
- [MDN: Touch Events](https://developer.mozilla.org/en-US/docs/Web/API/Touch_events)
- [WebGPU Best Practices](https://toji.dev/webgpu-best-practices/)
- [GPU Gems: Interactive Techniques](https://developer.nvidia.com/gpugems/gpugems/part-i-natural-effects)

恭喜！你现在已经掌握了在WebGPU应用中实现丰富交互的核心技能。继续实验，创造出属于你自己的独特交互效果吧！
