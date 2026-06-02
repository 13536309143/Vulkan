# Vulkan 示例解析：triangle.cpp 如何把一个三角形渲染到窗口上

本文分析的是 `examples/triangle/triangle.cpp`。这是 Sascha Willems Vulkan 示例中的基础案例，标题是 `Basic indexed triangle`。它的目标很直接：不依赖过多封装，手动展示 Vulkan 从初始化、创建资源、录制命令到提交并显示画面的完整路径。

这份代码虽然只画了一个三角形，但它已经覆盖了 Vulkan 图形渲染中最关键的概念：

- swapchain：窗口系统可显示的图像队列。
- framebuffer：把 swapchain image 和 depth image 组合成一次 render pass 的目标。
- render pass：描述本次渲染会写哪些 attachment，以及它们的布局转换。
- graphics pipeline：把 shader、顶点布局、光栅化、深度测试、混合等状态固定下来。
- vertex/index buffer：提供三角形几何数据。
- uniform buffer：给 vertex shader 提供投影、视图、模型矩阵。
- command buffer：记录 Vulkan 绘制命令。
- semaphore/fence：协调 CPU、GPU、swapchain image acquire、render、present 的时序。

## 1. 程序入口：从窗口到 render loop

Windows 入口在文件末尾：

```cpp
int APIENTRY WinMain(...)
{
    vulkanExample = new VulkanExample();
    vulkanExample->initVulkan();
    vulkanExample->setupWindow(hInstance, WndProc);
    vulkanExample->prepare();
    vulkanExample->renderLoop();
    delete(vulkanExample);
    return 0;
}
```

这几步可以理解为：

1. `new VulkanExample()`：创建示例对象，设置标题、相机和一些示例配置。
2. `initVulkan()`：在基类中创建 Vulkan instance、选择 physical device、创建 logical device、队列等基础 Vulkan 对象。
3. `setupWindow(...)`：创建平台窗口。
4. `prepare()`：创建三角形渲染需要的所有资源。
5. `renderLoop()`：进入消息循环，每帧调用 `render()`。

基类 `VulkanExampleBase::renderLoop()` 会处理窗口消息，然后调用 `nextFrame()`；`nextFrame()` 内部调用虚函数 `render()`。由于 `triangle.cpp` 里的 `VulkanExample` 重写了 `render()`，所以每帧真正执行的是本示例自己的渲染逻辑。

## 2. VulkanExample 类维护了哪些资源

示例类定义在：

```cpp
class VulkanExample : public VulkanExampleBase
```

它继承了基类中的 Vulkan 基础对象，比如 `device`、`queue`、`swapChain`、`renderPass`、`frameBuffers`、`width/height` 等。同时它自己维护了三角形示例专用的资源。

### 2.1 顶点格式

```cpp
struct Vertex {
    float position[3];
    float color[3];
};
```

每个顶点包含两个属性：

- `position`：三维位置。
- `color`：顶点颜色。

后面创建 graphics pipeline 时，会把这个 C++ 结构映射到 shader 的输入：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inColor;
```

也就是说，`position` 对应 shader 的 `location = 0`，`color` 对应 shader 的 `location = 1`。

### 2.2 顶点缓冲和索引缓冲

```cpp
struct {
    VkDeviceMemory memory;
    VkBuffer buffer;
} vertices;

