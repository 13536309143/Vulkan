# Vulkan 示例解析：triangle_light.cpp 如何用更少代码画出三角形

本文分析的是 `examples/triangle_light/triangle_light.cpp`。它是 `triangle.cpp` 的精简版，目标不是重新展示所有 Vulkan 底层细节，而是尽量复用 Sascha Willems 示例框架中已经存在的能力，把代码压缩到“画一个 indexed triangle 真正需要关心的部分”。

原版 `triangle.cpp` 更像是一篇 Vulkan 底层流程展开：它手写 command pool、command buffer、fence、semaphore、render pass、framebuffer、shader module 加载、staging buffer 上传和 present 流程。

`triangle_light.cpp` 则换了一个角度：如果我们已经在这个 sample 框架里，哪些东西可以直接交给 `VulkanExampleBase`？最后会发现，真正需要自己写的只有：

- 三角形的顶点和索引数据。
- uniform buffer 和 descriptor。
- graphics pipeline。
- 每帧 command buffer 里具体 draw 什么。

窗口、swapchain、默认 render pass、framebuffer、同步对象、acquire、submit、present 都复用了基类。

## 1. 这个 light 版精简了什么

先看文件开头的注释：

```cpp
/*
* Vulkan Example - Minimal indexed triangle using the sample framework helpers
*
* This keeps the original triangle shader and lets VulkanExampleBase handle the
* window, swapchain, default render pass, framebuffers and frame submission.
*/
```

这句话基本概括了设计目标：

1. 继续使用原 `triangle` 示例的 shader。
2. 让 `VulkanExampleBase` 负责窗口、swapchain、render pass、framebuffer 和 frame submission。
3. 示例本身只保留三角形相关资源。

因此，light 版没有再手写：

- `vkAcquireNextImageKHR`
- `vkQueueSubmit`
- `vkQueuePresentKHR`
- 自己的 command pool
- 自己的 per-frame command buffer 数组
- 自己的 present/render semaphore 数组
- 自己的 render pass 创建
- 自己的 framebuffer 创建
- 自己的 SPIR-V 文件读取函数
- staging buffer 传输流程

这些在原版里都是很好的学习材料，但如果目标是快速理解“一个示例项目里如何挂上自己的绘制逻辑”，它们会显得太重。

## 2. 类结构：仍然继承 VulkanExampleBase

light 版依然定义一个示例类：

```cpp
class VulkanExample : public VulkanExampleBase
```

这意味着它可以直接使用基类里的对象和函数，比如：

- `device`
- `vulkanDevice`
- `queue`
- `renderPass`
- `frameBuffers`
- `drawCmdBuffers`
- `currentBuffer`
- `currentImageIndex`
- `descriptorPool`
- `pipelineCache`
- `prepareFrame()`
- `submitFrame()`
- `loadShader()`

这就是代码能大幅缩短的主要原因。我们不再从零管理一整套 Vulkan 应用生命周期，而是在框架已经准备好的 frame 流程中插入自己的 draw call。

## 3. 顶点格式：从 float 数组换成 glm::vec3

light 版的顶点结构是：

```cpp
struct Vertex {
    glm::vec3 position;
    glm::vec3 color;
};
```

和原版的语义一样，仍然是一个位置加一个颜色。区别只是写法更贴近 GLM 和 shader：

- `position` 对应 vertex shader 里的 `layout(location = 0) in vec3 inPos`
- `color` 对应 vertex shader 里的 `layout(location = 1) in vec3 inColor`

shader 沿用原版：

```cpp
loadShader(getShadersPath() + "triangle/triangle.vert.spv", VK_SHADER_STAGE_VERTEX_BIT)
loadShader(getShadersPath() + "triangle/triangle.frag.spv", VK_SHADER_STAGE_FRAGMENT_BIT)
```

所以顶点输入布局必须和原 `triangle` shader 保持一致。

## 4. Uniform Buffer：保持和原 shader 一致

light 版定义了：

```cpp
struct UBO {
    glm::mat4 projectionMatrix;
    glm::mat4 modelMatrix;
    glm::mat4 viewMatrix;
};
```

