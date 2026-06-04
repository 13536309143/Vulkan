# Vulkan 示例解析：descriptorsets.cpp 如何用 Descriptor Set 给不同物体绑定不同资源

本文分析的是 `examples/descriptorsets/descriptorsets.cpp`。这个示例的主题非常基础，但也非常核心：**如何使用 descriptor set 把 CPU 侧创建的 buffer、image、sampler 等资源绑定到 shader 中使用**。

如果说 `pipelines.cpp` 重点讲的是“不同 pipeline 代表不同渲染状态”，那么 `descriptorsets.cpp` 讲的就是另一条 Vulkan 主线：**同一条 pipeline 可以通过绑定不同的 descriptor set，给 shader 提供不同的数据，从而画出不同的物体。**

这个示例最终绘制了两个立方体：

- 两个立方体使用同一个 glTF cube 模型。
- 两个立方体使用同一条 graphics pipeline。
- 两个立方体使用同一组 vertex / fragment shader。
- 但每个立方体有自己的 uniform buffer。
- 每个立方体有自己的 texture。
- 每个立方体每帧都有自己的 descriptor set。

绘制时，代码只需要在 draw call 前切换 descriptor set：

```cpp
vkCmdBindDescriptorSets(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    0,
    1,
    &cube.descriptorSets[currentBuffer],
    0,
    nullptr);

model.draw(cmdBuffer);
```

这就是 descriptor set 的核心意义：**它不是资源本身，而是一组 shader 可见资源绑定的集合。**

## 1. 为什么 Vulkan 需要 Descriptor Set

在 shader 中，我们经常需要访问外部资源，例如：

- uniform buffer：相机矩阵、模型矩阵、光照参数。
- storage buffer：大规模结构化数据。
- sampled image：纹理图像。
- sampler：纹理采样状态。
- combined image sampler：图像和采样器组合。
- storage image：可读写图像。

在 OpenGL 中，这类资源通常通过全局状态或绑定点逐个绑定。Vulkan 则采用更显式、更可预期的方式：先声明 shader 需要哪些资源，再把这些资源组织成 descriptor set。

可以把 Vulkan descriptor 系统理解成三层：

```text
Descriptor Set Layout
    描述这个 set 里有哪些 binding，每个 binding 是什么类型，哪些 shader stage 可以访问。

Descriptor Set
    按照 layout 分配出来的具体绑定表。

Descriptor
    set 中某个 binding 对应的具体资源引用，例如某个 uniform buffer 或某张 texture。
```

在 `descriptorsets.cpp` 中，shader 需要两个资源：

```glsl
layout (set = 0, binding = 0) uniform UBOMatrices {
    mat4 projection;
    mat4 view;
    mat4 model;
} uboMatrices;

layout (set = 0, binding = 1) uniform sampler2D samplerColorMap;
```

对应到 C++ 侧：

- set 0 binding 0：`VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER`
- set 0 binding 1：`VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER`

这两个 binding 被放在同一个 descriptor set 中。每个 cube 都有自己的 descriptor set，因此每个 cube 都可以有不同的矩阵和不同的纹理。

## 2. 示例整体结构

示例继承自框架基类：

```cpp
class VulkanExample : public VulkanExampleBase
```

`VulkanExampleBase` 已经处理了 Vulkan 初始化、swapchain、render pass、framebuffer、command buffer、同步和 UI overlay 等基础设施。这个示例只关注 descriptor set 的创建和绑定。

核心成员如下：

```cpp
bool animate = true;

struct Cube {
    struct Matrices {
        glm::mat4 projection;
        glm::mat4 view;
        glm::mat4 model;
    } matrices;

    vks::Texture2D texture;
    std::array<vks::Buffer, maxConcurrentFrames> uniformBuffers{};
    std::array<VkDescriptorSet, maxConcurrentFrames> descriptorSets{};
    glm::vec3 rotation{ 0.0f };
};

std::array<Cube, 2> cubes;

vkglTF::Model model;

VkPipeline pipeline{ VK_NULL_HANDLE };
VkPipelineLayout pipelineLayout{ VK_NULL_HANDLE };
VkDescriptorSetLayout descriptorSetLayout{ VK_NULL_HANDLE };
```

结构非常清晰。

`model` 是共享的 cube 几何体。两个立方体不需要各自加载一份 mesh。

`pipeline` 是共享的 graphics pipeline。两个立方体使用同样的 shader、vertex input、rasterizer、depth test 和 blend 状态。

每个 `Cube` 保存自己的：

- `matrices`：当前 cube 的 projection、view、model 矩阵。
- `texture`：当前 cube 的纹理。
- `uniformBuffers`：每个并发帧一份 uniform buffer。
- `descriptorSets`：每个并发帧一份 descriptor set。
- `rotation`：当前 cube 的动画旋转角。

这正好对应真实渲染器中常见的资源分层：

```text
共享资源：
    pipeline
    mesh
    shader

每个物体自己的资源：
    model matrix
    material texture
    descriptor set
```

## 3. 程序运行流程

初始化流程在 `prepare()` 中：

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

它按顺序完成：

1. 初始化基础 Vulkan 资源。
2. 加载 cube 模型和两张纹理。
3. 为每个 cube、每个并发帧创建 uniform buffer。
4. 创建 descriptor pool、descriptor set layout、descriptor set，并写入资源绑定。
5. 创建 pipeline layout 和 graphics pipeline。

