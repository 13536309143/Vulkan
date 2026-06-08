# Vulkan 示例解析：specializationconstants.cpp 如何用 Specialization Constants 创建 Shader 变体

本文分析的是 `examples/specializationconstants/specializationconstants.cpp`。这个示例展示了 Vulkan 中一种创建 shader 变体的方式：**Specialization Constants**。

它最终在一个窗口中把画面横向分成三份：

- 左侧：Phong 光照。
- 中间：Toon / cartoon 光照。
- 右侧：Textured 纹理光照。

三份画面使用的是同一个 glTF 场景、同一套 uniform buffer、同一套 descriptor set、同一份 vertex shader 和同一份 fragment shader。真正变化的是：创建 graphics pipeline 时给 fragment shader 传入了不同的 specialization constant 值。

这个示例最值得学习的地方是：**同一个 SPIR-V shader 模块，可以在 pipeline 创建阶段被特化成多个不同版本。** 这和每帧更新的 uniform buffer 或 push constants 不同，specialization constants 在 pipeline 创建完成后就固定下来，适合做 shader 功能开关、算法路径选择、固定数组大小和材质变体。

核心代码如下：

```cpp
specializationData.lightingModel = 0;
vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCI, nullptr, &pipelines.phong);

specializationData.lightingModel = 1;
vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCI, nullptr, &pipelines.toon);

specializationData.lightingModel = 2;
vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCI, nullptr, &pipelines.textured);
```

也就是说，程序没有写三份 fragment shader，而是用同一份 `uber.frag` 创建了三条不同 pipeline。

## 1. Specialization Constants 解决什么问题

在 Vulkan 中，把数据传给 shader 的常见方式包括：

- vertex attributes
- uniform buffer
- storage buffer
- sampled image / sampler
- push constants
- specialization constants

其中 specialization constants 的定位很特殊：它不是每帧变化的数据，而是 **pipeline 创建时用于特化 shader 的常量**。

如果一个 shader 中有这样的分支：

```glsl
switch (LIGHTING_MODEL) {
    case 0:
        // Phong
        break;
    case 1:
        // Toon
        break;
    case 2:
        // Textured
        break;
}
```

如果 `LIGHTING_MODEL` 是普通 uniform，那么 GPU 驱动通常只能把它当作运行时变量处理。shader 里的多个分支都需要保留，具体走哪条路径要等 draw 时才知道。

如果 `LIGHTING_MODEL` 是 specialization constant，那么它在 `vkCreateGraphicsPipelines()` 时就已经确定。驱动有机会把不可能执行的分支优化掉，或者至少按固定值生成更合适的 shader 变体。

这就是本示例要展示的核心思想：

```text
一份 uber shader 源码
    -> 多组 specialization constant 数据
    -> 多条 VkPipeline
    -> 多种渲染效果
```

## 2. 示例最终画了什么

示例加载了一个 glTF 场景：

```cpp
scene.loadFromFile(
    getAssetPath() + "models/color_teapot_spheres.gltf",
    vulkanDevice,
    queue,
    vkglTF::FileLoadingFlags::PreTransformVertices |
    vkglTF::FileLoadingFlags::PreMultiplyVertexColors |
    vkglTF::FileLoadingFlags::FlipY);
```

还加载了一张 KTX 纹理：

```cpp
colormap.loadFromFile(
    getAssetPath() + "textures/metalplate_nomips_rgba.ktx",
    VK_FORMAT_R8G8B8A8_UNORM,
    vulkanDevice,
    queue);
```

最终绘制时，同一个 `scene` 会被画三次：

```cpp
vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.phong);
scene.draw(cmdBuffer);

vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.toon);
scene.draw(cmdBuffer);

vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.textured);
scene.draw(cmdBuffer);
```

三次 draw 的 geometry、descriptor set 和 uniform buffer 都一样。不同的是每次绑定的 `VkPipeline` 不同，而这些 pipeline 来自同一份 fragment shader 的不同特化结果。

## 3. 程序整体结构

示例继承自框架基类：

```cpp
class VulkanExample: public VulkanExampleBase
```

