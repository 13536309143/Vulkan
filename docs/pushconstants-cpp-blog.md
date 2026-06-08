# Vulkan 示例解析：pushconstants.cpp 如何用 Push Constants 传递每个物体的小数据

本文分析的是 `examples/pushconstants/pushconstants.cpp`。这个示例展示了 Vulkan 中一种非常轻量的 shader 数据传递方式：**Push Constants**。

Push constants 适合传递很小、更新频繁、和 draw call 强相关的数据，例如：

- 当前对象的颜色。
- 当前对象的位置。
- 一个小型变换参数。
- 材质开关。
- object id。
- 少量标量参数。

这个示例最终绘制 16 个球体：

- 所有球体共享同一个 glTF sphere 模型。
- 所有球体共享同一条 graphics pipeline。
- 所有球体共享同一个 descriptor set，里面只有一个 uniform buffer。
- uniform buffer 存放 projection、view 和基础 model 矩阵。
- 每个球体自己的颜色和位置通过 push constants 传入。

绘制循环的核心代码是：

```cpp
for (uint32_t j = 0; j < spherecount; j++) {
    vkCmdPushConstants(
        cmdBuffer,
        pipelineLayout,
        VK_SHADER_STAGE_VERTEX_BIT,
        0,
        sizeof(SpherePushConstantData),
        &spheres[j]);

    model.draw(cmdBuffer);
}
```

也就是说，每次 draw 前推送一小块数据，shader 读取这块数据来决定当前球体的颜色和位置。

这篇文章会从 Vulkan 资源绑定模型、pipeline layout、push constant range、shader 声明、command buffer 状态、和 UBO/dynamic UBO 的对比几个角度完整拆解这个示例。

## 1. Push Constants 解决什么问题

在 Vulkan 中，把数据传给 shader 的常见方式包括：

- uniform buffer
- storage buffer
- sampled image / sampler
- push constants
- specialization constants
- vertex attributes

其中 push constants 的定位很特殊：它不是 descriptor，也不需要 descriptor set。它是一小块可以直接写入 command buffer 状态的 shader 可见数据。

源码开头的注释说明了它的用途：

```cpp
/*
* Using push constants it's possible to pass a small bit of static data to a shader,
* which is stored in the command buffer state.
* This is perfect for passing e.g. static per-object data or parameters without the need for descriptor sets.
* The sample uses these to push different static parameters for rendering multiple objects.
*/
```

这段话里有三个重点。

第一，数据很小：

```text
a small bit of data
```

Vulkan 规范只要求设备至少支持 128 bytes 的 push constants。实际设备可能支持更多，但不能假设它很大。

第二，它属于 command buffer state：

```text
stored in the command buffer state
```

调用 `vkCmdPushConstants()` 会把数据记录进 command buffer。它影响后续 draw call，直到被新的 push constant 写入覆盖。

第三，它不需要 descriptor set：

```text
without the need for descriptor sets
```

这就是 push constants 最大的便利点。对于每个 draw 都要变的一点小数据，没必要创建 buffer、descriptor pool、descriptor set layout、descriptor set，再更新 descriptor。

## 2. 示例最终画了什么

示例加载了一个 sphere 模型：

```cpp
model.loadFromFile(
    getAssetPath() + "models/sphere.gltf",
    vulkanDevice,
    queue,
    glTFLoadingFlags);
```

然后创建 16 份球体参数：

```cpp
std::array<SpherePushConstantData, 16> spheres;
```

每个球体有：

```cpp
struct SpherePushConstantData {
    glm::vec4 color;
    glm::vec4 position;
};
```

这正好是 32 bytes：

```text
vec4 color    = 16 bytes
vec4 position = 16 bytes
total         = 32 bytes
```

每个球体的位置被放到一个圆环上，颜色随机生成。最终画面就是一圈不同颜色的球体。

关键点在于：球体并没有各自的 uniform buffer，也没有各自的 descriptor set。它们的 per-object 数据完全通过 push constants 提供。

## 3. 程序整体结构

核心成员如下：