每帧渲染流程在 `render()` 中：

```cpp
void render()
{
    if (!prepared)
        return;

    if (animate && !paused) {
        cubes[0].rotation.x += 2.5f * frameTimer;
        cubes[1].rotation.y += 2.0f * frameTimer;
    }

    VulkanExampleBase::prepareFrame();
    updateUniformBuffers();
    buildCommandBuffer();
    VulkanExampleBase::submitFrame();
}
```

每帧主要做三件事：

- 更新两个 cube 的旋转。
- 更新当前帧的 uniform buffer。
- 录制 command buffer，绘制两个 cube。

这里 descriptor set 并不是每帧重新创建的。descriptor set 在初始化阶段创建并写入，渲染时只是绑定对应当前帧的 descriptor set。

## 4. 加载资源：一个模型，两张纹理

资源加载在 `loadAssets()` 中：

```cpp
const uint32_t glTFLoadingFlags =
    vkglTF::FileLoadingFlags::PreTransformVertices |
    vkglTF::FileLoadingFlags::PreMultiplyVertexColors |
    vkglTF::FileLoadingFlags::FlipY;

model.loadFromFile(
    getAssetPath() + "models/cube.gltf",
    vulkanDevice,
    queue,
    glTFLoadingFlags);

cubes[0].texture.loadFromFile(
    getAssetPath() + "textures/crate01_color_height_rgba.ktx",
    VK_FORMAT_R8G8B8A8_UNORM,
    vulkanDevice,
    queue);

cubes[1].texture.loadFromFile(
    getAssetPath() + "textures/crate02_color_height_rgba.ktx",
    VK_FORMAT_R8G8B8A8_UNORM,
    vulkanDevice,
    queue);
```

这里加载了同一个 cube 模型：

```text
models/cube.gltf
```

然后给两个 cube 分别加载两张不同纹理：

```text
textures/crate01_color_height_rgba.ktx
textures/crate02_color_height_rgba.ktx
```

这就为后续 descriptor set 演示准备好了条件：

```text
同一个 mesh + 同一个 pipeline + 不同 descriptor set = 不同对象外观
```

每个 `vks::Texture2D` 内部不仅持有 Vulkan image 和 image view，也会准备一个 descriptor 结构：

```cpp
cube.texture.descriptor
```

它通常包含：

```cpp
VkDescriptorImageInfo {
    VkSampler sampler;
    VkImageView imageView;
    VkImageLayout imageLayout;
}
```

后面更新 descriptor set 时，会把这个 `VkDescriptorImageInfo` 写入 binding 1。

## 5. samplerAnisotropy：纹理采样的可选设备特性

示例在 `getEnabledFeatures()` 中检查并启用各向异性过滤：

```cpp
void getEnabledFeatures()
{
    if (deviceFeatures.samplerAnisotropy) {
        enabledFeatures.samplerAnisotropy = VK_TRUE;
    };
}
```

`samplerAnisotropy` 是纹理采样相关的设备特性。如果设备支持，各向异性过滤可以改善斜视角下纹理的清晰度。

这个特性和 descriptor set 的关系是：descriptor set 绑定的是 combined image sampler，而 sampler 的创建可能会使用 anisotropy。Vulkan 中这类能力必须先在 physical device 上查询，然后在 logical device 创建时显式启用。

这也是 Vulkan 的典型模式：

```text
查询 feature
启用 feature
创建相关资源
在 shader 中通过 descriptor 使用资源
```

## 6. Uniform Buffer：每个 Cube 每帧一份矩阵

每个 cube 的 uniform 数据结构是：

```cpp
struct Matrices {
    glm::mat4 projection;
    glm::mat4 view;
    glm::mat4 model;
} matrices;
```

对应 vertex shader：

```glsl
layout (set = 0, binding = 0) uniform UBOMatrices {
    mat4 projection;
    mat4 view;
    mat4 model;
} uboMatrices;
```

uniform buffer 创建在 `prepareUniformBuffers()` 中：

```cpp
for (auto& cube : cubes) {
    for (auto& buffer : cube.uniformBuffers) {
        VK_CHECK_RESULT(vulkanDevice->createBuffer(
            VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
            VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
            VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
            &buffer,
            sizeof(Cube::Matrices)));

        VK_CHECK_RESULT(buffer.map());
    }
}
```

这里有两个层面的重复：

```text
每个 cube 一组 uniform buffer
每个并发帧一份 uniform buffer
```

假设有 2 个 cube，`maxConcurrentFrames` 是 2，那么会创建：

```text
2 * 2 = 4 个 uniform buffer
```

为什么要按 frame-in-flight 复制 uniform buffer？

因为 CPU 和 GPU 是并行工作的。CPU 在准备第 N+1 帧时，GPU 可能还在读取第 N 帧的 uniform buffer。如果只有一份 buffer，CPU 直接覆盖数据，就可能造成 GPU 读取到错误内容。

所以示例为每个并发帧准备独立 buffer：

```cpp
std::array<vks::Buffer, maxConcurrentFrames> uniformBuffers{};
```

更新当前帧时只写：

```cpp
cube.uniformBuffers[currentBuffer]
```