`VulkanExampleBase` 已经处理了 Vulkan instance、device、swapchain、render pass、framebuffer、command buffer、同步对象、UI overlay 和 camera 等基础设施。这个示例只需要关注 specialization constants、pipeline 创建和绘制逻辑。

核心成员可以分成几类。

首先是模型和纹理资源：

```cpp
vkglTF::Model scene;
vks::Texture2D colormap;
```

`scene` 是要绘制的 glTF 场景，`colormap` 是 textured 分支使用的纹理。

然后是每帧更新的 uniform 数据：

```cpp
struct UniformData {
    glm::mat4 projection;
    glm::mat4 modelView;
    glm::vec4 lightPos{ 0.0f, -2.0f, 1.0f, 0.0f };
} uniformData;

std::array<vks::Buffer, maxConcurrentFrames> uniformBuffers;
```

每个并发帧都有一个 uniform buffer。这样 CPU 更新当前帧矩阵时，不会覆盖 GPU 仍在读取的上一帧数据。

接着是 descriptor 和 pipeline layout：

```cpp
VkPipelineLayout pipelineLayout{ VK_NULL_HANDLE };
VkDescriptorSetLayout descriptorSetLayout{ VK_NULL_HANDLE };
std::array<VkDescriptorSet, maxConcurrentFrames> descriptorSets{};
```

最后是三条 pipeline：

```cpp
struct Pipelines{
    VkPipeline phong{ VK_NULL_HANDLE };
    VkPipeline toon{ VK_NULL_HANDLE };
    VkPipeline textured{ VK_NULL_HANDLE };
} pipelines;
```

这三条 pipeline 共享绝大多数状态，只在 fragment shader 的 specialization constant 值上不同。

## 4. 程序运行流程

初始化入口是 `prepare()`：

```cpp
void prepare()
{
    VulkanExampleBase::prepare();
    loadAssets();
    prepareUniformBuffers();
    setupDescriptors();
    preparePipelines();
    prepared = true;
}
```

这几步可以理解为：

1. `VulkanExampleBase::prepare()`：创建 swapchain、render pass、framebuffer 等基础对象。
2. `loadAssets()`：加载 glTF 模型和 KTX 纹理。
3. `prepareUniformBuffers()`：为每个并发帧创建并映射 uniform buffer。
4. `setupDescriptors()`：创建 descriptor pool、descriptor set layout 和 descriptor sets。
5. `preparePipelines()`：创建 pipeline layout，并创建三条使用不同 specialization constant 的 graphics pipeline。

每帧渲染入口是 `render()`：

```cpp
virtual void render()
{
    if (!prepared)
        return;
    VulkanExampleBase::prepareFrame();
    updateUniformBuffers();
    buildCommandBuffer();
    VulkanExampleBase::submitFrame();
}
```

这里的 `prepareFrame()` 和 `submitFrame()` 由基类负责，示例自己的主要逻辑是更新当前帧 uniform buffer，然后重新录制 command buffer。

## 5. Shader 接口：普通资源绑定和特化常量

顶点 shader 使用 binding 0 的 uniform buffer：

```glsl
layout (binding = 0) uniform UBO
{
    mat4 projection;
    mat4 model;
    vec4 lightPos;
} ubo;
```

fragment shader 使用 binding 1 的纹理：

```glsl
layout (binding = 1) uniform sampler2D samplerColormap;
```

这些是普通 descriptor 资源。它们通过 descriptor set 绑定，在 draw 时对 shader 可见。

fragment shader 中还声明了两个 specialization constants：

```glsl
layout (constant_id = 0) const int LIGHTING_MODEL = 0;
layout (constant_id = 1) const float PARAM_TOON_DESATURATION = 0.0f;
```

这里的 `constant_id` 和 descriptor `binding` 是两套完全不同的编号系统。

可以简单理解为：

```text
binding
    描述 shader 从 descriptor set 的哪个绑定点读取 buffer / image / sampler。

constant_id
    描述 pipeline 创建时要替换 shader 中哪个 specialization constant。

location
    描述 shader stage 输入输出变量的位置。
```

在这个示例里，`LIGHTING_MODEL` 决定 fragment shader 选择哪个光照分支，`PARAM_TOON_DESATURATION` 决定 toon 分支的去饱和强度。