这个结构必须和 shader 中的 uniform block 顺序一致：

```glsl
layout (binding = 0) uniform UBO
{
    mat4 projectionMatrix;
    mat4 modelMatrix;
    mat4 viewMatrix;
} ubo;
```

注意这里的顺序是：

1. `projectionMatrix`
2. `modelMatrix`
3. `viewMatrix`

而 vertex shader 实际使用时是：

```glsl
gl_Position = ubo.projectionMatrix * ubo.viewMatrix * ubo.modelMatrix * vec4(inPos.xyz, 1.0);
```

C++ 结构体字段顺序要匹配 shader 的 uniform block 内存布局，使用时的乘法顺序是 shader 自己决定的。

## 5. 成员变量：只保留三角形自己的资源

light 版成员变量很少：

```cpp
vks::Buffer vertexBuffer;
vks::Buffer indexBuffer;
std::array<vks::Buffer, maxConcurrentFrames> uniformBuffers;
std::array<VkDescriptorSet, maxConcurrentFrames> descriptorSets{};

VkDescriptorSetLayout descriptorSetLayout{ VK_NULL_HANDLE };
VkPipelineLayout pipelineLayout{ VK_NULL_HANDLE };
VkPipeline pipeline{ VK_NULL_HANDLE };
uint32_t indexCount{ 0 };
```

这里有一个重要变化：buffer 不再手写 `VkBuffer + VkDeviceMemory`，而是使用框架提供的 `vks::Buffer`。

`vks::Buffer` 封装了：

- `VkBuffer`
- `VkDeviceMemory`
- `VkDescriptorBufferInfo descriptor`
- `map()`
- `destroy()`
- `setupDescriptor()`

因此创建 uniform descriptor 时，可以直接使用：

```cpp
&uniformBuffers[i].descriptor
```

而不需要像原版那样手写 `VkDescriptorBufferInfo`。

## 6. 构造函数：只做示例配置和相机设置

```cpp
VulkanExample()
{
    title = "Lightweight indexed triangle";
    settings.overlay = false;
    camera.type = Camera::CameraType::lookat;
    camera.setPosition(glm::vec3(0.0f, 0.0f, -2.5f));
    camera.setPerspective(60.0f, static_cast<float>(width) / static_cast<float>(height), 1.0f, 256.0f);
}
```

这里做了几件事：

- 设置窗口标题。
- 关闭 UI overlay，让示例更纯粹。
- 使用 look-at 相机。
- 把相机放在 `z = -2.5`。
- 设置透视投影。

因为三角形顶点在 `z = 0` 平面上，相机位于负 z 方向，所以它能看到这个三角形。

## 7. prepare：整个初始化流程被压缩到四步

light 版的 `prepare()` 很短：

```cpp
void prepare() override
{
    VulkanExampleBase::prepare();
    prepareBuffers();
    setupDescriptors();
    preparePipeline();
    prepared = true;
}
```

第一步 `VulkanExampleBase::prepare()` 很关键。它会准备框架层面的 Vulkan 渲染资源，例如：

- surface
- swapchain
- command pool
- draw command buffers
- 同步对象
- depth stencil
- 默认 render pass
- framebuffer
- pipeline cache

所以本示例后面只需要补上三角形自己的资源：

- `prepareBuffers()`：创建 vertex/index/uniform buffer。
- `setupDescriptors()`：创建 descriptor pool、descriptor set layout、descriptor set。
- `preparePipeline()`：创建 graphics pipeline。

这就是 light 版和原版最大的结构差异。

## 8. prepareBuffers：直接创建 host-visible buffer

三角形数据：

```cpp
std::array<Vertex, 3> vertices = {
    Vertex{ glm::vec3( 1.0f,  1.0f, 0.0f), glm::vec3(1.0f, 0.0f, 0.0f) },
    Vertex{ glm::vec3(-1.0f,  1.0f, 0.0f), glm::vec3(0.0f, 1.0f, 0.0f) },
    Vertex{ glm::vec3( 0.0f, -1.0f, 0.0f), glm::vec3(0.0f, 0.0f, 1.0f) }
};
std::array<uint32_t, 3> indices = { 0, 1, 2 };
```