这样当前 CPU 写入不会破坏 GPU 正在使用的另一帧数据。

这个设计也解释了为什么每个 cube 每帧还需要一个 descriptor set：descriptor set 里 binding 0 指向的是当前帧对应的 uniform buffer。

## 7. updateUniformBuffers：对象变换如何进入 shader

每帧更新矩阵时，代码先给两个 cube 设置不同的位置：

```cpp
cubes[0].matrices.model =
    glm::translate(glm::mat4(1.0f), glm::vec3(-2.0f, 0.0f, 0.0f));

cubes[1].matrices.model =
    glm::translate(glm::mat4(1.0f), glm::vec3(1.5f, 0.5f, 0.0f));
```

然后为每个 cube 写入 projection、view、model：

```cpp
for (auto& cube : cubes) {
    cube.matrices.projection = camera.matrices.perspective;
    cube.matrices.view = camera.matrices.view;

    cube.matrices.model = glm::rotate(
        cube.matrices.model,
        glm::radians(cube.rotation.x),
        glm::vec3(1.0f, 0.0f, 0.0f));

    cube.matrices.model = glm::rotate(
        cube.matrices.model,
        glm::radians(cube.rotation.y),
        glm::vec3(0.0f, 1.0f, 0.0f));

    cube.matrices.model = glm::rotate(
        cube.matrices.model,
        glm::radians(cube.rotation.z),
        glm::vec3(0.0f, 0.0f, 1.0f));

    cube.matrices.model = glm::scale(
        cube.matrices.model,
        glm::vec3(0.25f));

    memcpy(
        cube.uniformBuffers[currentBuffer].mapped,
        &cube.matrices,
        sizeof(cube.matrices));
}
```

最终 vertex shader 用它计算顶点位置：

```glsl
gl_Position =
    uboMatrices.projection *
    uboMatrices.view *
    uboMatrices.model *
    vec4(inPos.xyz, 1.0);
```

两个 cube 使用同一份 mesh 顶点，但因为 descriptor set 中的 uniform buffer 不同，shader 读取到的 model matrix 不同，最终出现在屏幕上的位置和旋转也不同。

这就是 descriptor set 在对象级绘制中的作用：**同一个 draw 函数，只要绑定不同对象的 descriptor set，就能让 shader 看到不同对象的数据。**

## 8. Descriptor Pool：提前声明要分配多少 descriptor

descriptor set 不能凭空创建，它必须从 descriptor pool 中分配。`setupDescriptors()` 的第一步就是创建 pool。

示例需要两类 descriptor：

```cpp
std::array<VkDescriptorPoolSize, 2> descriptorPoolSizes{};
```

第一类是 uniform buffer：

```cpp
descriptorPoolSizes[0].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
descriptorPoolSizes[0].descriptorCount =
    static_cast<uint32_t>(cubes.size()) * maxConcurrentFrames;
```

每个 cube、每个并发帧需要一个 uniform buffer descriptor，所以数量是：

```text
cube 数量 * maxConcurrentFrames
```

第二类是 combined image sampler：

```cpp
descriptorPoolSizes[1].type =
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;

descriptorPoolSizes[1].descriptorCount =
    static_cast<uint32_t>(cubes.size()) * maxConcurrentFrames;
```

从资源本身看，texture 是静态的。每个 cube 只有一张 texture，不需要每帧复制 image。

但这个示例把 uniform buffer 和 texture 放在同一个 descriptor set 里，而 descriptor set 又按 cube 和 frame 复制。因此 image sampler descriptor 也被按 frame 复制了。

源码注释也明确说明了这一点：

```text
Images are static, and never update after initial upload,
but as they use the same descriptor set as buffers,
we need to duplicate the descriptor count here too.
```

然后创建 descriptor pool：

```cpp
VkDescriptorPoolCreateInfo descriptorPoolCI = {};
descriptorPoolCI.sType =
    VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
descriptorPoolCI.poolSizeCount =
    static_cast<uint32_t>(descriptorPoolSizes.size());
descriptorPoolCI.pPoolSizes = descriptorPoolSizes.data();
descriptorPoolCI.maxSets =
    static_cast<uint32_t>(cubes.size() * maxConcurrentFrames);

VK_CHECK_RESULT(vkCreateDescriptorPool(
    device,
    &descriptorPoolCI,
    nullptr,
    &descriptorPool));
```

这里有两个数量要区分：

`poolSizeCount` 和 `pPoolSizes` 描述的是 pool 中每种 descriptor 类型分别有多少个。

`maxSets` 描述的是这个 pool 最多能分配多少个 descriptor set。

在这个示例中，set 数量也是：

```text
cube 数量 * maxConcurrentFrames
```

因为每个 cube 每帧一个 descriptor set。

## 9. Descriptor Set Layout：shader 资源接口的声明

descriptor pool 只关心“有多少资源槽位”。真正描述 shader 绑定接口的是 descriptor set layout。

示例创建了两个 binding：

```cpp
std::array<VkDescriptorSetLayoutBinding, 2> setLayoutBindings{};
```

### 9.1 Binding 0：Vertex Shader 的 Uniform Buffer

```cpp
setLayoutBindings[0].descriptorType =
    VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
setLayoutBindings[0].binding = 0;
setLayoutBindings[0].stageFlags =
    VK_SHADER_STAGE_VERTEX_BIT;
setLayoutBindings[0].descriptorCount = 1;
```