## 6. Descriptor Set：每帧一份 UBO，同一张纹理

`setupDescriptors()` 创建了 descriptor pool：

```cpp
std::vector<VkDescriptorPoolSize> poolSizes = {
    vks::initializers::descriptorPoolSize(VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, maxConcurrentFrames),
    vks::initializers::descriptorPoolSize(VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, maxConcurrentFrames)
};
```

这里为每个并发帧准备一个 uniform buffer descriptor 和一个 combined image sampler descriptor。

descriptor set layout 有两个 binding：

```cpp
std::vector<VkDescriptorSetLayoutBinding> setLayoutBindings = {
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        VK_SHADER_STAGE_VERTEX_BIT,
        0),
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        VK_SHADER_STAGE_FRAGMENT_BIT,
        1),
};
```

binding 0 对应 vertex shader 的 UBO，binding 1 对应 fragment shader 的 colormap 纹理。

然后为每个并发帧分配 descriptor set：

```cpp
for (auto i = 0; i < uniformBuffers.size(); i++) {
    VK_CHECK_RESULT(vkAllocateDescriptorSets(device, &allocInfo, &descriptorSets[i]));
    ...
}
```

每个 descriptor set 写入自己的 uniform buffer：

```cpp
vks::initializers::writeDescriptorSet(
    descriptorSets[i],
    VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
    0,
    &uniformBuffers[i].descriptor)
```

纹理不需要按帧复制资源本身，但 descriptor set 中仍然要写入同一个纹理 descriptor：

```cpp
vks::initializers::writeDescriptorSet(
    descriptorSets[i],
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
    1,
    &colormap.descriptor)
```

这样每帧绑定自己的 descriptor set，就能拿到当前帧的 UBO，同时访问同一张纹理。

## 7. Pipeline Layout：把 Descriptor Set Layout 接到 Pipeline 上

创建 pipeline 前，先创建 pipeline layout：

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutCreateInfo =
    vks::initializers::pipelineLayoutCreateInfo(&descriptorSetLayout, 1);

VK_CHECK_RESULT(vkCreatePipelineLayout(
    device,
    &pipelineLayoutCreateInfo,
    nullptr,
    &pipelineLayout));
```

这个 pipeline layout 只包含一个 descriptor set layout，没有 push constant range。

它的意义是告诉 pipeline：

```text
set 0 binding 0 是 vertex shader 可见的 uniform buffer
set 0 binding 1 是 fragment shader 可见的 combined image sampler
```

三条 pipeline 都使用同一个 `pipelineLayout`，因为它们访问资源的方式完全相同。

## 8. Graphics Pipeline：共享大部分固定函数状态

`preparePipelines()` 创建 pipeline 时，先准备常规固定函数状态：

```cpp
VkPipelineInputAssemblyStateCreateInfo inputAssemblyState =
    vks::initializers::pipelineInputAssemblyStateCreateInfo(
        VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST,
        0,
        VK_FALSE);

VkPipelineRasterizationStateCreateInfo rasterizationState =
    vks::initializers::pipelineRasterizationStateCreateInfo(
        VK_POLYGON_MODE_FILL,
        VK_CULL_MODE_NONE,
        VK_FRONT_FACE_CLOCKWISE,
        0);

VkPipelineDepthStencilStateCreateInfo depthStencilState =
    vks::initializers::pipelineDepthStencilStateCreateInfo(
        VK_TRUE,
        VK_TRUE,
        VK_COMPARE_OP_LESS_OR_EQUAL);
```

这些状态对三条 pipeline 都一样。

动态状态包括 viewport、scissor 和 line width：

```cpp
std::vector<VkDynamicState> dynamicStateEnables = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR,
    VK_DYNAMIC_STATE_LINE_WIDTH
};
```

动态 viewport 很关键，因为示例要在同一个 render pass 里把同一个场景画到左、中、右三个区域。

顶点输入布局来自 glTF 模型工具类：

```cpp
pipelineCI.pVertexInputState =
    vkglTF::Vertex::getPipelineVertexInputState({
        vkglTF::VertexComponent::Position,
        vkglTF::VertexComponent::Normal,
        vkglTF::VertexComponent::UV,
        vkglTF::VertexComponent::Color
    });