struct {
    VkDeviceMemory memory;
    VkBuffer buffer;
    uint32_t count;
} indices;
```

`vertices.buffer` 存三角形顶点数据，`indices.buffer` 存索引数据。后续绘制时通过：

```cpp
vkCmdBindVertexBuffers(...);
vkCmdBindIndexBuffer(...);
vkCmdDrawIndexed(...);
```

告诉 GPU 用这些数据组装三角形。

### 2.3 Uniform Buffer

```cpp
struct ShaderData {
    glm::mat4 projectionMatrix;
    glm::mat4 modelMatrix;
    glm::mat4 viewMatrix;
};
```

这个结构和 vertex shader 中的 uniform block 对应：

```glsl
layout (binding = 0) uniform UBO
{
    mat4 projectionMatrix;
    mat4 modelMatrix;
    mat4 viewMatrix;
} ubo;
```

它负责把相机矩阵传给 shader。vertex shader 会用这些矩阵把模型坐标变换到裁剪空间：

```glsl
gl_Position = ubo.projectionMatrix * ubo.viewMatrix * ubo.modelMatrix * vec4(inPos.xyz, 1.0);
```

示例为每个并发帧准备一个 uniform buffer：

```cpp
std::array<UniformBuffer, MAX_CONCURRENT_FRAMES> uniformBuffers;
```

这样 CPU 更新当前帧 uniform buffer 时，不会覆盖 GPU 仍在使用的上一帧数据。

### 2.4 Pipeline、Descriptor 和同步对象

示例还维护：

```cpp
VkPipelineLayout pipelineLayout;
VkPipeline pipeline;
VkDescriptorSetLayout descriptorSetLayout;
```

其中：

- `descriptorSetLayout` 描述 shader 需要哪些资源。这里就是 `binding = 0` 的 uniform buffer。
- `pipelineLayout` 把 descriptor set layout 放进 pipeline 接口。
- `pipeline` 是最终用于绘制三角形的 graphics pipeline。

同步对象包括：

```cpp
std::vector<VkSemaphore> presentCompleteSemaphores;
std::vector<VkSemaphore> renderCompleteSemaphores;
std::array<VkFence, MAX_CONCURRENT_FRAMES> waitFences;
```

它们分别用于：

- 等 swapchain image acquire 完成。
- 等渲染命令执行完成后再 present。
- 等某一帧的 command buffer 执行完成后，CPU 才复用它。

## 3. prepare：渲染前的资源准备

本示例的 `prepare()` 如下：

```cpp
void prepare() override
{
    VulkanExampleBase::prepare();
    createSynchronizationPrimitives();
    createCommandBuffers();
    createVertexBuffer();
    createUniformBuffers();
    createDescriptorSetLayout();
    createDescriptorPool();
    createDescriptorSets();
    createPipelines();
    prepared = true;
}
```

这里有一个容易忽略的点：`VulkanExampleBase::prepare()` 会创建 surface、swapchain、基类默认 command buffer、基类默认同步对象，并调用虚函数创建 depth stencil、render pass、framebuffer。

在本示例中：

- `setupDepthStencil()` 是重写的，所以基类 `prepare()` 调用的是本文件的版本。
- `setupRenderPass()` 是重写的，所以 render pass 也是本文件定义的。
- `setupFrameBuffer()` 是重写的，所以 framebuffer 也是本文件定义的。
- `createCommandBuffers()` 和 `createSynchronizationPrimitives()` 在基类中不是虚函数，所以本示例在自己的 `prepare()` 中又创建了一套自己使用的 command buffer 和同步对象。

这样做的原因是这个例子想尽量展示完整的 Vulkan acquire、record、submit、present 流程，而不是依赖基类封装好的绘制路径。

## 4. 创建三角形数据：createVertexBuffer

三角形的三个顶点定义在 `createVertexBuffer()` 中：

```cpp
std::vector<Vertex> vertexBuffer{
    { {  1.0f,  1.0f, 0.0f }, { 1.0f, 0.0f, 0.0f } },
    { { -1.0f,  1.0f, 0.0f }, { 0.0f, 1.0f, 0.0f } },
    { {  0.0f, -1.0f, 0.0f }, { 0.0f, 0.0f, 1.0f } }
};
```

这三个点大致形成一个倒三角：

- 右上角：红色。
- 左上角：绿色。
- 下方中间：蓝色。

索引数据是：

```cpp
std::vector<uint32_t> indexBuffer{ 0, 1, 2 };
indices.count = static_cast<uint32_t>(indexBuffer.size());
```

这表示按照顶点 `0 -> 1 -> 2` 画一个三角形。

### 4.1 为什么要用 staging buffer

代码没有直接把数据放进最终的 GPU buffer，而是用了 staging buffer：

1. 创建 CPU 可见的 staging buffer。
2. `vkMapMemory()` 把 staging buffer 映射到 CPU 地址空间。
3. `memcpy()` 把顶点和索引数据拷进去。
4. 创建 GPU device local buffer。
5. 用 `vkCmdCopyBuffer()` 从 staging buffer 拷到 GPU buffer。
6. 等 copy 命令完成。
7. 销毁 staging buffer。

这么做是为了让最终绘制用的 buffer 位于 `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT` 内存中。它通常对 GPU 访问更快，但 CPU 不能直接写入，所以先借助 staging buffer 中转。

## 5. Render Pass 和 Framebuffer

本示例重写了 `setupRenderPass()`。它定义了两个 attachment：

```cpp
std::array<VkAttachmentDescription, 2> attachments{};
```

第一个 attachment 是 color attachment：

```cpp
attachments[0].format = swapChain.colorFormat;
attachments[0].loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
attachments[0].storeOp = VK_ATTACHMENT_STORE_OP_STORE;
attachments[0].initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
attachments[0].finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
```

关键点：

- `loadOp = CLEAR`：render pass 开始时清屏。
- `storeOp = STORE`：render pass 结束后保留颜色结果，因为要显示到窗口。
- `finalLayout = PRESENT_SRC_KHR`：最终转成可 present 的布局。

第二个 attachment 是 depth attachment：

```cpp
attachments[1].format = depthFormat;
attachments[1].loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
attachments[1].storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
attachments[1].finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```

深度图只在本次渲染中使用，渲染结束后不需要显示，所以 `storeOp` 是 `DONT_CARE`。

`setupFrameBuffer()` 为 swapchain 的每张 image 创建一个 framebuffer：

```cpp
attachments[0] = swapChain.imageViews[i];
attachments[1] = depthStencil.view;
```

也就是说，每个 framebuffer 包含：

- 当前 swapchain image 的 image view，作为颜色输出。
- 共用的 depth stencil image view，作为深度输出。

当某一帧 acquire 到 `imageIndex` 后，渲染时就使用：

```cpp
renderPassBeginInfo.framebuffer = frameBuffers[imageIndex];
```

这样 GPU 写入的就是当前要显示的 swapchain image。

## 6. Descriptor：把 Uniform Buffer 连接到 Shader

shader 中有：

```glsl
layout (binding = 0) uniform UBO { ... } ubo;
```

所以 C++ 侧需要创建 descriptor set layout：

```cpp
layoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
layoutBinding.descriptorCount = 1;
layoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
```

这说明：

- binding 0 是一个 uniform buffer。
- 这个 uniform buffer 被 vertex shader 使用。

然后 `createDescriptorSets()` 为每个并发帧分配一个 descriptor set，并把对应帧的 uniform buffer 写进去：

```cpp
bufferInfo.buffer = uniformBuffers[i].buffer;
bufferInfo.range = sizeof(ShaderData);