这表示：

```text
set = 0
binding = 0
type = uniform buffer
shader stage = vertex shader
array count = 1
```

对应 GLSL：

```glsl
layout (set = 0, binding = 0) uniform UBOMatrices {
    mat4 projection;
    mat4 view;
    mat4 model;
} uboMatrices;
```

这里 `stageFlags` 只给了：

```cpp
VK_SHADER_STAGE_VERTEX_BIT
```

因为矩阵只在 vertex shader 中使用，fragment shader 不需要访问。

### 9.2 Binding 1：Fragment Shader 的 Combined Image Sampler

```cpp
setLayoutBindings[1].descriptorType =
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
setLayoutBindings[1].binding = 1;
setLayoutBindings[1].stageFlags =
    VK_SHADER_STAGE_FRAGMENT_BIT;
setLayoutBindings[1].descriptorCount = 1;
```

这表示：

```text
set = 0
binding = 1
type = combined image sampler
shader stage = fragment shader
array count = 1
```

对应 GLSL：

```glsl
layout (set = 0, binding = 1) uniform sampler2D samplerColorMap;
```

这里 `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER` 表示 image 和 sampler 作为一个组合绑定。fragment shader 采样纹理时同时需要图像视图和采样器状态。

### 9.3 创建 Layout

两个 binding 填好后，创建 descriptor set layout：

```cpp
VkDescriptorSetLayoutCreateInfo descriptorLayoutCI{};
descriptorLayoutCI.sType =
    VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
descriptorLayoutCI.bindingCount =
    static_cast<uint32_t>(setLayoutBindings.size());
descriptorLayoutCI.pBindings = setLayoutBindings.data();

VK_CHECK_RESULT(vkCreateDescriptorSetLayout(
    device,
    &descriptorLayoutCI,
    nullptr,
    &descriptorSetLayout));
```

到这里，应用还没有绑定任何具体 buffer 或 texture。它只是创建了一个接口模板：

```text
这个 set 有两个槽：
    binding 0 是 vertex shader 用的 uniform buffer
    binding 1 是 fragment shader 用的 combined image sampler
```

后面每个 cube 的 descriptor set 都会按照这个 layout 分配。

## 10. Descriptor Set：按照 Layout 分配具体资源绑定表

示例为每个 cube、每个并发帧分配一个 descriptor set：

```cpp
for (auto& cube : cubes) {
    for (auto i = 0; i < cube.uniformBuffers.size(); i++) {
        VkDescriptorSetAllocateInfo allocateInfo{};
        allocateInfo.sType =
            VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
        allocateInfo.descriptorPool = descriptorPool;
        allocateInfo.descriptorSetCount = 1;
        allocateInfo.pSetLayouts = &descriptorSetLayout;

        VK_CHECK_RESULT(vkAllocateDescriptorSets(
            device,
            &allocateInfo,
            &cube.descriptorSets[i]));
    }
}
```

分配出来的 descriptor set 仍然是空的。它知道自己的 layout，但还不知道 binding 0 应该指向哪个 buffer，binding 1 应该指向哪张 texture。

下一步要通过 `vkUpdateDescriptorSets()` 写入具体 descriptor。

## 11. 写入 Binding 0：Uniform Buffer Descriptor

每个 uniform buffer 对象都有一个 descriptor：

```cpp
cube.uniformBuffers[i].descriptor
```

它通常是一个 `VkDescriptorBufferInfo`：

```cpp
VkDescriptorBufferInfo {
    VkBuffer buffer;
    VkDeviceSize offset;
    VkDeviceSize range;
}
```

示例把它写入 binding 0：

```cpp
writeDescriptorSets[0].sType =
    VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
writeDescriptorSets[0].dstSet = cube.descriptorSets[i];
writeDescriptorSets[0].dstBinding = 0;
writeDescriptorSets[0].descriptorType =
    VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
writeDescriptorSets[0].pBufferInfo =
    &cube.uniformBuffers[i].descriptor;
writeDescriptorSets[0].descriptorCount = 1;
```

这段代码的含义是：

```text
把当前 cube、当前 frame 的 uniform buffer
写入当前 descriptor set 的 binding 0
```

因此，当绘制某个 cube 并绑定它的 descriptor set 时，vertex shader 里的：

```glsl
uboMatrices
```

读取到的就是这个 cube 当前帧的矩阵。

## 12. 写入 Binding 1：Texture Descriptor

texture descriptor 写入 binding 1：

```cpp
writeDescriptorSets[1].sType =
    VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
writeDescriptorSets[1].dstSet = cube.descriptorSets[i];
writeDescriptorSets[1].dstBinding = 1;
writeDescriptorSets[1].descriptorType =
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
writeDescriptorSets[1].pImageInfo = &cube.texture.descriptor;
writeDescriptorSets[1].descriptorCount = 1;
```

这里使用的是：

```cpp
pImageInfo
```

而不是：

```cpp
pBufferInfo
```

因为 binding 1 是 image sampler，不是 buffer。

`cube.texture.descriptor` 中包含 sampler、image view 和 image layout。fragment shader 通过：

```glsl
texture(samplerColorMap, inUV)
```

读取对应纹理。

更新 descriptor set：