```

它对应 `uber.vert` 中的输入：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inNormal;
layout (location = 2) in vec2 inUV;
layout (location = 3) in vec3 inColor;
```

## 9. C++ 侧如何描述 Specialization Data

C++ 中先定义一块 host 数据：

```cpp
struct SpecializationData {
    uint32_t lightingModel{ 0 };
    float toonDesaturationFactor{ 0.5f };
} specializationData;
```

这块数据不是 uniform buffer，也不会在 draw 时绑定。它只会在 `vkCreateGraphicsPipelines()` 创建 pipeline 时被 Vulkan 读取。

字段含义是：

- `lightingModel`：选择 fragment shader 中的光照模型。
- `toonDesaturationFactor`：toon 光照分支中的去饱和参数。

对应 shader 中的声明：

```glsl
layout (constant_id = 0) const int LIGHTING_MODEL = 0;
layout (constant_id = 1) const float PARAM_TOON_DESATURATION = 0.0f;
```

C++ 字段和 shader 常量不是靠名字匹配，而是靠 `constant_id` 和 map entry 匹配。

## 10. VkSpecializationMapEntry：把 constant_id 映射到 host 数据

每个 specialization constant 都需要一个 `VkSpecializationMapEntry`：

```cpp
std::array<VkSpecializationMapEntry, 2> specializationMapEntries;
```

第一个 entry 对应 `LIGHTING_MODEL`：

```cpp
specializationMapEntries[0].constantID = 0;
specializationMapEntries[0].size = sizeof(specializationData.lightingModel);
specializationMapEntries[0].offset = 0;
```

第二个 entry 对应 `PARAM_TOON_DESATURATION`：

```cpp
specializationMapEntries[1].constantID = 1;
specializationMapEntries[1].size = sizeof(specializationData.toonDesaturationFactor);
specializationMapEntries[1].offset =
    offsetof(SpecializationData, toonDesaturationFactor);
```

三个字段分别表示：

```text
constantID
    shader 中 layout (constant_id = N) 的 N。

offset
    这个值在 pData 指向的数据块中的字节偏移。

size
    这个值要读取多少字节。
```

这里第二个字段使用 `offsetof` 很重要。C++ 结构体字段之间可能存在对齐填充，不能假设第二个字段一定紧跟在第一个字段后面。

## 11. VkSpecializationInfo：把映射表接到 shader stage 上

多个 map entry 会组合成一个 `VkSpecializationInfo`：

```cpp
VkSpecializationInfo specializationInfo{};
specializationInfo.dataSize = sizeof(specializationData);
specializationInfo.mapEntryCount =
    static_cast<uint32_t>(specializationMapEntries.size());
specializationInfo.pMapEntries = specializationMapEntries.data();
specializationInfo.pData = &specializationData;
```

这可以理解成一张表：

```text
pData
    指向 SpecializationData 数据块。

pMapEntries
    告诉 Vulkan 每个 constant_id 从 pData 的哪个 offset 读取多少字节。
```

接着把它挂到 fragment shader stage 上：

```cpp
shaderStages[1].pSpecializationInfo = &specializationInfo;
```

注意它不是挂到 `VkGraphicsPipelineCreateInfo` 上，而是挂到某个 `VkPipelineShaderStageCreateInfo` 上。因为 specialization constants 是按 shader stage 生效的。

本示例只有 fragment shader 用到了 `constant_id`，所以只给 fragment stage 设置 `pSpecializationInfo`。

## 12. 同一份 shader 如何创建三条 pipeline

shader 模块只加载一次：

```cpp
shaderStages[0] = loadShader(
    getShadersPath() + "specializationconstants/uber.vert.spv",
    VK_SHADER_STAGE_VERTEX_BIT);

shaderStages[1] = loadShader(
    getShadersPath() + "specializationconstants/uber.frag.spv",
    VK_SHADER_STAGE_FRAGMENT_BIT);
```

之后连续创建三条 pipeline。

第一条 pipeline 使用 Phong：