writeDescriptorSet.dstSet = uniformBuffers[i].descriptorSet;
writeDescriptorSet.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
writeDescriptorSet.pBufferInfo = &bufferInfo;
writeDescriptorSet.dstBinding = 0;

vkUpdateDescriptorSets(device, 1, &writeDescriptorSet, 0, nullptr);
```

每帧绘制时，代码绑定当前帧的 descriptor set：

```cpp
vkCmdBindDescriptorSets(
    commandBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    0,
    1,
    &uniformBuffers[currentFrame].descriptorSet,
    0,
    nullptr);
```

这样 shader 使用的就是当前帧对应的矩阵数据。

## 7. Graphics Pipeline：固定一次绘制所需的大部分状态

Vulkan 的 pipeline 可以理解为一次绘制的完整状态包。`createPipelines()` 里创建了本示例唯一的 graphics pipeline。

### 7.1 Pipeline Layout

```cpp
pipelineLayoutCI.setLayoutCount = 1;
pipelineLayoutCI.pSetLayouts = &descriptorSetLayout;
vkCreatePipelineLayout(device, &pipelineLayoutCI, nullptr, &pipelineLayout);
```

pipeline layout 定义了 shader 能访问哪些 descriptor set。这里就是前面创建的 uniform buffer layout。

### 7.2 Input Assembly

```cpp
inputAssemblyStateCI.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
```

这表示 GPU 按三角形列表解释输入顶点。配合索引 `{0, 1, 2}`，最终只生成一个三角形。

### 7.3 Rasterization

```cpp
rasterizationStateCI.polygonMode = VK_POLYGON_MODE_FILL;
rasterizationStateCI.cullMode = VK_CULL_MODE_NONE;
rasterizationStateCI.frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE;
```

这里的设置表示：

- 用填充模式画面，而不是线框。
- 不做背面剔除。
- 逆时针为正面，但由于不剔除，所以这个三角形无论正反都能画出来。

### 7.4 Depth 和 Blend

```cpp
depthStencilStateCI.depthTestEnable = VK_TRUE;
depthStencilStateCI.depthWriteEnable = VK_TRUE;
depthStencilStateCI.depthCompareOp = VK_COMPARE_OP_LESS_OR_EQUAL;
```

虽然只有一个三角形，但示例仍启用了深度测试和深度写入。这是为了展示完整的常规 3D pipeline 配置。

颜色混合关闭：

```cpp
blendAttachmentState.blendEnable = VK_FALSE;
```

fragment shader 输出什么颜色，color attachment 就写什么颜色。

### 7.5 Viewport 和 Scissor 是动态状态

代码把 viewport 和 scissor 加入 dynamic state：

```cpp
dynamicStateEnables.push_back(VK_DYNAMIC_STATE_VIEWPORT);
dynamicStateEnables.push_back(VK_DYNAMIC_STATE_SCISSOR);
```

这表示 pipeline 创建时不用固定窗口大小。每帧录制 command buffer 时再设置：

```cpp
vkCmdSetViewport(commandBuffer, 0, 1, &viewport);
vkCmdSetScissor(commandBuffer, 0, 1, &scissor);
```

窗口大小变化时更方便。

### 7.6 顶点输入布局

```cpp
VkVertexInputBindingDescription vertexInputBinding{};
vertexInputBinding.binding = 0;
vertexInputBinding.stride = sizeof(Vertex);
vertexInputBinding.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
```

这说明 binding 0 是按顶点读取，每个顶点大小是 `sizeof(Vertex)`。

两个 attribute：

```cpp
vertexInputAttributs[0].location = 0;
vertexInputAttributs[0].format = VK_FORMAT_R32G32B32_SFLOAT;
vertexInputAttributs[0].offset = offsetof(Vertex, position);