```cpp
vkglTF::Model model;

struct SpherePushConstantData {
    glm::vec4 color;
    glm::vec4 position;
};
std::array<SpherePushConstantData, 16> spheres;

struct UniformData {
    glm::mat4 projection;
    glm::mat4 model;
    glm::mat4 view;
} uniformData;

std::array<vks::Buffer, maxConcurrentFrames> uniformBuffers;

VkPipeline pipeline{ VK_NULL_HANDLE };
VkPipelineLayout pipelineLayout{ VK_NULL_HANDLE };
VkDescriptorSetLayout descriptorSetLayout{ VK_NULL_HANDLE };
std::array<VkDescriptorSet, maxConcurrentFrames> descriptorSets{};
```

这里有两类 shader 数据。

第一类是全局或每帧数据：

```cpp
UniformData
```

它包括 projection、model、view 矩阵，通过普通 uniform buffer 传入。

第二类是每个球体自己的小数据：

```cpp
SpherePushConstantData
```

它包括 color 和 position，通过 push constants 传入。

这正是这个示例的教学重点：**不是所有 shader 数据都要放进同一种资源里。**

可以按更新频率和数据大小分层：

```text
相机矩阵:
    每帧更新，所有对象共享，适合 UBO。

球体颜色和位置:
    每个 draw 不同，只有 32 bytes，适合 push constants。
```

## 4. 初始化流程

`prepare()` 函数组织了整个初始化：

```cpp
void prepare()
{
    VulkanExampleBase::prepare();
    loadAssets();
    setupSpheres();
    prepareUniformBuffers();
    setupDescriptors();
    preparePipelines();
    prepared = true;
}
```

顺序是：

1. 初始化基础 Vulkan 资源。
2. 加载 sphere glTF 模型。
3. 初始化 16 个球体的颜色和位置。
4. 创建每帧 uniform buffer。
5. 创建 descriptor set layout、descriptor set，把 UBO 绑定给 shader。
6. 创建包含 push constant range 的 pipeline layout 和 graphics pipeline。

这里要注意：push constants 不需要 descriptor pool，也不需要 descriptor set。它们只需要在 pipeline layout 中声明 range。

## 5. setupSpheres：生成每个球体的 Push Constant 数据

每个球体的数据在 `setupSpheres()` 中生成：

```cpp
void setupSpheres()
{
    std::random_device rndDevice;
    std::default_random_engine rndEngine(
        benchmark.active ? 0 : rndDevice());

    std::uniform_real_distribution<float> rndDist(0.1f, 1.0f);

    for (uint32_t i = 0; i < spheres.size(); i++) {
        spheres[i].color = glm::vec4(
            rndDist(rndEngine),
            rndDist(rndEngine),
            rndDist(rndEngine),
            1.0f);

        const float rad = glm::radians(
            i * 360.0f /
            static_cast<uint32_t>(spheres.size()));

        spheres[i].position =
            glm::vec4(glm::vec3(sin(rad), cos(rad), 0.0f) * 3.5f, 1.0f);
    }
}
```

颜色是随机的：

```cpp
spheres[i].color = glm::vec4(r, g, b, 1.0f);
```

位置在圆环上：

```cpp
sin(rad), cos(rad), 0.0f
```

半径是：

```cpp
3.5f
```

如果 benchmark 模式开启，随机种子固定为 0，保证结果可复现。否则使用 `std::random_device`。

这组数据后面不会写入 uniform buffer，也不会写入 descriptor set，而是在 draw loop 中直接 push 到 command buffer。

## 6. Uniform Buffer：相机矩阵仍然走 Descriptor Set

这个示例没有把所有数据都放进 push constants。相机矩阵仍然使用 uniform buffer。

数据结构：

```cpp
struct UniformData {
    glm::mat4 projection;
    glm::mat4 model;
    glm::mat4 view;
} uniformData;
```

对应 vertex shader：

```glsl
layout (binding = 0) uniform UBO
{
    mat4 projection;
    mat4 model;
    mat4 view;
} ubo;
```

创建 uniform buffers：

```cpp
for (auto& buffer : uniformBuffers) {
    VK_CHECK_RESULT(vulkanDevice->createBuffer(
        VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
        VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
        VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
        &buffer,
        sizeof(UniformData),
        &uniformData));

    VK_CHECK_RESULT(buffer.map());
}
```