```cpp
specializationData.lightingModel = 0;
VK_CHECK_RESULT(vkCreateGraphicsPipelines(
    device,
    pipelineCache,
    1,
    &pipelineCI,
    nullptr,
    &pipelines.phong));
```

第二条 pipeline 使用 Toon：

```cpp
specializationData.lightingModel = 1;
VK_CHECK_RESULT(vkCreateGraphicsPipelines(
    device,
    pipelineCache,
    1,
    &pipelineCI,
    nullptr,
    &pipelines.toon));
```

第三条 pipeline 使用 Textured：

```cpp
specializationData.lightingModel = 2;
VK_CHECK_RESULT(vkCreateGraphicsPipelines(
    device,
    pipelineCache,
    1,
    &pipelineCI,
    nullptr,
    &pipelines.textured));
```

这里有一个容易误解的点：三次调用都复用了同一个 `specializationInfo`，而 `specializationInfo.pData` 指向同一个 `specializationData`。

这是可以的，因为 `vkCreateGraphicsPipelines()` 会在 pipeline 创建过程中读取当时的数据。pipeline 创建完成后，再修改 `specializationData.lightingModel` 不会影响已经创建好的 pipeline。

所以这段代码的实际效果是：

| Pipeline | `LIGHTING_MODEL` | Fragment shader 分支 |
| --- | --- | --- |
| `pipelines.phong` | `0` | Phong |
| `pipelines.toon` | `1` | Toon |
| `pipelines.textured` | `2` | Textured |

## 13. Fragment Shader 中的三种光照路径

`uber.frag` 通过 `LIGHTING_MODEL` 选择不同分支：

```glsl
switch (LIGHTING_MODEL) {
    case 0:
        ...
        break;
    case 1:
        ...
        break;
    case 2:
        ...
        break;
}
```

Phong 分支使用顶点颜色、法线、光照向量和观察向量计算 ambient、diffuse、specular：

```glsl
vec3 ambient = inColor * vec3(0.25);
vec3 diffuse = max(dot(N, L), 0.0) * inColor;
vec3 specular = pow(max(dot(R, V), 0.0), 32.0) * vec3(0.75);
outFragColor = vec4(ambient + diffuse * 1.75 + specular, 1.0);
```

Toon 分支把光照强度分成几个离散档位：

```glsl
float intensity = dot(N,L);
if (intensity > 0.98)
    color = inColor * 1.5;
else if  (intensity > 0.9)
    color = inColor * 1.0;
else if (intensity > 0.5)
    color = inColor * 0.6;
else if (intensity > 0.25)
    color = inColor * 0.4;
else
    color = inColor * 0.2;
```

然后用另一个 specialization constant 控制去饱和：

```glsl
color = vec3(mix(
    color,
    vec3(dot(vec3(0.2126,0.7152,0.0722), color)),
    PARAM_TOON_DESATURATION));
```

Textured 分支从纹理采样，并参与光照计算：

```glsl
vec4 color = texture(samplerColormap, inUV).rrra;
vec3 diffuse = max(dot(N, L), 0.0) * color.rgb;
float specular = pow(max(dot(R, V), 0.0), 32.0) * color.a;
outFragColor = vec4(ambient + diffuse + vec3(specular), 1.0);
```

这三个分支都在同一份 shader 源码里，但最终由不同 pipeline 固定选择。

## 14. Uniform Buffer 仍然负责每帧变化的数据

Specialization constants 不适合做每帧变化的数据。这个示例中，相机矩阵仍然通过 uniform buffer 传入：

```cpp
void updateUniformBuffers()
{
    camera.setPerspective(
        60.0f,
        ((float)width / 3.0f) / (float)height,
        0.1f,
        512.0f);

    uniformData.projection = camera.matrices.perspective;
    uniformData.modelView = camera.matrices.view;
    memcpy(uniformBuffers[currentBuffer].mapped, &uniformData, sizeof(UniformData));
}
```

这里的宽高比使用 `width / 3.0f`，因为每个模型只渲染到屏幕三分之一宽的 viewport。如果用整个窗口宽度计算投影，单个区域里的模型比例会不对。

这个设计体现了两类数据的分工：

```text
Uniform buffer
    每帧变化，例如矩阵、光源位置。

Specialization constants
    pipeline 创建时固定，例如选择哪种光照模型。
```