vertexInputAttributs[1].location = 1;
vertexInputAttributs[1].format = VK_FORMAT_R32G32B32_SFLOAT;
vertexInputAttributs[1].offset = offsetof(Vertex, color);
```

这把 C++ 里的 `Vertex` 布局和 shader 输入绑定起来：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inColor;
```

### 7.7 Shader Stage

```cpp
shaderStages[0].stage = VK_SHADER_STAGE_VERTEX_BIT;
shaderStages[0].module = loadSPIRVShader(getShadersPath() + "triangle/triangle.vert.spv");

shaderStages[1].stage = VK_SHADER_STAGE_FRAGMENT_BIT;
shaderStages[1].module = loadSPIRVShader(getShadersPath() + "triangle/triangle.frag.spv");
```

Vulkan pipeline 使用的是 SPIR-V 二进制 shader。`loadSPIRVShader()` 会读取 `.spv` 文件，然后调用 `vkCreateShaderModule()` 创建 shader module。

最后：

```cpp
vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCI, nullptr, &pipeline);
```

到这里，一个能画三角形的 graphics pipeline 就创建好了。

## 8. Shader：顶点如何变成颜色

顶点 shader：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inColor;

layout (binding = 0) uniform UBO
{
    mat4 projectionMatrix;
    mat4 modelMatrix;
    mat4 viewMatrix;
} ubo;

layout (location = 0) out vec3 outColor;

void main()
{
    outColor = inColor;
    gl_Position = ubo.projectionMatrix * ubo.viewMatrix * ubo.modelMatrix * vec4(inPos.xyz, 1.0);
}
```

它做了两件事：

1. 把顶点颜色传给 fragment shader。
2. 把顶点位置乘以 `projection * view * model`，输出到 `gl_Position`。

fragment shader：

```glsl
layout (location = 0) in vec3 inColor;
layout (location = 0) out vec4 outFragColor;

void main()
{
    outFragColor = vec4(inColor, 1.0);
}
```

fragment shader 直接输出插值后的顶点颜色。由于三个顶点分别是红、绿、蓝，三角形内部会出现平滑渐变。

## 9. 每帧渲染：render 函数的完整流程

`render()` 是这份代码最核心的函数。它每帧完成：

1. 等上一轮使用的 command buffer 执行完。
2. 从 swapchain 获取下一张 image。
3. 更新当前帧 uniform buffer。
4. 录制 command buffer。
5. 提交 command buffer 到 graphics queue。
6. 把渲染完成的 image present 到窗口。

### 9.1 等待当前帧资源可复用

```cpp
vkWaitForFences(device, 1, &waitFences[currentFrame], VK_TRUE, UINT64_MAX);
vkResetFences(device, 1, &waitFences[currentFrame]);
```

`currentFrame` 在 `0` 和 `1` 之间循环，因为：

```cpp
constexpr auto MAX_CONCURRENT_FRAMES = 2;
```

这允许 CPU 最多提前准备两个 frame。复用某个 frame 的 command buffer 和 uniform buffer 前，需要确认 GPU 已经不用它了。

### 9.2 获取 swapchain image

```cpp
uint32_t imageIndex;
VkResult result = vkAcquireNextImageKHR(
    device,
    swapChain.swapChain,
    UINT64_MAX,
    presentCompleteSemaphores[currentFrame],
    VK_NULL_HANDLE,
    &imageIndex);