```cpp
vkUpdateDescriptorSets(
    device,
    static_cast<uint32_t>(writeDescriptorSets.size()),
    writeDescriptorSets.data(),
    0,
    nullptr);
```

从这一刻开始，这个 descriptor set 就变成了一张完整的资源绑定表：

```text
descriptor set for cube N, frame F
    binding 0 -> cube N 的 frame F uniform buffer
    binding 1 -> cube N 的 texture
```

## 13. 为什么静态 Texture 也被复制到每帧 Descriptor Set

从严格工程角度看，这个示例有一个刻意简化的地方：texture 是静态资源，不需要每帧复制 descriptor。

但是示例的 descriptor set layout 把两个资源放在了同一个 set 中：

```text
set 0:
    binding 0 = per-frame uniform buffer
    binding 1 = per-object texture
```

binding 0 需要按 frame 复制，因为 uniform buffer 每帧不同。

binding 1 本来不需要按 frame 复制，因为同一个 cube 的 texture 不变。

但 descriptor set 是整体分配和绑定的。如果每个 frame 都要有自己的 set，那么这个 set 中的所有 binding 都要写完整，所以 texture descriptor 也跟着复制了一份。

源码注释给出了另一个更工程化的方案：

```text
Another option is to put image descriptors into a completely separate set.
```

也就是说，可以把资源分成两组：

```text
set 0: per-frame / per-object matrices
set 1: per-material texture
```

这样每帧变化的 uniform buffer 和静态 texture 就可以分开管理。

例如真实项目中经常会这样组织：

```text
set 0: frame data
    camera
    lights
    time

set 1: material data
    base color texture
    normal texture
    roughness texture
    sampler

set 2: object data
    model matrix
    skinning matrices
```

这个示例没有这样拆，是为了让 descriptor set 教学更直接：一个 cube 一个 set，绑定一次就包含它需要的所有资源。

## 14. Pipeline Layout：把 Descriptor Set Layout 接到 Pipeline 上

descriptor set layout 创建好后，还不能直接用于 draw。graphics pipeline 需要通过 pipeline layout 知道自己会使用哪些 descriptor set layout。

`preparePipelines()` 开头创建 pipeline layout：

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutCI{};
pipelineLayoutCI.sType =
    VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutCI.setLayoutCount = 1;
pipelineLayoutCI.pSetLayouts = &descriptorSetLayout;

VK_CHECK_RESULT(vkCreatePipelineLayout(
    device,
    &pipelineLayoutCI,
    nullptr,
    &pipelineLayout));
```

这里 `setLayoutCount = 1`，表示 pipeline 使用一个 descriptor set layout，也就是 set 0。

这和 shader 中的声明对应：

```glsl
layout (set = 0, binding = 0) uniform UBOMatrices ...
layout (set = 0, binding = 1) uniform sampler2D ...
```

如果 shader 中使用了 set 1，但 pipeline layout 只声明了一个 set layout，就会不匹配。

可以把 pipeline layout 理解为：

```text
pipeline 的资源接口布局
```

它告诉 Vulkan：这条 pipeline 的 shader 期望哪些 descriptor set，以及每个 set 的 layout 是什么。

## 15. Graphics Pipeline：所有 Cube 共用一条 Pipeline

`preparePipelines()` 中创建了一条 graphics pipeline：

```cpp
VkGraphicsPipelineCreateInfo pipelineCI =
    vks::initializers::pipelineCreateInfo(
        pipelineLayout,
        renderPass,
        0);
```

这条 pipeline 使用：

```cpp
descriptorsets/cube.vert.spv
descriptorsets/cube.frag.spv
```

并设置标准的绘制状态：

```cpp
VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST
VK_POLYGON_MODE_FILL
VK_CULL_MODE_BACK_BIT
VK_FRONT_FACE_COUNTER_CLOCKWISE
VK_COMPARE_OP_LESS_OR_EQUAL
VK_SAMPLE_COUNT_1_BIT
```

动态状态包括：

```cpp
VK_DYNAMIC_STATE_VIEWPORT
VK_DYNAMIC_STATE_SCISSOR
```

顶点输入来自 glTF helper：

```cpp
pipelineCI.pVertexInputState =
    vkglTF::Vertex::getPipelineVertexInputState({
        vkglTF::VertexComponent::Position,
        vkglTF::VertexComponent::Normal,
        vkglTF::VertexComponent::UV,
        vkglTF::VertexComponent::Color
    });
```

对应 vertex shader 输入：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inNormal;
layout (location = 2) in vec2 inUV;
layout (location = 3) in vec3 inColor;
```

最后创建 pipeline：

```cpp
shaderStages[0] = loadShader(
    getShadersPath() + "descriptorsets/cube.vert.spv",
    VK_SHADER_STAGE_VERTEX_BIT);

shaderStages[1] = loadShader(
    getShadersPath() + "descriptorsets/cube.frag.spv",
    VK_SHADER_STAGE_FRAGMENT_BIT);

VK_CHECK_RESULT(vkCreateGraphicsPipelines(
    device,
    pipelineCache,
    1,
    &pipelineCI,
    nullptr,
    &pipeline));
```

注意：这个示例只有一条 pipeline。两个 cube 的差异不来自 pipeline，而来自 descriptor set。

这和 `pipelines.cpp` 正好形成对照：