## 15. Command Buffer：同一个场景，切换三次 pipeline

`buildCommandBuffer()` 中先开始 render pass，然后设置 scissor：

```cpp
VkRect2D scissor = vks::initializers::rect2D(width, height, 0, 0);
vkCmdSetScissor(cmdBuffer, 0, 1, &scissor);
```

接着绑定当前帧 descriptor set：

```cpp
vkCmdBindDescriptorSets(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    0,
    1,
    &descriptorSets[currentBuffer],
    0,
    nullptr);
```

之后绘制左侧区域：

```cpp
VkViewport viewport =
    vks::initializers::viewport((float)width / 3.0f, (float)height, 0.0f, 1.0f);

vkCmdSetViewport(cmdBuffer, 0, 1, &viewport);
vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.phong);
scene.draw(cmdBuffer);
```

绘制中间区域时，只移动 viewport 的 `x`，然后切换 pipeline：

```cpp
viewport.x = (float)width / 3.0f;
vkCmdSetViewport(cmdBuffer, 0, 1, &viewport);
vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.toon);
scene.draw(cmdBuffer);
```

绘制右侧区域同理：

```cpp
viewport.x = (float)width / 3.0f + (float)width / 3.0f;
vkCmdSetViewport(cmdBuffer, 0, 1, &viewport);
vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.textured);
scene.draw(cmdBuffer);
```

这段代码说明了 Vulkan command buffer 的状态模型：

- descriptor set 绑定一次即可复用。
- viewport 是动态状态，可以在同一条 pipeline 或不同 pipeline 之间切换。
- pipeline 绑定会改变 shader 和固定函数状态。
- 同一个 model 可以在同一个 render pass 中用不同 pipeline 绘制多次。

## 16. Specialization Constants 和 Push Constants 的区别

这两个名字都带 constants，但用途完全不同。

Push constants 是 command buffer 级别的数据：

```text
vkCmdPushConstants()
    在录制 command buffer 时写入。
    可以在 draw call 之间频繁改变。
    不需要重建 pipeline。
```

Specialization constants 是 pipeline 创建级别的数据：

```text
VkSpecializationInfo
    在 vkCreateGraphicsPipelines() 时传入。
    pipeline 创建后固定。
    想换值通常需要另一条 pipeline。
```

因此：

- 每个物体的颜色、位置、小型材质参数，更适合 push constants 或 uniform buffer。
- 是否启用某个 shader 分支、使用哪种算法、固定采样数，更适合 specialization constants。

本示例里的 `LIGHTING_MODEL` 是典型的 specialization constant，因为它决定的是 shader 变体，而不是每个 draw 临时变化的小数据。

## 17. Specialization Constants 和宏编译的关系

没有 specialization constants 时，想从一份 shader 源码得到多个变体，常见做法是使用预处理宏：

```glsl
#if LIGHTING_MODEL == 0
    ...
#elif LIGHTING_MODEL == 1
    ...
#endif
```

然后用不同宏值编译出多个 SPIR-V 文件。

Specialization constants 提供了另一种方式：

```glsl
layout (constant_id = 0) const int LIGHTING_MODEL = 0;
```

同一个 SPIR-V 模块可以在 pipeline 创建时指定不同值。这样可以减少 shader 源文件和 SPIR-V 文件数量，也更适合运行时根据设备能力或材质配置创建变体。

但它并不意味着 pipeline 可以无限制动态切换常量值。只要 specialization constant 值不同，就应该把它看作不同 pipeline 变体。

## 18. 资源生命周期

析构函数中释放了本示例创建的 Vulkan 对象：

```cpp
vkDestroyPipeline(device, pipelines.phong, nullptr);
vkDestroyPipeline(device, pipelines.textured, nullptr);
vkDestroyPipeline(device, pipelines.toon, nullptr);
vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);
colormap.destroy();
for (auto& buffer : uniformBuffers) {
    buffer.destroy();
}
```

这里有几个顺序关系值得注意：