这里每个并发帧一份 UBO：

```cpp
std::array<vks::Buffer, maxConcurrentFrames> uniformBuffers;
```

这是为了避免 CPU 更新当前帧 uniform 数据时覆盖 GPU 仍在读取的上一帧数据。

每帧更新：

```cpp
void updateUniformBuffers()
{
    uniformData.projection = camera.matrices.perspective;
    uniformData.view = camera.matrices.view;
    uniformData.model =
        glm::scale(glm::mat4(1.0f), glm::vec3(0.5f));

    memcpy(
        uniformBuffers[currentBuffer].mapped,
        &uniformData,
        sizeof(UniformData));
}
```

这里 `model` 矩阵只做统一缩放：

```cpp
glm::scale(..., glm::vec3(0.5f))
```

每个球体的位置偏移没有放进这个 UBO，而是通过 push constants 提供。

## 7. Descriptor Set：这里只负责绑定 UBO

`setupDescriptors()` 创建 descriptor pool：

```cpp
std::vector<VkDescriptorPoolSize> poolSizes = {
    vks::initializers::descriptorPoolSize(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        maxConcurrentFrames),
};

VkDescriptorPoolCreateInfo descriptorPoolInfo =
    vks::initializers::descriptorPoolCreateInfo(
        poolSizes,
        maxConcurrentFrames);

VK_CHECK_RESULT(vkCreateDescriptorPool(
    device,
    &descriptorPoolInfo,
    nullptr,
    &descriptorPool));
```

这里只需要普通 uniform buffer descriptor，因为 push constants 不从 descriptor pool 分配。

descriptor set layout 也只有一个 binding：

```cpp
std::vector<VkDescriptorSetLayoutBinding> setLayoutBindings = {
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        VK_SHADER_STAGE_VERTEX_BIT,
        0),
};
```

含义是：

```text
set 0 binding 0
type = uniform buffer
stage = vertex shader
```

对应 shader：

```glsl
layout (binding = 0) uniform UBO
```

然后每个并发帧分配一个 descriptor set：

```cpp
for (auto i = 0; i < uniformBuffers.size(); i++) {
    VK_CHECK_RESULT(vkAllocateDescriptorSets(
        device,
        &allocInfo,
        &descriptorSets[i]));

    std::vector<VkWriteDescriptorSet> writeDescriptorSets = {
        vks::initializers::writeDescriptorSet(
            descriptorSets[i],
            VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
            0,
            &uniformBuffers[i].descriptor),
    };

    vkUpdateDescriptorSets(
        device,
        static_cast<uint32_t>(writeDescriptorSets.size()),
        writeDescriptorSets.data(),
        0,
        nullptr);
}
```

到这里，descriptor set 只解决一件事：让 vertex shader 能读取当前帧的 projection/model/view 矩阵。

球体颜色和位置不在 descriptor set 中。

## 8. Push Constant Range：在 Pipeline Layout 中声明

Push constants 的关键不是 descriptor set layout，而是 pipeline layout。

`preparePipelines()` 开头创建了一个 `VkPushConstantRange`：

```cpp
VkPushConstantRange pushConstantRange{};
pushConstantRange.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
pushConstantRange.offset = 0;
pushConstantRange.size = sizeof(SpherePushConstantData);
```

这里声明了三件事。

第一，哪些 shader stage 可以访问这段 push constants：

```cpp
VK_SHADER_STAGE_VERTEX_BIT
```

这个示例中只有 vertex shader 读取 push constants，所以只给 vertex stage。

第二，offset：

```cpp
0
```

表示这段 push constant range 从 push constant 存储的第 0 字节开始。

第三，size：

```cpp
sizeof(SpherePushConstantData)
```

也就是 32 bytes。

然后把 range 挂到 pipeline layout 上：

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutCreateInfo =
    vks::initializers::pipelineLayoutCreateInfo(
        &descriptorSetLayout,
        1);

pipelineLayoutCreateInfo.pushConstantRangeCount = 1;
pipelineLayoutCreateInfo.pPushConstantRanges = &pushConstantRange;

