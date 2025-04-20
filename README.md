# Gargantua 黑洞可视化效果

这是一个基于WebGL的交互式黑洞渲染项目，展示了类似于电影《星际穿越》中 Gargantua 黑洞的视觉效果。项目通过片元着色器实现了黑洞引力透镜效应、吸积盘以及其周围的扭曲空间等物理现象的模拟。

## 在线预览

点击 [github.io/Gargantua](https://haskely.github.io/Gargantua/) 查看实时渲染效果。

![previce](./preview.png)

## 原始来源

本项目改编自 Shadertoy 上的着色器作品：
- 原始链接：[https://www.shadertoy.com/view/lstSRS](https://www.shadertoy.com/view/lstSRS)
- 原始作者：[sonicether](https://www.shadertoy.com/user/sonicether)

## 如何本地运行

1. 确保您的系统已安装了支持运行shell脚本的环境
2. 执行以下命令启动本地服务器：
   ```
   sh run_server.sh
   ```
3. 在浏览器中访问服务器提供的URL

或者，您可以使用任何静态网页服务器托管这些文件。

## 项目结构

```
.
├── index.html              # 主要HTML文件，包含着色器代码和JavaScript逻辑
├── README.md               # 本文档
├── run_server.sh           # 启动本地服务器的脚本
├── task.md                 # 项目任务描述
├── texture_sample.html     # 纹理示例文件
└── textures/               # 纹理图像文件夹
    ├── organic_1_1024.jpg  # 有机纹理图像
    └── rgba_noise_medium_256.png  # 噪声纹理图像
```

## 交互控制

- **鼠标移动**：调整视角和黑洞位置
- **FPS滑块**：调整最大帧率（位于屏幕左上角）

## 技术实现

本项目使用以下技术：
- WebGL2 用于GPU加速渲染
- GLSL 着色器语言实现黑洞物理效果
- 纹理图像用于增强视觉效果
- JavaScript 用于管理WebGL上下文和用户交互

### Todo

暂未实现时间平滑和 Bloom HDR 效果。