这和原版三角形完全一致：三个顶点，三个索引，一个三角形。

然后创建 vertex buffer：

```cpp
vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_VERTEX_BUFFER_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
    &vertexBuffer,
    sizeof(Vertex) * vertices.size(),
    vertices.data());
```

创建 index buffer：

```cpp
vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_INDEX_BUFFER_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
    &indexBuffer,
    sizeof(uint32_t) * indices.size(),
    indices.data());
```

这里故意没有使用 staging buffer。原因是 light 版追求代码最小化，直接创建 CPU 可见、host coherent 的 buffer，并把数据写进去。

这有一个明确的取舍：

- 优点：代码短，概念少，适合理解绘制主链路。
- 缺点：顶点和索引数据没有放到 device local memory，性能上不如原版的 staging 上传方式。

对于三个顶点的教学示例，这个取舍是合理的。

## 9. 每帧一个 Uniform Buffer

uniform buffer 创建代码：

```cpp
for (auto& buffer : uniformBuffers) {
    vulkanDevice->createBuffer(
        VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
        VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
        &buffer,
        sizeof(UBO),
        nullptr);
    buffer.map();
}
```

`uniformBuffers` 的大小是：

```cpp
std::array<vks::Buffer, maxConcurrentFrames> uniformBuffers;
```

`maxConcurrentFrames` 来自基类，通常是 2。

这么做的目的和原版一样：每个 in-flight frame 使用自己的 uniform buffer，避免 CPU 更新当前帧矩阵时覆盖 GPU 仍在读取的上一帧矩阵。

`buffer.map()` 只调用一次，之后每帧直接 `memcpy` 到 `mapped` 指针即可。

## 10. setupDescriptors：把 UBO 绑定给 vertex shader

descriptor pool：

```cpp
std::vector<VkDescriptorPoolSize> poolSizes = {
    vks::initializers::descriptorPoolSize(VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, maxConcurrentFrames)
};
VkDescriptorPoolCreateInfo poolInfo =
    vks::initializers::descriptorPoolCreateInfo(poolSizes, maxConcurrentFrames);
vkCreateDescriptorPool(device, &poolInfo, nullptr, &descriptorPool);
```

这里创建的 descriptor pool 只支持一种 descriptor：uniform buffer。

descriptor set layout：

```cpp
std::vector<VkDescriptorSetLayoutBinding> bindings = {
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        VK_SHADER_STAGE_VERTEX_BIT,
        0)
};
```

含义是：

- binding 0
- descriptor 类型是 uniform buffer
- 只在 vertex shader 使用

这正好对应 shader：

```glsl
layout (binding = 0) uniform UBO
```

然后为每个 frame 分配一个 descriptor set：

```cpp
for (uint32_t i = 0; i < maxConcurrentFrames; i++) {
    vkAllocateDescriptorSets(device, &allocInfo, &descriptorSets[i]);
    std::vector<VkWriteDescriptorSet> writes = {
        vks::initializers::writeDescriptorSet(
            descriptorSets[i],
            VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
            0,
            &uniformBuffers[i].descriptor)
    };
    vkUpdateDescriptorSets(device, static_cast<uint32_t>(writes.size()), writes.data(), 0, nullptr);
}
```

因为 `vks::Buffer` 已经内置了 `descriptor`，这里比原版少了不少手写结构体代码。

## 11. preparePipeline：使用 helper 创建 graphics pipeline

pipeline layout：

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutInfo =
    vks::initializers::pipelineLayoutCreateInfo(&descriptorSetLayout, 1);
vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout);
```

pipeline layout 的作用是告诉 pipeline：shader 可以访问哪套 descriptor set layout。

### 11.1 顶点输入布局

```cpp
std::vector<VkVertexInputBindingDescription> bindingDescriptions = {
    vks::initializers::vertexInputBindingDescription(0, sizeof(Vertex), VK_VERTEX_INPUT_RATE_VERTEX)
};
```

表示 binding 0 每次读取一个 `Vertex`。

两个 attribute：

```cpp
std::vector<VkVertexInputAttributeDescription> attributeDescriptions = {
    vks::initializers::vertexInputAttributeDescription(
        0, 0, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, position)),
    vks::initializers::vertexInputAttributeDescription(
        0, 1, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, color))
};
```

参数可以这样理解：

```text
binding, location, format, offset
```

也就是：

- binding 0, location 0：从 `position` 读取 vec3。
- binding 0, location 1：从 `color` 读取 vec3。

这和 shader 的输入完全对齐。

### 11.2 固定功能状态

light 版通过 `vks::initializers` 创建 pipeline 状态：

```cpp
pipelineInputAssemblyStateCreateInfo(VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST, ...)
pipelineRasterizationStateCreateInfo(VK_POLYGON_MODE_FILL, VK_CULL_MODE_NONE, ...)
pipelineColorBlendAttachmentState(0xf, VK_FALSE)
pipelineDepthStencilStateCreateInfo(VK_TRUE, VK_TRUE, VK_COMPARE_OP_LESS_OR_EQUAL)
pipelineViewportStateCreateInfo(1, 1, 0)
pipelineMultisampleStateCreateInfo(VK_SAMPLE_COUNT_1_BIT)
```

它们对应：

- 输入装配：三角形列表。
- 光栅化：填充模式，不剔除背面。
- 颜色写入：写 RGBA，不启用混合。
- 深度：启用深度测试和深度写入。
- viewport/scissor：各一个。
- MSAA：不启用多重采样。

这些设置和原版三角形基本一致，只是写法更短。

### 11.3 动态 viewport 和 scissor

```cpp
std::vector<VkDynamicState> dynamicStates = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR
};
```

viewport 和 scissor 不固定在 pipeline 里，而是在 command buffer 中动态设置。这样窗口大小变化时不需要因为尺寸变化重新创建 pipeline。

### 11.4 复用原 triangle shader

```cpp
std::array<VkPipelineShaderStageCreateInfo, 2> shaderStages = {
    loadShader(getShadersPath() + "triangle/triangle.vert.spv", VK_SHADER_STAGE_VERTEX_BIT),
    loadShader(getShadersPath() + "triangle/triangle.frag.spv", VK_SHADER_STAGE_FRAGMENT_BIT)
};
```

这里用的是基类 `loadShader()`，不是原版 `triangle.cpp` 自己写的 `loadSPIRVShader()`。

这又省掉了一段手动读取 `.spv` 文件和创建 `VkShaderModule` 的代码。shader module 生命周期也交给基类维护。

最后创建 pipeline：

```cpp
VkGraphicsPipelineCreateInfo pipelineInfo =
    vks::initializers::pipelineCreateInfo(pipelineLayout, renderPass);

pipelineInfo.pVertexInputState = &vertexInputState;
pipelineInfo.pInputAssemblyState = &inputAssemblyState;
pipelineInfo.pRasterizationState = &rasterizationState;
pipelineInfo.pColorBlendState = &colorBlendState;
pipelineInfo.pMultisampleState = &multisampleState;
pipelineInfo.pViewportState = &viewportState;
pipelineInfo.pDepthStencilState = &depthStencilState;
pipelineInfo.pDynamicState = &dynamicState;
pipelineInfo.stageCount = static_cast<uint32_t>(shaderStages.size());
pipelineInfo.pStages = shaderStages.data();

vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineInfo, nullptr, &pipeline);
```

注意 `renderPass` 来自基类。light 版没有自己创建 render pass，而是复用框架默认 render pass。

## 12. updateUniformBuffer：每帧更新矩阵

```cpp
void updateUniformBuffer()
{
    UBO ubo{};
    ubo.projectionMatrix = camera.matrices.perspective;
    ubo.modelMatrix = glm::mat4(1.0f);
    ubo.viewMatrix = camera.matrices.view;
    std::memcpy(uniformBuffers[currentBuffer].mapped, &ubo, sizeof(ubo));
}
```

`currentBuffer` 是基类管理的当前 in-flight frame 下标。

每帧更新的内容是：

- 当前相机投影矩阵。
- 单位 model 矩阵。
- 当前相机 view 矩阵。

由于 uniform buffer 使用 host coherent memory，写入后不需要手动 flush。

## 13. buildCommandBuffer：真正录制 draw call

light 版没有自己创建 command buffer，而是使用基类的：

```cpp
VkCommandBuffer cmdBuffer = drawCmdBuffers[currentBuffer];
```

这意味着 command buffer 的分配、复用和提交都由基类流程处理。本函数只负责“这一帧要往 command buffer 里录什么命令”。

### 13.1 开始 Render Pass

```cpp
std::array<VkClearValue, 2> clearValues{};
clearValues[0].color = defaultClearColor;
clearValues[1].depthStencil = { 1.0f, 0 };
```

`defaultClearColor` 来自基类。

```cpp
VkRenderPassBeginInfo renderPassBeginInfo =
    vks::initializers::renderPassBeginInfo();
renderPassBeginInfo.renderPass = renderPass;
renderPassBeginInfo.framebuffer = frameBuffers[currentImageIndex];
renderPassBeginInfo.renderArea.extent.width = width;
renderPassBeginInfo.renderArea.extent.height = height;
renderPassBeginInfo.clearValueCount = static_cast<uint32_t>(clearValues.size());
renderPassBeginInfo.pClearValues = clearValues.data();
```

这里的关键是：

- `renderPass` 来自基类。
- `frameBuffers[currentImageIndex]` 来自基类。
- `currentImageIndex` 是 `prepareFrame()` acquire swapchain image 后设置的。

所以本示例不需要自己处理 swapchain image 获取逻辑，只要使用当前 image 对应的 framebuffer。

### 13.2 设置 viewport 和 scissor

```cpp
VkViewport viewport =
    vks::initializers::viewport(static_cast<float>(width), static_cast<float>(height), 0.0f, 1.0f);
VkRect2D scissor =
    vks::initializers::rect2D(width, height, 0, 0);
vkCmdSetViewport(cmdBuffer, 0, 1, &viewport);
vkCmdSetScissor(cmdBuffer, 0, 1, &scissor);
```

覆盖整个窗口。

### 13.3 绑定资源并绘制

```cpp
VkDeviceSize offset = 0;
vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
vkCmdBindDescriptorSets(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout, 0, 1, &descriptorSets[currentBuffer], 0, nullptr);
vkCmdBindVertexBuffers(cmdBuffer, 0, 1, &vertexBuffer.buffer, &offset);
vkCmdBindIndexBuffer(cmdBuffer, indexBuffer.buffer, 0, VK_INDEX_TYPE_UINT32);
vkCmdDrawIndexed(cmdBuffer, indexCount, 1, 0, 0, 0);
```

这几行就是整个例子的核心：

1. 绑定 pipeline，确定 shader 和固定功能状态。
2. 绑定当前 frame 的 descriptor set，给 shader 提供矩阵。
3. 绑定 vertex buffer，提供顶点位置和颜色。
4. 绑定 index buffer，提供索引顺序。
5. 调用 `vkCmdDrawIndexed` 画出三角形。

因为 `indexCount` 是 3，所以只绘制一个三角形。

## 14. render：每帧流程交给基类串起来

```cpp
void render() override
{
    if (!prepared) {
        return;
    }
    VulkanExampleBase::prepareFrame();
    updateUniformBuffer();
    buildCommandBuffer();
    VulkanExampleBase::submitFrame();
}
```

这一段是 light 版最能体现框架复用价值的地方。

`prepareFrame()` 会做：

- 等待当前 frame 的 fence。
- 更新 UI overlay，如果启用的话。
- acquire 下一张 swapchain image。
- 设置 `currentImageIndex`。

`submitFrame()` 会做：

- 提交 `drawCmdBuffers[currentBuffer]`。
- 等待 image acquire semaphore。
- signal render complete semaphore。
- present 当前 swapchain image。
- 推进 `currentBuffer`。

所以 light 版只需要在中间插入两件事：

```cpp
updateUniformBuffer();
buildCommandBuffer();
```

也就是更新 shader 数据，然后录制本帧绘制命令。

## 15. 原版和 light 版的关键区别

可以把两者的责任边界概括成这样：

```text
原 triangle.cpp：
    手写完整 Vulkan 渲染闭环
    create resources -> acquire -> record -> submit -> present