```text
pipelines.cpp:
    同一资源，切换 pipeline，得到不同 shading 风格。

descriptorsets.cpp:
    同一 pipeline，切换 descriptor set，得到不同对象数据和纹理。
```

## 16. Shader 如何消费 Descriptor

vertex shader：

```glsl
layout (set = 0, binding = 0) uniform UBOMatrices {
    mat4 projection;
    mat4 view;
    mat4 model;
} uboMatrices;

void main()
{
    outNormal = inNormal;
    outColor = inColor;
    outUV = inUV;

    gl_Position =
        uboMatrices.projection *
        uboMatrices.view *
        uboMatrices.model *
        vec4(inPos.xyz, 1.0);
}
```

它通过 binding 0 读取矩阵，完成顶点变换。

fragment shader：

```glsl
layout (set = 0, binding = 1) uniform sampler2D samplerColorMap;

void main()
{
    outFragColor =
        texture(samplerColorMap, inUV) *
        vec4(inColor, 1.0);
}
```

它通过 binding 1 采样纹理，再乘以顶点颜色。

因此，绑定不同 descriptor set 会同时改变两个 shader stage 的输入：

```text
vertex shader 看到不同矩阵
fragment shader 看到不同纹理
```

这就是两个 cube 可以使用同一份 shader，却显示为不同位置、不同旋转、不同纹理的原因。

## 17. buildCommandBuffer：绑定一次 Pipeline，多次绑定 Descriptor Set

绘制逻辑在 `buildCommandBuffer()` 中。

开始 render pass 后，先绑定 pipeline：

```cpp
vkCmdBindPipeline(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipeline);
```

设置 viewport 和 scissor：

```cpp
VkViewport viewport =
    vks::initializers::viewport(
        (float)width,
        (float)height,
        0.0f,
        1.0f);

vkCmdSetViewport(cmdBuffer, 0, 1, &viewport);

VkRect2D scissor =
    vks::initializers::rect2D(width, height, 0, 0);

vkCmdSetScissor(cmdBuffer, 0, 1, &scissor);
```

绑定共享模型 buffer：

```cpp
model.bindBuffers(cmdBuffer);
```

然后进入最关键的循环：

```cpp
for (auto cube : cubes) {
    vkCmdBindDescriptorSets(
        cmdBuffer,
        VK_PIPELINE_BIND_POINT_GRAPHICS,
        pipelineLayout,
        0,
        1,
        &cube.descriptorSets[currentBuffer],
        0,
        nullptr);

    model.draw(cmdBuffer);
}
```

这段命令流可以翻译成：

```text
绑定 pipeline
绑定 cube mesh 的 vertex/index buffer

绑定 cube 0 当前帧 descriptor set
绘制 cube mesh

绑定 cube 1 当前帧 descriptor set
绘制 cube mesh
```

两次 draw 使用的是同一个 `model.draw(cmdBuffer)`，但当前 descriptor set 不同，所以 shader 读取到的矩阵和纹理不同。

这就是 descriptor set 在 draw call 级别的作用。

一个小的工程细节是，这里写的是：

```cpp
for (auto cube : cubes)
```

这会复制 `Cube` 对象。示例里只是读取 descriptor handle 和绘制，因此没有行为问题。但真实项目中更推荐写成：

```cpp
for (const auto& cube : cubes)
```

这样可以避免复制包含 buffer、texture、descriptor set 等 Vulkan 资源句柄的结构体，也更符合资源对象的语义。

## 18. Descriptor Set 的绑定参数拆解

这行调用值得单独拆开：

```cpp
vkCmdBindDescriptorSets(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    0,
    1,
    &cube.descriptorSets[currentBuffer],
    0,
    nullptr);
```

参数含义如下。

`cmdBuffer`：要录制绑定命令的 command buffer。

`VK_PIPELINE_BIND_POINT_GRAPHICS`：绑定到 graphics pipeline，而不是 compute pipeline。

`pipelineLayout`：必须和当前 pipeline 使用的 layout 兼容。

`0`：firstSet，表示从 set 0 开始绑定。

`1`：descriptorSetCount，表示绑定一个 descriptor set。

`&cube.descriptorSets[currentBuffer]`：要绑定的 descriptor set。

`0` 和 `nullptr`：dynamic offset 数量和数组。这个示例没有使用 dynamic uniform buffer，所以不需要 dynamic offset。

绑定后，shader 中所有 `set = 0` 的资源引用都会解析到这个 descriptor set。

如果后续又绑定了另一个 set 0 descriptor set，那么下一次 draw 使用的资源就会变成新的 set。

这也解释了为什么 draw loop 中的顺序必须是：

```text
bind descriptor set
draw
bind another descriptor set
draw
```

descriptor set 绑定是 command buffer 状态的一部分，它影响的是之后的 draw call。

## 19. Descriptor Set、Pipeline Layout、Shader Binding 三者必须匹配

理解这个示例时，最重要的是把三侧关系对齐。

shader 侧：

```glsl
layout (set = 0, binding = 0) uniform UBOMatrices ...
layout (set = 0, binding = 1) uniform sampler2D ...
```

descriptor set layout 侧：

```cpp
binding 0:
    type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER
    stage = VK_SHADER_STAGE_VERTEX_BIT

binding 1:
    type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER
    stage = VK_SHADER_STAGE_FRAGMENT_BIT
```