- pipeline 依赖 pipeline layout，所以先销毁 pipeline，再销毁 pipeline layout。
- descriptor set layout 由 descriptor set 和 pipeline layout 使用，示例退出时统一销毁。
- uniform buffer 和 texture 是 descriptor 引用的资源，程序结束时释放。

descriptor pool 在基类中管理，不在这个析构函数里显式销毁。

## 19. 常见错误和调试方向

如果修改这个示例时画面不符合预期，可以优先检查这些点。

第一，`constantID` 是否和 shader 的 `constant_id` 一致：

```cpp
specializationMapEntries[0].constantID = 0;
```

必须对应：

```glsl
layout (constant_id = 0) const int LIGHTING_MODEL = 0;
```

第二，`offset` 是否正确。结构体字段偏移最好用 `offsetof`，不要手算。

第三，`size` 是否和 C++ 字段类型匹配。shader 中 `int` 对应这里使用的 `uint32_t lightingModel`，实际项目中更建议让 C++ 类型和 shader 类型保持明确一致。

第四，是否把 `VkSpecializationInfo` 挂到了正确的 shader stage 上。本示例的 `constant_id` 在 fragment shader 里，所以要设置：

```cpp
shaderStages[1].pSpecializationInfo = &specializationInfo;
```

第五，修改 `.frag` 或 `.vert` 后是否重新编译了 `.spv`。示例运行时加载的是：

```text
shaders/glsl/specializationconstants/uber.vert.spv
shaders/glsl/specializationconstants/uber.frag.spv
```

不是直接加载 GLSL 文本文件。

## 20. 从这个示例提炼出的工程原则

第一，specialization constants 适合表达固定变体，不适合表达高频参数。

例如 `LIGHTING_MODEL` 适合 specialization constant，因为它决定 shader 结构。相机矩阵不适合，因为它每帧都变。

第二，多条 pipeline 可以共享绝大多数状态。

本示例三条 pipeline 共享 descriptor layout、pipeline layout、vertex input、render pass、固定函数状态和 shader 模块，只通过 specialization data 产生差异。

第三，uber shader 可以降低源码数量，但不能消除 pipeline 变体管理。

你仍然需要为不同 specialization constant 组合创建不同 pipeline。实际项目中通常需要 pipeline cache 和变体 key，例如：

```text
MaterialPipelineKey {
    lightingModel
    useNormalMap
    alphaMode
    sampleCount
}
```

第四，descriptor binding、vertex location、specialization constant id 要分开管理。

这三套编号都出现在 shader 和 C++ 之间，但语义不同。混淆它们是调试 Vulkan shader 接口时很常见的问题。

## 21. 一个简化版 mental model

可以把本示例理解成下面这条数据流：

```text
uber.frag
    声明 layout (constant_id = 0) const int LIGHTING_MODEL

SpecializationData
    C++ 侧保存 lightingModel = 0 / 1 / 2

VkSpecializationMapEntry
    告诉 Vulkan constant_id 0 从 SpecializationData 的哪个 offset 读取

VkSpecializationInfo
    把数据块和映射表组合起来

VkPipelineShaderStageCreateInfo::pSpecializationInfo
    把 specialization info 接到 fragment shader stage

vkCreateGraphicsPipelines()
    用当前 specializationData 创建一条固定变体 pipeline

vkCmdBindPipeline()
    绘制时绑定不同 pipeline，得到不同光照效果
```

这条链路就是 specialization constants 的完整使用路径。

## 22. 总结

`specializationconstants.cpp` 展示了 Vulkan shader 变体管理中的一个基础技巧：用 specialization constants 在 pipeline 创建阶段固定 shader 常量，从同一份 shader 模块创建多条行为不同的 pipeline。

这个示例中：

- `LIGHTING_MODEL` 决定 Phong、Toon、Textured 三种光照路径。
- `PARAM_TOON_DESATURATION` 控制 Toon 分支的去饱和程度。
- 三条 pipeline 共用同一份 shader 文件和大部分 pipeline 状态。
- command buffer 通过切换 viewport 和 pipeline，在同一个 render pass 中绘制三份对比画面。

如果要在真实项目中使用这个模式，可以把 specialization constants 当作 shader variant key 的一部分。它适合固定功能开关和算法路径，不适合每帧或每个 draw 都变化的参数。