```

`imageIndex` 是本帧要写入的 swapchain image 下标。

`presentCompleteSemaphores[currentFrame]` 会在 image 可用时 signal。后面提交绘制命令时会等待这个 semaphore，确保 GPU 不会在 image 还不可用时就开始写它。

### 9.3 更新矩阵 Uniform Buffer

```cpp
ShaderData shaderData{};
shaderData.projectionMatrix = camera.matrices.perspective;
shaderData.viewMatrix = camera.matrices.view;
shaderData.modelMatrix = glm::mat4(1.0f);

memcpy(uniformBuffers[currentFrame].mapped, &shaderData, sizeof(ShaderData));
```

这里每帧把相机矩阵写入当前帧的 uniform buffer。因为创建 uniform buffer 时选择了：

```cpp
VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT
```

所以 CPU 写入后对 GPU 可见，不需要额外 flush。

### 9.4 开始录制 command buffer

```cpp
vkResetCommandBuffer(commandBuffers[currentFrame], 0);
vkBeginCommandBuffer(commandBuffer, &cmdBufInfo);
```

Vulkan 不是立即执行 `draw`，而是先把命令录进 command buffer。录完后再提交给 queue。

### 9.5 开始 Render Pass

```cpp
VkClearValue clearValues[2]{};
clearValues[0].color = { { 0.0f, 0.0f, 0.2f, 1.0f } };
clearValues[1].depthStencil = { 1.0f, 0 };
```

颜色 attachment 会被清成深蓝色，深度 attachment 会被清成 `1.0`。

```cpp
renderPassBeginInfo.renderPass = renderPass;
renderPassBeginInfo.framebuffer = frameBuffers[imageIndex];
renderPassBeginInfo.renderArea.extent.width = width;
renderPassBeginInfo.renderArea.extent.height = height;
renderPassBeginInfo.pClearValues = clearValues;