VK_CHECK_RESULT(vkCreatePipelineLayout(
    device,
    &pipelineLayoutCreateInfo,
    nullptr,
    &pipelineLayout));
```

这一步非常关键。

shader 里虽然写了：

```glsl
layout(push_constant) uniform PushConsts
```

但 Vulkan pipeline layout 也必须声明对应的 push constant range。否则 pipeline layout 和 shader 接口不匹配。

可以把 pipeline layout 理解为这条 pipeline 的资源接口声明：

```text
descriptor set layouts:
    set 0 binding 0 = uniform buffer

push constant ranges:
    offset 0, size 32, vertex shader 可见
```

## 9. Shader 中如何声明 Push Constants

vertex shader 位于：

```text
shaders/glsl/pushconstants/pushconstants.vert
```

普通 UBO：

```glsl
layout (binding = 0) uniform UBO
{
    mat4 projection;
    mat4 model;
    mat4 view;
} ubo;
```

push constants：

```glsl
layout(push_constant) uniform PushConsts {
    vec4 color;
    vec4 position;
} pushConsts;
```

这里的 layout 和 C++ 结构体对应：

```cpp
struct SpherePushConstantData {
    glm::vec4 color;
    glm::vec4 position;
};
```

二者都按：

```text
vec4 color
vec4 position
```

排列。

在 shader 中使用：

```glsl
outColor = inColor * pushConsts.color.rgb;

vec3 locPos = vec3(ubo.model * vec4(inPos, 1.0));
vec3 worldPos = locPos + pushConsts.position.xyz;

gl_Position =
    ubo.projection *
    ubo.view *
    vec4(worldPos, 1.0);
```

也就是说：

- UBO 的 `model` 负责统一缩放 sphere。
- push constant 的 `position` 负责把每个 sphere 放到不同位置。
- push constant 的 `color` 负责给每个 sphere 乘上不同颜色。

fragment shader 很简单：

```glsl
layout (location = 0) in vec3 inColor;
layout (location = 0) out vec4 outFragColor;

void main()
{
    outFragColor.rgb = inColor;
}
```

它只输出 vertex shader 传来的颜色。严格来说，这里只写了 `rgb`，没有显式写 alpha。因为示例没有启用 blending，alpha 对最终颜色没有实际影响。但在工程代码中，更稳妥的写法通常是：

```glsl
outFragColor = vec4(inColor, 1.0);
```

## 10. Graphics Pipeline：Push Constants 不改变固定功能状态

除了 pipeline layout 多了 push constant range，graphics pipeline 的创建过程比较常规。

输入装配：

```cpp
VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST
```

光栅化：

```cpp
VK_POLYGON_MODE_FILL
VK_CULL_MODE_BACK_BIT
VK_FRONT_FACE_COUNTER_CLOCKWISE
```

深度测试：

```cpp
VK_TRUE,
VK_TRUE,
VK_COMPARE_OP_LESS_OR_EQUAL
```

动态状态：

```cpp
VK_DYNAMIC_STATE_VIEWPORT
VK_DYNAMIC_STATE_SCISSOR
```

顶点输入使用 glTF helper：

```cpp
pipelineCI.pVertexInputState =
    vkglTF::Vertex::getPipelineVertexInputState({
        vkglTF::VertexComponent::Position,
        vkglTF::VertexComponent::Normal,
        vkglTF::VertexComponent::Color
    });
```

对应 shader 输入：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inNormal;
layout (location = 2) in vec3 inColor;
```

加载 shader：

```cpp
shaderStages[0] = loadShader(
    getShadersPath() + "pushconstants/pushconstants.vert.spv",
    VK_SHADER_STAGE_VERTEX_BIT);

shaderStages[1] = loadShader(
    getShadersPath() + "pushconstants/pushconstants.frag.spv",
    VK_SHADER_STAGE_FRAGMENT_BIT);
```

最后创建 pipeline：

```cpp
VK_CHECK_RESULT(vkCreateGraphicsPipelines(
    device,
    pipelineCache,
    1,
    &pipelineCI,
    nullptr,
    &pipeline));
```