pipeline layout 侧：

```cpp
setLayoutCount = 1
pSetLayouts = &descriptorSetLayout
```

command buffer 侧：

```cpp
vkCmdBindDescriptorSets(..., pipelineLayout, 0, 1, &descriptorSet, ...);
```

这四处必须共同描述同一个资源接口。

常见错误包括：

- shader 用了 binding 1，但 descriptor set layout 没有 binding 1。
- shader 声明 sampler2D，但 C++ 写成 uniform buffer。
- shader 在 fragment stage 使用资源，但 layout 的 `stageFlags` 只写了 vertex stage。
- pipeline layout 没有包含对应 set layout。
- command buffer 绑定 descriptor set 时使用了不兼容的 pipeline layout。
- 当前帧绑定了错误 frame index 的 descriptor set。

Vulkan validation layer 对这些错误通常能给出明确提示，但理解这套关系能让调试效率高很多。

## 20. 为什么 Descriptor Set 是 Vulkan 资源绑定的核心

这个示例虽然只画两个 cube，但它已经包含 Vulkan descriptor 体系的主要概念：

```text
VkDescriptorPool
VkDescriptorSetLayout
VkDescriptorSet
VkWriteDescriptorSet
VkDescriptorBufferInfo
VkDescriptorImageInfo
VkPipelineLayout
vkCmdBindDescriptorSets
```

它们分别承担不同职责。

`VkDescriptorPool` 负责分配 descriptor set，并预留各种 descriptor 类型的容量。

`VkDescriptorSetLayout` 负责声明 shader 资源接口。

`VkDescriptorSet` 是按照 layout 分配出来的具体资源绑定表。

`VkWriteDescriptorSet` 描述一次写入操作，把具体 buffer 或 image 写到某个 set 的某个 binding。

`VkDescriptorBufferInfo` 描述 buffer descriptor 的资源信息。

`VkDescriptorImageInfo` 描述 image/sampler descriptor 的资源信息。

`VkPipelineLayout` 把 descriptor set layout 接到 pipeline 上。

`vkCmdBindDescriptorSets` 在 command buffer 中选择当前 draw call 使用哪组资源。

很多 Vulkan 初学者会觉得 descriptor 系统繁琐，是因为它把传统 API 中隐含的全局绑定状态拆成了显式对象。但拆开以后，资源绑定关系更清楚，也更利于多线程录制、批处理和减少运行时不确定性。

## 21. 和真实项目的资源分层

`descriptorsets.cpp` 采用的是最适合教学的组织方式：

```text
每个 cube 每帧一个 descriptor set
set 内同时包含矩阵和纹理
```

真实项目中，通常会根据更新频率拆分 descriptor set。

例如：

```text
set 0: per-frame
    projection
    view
    camera position
    lights

set 1: per-material
    base color texture
    normal texture
    metallic roughness texture
    sampler

set 2: per-object
    model matrix
    object id
    skinning buffer
```

这样做的好处是：

- per-frame set 每帧绑定一次或少量次数。
- per-material set 只有材质变化时切换。
- per-object set 只有对象变化时切换。
- 静态纹理 descriptor 不需要按 frame 重复写入。
- 可以根据更新频率设计 descriptor allocator 和缓存策略。

如果套回这个示例，两个 cube 可以改成：

```text
set 0: 当前帧 camera 数据
set 1: cube 材质纹理
set 2: cube model matrix
```

但这会增加代码复杂度，不适合作为入门示例。当前写法牺牲了一点资源复用效率，换来了非常清晰的教学结构。

## 22. Uniform Buffer 的另一种组织方式

源码注释中还提到：

```text
Another option instead of using separate buffers could be using one large buffer with separate ranges per cube and frame
```

也就是说，除了为每个 cube 每帧创建一个独立 buffer，还可以创建一个大 uniform buffer，然后为不同 cube 和 frame 使用不同 offset/range。

例如：

```text
big uniform buffer
    frame 0 cube 0 matrices
    frame 0 cube 1 matrices
    frame 1 cube 0 matrices
    frame 1 cube 1 matrices
```

descriptor 可以指向不同 range：

```cpp
VkDescriptorBufferInfo {
    buffer = bigBuffer;
    offset = objectFrameOffset;
    range = sizeof(Cube::Matrices);
}
```

或者使用 dynamic uniform buffer：

```cpp
VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC
```

然后在 `vkCmdBindDescriptorSets()` 传入 dynamic offset。

这种方式可以减少 buffer 数量，但需要处理 uniform buffer offset alignment，例如：

```cpp
minUniformBufferOffsetAlignment
```

教学示例选择多个小 buffer，是为了避免对齐和动态 offset 干扰 descriptor set 的核心概念。

## 23. Descriptor 更新和渲染时绑定的区别

这个示例还很适合区分两个操作：

```text
vkUpdateDescriptorSets
vkCmdBindDescriptorSets
```

`vkUpdateDescriptorSets` 是 CPU 侧操作，用来把具体资源写进 descriptor set。

它发生在初始化阶段：

```cpp
vkUpdateDescriptorSets(device, writeCount, writes, 0, nullptr);
```

`vkCmdBindDescriptorSets` 是 command buffer 录制操作，用来指定后续 draw call 使用哪个 descriptor set。