vkCmdBeginRenderPass(commandBuffer, &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
```

这里的 `frameBuffers[imageIndex]` 很关键：它让本帧渲染命令写入刚刚从 swapchain acquire 到的 image。

### 9.6 设置 viewport 和 scissor

```cpp
VkViewport viewport{};
viewport.width = (float)width;
viewport.height = (float)height;
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
vkCmdSetViewport(commandBuffer, 0, 1, &viewport);

VkRect2D scissor{};
scissor.extent.width = width;
scissor.extent.height = height;
vkCmdSetScissor(commandBuffer, 0, 1, &scissor);
```

这决定了渲染结果映射到窗口的哪个区域。本例覆盖整个窗口。

### 9.7 绑定资源并发出 draw call

```cpp
vkCmdBindDescriptorSets(...);
vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
vkCmdBindVertexBuffers(commandBuffer, 0, 1, &vertices.buffer, offsets);
vkCmdBindIndexBuffer(commandBuffer, indices.buffer, 0, VK_INDEX_TYPE_UINT32);
vkCmdDrawIndexed(commandBuffer, indices.count, 1, 0, 0, 0);
```

这一段就是三角形真正被绘制出来的地方。

绑定顺序可以理解为：

1. `vkCmdBindDescriptorSets`：告诉 shader 用哪个 uniform buffer。
2. `vkCmdBindPipeline`：告诉 GPU 使用哪套图形管线和 shader。
3. `vkCmdBindVertexBuffers`：告诉 GPU 从哪里读顶点位置和颜色。
4. `vkCmdBindIndexBuffer`：告诉 GPU 从哪里读索引。
5. `vkCmdDrawIndexed`：真正发出绘制命令。

因为 `indices.count == 3`，所以这次 draw call 只画一个三角形。

### 9.8 结束 Render Pass 和 Command Buffer

```cpp
vkCmdEndRenderPass(commandBuffer);
vkEndCommandBuffer(commandBuffer);
```

render pass 结束时，color attachment 会根据 render pass 中的配置转换到：

```cpp
VK_IMAGE_LAYOUT_PRESENT_SRC_KHR
```

也就是可用于 present 的布局。

### 9.9 提交到 Graphics Queue

```cpp
VkPipelineStageFlags waitStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;

submitInfo.pWaitSemaphores = &presentCompleteSemaphores[currentFrame];
submitInfo.waitSemaphoreCount = 1;
submitInfo.pSignalSemaphores = &renderCompleteSemaphores[imageIndex];
submitInfo.signalSemaphoreCount = 1;
submitInfo.pCommandBuffers = &commandBuffer;

vkQueueSubmit(queue, 1, &submitInfo, waitFences[currentFrame]);
```

这里有两个重要同步点：

- 等 `presentCompleteSemaphores[currentFrame]`：确保 swapchain image 已经可写。
- signal `renderCompleteSemaphores[imageIndex]`：告诉后面的 present，本帧渲染已经完成。

`waitFences[currentFrame]` 会在 GPU 执行完这次提交后 signal。下一次复用这个 `currentFrame` 时，CPU 会等待这个 fence。

### 9.10 Present 到窗口

```cpp
presentInfo.pWaitSemaphores = &renderCompleteSemaphores[imageIndex];
presentInfo.pSwapchains = &swapChain.swapChain;
presentInfo.pImageIndices = &imageIndex;

vkQueuePresentKHR(queue, &presentInfo);
```

present 阶段等待 `renderCompleteSemaphores[imageIndex]`，确保 image 已经渲染完成，然后把这张 swapchain image 交给窗口系统显示。

最后：

```cpp
currentFrame = (currentFrame + 1) % MAX_CONCURRENT_FRAMES;
```

切换到下一套 per-frame 资源。

## 10. 一帧渲染的数据流总结

可以把整个过程简化成下面这条链路：

```text
CPU 创建顶点/索引数据
        |
        v
staging buffer
        |
        v
GPU vertex buffer / index buffer
        |
        v
command buffer 绑定 pipeline + descriptor + vertex/index buffer
        |
        v
vkCmdDrawIndexed
        |
        v
vertex shader：位置乘 MVP，颜色传递
        |
        v
rasterizer：三角形变成片元，颜色插值
        |
        v
fragment shader：输出颜色
        |
        v
color attachment，也就是当前 swapchain image
        |
        v
vkQueuePresentKHR 显示到窗口
```

## 11. 从代码角度看，真正让三角形出现的最小闭环

如果只看“为什么窗口上出现了三角形”，最关键的是这几个条件同时成立：

1. `createVertexBuffer()` 创建了 3 个顶点和 3 个索引。
2. `createPipelines()` 设置了三角形列表拓扑、顶点输入布局和 vertex/fragment shader。
3. `createDescriptorSets()` 把 uniform buffer 绑定给 vertex shader。
4. `render()` acquire 到 swapchain image，并选择对应 framebuffer。
5. command buffer 中绑定 pipeline、descriptor、vertex buffer、index buffer。
6. `vkCmdDrawIndexed(..., indices.count, ...)` 发出绘制命令。
7. `vkQueueSubmit()` 执行命令。
8. `vkQueuePresentKHR()` 把渲染结果显示到窗口。

三角形本身来自顶点缓冲，颜色来自顶点颜色经过 shader 插值，窗口显示来自 swapchain present。

## 12. 这个例子适合学习什么

这份 `triangle.cpp` 的价值不在于图形复杂，而在于它把 Vulkan 的基础图形渲染路径摊开了：

- 如何把 CPU 数据上传到 GPU buffer。
- 如何描述 shader 需要的 uniform buffer。
- 如何配置 graphics pipeline。
- 如何定义 render pass 和 framebuffer。
- 如何每帧 acquire swapchain image。
- 如何录制 command buffer。
- 如何同步 acquire、render、present。

如果后续要扩展这个例子，可以按下面顺序继续学习：

1. 修改顶点数据，画更多三角形。
2. 修改 fragment shader，输出固定颜色、渐变或简单光照。
3. 添加纹理，学习 sampled image 和 sampler descriptor。
4. 添加多个 uniform 或 push constants。
5. 添加模型加载，把 vertex/index buffer 换成真实 mesh。

理解这个例子后，再看 Vulkan 中的模型渲染、纹理采样、深度测试、多 pipeline、多 descriptor set，会容易很多。