Push constants 不是 pipeline 的固定功能状态。它们是 shader 资源接口的一部分，所以体现在 pipeline layout 里，而不是 rasterization、depth、blend 这些状态里。

## 11. Command Buffer：每个 Draw 前 Push 一份小数据

绘制逻辑在 `buildCommandBuffer()` 中。

先开始 command buffer 和 render pass，然后设置 viewport/scissor：

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

绑定 pipeline：

```cpp
vkCmdBindPipeline(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipeline);
```

绑定当前帧 descriptor set：

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

这个 descriptor set 提供 UBO 中的 projection/model/view。

然后循环绘制 16 个 sphere：

```cpp
uint32_t spherecount =
    static_cast<uint32_t>(spheres.size());

for (uint32_t j = 0; j < spherecount; j++) {
    vkCmdPushConstants(
        cmdBuffer,
        pipelineLayout,
        VK_SHADER_STAGE_VERTEX_BIT,
        0,
        sizeof(SpherePushConstantData),
        &spheres[j]);

    model.draw(cmdBuffer);
}
```

每次循环做两件事：

```text
push 当前 sphere 的 color + position
draw sphere mesh
```

这意味着 `vkCmdPushConstants()` 的数据会影响紧跟着的 `model.draw(cmdBuffer)`。

下一轮循环又推送新的数据，覆盖同一段 push constant range，然后再绘制下一个 sphere。

命令流可以概括成：

```text
bind pipeline
bind descriptor set with UBO

push sphere 0 color/position
draw sphere

push sphere 1 color/position
draw sphere

push sphere 2 color/position
draw sphere

...
```

这就是 push constants 最典型的使用方式。

## 12. vkCmdPushConstants 参数拆解

核心调用如下：

```cpp
vkCmdPushConstants(
    cmdBuffer,
    pipelineLayout,
    VK_SHADER_STAGE_VERTEX_BIT,
    0,
    sizeof(SpherePushConstantData),
    &spheres[j]);
```

逐个看参数。

`cmdBuffer`：当前正在录制的 command buffer。

`pipelineLayout`：必须包含与这次 push 操作兼容的 push constant range。

`VK_SHADER_STAGE_VERTEX_BIT`：这次写入的数据对 vertex shader 可见。它必须被 pipeline layout 中对应 range 的 stageFlags 覆盖。

`0`：写入 push constant 存储的起始 offset。

`sizeof(SpherePushConstantData)`：写入大小，当前是 32 bytes。

`&spheres[j]`：源数据指针。

这里 offset 和 size 都要符合 Vulkan 对 push constants 的要求。常见约束包括：

- offset 必须是 4 的倍数。
- size 必须是 4 的倍数。
- offset + size 不能超过设备的 `maxPushConstantsSize`。
- 写入的 stage flags 必须和 pipeline layout 中声明的 range 兼容。

示例使用两个 `vec4`，天然满足 4 字节对齐和大小要求。

## 13. Push Constants 是 Command Buffer State

理解 push constants 时，一个关键点是：它们是 command buffer state。

调用：

```cpp
vkCmdPushConstants(...)
```

不是立即把数据传给 GPU 执行，而是把“写入 push constant 数据”这条命令记录到 command buffer 中。

当 command buffer 被提交并执行时，GPU 会按录制顺序看到：

```text
push sphere 0 data
draw
push sphere 1 data
draw
push sphere 2 data
draw
```

push constant 的状态会被后续 draw 使用，直到同一范围的数据被新的 `vkCmdPushConstants()` 覆盖。

因此，如果你写成：

```cpp
vkCmdPushConstants(..., sphere0);
vkCmdPushConstants(..., sphere1);
draw;
```

这个 draw 使用的是 sphere1，而不是 sphere0。

顺序非常重要。

## 14. Push Constants 和 Descriptor Set 的关系

这个示例同时使用了 descriptor set 和 push constants。

descriptor set 负责：

```text
binding 0 -> UniformData
```

push constants 负责：

```text
color
position
```

二者互不替代，而是互补。

可以这样理解：

```text
Descriptor set:
    适合绑定 buffer、image、sampler 等资源。
    适合较大数据或需要长期存在的资源。
    需要 layout、pool、set、update。

Push constants:
    适合很小的数据块。
    不需要 descriptor。
    直接记录进 command buffer。
    大小有限，但更新非常轻。
```