triangle_light.cpp：
    使用 VulkanExampleBase 管理 frame 闭环
    只实现资源绑定和 draw command
```

更具体地说：

| 功能 | 原 triangle.cpp | triangle_light.cpp |
| --- | --- | --- |
| 窗口创建 | 基类 | 基类 |
| swapchain | 基类 | 基类 |
| render pass | 示例自己重写 | 基类默认 |
| framebuffer | 示例自己重写 | 基类默认 |
| command buffer | 示例自己创建 | 基类 `drawCmdBuffers` |
| fence/semaphore | 示例自己创建 | 基类管理 |
| acquire image | 示例自己调用 | `prepareFrame()` |
| submit queue | 示例自己调用 | `submitFrame()` |
| present | 示例自己调用 | `submitFrame()` |
| shader 加载 | 示例自己写 `loadSPIRVShader()` | 基类 `loadShader()` |
| vertex/index buffer | 手写 buffer 和 memory，使用 staging | `vks::Buffer`，直接 host-visible |
| descriptor | 手写结构较多 | 使用 `vks::initializers` 和 `vks::Buffer::descriptor` |

这也是为什么 light 版更适合作为“在已有 Vulkan 框架里添加一个简单 draw pass”的入门样例。

## 16. 一帧的执行链路

light 版每帧可以总结为：

```text
render()
  |
  v
VulkanExampleBase::prepareFrame()
  - 等 fence
  - acquire swapchain image
  - 得到 currentImageIndex
  |
  v
updateUniformBuffer()
  - 写 projection/model/view 到当前 frame 的 UBO
  |
  v
buildCommandBuffer()
  - begin command buffer
  - begin render pass
  - bind pipeline
  - bind descriptor set
  - bind vertex/index buffer
  - vkCmdDrawIndexed
  - end render pass
  - end command buffer
  |
  v
VulkanExampleBase::submitFrame()
  - vkQueueSubmit
  - vkQueuePresentKHR
  - currentBuffer 前进
```

最终，三角形仍然是这样生成的：

```text
Vertex buffer + Index buffer
        |
        v
Vertex shader: MVP 变换，并传出颜色
        |
        v
Rasterizer: 三角形变成片元，颜色插值
        |
        v
Fragment shader: 输出 vec4(inColor, 1.0)
        |
        v
当前 swapchain image
        |
        v
present 到窗口
```

## 17. 这个 light 版适合学习什么

如果原版 `triangle.cpp` 适合学习 Vulkan 底层 API 的完整流程，那么 `triangle_light.cpp` 更适合学习：

- 如何在 Sascha Willems 示例框架中新增一个示例。
- 如何复用 `VulkanExampleBase` 的 frame lifecycle。
- 如何使用 `vks::Buffer` 简化 buffer 创建和销毁。
- 如何使用 `vks::initializers` 减少 Vulkan 结构体样板代码。
- 如何复用已有 shader。
- 如何把重点集中在 pipeline、descriptor 和 draw call 上。

它不是原版的替代品，而是一个更轻的入口。先看 light 版，可以更快抓住“画三角形需要自己提供什么”；再看原版，可以补上 Vulkan 同步、staging buffer、render pass 和 present 的完整底层细节。

## 18. 后续可以怎么扩展

这个 light 版很适合作为小实验的起点：

1. 修改顶点坐标，观察裁剪空间和相机矩阵的影响。
2. 修改顶点颜色，理解 fragment shader 插值。
3. 增加更多顶点和索引，绘制矩形或多个三角形。
4. 把 host-visible vertex buffer 改回 staging + device local，比较代码复杂度和性能取舍。
5. 增加 push constants，把 model matrix 从 uniform buffer 中拆出来。
6. 新增自己的 shader 目录，让 `triangle_light` 不再复用 `triangle` shader。

理解这个文件之后，就能比较自然地从“框架里画一个三角形”过渡到“框架里画一个 mesh、一个带纹理的物体、一个完整场景”。
