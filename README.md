# Vulkan C++ 示例和演示

一个全面的开源 C++ 示例集合，用于 [Vulkan](https://www.vulkan.org)，即 Khronos 推出的底层图形与计算 API(https://github.com/SaschaWillems/Vulkan)。

[![捐赠](https://img.shields.io/badge/Donate-PayPal-green.svg)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=BHXPMV6ZKPH9E)

## 目录
+ [2026 年如何使用 Vulkan](#how-to-vulkan-in-2026)
+ [Khronos 官方 Vulkan 示例](#official-khronos-vulkan-samples)
+ [克隆](#Cloning)
+ [资源](#Assets)
+ [构建](#Building)
+ [运行](#Running)
+ [着色器](#Shaders)
+ [同步说明](#a-note-on-synchronization)
+ [示例](#Examples)
    + [基础](#Basics)
    + [glTF](#glTF)
    + [高级](#Advanced)
    + [性能](#Performance)
    + [基于物理的渲染](#physically-based-rendering)
    + [延迟渲染](#Deferred)
    + [计算着色器](#compute-shader)
    + [几何着色器](#geometry-shader)
    + [细分着色器](#tessellation-shader)
    + [硬件加速光线追踪](#hardware-accelerated-ray-tracing)
    + [无头模式](#Headless)
    + [用户界面](#user-interface)
    + [效果](#Effects)
    + [扩展](#Extensions)
    + [其他](#Misc)
+ [致谢与归属](#credits-and-attributions)

<a id="how-to-vulkan-in-2026"></a>
## 2026 年如何使用 Vulkan

如果你想了解 2026 年如何使用 Vulkan 做图形开发，请参阅[这个仓库](https://github.com/SaschaWillems/HowToVulkan)。如果你还没有太多 Vulkan 经验，并且想理解 Vulkan 以及这些示例的工作方式，它会特别有帮助。

<a id="official-khronos-vulkan-samples"></a>
## Khronos 官方 Vulkan 示例

Khronos 已经公开了官方 Vulkan Samples 仓库（[新闻稿](https://www.khronos.org/blog/vulkan-releases-unified-samples-repository?utm_source=Khronos%20Blog&utm_medium=Twitter&utm_campaign=Vulkan%20Repository)）。

你可以在这里找到该仓库：https://github.com/KhronosGroup/Vulkan-Samples

由于我参与了官方仓库的启动工作，从现在开始我会主要向该仓库贡献内容，但仍可能在这里添加不适合放入官方仓库的示例，并且我当然也会继续维护这些示例。

<a id="Cloning"></a>
## 克隆

本仓库包含用于外部依赖和资源的子模块，因此全新克隆时需要递归克隆：

```sh
git clone --recursive https://github.com/SaschaWillems/Vulkan.git
```

已有仓库可以手动更新：

```sh
git submodule init
git submodule update
```

<a id="Building"></a>
## 构建

本仓库包含在 Windows、Android、iOS 和 macOS（使用 MoltenVK）上编译并构建示例所需的全部内容，需要支持 C++20 的 C++ 编译器。

有关不同平台构建方式的详细信息，请参阅 [BUILD.md](BUILD.md)。

<a id="Running"></a>
## 运行

构建完成后，可以从 `bin` 目录运行示例。可用命令行选项列表可通过 `--help` 查看：

```text
 --help: 显示帮助
 -h, --height: 设置窗口高度
 -v, --validation: 启用验证层
 -vs, --vsync: 启用 V-Sync
 -f, --fullscreen: 以全屏模式启动
 -w, --width: 设置窗口宽度
 -s, --shaders: 选择要使用的着色器类型（glsl、slang、hlsl）
 -g, --gpu: 选择用于运行的 GPU
 -gl, --listgpus: 显示可用 Vulkan 设备列表
 -b, --benchmark: 以基准测试模式运行示例
 -bw, --benchwarmup: 设置基准测试模式的预热时间，单位为秒
 -br, --benchruntime: 设置基准测试模式的持续时间，单位为秒
 -bf, --benchfilename: 设置基准测试结果文件名
 -bt, --benchframetimes: 将帧时间保存到基准测试结果文件
 -bfs, --benchmarkframes: 仅渲染给定数量的帧
 -rp, --resourcepath: 设置包含 assets 和 shaders 文件夹的目录路径
```

注意，某些示例需要特定的设备特性。如果你使用的是多 GPU 系统，可能需要使用 `-gl` 和 `-g` 来选择支持这些特性的 GPU。

<a id="Shaders"></a>
## 着色器

Vulkan 使用一种名为 SPIR-V 的中间表示形式来消费着色器。这使得不同着色器语言都可以通过编译到该字节码格式来使用。这里主要使用的着色器语言是 [GLSL](shaders/glsl)，大多数示例也提供 [slang](shaders/slang/) 和 [HLSL](shaders/hlsl) 着色器源代码，便于比较这些着色语言之间的差异。[Rust GPU](https://rust-gpu.github.io/) 项目在一个[独立仓库](https://github.com/Rust-GPU/VulkanShaderExamples/tree/master/shaders/rust)中维护 [Rust](https://www.rust-lang.org/) 着色器源代码。

<a id="Examples"></a>
## 示例

<a id="Basics"></a>
### 基础

- [使用 Vulkan 1.0 的基础三角形](examples/triangle/)

    使用 Vulkan 将彩色三角形渲染到屏幕上的基础且详细的示例。它旨在作为从零开始学习 Vulkan 的起点。大量代码都是样板代码，在后续示例中会被抽象掉。

- [使用 Vulkan 1.3 的基础三角形](examples/trianglevulkan13//)

    基础且详细的彩色三角形渲染示例的 Vulkan 1.3 版本。它使用了动态渲染等特性来简化 API 使用。

- [管线](examples/pipelines/)

    使用管线状态对象（PSO），将状态信息（光栅化状态、剔除模式等）和着色器一起烘焙到单个对象中，使实现更容易优化使用方式（相较于 OpenGL 的动态状态机）。同时演示管线派生的用法。

- [描述符集](examples/descriptorsets)

    描述符用于向着色器绑定点传递数据。该示例设置描述符集、布局和池，基于集合布局创建单个管线，并使用不同描述符集渲染多个对象。

- [动态 uniform 缓冲区](examples/dynamicuniformbuffer/)

    动态 uniform 缓冲区用于渲染多个对象，并将多个矩阵存储在单个 uniform 缓冲区对象中。各个矩阵会在描述符绑定时动态寻址，从而减少所需描述符集数量。

- [Push constants](examples/pushconstants/)

    使用 push constants，也就是存储在命令缓冲区中的小块 uniform 数据，在不需要 uniform 缓冲区的情况下向着色器传递数据。

- [特化常量](examples/specializationconstants/)

    使用 SPIR-V 特化常量，从单个“uber”着色器创建多条具有不同光照路径的管线。

- [纹理映射](examples/texture/)

    从磁盘加载 2D 纹理（包括所有 mip 级别），使用 staging 将其上传到显存，并通过组合图像采样器进行采样。

- [纹理数组](examples/texturearray/)

    加载包含多个 2D 纹理切片的 2D 纹理数组（每个切片都有自己的 mip 链），并渲染多个网格，每个网格从纹理的不同层采样。2D 纹理数组不会在切片之间做插值。

- [立方体贴图纹理](examples/texturecubemap/)

    从磁盘加载包含六个不同面的立方体贴图纹理。所有面和 mip 级别都会上传到显存，cubemap 会作为背景显示在 skybox 上，并作为反射显示在 3D 模型上。

- [立方体贴图数组](examples/texturecubemaparray/)

    从单个文件加载立方体贴图纹理数组。所有立方体贴图及其各个面和 mip 级别都会上传到显存，选中的 cubemap 会作为背景显示在 skybox 上，并作为反射显示在 3D 模型上。

- [3D 纹理](examples/texture3d/)

    在 CPU 上生成 3D 纹理（使用 Perlin 噪声），将其上传到设备并采样以渲染动画。3D 纹理存储体积数据，并在三个维度上进行插值。

- [输入附件](examples/inputattachments)

    使用输入附件，在单个 render pass 中同一像素位置读取前一个 subpass 的帧缓冲内容。它可用于基础后处理或图像合成（[博客文章](https://www.saschawillems.de/tutorials/vulkan/input_attachments_subpasses)）。

- [Subpass](examples/subpasses/)

    高级示例，使用 subpass 和输入附件，在单个 render pass 中向帧缓冲附件写入并回读数据（仅限相同位置）。它用于在单次 pass 中实现带有前向透明度的延迟渲染合成。

- [离屏渲染](examples/offscreen/)

    两个 pass 的基础离屏渲染。第一个 pass 将镜像场景渲染到带有颜色和深度附件的独立帧缓冲；第二个 pass 从该颜色附件采样，用于渲染镜面表面。

- [CPU 粒子系统](examples/particlesystem/)

    实现一个简单的基于 CPU 的粒子系统。粒子数据存储在主机内存中，每帧在 CPU 上更新，并在使用预乘 alpha 渲染前与设备同步。

- [模板缓冲区](examples/stencilbuffer/)

    使用模板缓冲区及其比较功能，为 3D 模型渲染动态轮廓。

- [顶点属性](examples/vertexattributes/)

    演示向顶点着色器传递顶点的两种不同方式：交错顶点属性或独立顶点属性。

<a id="glTF"></a>
### glTF

这些示例详细展示了如何实现 [glTF 2.0 3D 格式](https://www.khronos.org/gltf/)这一 3D 传输文件格式的不同特性。

- [glTF 模型加载和渲染](examples/gltfloading/)

    展示如何从 [glTF 2.0](https://github.com/KhronosGroup/glTF) 文件加载完整场景。glTF 2.0 场景结构会被转换为使用 Vulkan 渲染该场景所需的数据结构。

- [glTF 顶点蒙皮](examples/gltfskinning/)

    演示如何根据存储在 [glTF 2.0](https://github.com/KhronosGroup/glTF) 模型中的动画数据执行 GPU 顶点蒙皮。除了读取顶点蒙皮所需的全部数据结构外，该示例还展示如何将动画数据上传到 GPU，以及如何使用着色器进行渲染。

- [glTF 场景渲染](examples/gltfscenerendering/)

    渲染从 [glTF 2.0](https://github.com/KhronosGroup/glTF) 文件加载的完整场景。该示例基于 glTF 模型加载示例，并增加了渲染更复杂场景所需的数据结构、函数和着色器，使用 Crytek 的 Sponza 模型，支持逐材质管线和法线贴图。

<a id="Advanced"></a>
### 高级

- [多重采样](examples/multisampling/)

    使用带有多重采样附件和 resolve 附件的 render pass 实现多重采样抗锯齿（MSAA），resolve 附件会解析到可见帧缓冲。

- [带 alpha to coverage 的多重采样](examples/multisamplingalphatocoverage/)

    使用带 alpha to coverage 的多重采样，实现一种与顺序无关的重叠透明对象渲染方式。

- [高动态范围](examples/hdr/)

    使用 16/32 位浮点精度作为所有内部格式、纹理和计算的精度，实现高动态范围渲染管线，包括 bloom pass、手动曝光和色调映射。

- [阴影映射](examples/shadowmapping/)

    为方向光源渲染阴影。第一个 pass 从光源视角存储深度值，第二个 pass 与这些深度值比较以判断片段是否处于阴影中。使用 depth bias 避免阴影伪影，并应用 PCF 滤波平滑阴影边缘。

- [级联阴影映射](examples/shadowmappingcascade/)

    使用多个阴影贴图（存储为分层纹理）提升大型场景的阴影分辨率。相机视锥被拆分为多个级联，对应阴影贴图中的不同层。阴影深度比较时的层选择通过将片段深度与各级联深度范围比较完成。

- [全向阴影映射](examples/shadowmappingomni/)

    使用动态浮点立方体贴图，为向所有方向投射阴影的点光源实现阴影。该立方体贴图每帧更新，并为每个片段存储到光源的距离，用于判断片段是否处于阴影中。

- [运行时 mipmap 生成](examples/texturemipmapgen/)

    在运行时生成完整 mip 链，而不是从文件加载。它从一个 mip 级别 blit 到下一个更小尺寸的级别，从实际纹理图像开始，一直到 mip 链末端的 1x1 像素。

- [截屏捕获](examples/screenshot/)

    场景渲染完成后捕获并保存图像，使用 blit 将最后一个 swapchain 图像从最优设备布局复制到主机本地线性内存，从而可以将其存储为 ppm 图像。

- [顺序无关透明](examples/oit)

    基于链表实现顺序无关透明。为此，该示例结合使用存储缓冲区，以及片段着色器中的图像 load/store 原子操作。

<a id="Performance"></a>
### 性能

- [多线程命令缓冲区生成](examples/multithreading/)

    多线程并行生成命令缓冲区。该示例不预构建并复用同一批命令缓冲区，而是使用多个硬件线程，演示每帧并行重新创建 secondary command buffer，并在所有线程完成后在 primary buffer 中执行和提交。

- [实例化](examples/instancing/)

    使用实例化特性，从单个顶点缓冲区渲染同一网格的多个实例，每个实例带有可变参数和纹理（索引分层纹理）。实例数据通过第二个顶点缓冲区传递。

- [间接绘制](examples/indirectdraw/)

    使用单个间接绘制调用渲染数千个具有不同几何体的实例化对象，而不是发出多个独立 draw。所有要执行的 draw 命令都存储在专用间接绘制缓冲区对象中（存储索引数量、偏移、实例数量等），上传到设备后由间接绘制命令读取用于渲染。

- [遮挡查询](examples/occlusionquery/)

    使用查询池对象获取已渲染图元通过的采样数量，用于判断屏幕可见性。

- [管线统计](examples/pipelinestatistics/)

    使用查询池对象收集管线不同阶段的统计信息，例如顶点、片段着色器和细分评估着色器的调用次数，具体取决于负载。

<a id="physically-based-rendering"></a>
### 基于物理的渲染

基于物理的渲染是一种光照技术，它基于测量得到的真实世界材质参数和环境光照，对双向反射分布函数进行近似，从而获得更真实且动态的外观。

- [PBR 基础](examples/pbrbasic/)

    演示基础的镜面 BRDF 实现，在一组具有不同材质参数的对象网格上使用实体材质和固定光源，展示金属反射率和表面粗糙度如何影响 PBR 光照对象的外观。

- [PBR 基于图像的光照](examples/pbribl/)

    将来自 HDR 环境 cubemap 的基于图像的光照加入 PBR 方程，使用周围环境作为光源。由于材质使用的光照贡献现在由环境控制，场景会获得更真实的外观。还展示如何从环境贴图生成 BRDF 2D-LUT、辐照度 cubemap 和预过滤 cubemap。

- [带 IBL 的纹理 PBR](examples/pbrtexture/)

    渲染一个专门为金属度-粗糙度 PBR 工作流制作的模型，使用纹理定义 PBR 方程的材质参数（反照率、金属度、粗糙度、烘焙环境遮蔽、法线贴图），并放置在基于图像的光照环境中。

<a id="Deferred"></a>
### 延迟渲染

这些示例使用[延迟着色](https://en.wikipedia.org/wiki/Deferred_shading)设置。

- [延迟着色基础](examples/deferred/)

    使用多个渲染目标，在单个 pass 中填充 G-Buffer 所需的全部附件（反照率、法线、位置、深度）。随后延迟 pass 使用这些数据在屏幕空间中计算着色和光照，因此计算只需针对可见片段执行，并且与光源数量无关。

- [延迟多重采样](examples/deferredmultisampling/)

    通过在片段着色器中手动 resolve，为延迟渲染器加入多重采样。

- [延迟着色阴影映射](examples/deferredshadows/)

    使用分层深度附件向延迟渲染器加入来自多个聚光灯的阴影，该深度附件通过多次几何着色器调用在一个 pass 中填充。

- [屏幕空间环境遮蔽](examples/ssao/)

    为 3D 场景加入屏幕空间环境遮蔽。来自前一个延迟 pass 的深度值用于生成环境遮蔽纹理，该纹理会在最终合成路径中应用到场景之前进行模糊处理。

<a id="compute-shader"></a>
### 计算着色器

所有 Vulkan 实现都支持计算着色器，这是一种在 GPU 上执行工作负载的更通用方式。这些示例演示如何使用这些计算着色器。

- [图像处理](examples/computeshader/)

    使用计算着色器和独立计算队列，实时向输入图像应用不同卷积核（和效果）。

- [GPU 粒子系统](examples/computeparticles/)

    使用计算着色器实现基于吸引力的 2D GPU 粒子系统。粒子数据存储在着色器存储缓冲区中，并且只在 GPU 上修改；通过内存屏障同步计算粒子更新和图形管线顶点访问。

- [N-body 模拟](examples/computenbody/)

    基于 N-body 模拟的粒子系统，支持多个吸引体和粒子间相互作用，使用两个 pass 分离粒子运动计算和最终积分。共享计算着色器内存用于加速计算。

- [光线追踪](examples/computeraytracing/)

    使用计算着色器实现带阴影和反射的简单 GPU 光线追踪器。图形 pass 中不会渲染场景几何体。

- [布料模拟](examples/computecloth/)

    基于质点-弹簧的 GPU 布料系统，使用计算着色器计算并积分弹簧力，同时实现与固定场景对象的基础碰撞。

- [剔除和 LOD](examples/computecullandlod/)

    纯 GPU 的视锥可见性剔除和细节层级系统。计算着色器用于修改存储在间接绘制命令缓冲区中的 draw 命令，以切换模型可见性并根据相机距离选择细节层级，不需要在 CPU 上执行计算或与 CPU 同步。

<a id="geometry-shader"></a>
### 几何着色器

- [法线调试](examples/geometryshader/)

    可视化逐顶点模型法线（用于调试）。第一个 pass 渲染普通模型，第二个 pass 使用几何着色器基于逐顶点模型法线生成彩色线段。

- [视口数组](examples/viewportarray/)

    使用几何着色器在一个 pass 中将场景渲染到多个视口，并对每个视口应用不同矩阵以模拟立体渲染（左/右）。需要设备支持 `multiViewport`。

<a id="tessellation-shader"></a>
### 细分着色器

- [位移映射](examples/displacement/)

    使用高度图为低多边形网格动态生成并位移额外几何细节。

- [动态地形细分](examples/terraintessellation/)

    使用细分着色器渲染地形，实现高度位移（基于 16 位高度图）、动态细节层级（基于三角形屏幕空间尺寸）和逐 patch 视锥剔除。

- [模型细分](examples/tessellation/)

    使用曲面 PN 三角形（[论文](http://alex.vlachos.com/graphics/CurvedPNTriangles.pdf)）为低多边形模型添加细节。

<a id="hardware-accelerated-ray-tracing"></a>
### 硬件加速光线追踪

Vulkan 支持配备专用光线追踪硬件的 GPU。这些示例展示该功能的不同部分。

- [基础光线追踪](examples/raytracingbasic)

    使用 `VK_KHR_acceleration_structure` 和 `VK_KHR_ray_tracing_pipeline` 扩展进行硬件加速光线追踪的基础示例。展示如何设置加速结构、光线追踪管线，以及执行实际光线追踪所需的 shader binding table。

- [光线追踪阴影](examples/raytracingshadows)

    使用新的光线追踪扩展，为更复杂的场景添加光线追踪阴影投射。展示如何添加多个 hit 和 miss 着色器，以及如何修改现有着色器来加入阴影计算。

- [光线追踪反射](examples/raytracingreflections)

    使用新的光线追踪扩展渲染带反射表面的复杂场景。展示如何在光线追踪着色器内部使用递归实现实时反射。

- [光线追踪纹理映射](examples/raytracingtextures)

    使用新的光线追踪扩展渲染带透明度的纹理映射四边形。展示如何在 closest hit 着色器中进行纹理映射，如何在 any hit 着色器中为透明度取消交点，以及如何使用 buffer device address 在这些着色器中访问网格数据。

- [Callable 光线追踪着色器](examples/raytracingcallable)

    Callable 着色器可以从其他光线追踪着色器内部动态调用，以便根据动态条件执行不同着色器。该示例对多个几何体进行光线追踪，每个几何体都从 closest hit 着色器调用不同的 callable 着色器。

- [光线追踪相交着色器](examples/raytracingintersection)

    使用 intersection 着色器处理程序化几何体。该示例不使用实际几何体，而是传入包围盒和对象定义；随后使用 intersection 着色器对程序化对象进行追踪。

- [光线追踪 glTF](examples/raytracinggltf/)

    使用光线追踪而不是光栅化渲染带纹理的 glTF 模型。利用帧累积实现透明度和抗锯齿。

- [Ray query](examples/rayquery)

    Ray query 为非光线追踪着色器阶段添加加速结构相交功能，从而可以结合光线追踪和光栅化。该示例在片段着色器中使用 ray query，为光栅化示例添加 ray cast 阴影。

- [位置获取](examples/raytracingpositionfetch/)

    使用 `VK_KHR_ray_tracing_position_fetch` 扩展，在着色器内部从加速结构获取顶点位置数据，而不需要手动解包顶点信息。

<a id="Headless"></a>
### 无头模式

运行一次性任务且不使用可视输出（没有窗口系统集成）的示例。它们可以在没有用户界面的环境中运行（[博客文章](https://www.saschawillems.de/tutorials/vulkan/headless_examples)）。

- [渲染](examples/renderheadless)

    将基础场景渲染到一个（不可见的）帧缓冲附件，将其读回主机内存并存储到磁盘，不进行任何屏幕显示，同时展示设备到主机图像同步所需内存屏障的正确用法。

- [计算](examples/computeheadless)

    仅使用计算着色器能力，对输入数据集（通过 SSBO 传入）运行计算。基于输入数据通过计算着色器计算 Fibonacci 行，写回后通过命令行显示。

<a id="user-interface"></a>
### 用户界面

- [文本渲染](examples/textoverlay/)

    加载并渲染由 [stb 字体文件](https://nothings.org/stb/font/)的位图字形数据创建的 2D 文本叠加层。该数据会作为纹理上传，并用于在第二个 pass 中将文本显示在 3D 场景之上。

- [距离场字体](examples/distancefieldfonts/)

    使用一种按字符存储有符号距离场信息的纹理，并配合特殊片段着色器基于该距离数据计算输出。这样可以获得与字体大小和缩放无关的清晰高质量字体渲染。

- [ImGui 叠加层](examples/imgui/)

    在 3D 场景之上生成并渲染包含多个窗口、控件和用户交互的复杂用户界面。UI 使用 [Dear ImGUI](https://github.com/ocornut/imgui) 生成，并每帧更新。

<a id="Extensions"></a>
### 扩展

Vulkan 是一个可扩展 API，许多功能通过扩展加入。这些示例演示此类扩展的用法。

**注意：** 某些扩展可能会在较新的 Vulkan 版本中成为核心功能。这些示例仍然可以与其配合使用。

- [保守光栅化](examples/conservativeraster/) - `VK_EXT_conservative_rasterization`

    使用保守光栅化改变 GPU 生成片段的方式。该示例启用 overestimation，为每个被触及的像素生成片段，而不是仅为完全覆盖的像素生成片段（[博客文章](https://www.saschawillems.de/tutorials/vulkan/conservative_rasterization)）。

- [Push descriptors](examples/pushdescriptors/) - `VK_KHR_push_descriptor`

    使用 push descriptors，将 push constants 概念应用到描述符集。该示例不为多个对象创建逐对象描述符集，而是在命令缓冲区创建时传入描述符。

- [Inline uniform blocks](examples/inlineuniformblocks/) - `VK_EXT_inline_uniform_block`

    使用 inline uniform blocks 在描述符集创建时直接传递 uniform 数据，同时演示如何在运行时更新这些描述符的数据。

- [Multiview 渲染](examples/multiview/) - `VK_KHR_multiview`

    将场景渲染到单个帧缓冲的多个 view（层），以在一个 pass 中模拟立体渲染。向各 view 的广播在顶点着色器中通过 `gl_ViewIndex` 完成。

- [条件渲染](examples/conditionalrender) - `VK_EXT_conditional_rendering`

    演示 VK_EXT_conditional_rendering 的用法，根据专用缓冲区中的值有条件地分派渲染命令。这允许实现例如可见性开关，而不必重建命令缓冲区（[博客文章](https://www.saschawillems.de/tutorials/vulkan/conditional_rendering)）。

- [调试着色器 printf](examples/debugprintf/) - `VK_KHR_shader_non_semantic_info`

    展示如何在着色器中使用 printf，按调用输出额外信息。这些信息可帮助在 RenderDoc 等工具中调试着色器相关问题。

    **注意：** 此示例应从 RenderDoc 等图形调试器运行。

- [Debug utils](examples/debugutils/) - `VK_EXT_debug_utils`

    展示如何使用 debug utils 为 Vulkan 对象添加标签和颜色，供图形调试器使用。这些信息有助于在 RenderDoc 等工具中识别资源。

    **注意：** 此示例应从 RenderDoc 等图形调试器运行。

- [负 viewport 高度](examples/negativeviewportheight/) - `VK_KHR_Maintenance1` 或 `Vulkan 1.1`

    展示如何使用负 viewport 高度渲染场景，使 Vulkan 渲染设置更接近 OpenGL 等其他 API。它还提供多个选项，用于更改相关管线状态，并以 OpenGL 或 Vulkan 风格坐标显示网格。详情见[本教程](https://www.saschawillems.de/tutorials/vulkan/flipping-viewport)。

- [可变速率着色](examples/variablerateshading/) - `VK_KHR_fragment_shading_rate`

    使用包含可变着色率的特殊图像，在帧缓冲范围内改变片段着色器调用次数。这样可以为帧缓冲中较不重要或噪声较少的区域降低片段着色器调用量。

- [描述符索引](examples/descriptorindexing/) - `VK_EXT_descriptor_indexing`

    演示如何使用 VK_EXT_descriptor_indexing 创建大小可变的描述符集，并在着色器中通过 `GL_EXT_nonuniform_qualifier` 和 `SPV_EXT_descriptor_indexing` 动态索引。

- [动态渲染](examples/dynamicrendering/) - `VK_KHR_dynamic_rendering`

    展示 VK_KHR_dynamic_rendering 扩展的用法。该扩展通过不再要求 render pass 对象或帧缓冲来简化渲染设置。

- [动态渲染本地读取](examples/dynamicrenderinglocalread/) - `VK_KHR_dynamic_rendering_local_read`

    展示 VK_KHR_dynamic_rendering 与 VK_KHR_dynamic_rendering_local_read 结合使用，以替代 render pass 和 subpass。Local read 确保读取保持在 tile memory 本地。

- [带多重采样的动态渲染](examples/dynamicrenderingmultisampling/) - `VK_KHR_dynamic_rendering`

    基于动态渲染示例，该示例展示如何使用动态渲染实现多重采样。

- [图形管线库](./examples/graphicspipelinelibrary) - `VK_EXT_graphics_pipeline_library`

    使用 graphics pipeline library 扩展改进运行时管线创建。该示例不会一次性创建完整管线，而是预先构建共享管线部分，例如顶点输入状态和片段输出状态。之后这些部分会在运行时用于创建完整管线，从而减少构建时间和可能出现的卡顿。

- [Mesh shaders](./examples/meshshader) - `VK_EXT_mesh_shader`

    基础示例，演示如何使用 mesh shading 管线替代传统顶点管线。

- [Descriptor heap](./examples/descriptorheap/) - `VK_EXT_descriptor_heap`

    基础示例，展示如何使用 descriptor heap，它会完全替代 Vulkan 原有描述符系统。[Khronos 博客](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension#descriptor_heaps)。

- [Descriptor buffers](./examples/descriptorbuffer/) - `VK_EXT_descriptor_buffer`

    基础示例，展示如何使用 descriptor buffer 替代描述符集。

    **注意：** Descriptor buffer 已被 descriptor heap 取代。

- [Shader objects](./examples/shaderobjects/) - `VK_EXT_shader_object`

    基础示例，展示如何使用 shader object 替代管线状态对象。它不会把所有状态烘焙到 PSO 中，而是显式加载并绑定着色器作为独立对象，并使用动态状态扩展设置状态。该示例还会存储二进制 shader object，并在后续运行中加载它们。

- [Host image copy](./examples/hostimagecopy/) - `VK_EXT_host_image_copy`

    展示如何执行 host image copy。它通过完全跳过 staging 过程，大幅简化主机到设备的图像流程。

- [Buffer device address](./examples/bufferdeviceaddress/) - `VK_KHR_buffer_device_addres`

    演示如何使用虚拟 GPU 地址在着色器中直接访问缓冲区数据。与使用描述符访问 uniform 等方式不同，借助该扩展，你只需向着色器提供要读取内存的地址，该地址可以通过 push constant 等方式任意更改。

- [Timeline semaphore](./examples/timelinesemaphore/) - `VK_KHR_timeline_semaphore`

    展示如何使用一种新的 semaphore 类型，它能够设置并标识时间线上的给定点。相比核心二值 semaphore，这简化了同步，因为单个 timeline semaphore 可以替代多个二值 semaphore。

- [片段着色器重心坐标](./examples/fragmentshaderbarycentrics/) - `VK_KHR_fragment_shader_barycentric`

    演示如何在片段着色器中访问重心坐标，以创建线框视觉效果。

<a id="Effects"></a>
### 效果

一组展示图形效果的示例，这些效果并非 Vulkan 专有。

- [全屏径向模糊](examples/radialblur/)

    演示全屏着色器效果的基础。场景会以较低分辨率渲染到离屏帧缓冲，然后使用径向模糊片段着色器作为全屏四边形叠加渲染到场景之上。

- [Bloom](examples/bloom/)

    高级全屏效果示例，为场景添加 bloom 效果。发光场景部分会渲染到低分辨率离屏帧缓冲，并通过两 pass 分离高斯模糊叠加应用到场景之上。

- [视差映射](examples/parallaxmapping/)

    实现多种纹理映射方法，根据纹理信息模拟深度：法线映射、视差映射、陡峭视差映射和视差遮蔽映射（质量最好，性能最差）。

- [球面环境映射](examples/sphericalenvmapping/)

    使用定义环境光照和反射信息的球面材质捕获纹理数组来模拟复杂光照。

<a id="Misc"></a>
### 其他

- [Vulkan Gears](examples/gears/)

    glxgears 的 Vulkan 版本。程序化生成并动画化多个齿轮。

- [Vulkan 演示场景](examples/vulkanscene/)

    渲染包含 logo 和吉祥物的 Vulkan 演示场景。它不是实际示例，更像是一个实验场和展示场景。

<a id="credits-and-attributions"></a>
## 致谢与归属

更多致谢与归属信息请参阅 [CREDITS.md](CREDITS.md)。