在这个示例中，如果把 16 个 sphere 的 color/position 都放进 uniform buffer，也完全可以。但那样需要额外管理 per-object buffer 或数组索引。对于 32 bytes 的 per-draw 数据，push constants 更直接。

## 15. Push Constants 和 Dynamic Uniform Buffer 的区别

`dynamicuniformbuffer.cpp` 用的是：

```cpp
VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC
```

它的模式是：

```text
一个大 uniform buffer
每个对象的数据在 buffer 中按 alignment 排列
每次 draw 传 dynamic offset 选择其中一段
```

`pushconstants.cpp` 的模式是：

```text
每次 draw 直接把一小块数据写入 command buffer 状态
shader 直接读取这块 push constant 数据
```

二者对比：

```text
Push constants:
    不需要 buffer
    不需要 descriptor
    数据量很小
    每次 push 都记录到 command buffer
    适合几十字节级别的 per-draw 参数

Dynamic UBO:
    需要 buffer 和 descriptor
    数据量可以比 push constants 更大
    需要处理 minUniformBufferOffsetAlignment
    适合很多对象的结构化 uniform 数据
```

如果每个对象只有颜色、ID、少量开关，push constants 很合适。

如果每个对象有 model matrix、normal matrix、材质参数等一组较大的数据，dynamic UBO 或 storage buffer 更合适。

## 16. Push Constants 和 Specialization Constants 的区别

名字相近，但用途完全不同。

Push constants 是 draw-time 或 command-buffer-time 数据：

```text
vkCmdPushConstants
draw
vkCmdPushConstants
draw
```

它可以在 command buffer 中频繁变化。

Specialization constants 是 pipeline 创建时用于 shader 特化的数据。它们在 pipeline 创建后固定，不能每个 draw 改。

适用场景不同：

```text
Push constants:
    当前对象颜色
    当前 draw 参数
    object id

Specialization constants:
    是否启用某个 shader 分支
    光源数量上限
    kernel size
    编译期常量变体
```

不要因为名字里都有 constants 就把它们混在一起。

## 17. Push Constant Range 可以拆分

这个示例只有一个 range：

```cpp
offset = 0
size = sizeof(SpherePushConstantData)
stageFlags = VK_SHADER_STAGE_VERTEX_BIT
```

真实项目中可以有多个 range，或者同一个 push constant block 的不同部分给不同 shader stage 使用。

例如：

```text
offset 0, size 64:
    vertex shader 读取 model matrix

offset 64, size 16:
    fragment shader 读取 material color
```

C++ 侧可以分别 push：

```cpp
vkCmdPushConstants(..., VK_SHADER_STAGE_VERTEX_BIT, 0, 64, &model);
vkCmdPushConstants(..., VK_SHADER_STAGE_FRAGMENT_BIT, 64, 16, &color);
```

这样可以减少不必要的数据写入，也能清楚表达不同 stage 的访问范围。

不过要注意，pipeline layout 中声明的 ranges 必须覆盖这些 offset、size 和 stageFlags。

## 18. Pipeline Layout 兼容性

Push constants 和 pipeline layout 绑定得很紧。

`vkCmdPushConstants()` 需要传入：

```cpp
pipelineLayout
```

这不是随便传的。它必须和后续绑定的 pipeline layout 兼容，并且包含对应的 push constant range。

如果你在一个 command buffer 中切换 pipeline，需要关注：

- 新 pipeline layout 是否包含同样的 push constant range。
- 已经 push 的数据是否仍然对新 pipeline 有效。
- stage flags 和 offset/size 是否匹配。

在这个示例里只有一条 pipeline，所以不存在复杂兼容问题。

真实 renderer 中，如果多个 pipeline 都使用相同的 push constant layout，建议统一设计 pipeline layout，避免频繁遇到兼容性问题。

## 19. Push Constants 的大小限制

源码注释提醒：

```cpp
// Note that the spec only requires a minimum of 128 bytes,
// so for passing larger blocks of data you'd use UBOs or SSBOs
```

Vulkan 设备限制中有：

