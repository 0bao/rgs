# RGS · C++ 软件光栅渲染器

> 一个在 Win32 下用 C++17 从零实现的**软件光栅渲染器**：自建光栅化管线、自建数学库、自建多线程任务系统，在此之上实现延迟渲染、PBR / IBL、BVH 光线求交与 **ReSTIR 实时全局光照**。

<p>
  <img alt="language" src="https://img.shields.io/badge/C%2B%2B-17-blue">
  <img alt="platform" src="https://img.shields.io/badge/platform-Windows-lightgrey">
  <img alt="api" src="https://img.shields.io/badge/window%20%2F%20UI-Win32%20%2B%20ImGui-green">
  <img alt="license" src="https://img.shields.io/badge/license-MIT-orange">
</p>

---

## 介绍

RGS 指向两个版本：

- **学习版**：面向渲染管线入门的教学项目，配有[完整视频讲解](https://www.bilibili.com/video/BV1bH4y1L75Q/)。实现了**光栅化渲染管线**（顶点装配 → 顶点着色器 → 图元组装 → 光栅化 → 片段着色器）、**Blinn-Phong 光照模型**、**可自定义着色器**，以及帧缓存、基于 stb_image 的材质、相机、模型加载、自建数学库与 Win32 窗口管理。
- **完整版**（本仓库）：在学习版基础上持续迭代的个人项目，规模更大，不提供教学支持。当前已具备下表能力。

> 项目仍在持续迭代，最新规划见 [`Docs/Roadmap.md`](./Roadmap.md)。

## 特性一览

### 渲染管线

| 能力 | 说明 |
|---|---|
| 可插拔 Pass 框架 | `RenderPass` / `RenderPipeline` + 帧级资源黑板 `FrameResources`，Pass 只声明读写、资源不落 Layer |
| 延迟渲染 | MRT G-Buffer（WorldPos / Normal / Albedo+Metallic / Velocity）+ 全屏光照 Pass |
| 前向路径 | `ForwardOpaquePass` / `ForwardTransparentPass` + `ResolvePass`，与延迟路径并存 |
| 通用 RenderTarget | `Attachment`（RGB32F / RGBA32F / R32F）泛型数据块，采样数下沉为 Attachment 属性（1–8 采样统一路径） |
| 抗锯齿 | MSAA（多样本 Attachment + target-to-target Resolve）与后处理 FXAA |
| HDR 输出 | HDR 中间缓冲 + 色调映射（`HdrPass` / `HdrConfig`） |

### 光照与全局光照

| 能力 | 说明 |
|---|---|
| 着色模型 | Blinn-Phong、PBR、IBL PBR、天空盒、FlatColor |
| IBL 预计算 | irradiance / prefilter mip 链 / BRDF LUT 全套烘焙（`IBLBaker`），支持磁盘缓存、后台烘焙与进度上报 |
| 阴影 | BVH 光线加速结构 + shadow ray 硬阴影；`SampleContext` per-pixel RNG 抖动采样面光源软阴影 |
| ReSTIR DI | Initial / Temporal / Spatial / Shading 四段式直接光重用（`Reservoir`） |
| ReSTIR GI | Initial / Temporal 间接光重用（`GiReservoir`） |
| 时序与降噪 | 速度缓冲 + Temporal Reprojection（邻域校验抑制鬼影）、A-trous 降噪 |

### 场景与资产

| 能力 | 说明 |
|---|---|
| Scene 数据中枢 | `Camera` / `Light` / `MeshInstance` 从 Layer 抽出，由 Pass 共享 |
| glTF 2.0 | 基于 cgltf 的场景加载，支持 PBR 材质、法线贴图、emissive / occlusion |
| 环境贴图管理 | HDR 加载、环境预设与质量档位、`.ibl_cache` 磁盘缓存复用 |
| 纹理与数学 | 基于 stb_image 的纹理（含 wrap 模式）、自建向量 / 矩阵 / 投影数学库 |

### 工程基建

| 能力 | 说明 |
|---|---|
| JobSystem | worker 线程池 + 全局任务队列 + `Event` 同步原语（计数 / 续体 / 一次性三态），光栅化按像素组分发到多线程 |
| 任务链 | 声明式依赖：`Then` / `ThenDispatch` 先注册后派发；`Draw` 采用 fence 式完成信号（调用方预创建 `Event` 传入） |
| 原子深度 | 可选并行深度测试：`LOCK CMPXCHG` 实现深度抢占（CAS），替代逐实例串行化，编译开关 `RGS_ATOMIC_DEPTH` |
| 窗口与 UI | Win32 窗口 + GDI 位图显示；ImGui（D3D11 后端）控制面板，含材质 / 光源 / 相机 / 环境 / 性能面板 |
| Layer 架构 | Layer 栈管理交互与 Demo，支持运行时切换 Demo（`DemoManagerLayer`） |
| 回归测试 | 无头测试目标 `JobSystemTest`（事件链 / 声明式 / fence 语义共 7 项） |

## 内置 Demo

启动后可在 ImGui 的 **Demo** 面板运行时切换：

| Demo | 内容 |
|---|---|
| **Deferred** | 延迟渲染管线端到端：G-Buffer + 全屏光照 + 硬阴影 + IBL / 天空盒 |
| **ReSTIR** | ReSTIR DI + GI 实时全局光照，含时序重用、空间重用、A-trous 降噪与色调映射 |
| **IBL PBR** | 前向 PBR + IBL 预计算材质展示（金属度 / 粗糙度扫掠） |
| **glTF** | glTF 2.0 场景加载与 PBR 材质、法线贴图、emissive / occlusion 展示 |

## 效果

- PBR 球体渲染（金属度 / 粗糙度扫掠）

<div align=center>
  <img src="./assets/pbr_sphere_m000r000.png" alt="pbr_sphere_m00r000" width="200" />
  <img src="./assets/pbr_sphere_m100r000.png" alt="pbr_sphere_m100r000" width="200" />
</div>

<div align=center>
  <img src="./assets/pbr_sphere_m075r025.png" alt="pbr_sphere_m075r025" width="200" />
  <img src="./assets/pbr_sphere_m025r025.png" alt="pbr_sphere_m025r025" width="200" />
</div>

- Blinn-Phong 模型

<div align=center>
  <img src="./assets/blinn.jpg" alt="blinn" width="400" />
</div>

- JobSystem 多线程提速（左：单线程约 8 fps；右：多线程约 67 fps）

<div align=center>
  <img src="./assets/single_thread.png" alt="single_thread" width="800" />
</div>
<div align=center>
  <img src="./assets/jobsystem.png" alt="jobsystem" width="800" />
</div>

- MSAA 抗走样

<div align=center>
  <img src="./assets/msaa.png" alt="msaa" width="800" />
</div>

## 快速开始

**环境要求**

```
CPU：AMD R7-7840HS（测试环境，任意多核 x64 CPU 均可）
操作系统：Windows 10 / 11
编译器：MSVC（VS 2022）或 Clang
构建工具：CMake 3.15+
```

第三方依赖（Dear ImGui v1.91.9b、stb、cgltf v1.14）由 CMake `FetchContent` 自动拉取，无需手动准备。

**构建**

```powershell
git clone https://github.com/seehours/rgs
cd rgs

# 生成 Visual Studio 工程（多配置生成器）
cmake -S . -B build -G "Visual Studio 17 2022" -A x64

# 构建主程序（RelWithDebInfo 为日常开发配置）
cmake --build build --config RelWithDebInfo --target RGS
```

**运行**

```powershell
.\build\RelWithDebInfo\RGS.exe
```

构建后会自动把 `Assets/` 拷贝到输出目录；若仓库内存在 `Nan/vertex_baker/glTF/Lantern.{gltf,bin}`，也会一并复制为 glTF 示例场景。

**回归测试**

```powershell
cmake --build build --config RelWithDebInfo --target JobSystemTest
.\build\RelWithDebInfo\JobSystemTest.exe
```

## 项目结构

```
RGS/src/
├── RGS/
│   ├── Base/            数学库、断言、性能插桩
│   ├── JobSystem.h/.cpp 多线程任务系统（worker 池 + 事件同步 + 任务链）
│   ├── Render/
│   │   ├── Renderer.h   光栅化核心（裁剪、光栅化、深度、MRT、原子深度 CAS）
│   │   ├── RenderTarget/Attachment、MSAA 采样数、Resolve / Blit
│   │   ├── RenderPass / RenderPipeline / FrameResources
│   │   ├── ComputePass / SampleContext（per-pixel RNG）
│   │   ├── IBLBaker / EnvironmentManager（IBL 烘焙与环境缓存）
│   │   ├── Forward/     前向 Pass（Opaque / Transparent / Resolve）
│   │   └── Lighting/ReSTIR/  ReSTIR DI / GI 全部 Pass 与 Reservoir
│   ├── Passes/          G-Buffer、延迟光照、HDR、FXAA、A-trous、时序重投影
│   ├── Shader/          Blinn、PBR、IBL PBR、天空盒、G-Buffer、延迟光照、卷积与 BRDF
│   ├── Scene/           Camera / Light / MeshInstance / Scene / BVH / glTF 加载
│   ├── Layer/           相机、环境、Demo 管理与四个 Demo Layer
│   ├── Texture.h        纹理与 HDR 加载
│   └── Window / Platform / Timer
├── ImGui/               ImGui（D3D11）封装
└── Windows/             Win32 窗口与 GDI 显示

Test/src/                无头回归测试（JobSystemTest 等）
Docs/                    路线图、分步方案文档与算法详解
Assets/                  HDR 环境贴图、纹理、模型
```

## 文档

| 文档 | 内容 |
|---|---|
| [`Docs/Roadmap.md`](./Roadmap.md) | 渲染器通用化改造总方案与路线图（Step 1–11） |
| [`Docs/AtomicDepthCAS_Algorithm.md`](./AtomicDepthCAS_Algorithm.md) | 原子深度 CAS 算法详解（逐段实现对照） |
| [`Docs/AtomicDepthCAS_Plan.md`](./AtomicDepthCAS_Plan.md) | 原子深度并行化方案与 A/B 验证结论 |
| [`Docs/JobSystem_TaskChain_Plan.md`](./JobSystem_TaskChain_Plan.md) | JobSystem 任务链（声明式事件依赖 / fence 式 Draw）方案 |
| `Docs/StepN_*_Plan.md`（N = 1…11） | 分步实施方案：FragmentOutput、RenderTarget、MRT G-Buffer、RayAccelerator、Scene、RenderPipeline、Deferred Lighting、Attachment 采样统一、软阴影、ReSTIR DI |
| [`Docs/ReSTIR_DI_Tutorial.md`](./ReSTIR_DI_Tutorial.md) · [`Docs/ReSTIR_GI_Tutorial.md`](./ReSTIR_GI_Tutorial.md) | ReSTIR DI / GI 原理与实现教程 |

## 参考与致谢

- [zauonlok/renderer](https://github.com/zauonlok/renderer)：从零实现的 C89 软件渲染器
- [LearnOpenGL](https://learnopengl-cn.github.io/)：OpenGL 理论学习资料
- [星河滚烫兮](https://space.bilibili.com/524338786) 的 [openrenderer](https://github.com/1229282331/openrenderer.git) 及其[介绍视频](https://www.bilibili.com/video/BV1vwYdeREEG)
- [Dear ImGui](https://github.com/ocornut/imgui) · [stb](https://github.com/nothings/stb) · [cgltf](https://github.com/jkuhlmann/cgltf)

## 许可证

[MIT License](./../LICENSE) © 2024 seehours