它发生在绘制阶段：

```cpp
vkCmdBindDescriptorSets(cmdBuffer, ..., &descriptorSet, ...);
```

二者不是一回事。

可以把 descriptor set 想象成一张资源表：

```text
vkUpdateDescriptorSets:
    修改资源表里的内容

vkCmdBindDescriptorSets:
    选择当前 draw 使用哪张资源表
```

在这个示例中，descriptor set 内容初始化后基本不变。每帧变化的是 uniform buffer 中的数据，而不是 descriptor set 指向的 buffer。

换句话说：

```text
descriptor set 仍然指向同一个当前帧 buffer
buffer 内容每帧被 CPU 更新
```

这也是 Vulkan 中很常见的做法：descriptor 绑定关系相对稳定，buffer 内容动态更新。

## 24. 资源生命周期

析构函数中销毁了示例自己创建的资源：

```cpp
vkDestroyPipeline(device, pipeline, nullptr);
vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);

for (auto& cube : cubes) {
    cube.texture.destroy();
    for (auto& buffer : cube.uniformBuffers) {
        buffer.destroy();
    }
}
```

这里销毁了：

- graphics pipeline。
- pipeline layout。
- descriptor set layout。
- 每个 cube 的 texture。
- 每个 cube 的 uniform buffer。

descriptor set 本身通常不需要逐个释放，除非 descriptor pool 创建时使用了允许单独释放的 flag。很多示例会让 descriptor set 随 descriptor pool 一起销毁。

这个示例中的 `descriptorPool` 来自框架基类，生命周期由示例框架统一管理。

一个关键原则是：descriptor set 中引用的资源必须在 GPU 使用期间保持有效。例如 descriptor set 指向某个 uniform buffer，那么这个 buffer 不能在相关 command buffer 执行完成前被销毁。

## 25. 从这个示例提炼出的工程原则

第一，descriptor set layout 应该和 shader binding 精确对应。

shader 写了：

```glsl
layout (set = 0, binding = 0)
```

C++ 侧就必须创建 set 0 中 binding 0 的 layout，并且 descriptor 类型、数量、stage flags 都要合理匹配。

第二，descriptor pool 要按实际需求估算容量。

这个示例需要：

```text
uniform buffer descriptor: cube 数量 * frame 数量
image sampler descriptor: cube 数量 * frame 数量
descriptor set: cube 数量 * frame 数量
```

真实项目中 descriptor pool 容量不足是常见错误，尤其是在材质、纹理和 frame-in-flight 数量增加后。

第三，按更新频率拆 descriptor set。

示例为了简单，把 per-frame uniform 和 per-object texture 放在同一个 set。真实项目中通常要拆分成 frame、material、object 等层级，减少重复 descriptor 和绑定成本。

第四，尽量让 descriptor set 内容稳定。

频繁调用 `vkUpdateDescriptorSets()` 会增加 CPU 管理成本。很多项目会在资源加载或材质创建时写好 descriptor set，渲染时只绑定。

第五，descriptor set 切换是 draw call 组织的一部分。

如果大量 draw 都使用相同 pipeline，但不同材质，那么渲染器需要考虑如何排序 draw call，减少 pipeline 切换和 descriptor set 切换。

## 26. 一个简化版 mental model

理解这个示例，可以把绘制两个 cube 的过程想成下面这样：

```text
初始化：
    创建 cube mesh
    加载 texture A
    加载 texture B
    创建 pipeline

    为 cube A 创建 descriptor set:
        binding 0 -> cube A uniform buffer
        binding 1 -> texture A

    为 cube B 创建 descriptor set:
        binding 0 -> cube B uniform buffer
        binding 1 -> texture B

每帧：
    更新 cube A uniform buffer
    更新 cube B uniform buffer

    绑定 pipeline
    绑定 mesh buffer

    绑定 cube A descriptor set
    draw cube

    绑定 cube B descriptor set
    draw cube
```

这就是 descriptor set 最朴素、也最重要的用法。

## 27. 总结

`descriptorsets.cpp` 是 Vulkan descriptor 系统的入门核心示例。

它通过两个立方体展示了：

- 如何创建 descriptor pool。
- 如何描述 descriptor set layout。
- 如何为每个对象、每个并发帧分配 descriptor set。
- 如何把 uniform buffer 写入 binding 0。
- 如何把 combined image sampler 写入 binding 1。
- 如何通过 pipeline layout 把 descriptor set layout 接到 graphics pipeline。
- 如何在 command buffer 中为每个 draw call 绑定不同 descriptor set。
- 如何让同一个 mesh、同一条 pipeline、同一组 shader 绘制出不同对象。

它的核心思想可以浓缩成一句话：

```text
Pipeline 决定怎么画，descriptor set 决定用哪些数据画。
```

在这个示例中，两个 cube 的“怎么画”完全相同，因为它们共享 pipeline；两个 cube 的“用哪些数据画”不同，因为它们绑定了不同 descriptor set。

理解这一点之后，再看更复杂的 Vulkan 示例，比如 glTF 材质、PBR、阴影、后处理、bindless texture、descriptor indexing，就能更容易看出它们本质上都在解决同一个问题：**如何把大量 GPU 资源用清晰、可扩展、低开销的方式暴露给 shader。**