```cpp
maxPushConstantsSize
```

规范保证至少 128 bytes，但实际设备可能是 128、256 或更大。

这个示例只使用：

```text
2 * vec4 = 32 bytes
```

非常安全。

但如果你想把完整对象数据都塞进 push constants，例如：

```text
mat4 model        64 bytes
mat4 normal       64 bytes
vec4 color        16 bytes
vec4 material     16 bytes
```

总共已经 160 bytes，可能超过某些设备的最低保证。

工程上应该遵循：

```text
Push constants 放最小、最热、最频繁变化的数据。
大块数据放 UBO / SSBO。
```

## 20. Push Constants 的性能理解

Push constants 通常被认为是非常轻量的 per-draw 数据传递方式，因为：

- 不需要分配 buffer。
- 不需要更新 descriptor set。
- 不需要 dynamic offset 对齐管理。
- 数据直接作为 command buffer 状态记录。

但这不意味着可以无节制使用。

每次 `vkCmdPushConstants()` 仍然是一次 command buffer 记录操作。数据也会进入命令流。大量 draw call、每次 push 较大数据时，CPU 录制成本和命令缓冲大小都可能增加。

因此它最适合：

```text
小数据
高频变化
和 draw call 强绑定
```

不适合：

```text
大量数组数据
纹理资源
大型结构体
跨许多 draw 共享的大块参数
```

## 21. 和真实项目的常见用法

在真实 Vulkan renderer 中，push constants 常见用法包括：

```text
object id:
    shader 用它去 SSBO 或 bindless table 中索引对象数据。

material id:
    shader 用它索引材质 buffer。

draw flags:
    控制某些小分支，例如是否启用 skinning、是否选中高亮。

transform index:
    间接访问对象矩阵数组。

small transform:
    一个 mat4 或 mat3x4，前提是大小限制允许。

debug color:
    editor/debug view 中临时覆盖颜色。
```

一个现代 renderer 可能会这样组织：

```text
set 0: frame data
    camera
    lights

set 1: bindless resources
    textures
    buffers

push constants:
    objectIndex
    materialIndex
    drawFlags
```

shader 根据 push constants 中的 index 去大 buffer 里读取真正的对象和材质数据。

这种模式比把完整对象数据都 push 进去更可扩展。

## 22. 为什么示例把位置和颜色放 Push Constants

这个示例中，位置和颜色都是 per-object 数据：

```cpp
glm::vec4 color;
glm::vec4 position;
```

它们符合 push constants 的理想条件：

- 数据量小，只有 32 bytes。
- 每个 draw 都不同。
- 不需要被多个 shader stage 大范围共享。
- 不需要作为数组随机访问。
- 不需要额外资源生命周期管理。

如果不用 push constants，可以有几种替代方案。

方案一：每个 sphere 一个 uniform buffer 和 descriptor set。

这会让 16 个对象产生更多 descriptor 管理代码，对于 32 bytes 数据太重。

方案二：一个 dynamic uniform buffer。

这也可行，但需要处理 `minUniformBufferOffsetAlignment`。如果 alignment 是 256，每个 sphere 只有 32 bytes 数据，却要占 256 bytes 槽位。

方案三：storage buffer + object index。

也可行，但对这个简单示例来说过于复杂。

所以 push constants 是最合适的教学选择。

## 23. 代码中的数据流

完整数据流可以概括成：

```text
CPU:
    setupSpheres()
        生成 spheres[16]，每个元素包含 color 和 position

    updateUniformBuffers()
        更新 projection/model/view 到当前帧 UBO

Command buffer:
    bind descriptor set
        shader binding 0 -> 当前帧 UBO

    for each sphere:
        vkCmdPushConstants(...)
            push color + position
        model.draw(...)

Vertex shader:
    从 UBO 读取 projection/model/view
    从 push constants 读取 color/position
    计算 gl_Position 和 outColor

Fragment shader:
    输出 outColor
```

其中 UBO 是持久资源绑定，push constants 是每次 draw 的临时小参数。

## 24. Command Buffer 中的状态顺序

这个示例的命令顺序是：

```text
Begin command buffer
Begin render pass
Set viewport/scissor
Bind pipeline
Bind descriptor set

Push sphere 0 constants
Draw sphere

Push sphere 1 constants
Draw sphere

...

Draw UI
End render pass
End command buffer
```

如果把 push 和 draw 顺序写反，例如：

```cpp
model.draw(cmdBuffer);
vkCmdPushConstants(...);
```

那么这次 push 不会影响刚刚已经录制的 draw，只会影响后续 draw。

Vulkan command buffer 是严格顺序语义。push constants 是状态更新命令，draw 是使用当前状态的绘制命令。

## 25. 常见错误和调试方向

第一，pipeline layout 没有声明 push constant range。

shader 中使用了：

```glsl
layout(push_constant)
```

但 C++ 中没有设置：

```cpp
pipelineLayoutCreateInfo.pushConstantRangeCount
pipelineLayoutCreateInfo.pPushConstantRanges
```

这会导致 pipeline layout 与 shader 接口不匹配。

第二，stage flags 不匹配。

如果 shader 在 vertex stage 读取 push constants，range 中必须包含：

```cpp
VK_SHADER_STAGE_VERTEX_BIT
```

`vkCmdPushConstants()` 调用中的 stage flags 也要与 layout range 兼容。

第三，C++ 结构体布局和 GLSL 不一致。

示例用两个 `vec4`，对应两个 `glm::vec4`，比较安全。

如果使用 `vec3`、`mat3`、混合标量等，就要非常小心对齐和布局。工程上常用 `vec4` 对齐字段，避免跨语言布局问题。

第四，push constants 超过设备限制。

需要检查：

```cpp
VkPhysicalDeviceLimits::maxPushConstantsSize
```

不要假设所有设备都支持超过 128 bytes。

第五，以为 push constants 是全局状态。

Push constants 是 command buffer 记录的状态，不是某种全局变量。它影响的是命令缓冲中后续使用兼容 pipeline layout 的 draw/dispatch。

第六，在多个 pipeline layout 之间切换时忽略兼容性。

如果 pipeline layout 的 push constant range 不兼容，之前 push 的数据未必能按预期用于新 pipeline。

## 26. 这个示例可以如何扩展

第一，把 fragment shader 的 alpha 显式写为 1.0：

```glsl
outFragColor = vec4(inColor, 1.0);
```

虽然当前示例没有 blending，alpha 不影响最终画面，但完整写出更稳妥。

第二，让 fragment shader 也读取 push constants。

例如把颜色留给 fragment stage 使用，range 的 stageFlags 可以改成：

```cpp
VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT
```

或者拆成两个 range。

第三，用 push constants 传 object index。

把完整 color/position 放到 storage buffer 中，push constants 只传：

```cpp
uint objectIndex;
```

这更接近现代大场景 renderer 的做法。

第四，比较 push constants、dynamic UBO、SSBO 三种方案。

同样绘制 16、100、1000 个对象，观察 CPU command recording、descriptor 管理和 GPU 访问模式的差异，会更容易理解每种方式的边界。

## 27. 总结

`pushconstants.cpp` 是理解 Vulkan push constants 的经典示例。

它展示了：

- 如何在 C++ 中定义一小块 per-object 数据。
- 如何在 shader 中用 `layout(push_constant)` 声明 push constant block。
- 如何在 pipeline layout 中声明 `VkPushConstantRange`。
- 如何通过 `vkCmdPushConstants()` 在每个 draw 前推送当前对象参数。
- 如何让 UBO 和 push constants 分工：UBO 放相机矩阵，push constants 放球体颜色和位置。
- 为什么 push constants 适合小数据、高频更新、per-draw 参数。

它的核心思想可以浓缩成一句话：

```text
Descriptor set 绑定长期资源，push constants 传递当前 draw 的小参数。
```

再放到 Vulkan renderer 的整体模型里：

```text
pipeline 决定怎么画
descriptor set 决定绑定哪些资源
push constants 决定这次 draw 的少量即时参数
```

理解这个示例之后，就能更清楚地判断：哪些数据应该放 UBO，哪些数据应该放 dynamic UBO 或 SSBO，哪些数据只需要用 push constants 轻量传过去。

