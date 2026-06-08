# Vulkan 示例解析：offscreen.cpp 如何把离屏渲染结果作为纹理再采样

本文分析的是 `examples/offscreen/offscreen.cpp`。这个示例展示了 Vulkan 中非常常见的两阶段渲染模式：**先渲染到一个独立的离屏 framebuffer，再把离屏 color attachment 当作纹理，在主场景中采样使用。**

这个示例最终实现的是一个镜面反射效果：

- 第一遍 render pass：把翻转后的龙模型渲染到 512 x 512 的 offscreen framebuffer。
- 第二遍 render pass：渲染正常场景，并把第一遍得到的 color attachment 采样到地面平面上，形成镜面反射。
- UI 可以切换 debug 模式，直接把 offscreen render target 显示到屏幕上。

它的核心不是复杂光照，而是 Vulkan 资源和同步链路：

```text
offscreen color VkImage
    -> 作为 color attachment 写入
    -> render pass finalLayout 转成 SHADER_READ_ONLY_OPTIMAL
    -> VkDescriptorImageInfo
    -> sampler2D
    -> mirror.frag 中采样
```

这篇文章会按代码执行顺序拆解：离屏 framebuffer、color/depth attachment、独立 render pass、subpass dependency、descriptor 组织、多套 uniform buffer、四条 pipeline、两次 render pass 和 shader 采样逻辑。

## 1. Offscreen Rendering 解决什么问题

很多渲染效果不是直接把所有东西画到 swapchain image 上，而是先画到中间图像，再把中间图像作为输入继续处理。例如：

- 镜面反射。
- 后处理效果。
- 阴影贴图。
- 屏幕空间反射。
- 延迟渲染 G-buffer。
- 物体描边或模糊。
- UI / 小地图 / 监视器画面。

`offscreen.cpp` 演示的就是这种模式：

```text
Pass 1
    渲染镜像模型到离屏 color attachment。

Pass 2
    渲染屏幕场景。
    平面 fragment shader 采样 Pass 1 的 color attachment。
```

离屏渲染的关键点是：同一张 image 在不同阶段有不同身份。

```text
第一阶段
    它是 color attachment。

第二阶段
    它是 sampled image。
```

因此 image 的 usage、layout transition、render pass finalLayout 和 descriptor 中的 imageLayout 都必须匹配。

## 2. 程序整体结构

示例继承自框架基类：

```cpp
class VulkanExample : public VulkanExampleBase
```

核心开关是：

```cpp
bool debugDisplay = false;
```

打开 debug 后，第二个 pass 不画镜面和模型，而是直接全屏显示离屏 render target。

模型资源有两个：

```cpp
struct {
    vkglTF::Model example;
    vkglTF::Model plane;
} models;
```

- `example`：被渲染的龙模型。
- `plane`：接收镜面反射纹理的平面。

全局 uniform 数据结构如下：

```cpp
struct UniformData {
    glm::mat4 projection;
    glm::mat4 view;
    glm::mat4 model;
    glm::vec4 lightPos = glm::vec4(0.0f, 0.0f, 0.0f, 1.0f);
} uniformData;
```

每个并发帧有三份 UBO：

```cpp
struct UniformBuffers {
    vks::Buffer model;
    vks::Buffer mirror;
    vks::Buffer offscreen;
};

std::array<UniformBuffers, maxConcurrentFrames> uniformBuffers;
```

三份 UBO 分别服务于：

```text
model
    主 pass 中正常绘制龙模型。

mirror
    主 pass 中绘制镜面平面。

offscreen
    offscreen pass 中绘制翻转后的龙模型。
```

## 3. 离屏 framebuffer 结构

示例自己定义了一个 `OffscreenPass`：

```cpp
struct FrameBufferAttachment {
    VkImage image;
    VkDeviceMemory mem;
    VkImageView view;
};

struct OffscreenPass {
    int32_t width, height;
    VkFramebuffer frameBuffer;
    FrameBufferAttachment color, depth;
    VkRenderPass renderPass;
    VkSampler sampler;
    VkDescriptorImageInfo descriptor;
} offscreenPass{};
```

它包含：

- color attachment：保存离屏渲染结果，后面会被 fragment shader 采样。
- depth attachment：离屏 pass 的深度测试使用。
- render pass：专门用于离屏渲染。
- framebuffer：把 color/depth image view 组合起来。
- sampler：主 pass 采样 color attachment 时使用。
- descriptor：后面写入 descriptor set 的 `VkDescriptorImageInfo`。

离屏 framebuffer 尺寸固定为：

```cpp
#define FB_DIM 512
#define FB_COLOR_FORMAT VK_FORMAT_R8G8B8A8_UNORM
```

也就是 512 x 512，颜色格式为 RGBA8 UNORM。

## 4. 程序运行流程

初始化入口是 `prepare()`：

```cpp
void prepare()
{
    VulkanExampleBase::prepare();
    loadAssets();
    prepareOffscreen();
    prepareUniformBuffers();
    setupDescriptors();
    preparePipelines();
    prepared = true;
}
```

执行顺序很重要：

1. `loadAssets()`：加载平面和龙模型。
2. `prepareOffscreen()`：创建离屏 color/depth image、render pass、framebuffer、sampler 和 image descriptor。
3. `prepareUniformBuffers()`：为每个并发帧创建三份 UBO。
4. `setupDescriptors()`：创建普通 shaded layout 和 textured layout，并把 offscreen color attachment 写入 mirror descriptor。
5. `preparePipelines()`：创建 debug、mirror、shaded、shadedOffscreen 四条 pipeline。

每帧渲染入口是：

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

每帧先更新矩阵，再录制两个 render pass 的命令。

## 5. 为什么启用 shaderClipDistance

构造函数中有一行：

```cpp
enabledFeatures.shaderClipDistance = VK_TRUE;
```

源码注释说明：

```cpp
// The scene shader uses a clipping plane, so this feature has to be enabled
```

对应 shader 中的：

```glsl
gl_ClipDistance[0] = dot(vec4(inPos, 1.0), clipPlane);
```

`gl_ClipDistance` 是 shader clip distance 功能，需要设备特性支持并在 logical device 创建时启用。

这个示例当前 shader 里的 `clipPlane` 是零向量：

```glsl
vec4 clipPlane = vec4(0.0, 0.0, 0.0, 0.0);
```

所以它没有实际裁剪掉几何体，但代码仍展示了如果 shader 使用 `gl_ClipDistance`，C++ 侧必须启用对应 feature。

## 6. 加载资源

`loadAssets()` 加载两个 glTF 模型：

```cpp
const uint32_t glTFLoadingFlags =
    vkglTF::FileLoadingFlags::PreTransformVertices |
    vkglTF::FileLoadingFlags::PreMultiplyVertexColors |
    vkglTF::FileLoadingFlags::FlipY;

models.plane.loadFromFile(
    getAssetPath() + "models/plane.gltf",
    vulkanDevice,
    queue,
    glTFLoadingFlags);

models.example.loadFromFile(
    getAssetPath() + "models/chinesedragon.gltf",
    vulkanDevice,
    queue,
    glTFLoadingFlags);
```

平面用于显示镜面反射，龙模型会被绘制两次：

```text
offscreen pass
    绘制翻转后的龙，用作镜面反射内容。

main pass
    绘制正常龙，作为场景主体。
```

## 7. 创建 Offscreen Color Attachment

`prepareOffscreen()` 先设置离屏尺寸：

```cpp
offscreenPass.width = FB_DIM;
offscreenPass.height = FB_DIM;
```

然后创建 color attachment image：

```cpp
VkImageCreateInfo image = vks::initializers::imageCreateInfo();
image.imageType = VK_IMAGE_TYPE_2D;
image.format = FB_COLOR_FORMAT;
image.extent.width = offscreenPass.width;
image.extent.height = offscreenPass.height;
image.extent.depth = 1;
image.mipLevels = 1;
image.arrayLayers = 1;
image.samples = VK_SAMPLE_COUNT_1_BIT;
image.tiling = VK_IMAGE_TILING_OPTIMAL;
image.usage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
```

最关键的是 `usage`：

```cpp
VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT
```

这表示同一张 image 有两个用途：

```text
COLOR_ATTACHMENT
    第一遍 render pass 写入。

SAMPLED
    第二遍 render pass 中作为 sampler2D 读取。
```

如果缺少 `VK_IMAGE_USAGE_SAMPLED_BIT`，后面把它写进 combined image sampler descriptor 就是不合法的。

创建 image、分配 device local memory 并绑定：

```cpp
vkCreateImage(device, &image, nullptr, &offscreenPass.color.image);
vkGetImageMemoryRequirements(device, offscreenPass.color.image, &memReqs);
memAlloc.memoryTypeIndex =
    vulkanDevice->getMemoryType(
        memReqs.memoryTypeBits,
        VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);
vkAllocateMemory(device, &memAlloc, nullptr, &offscreenPass.color.mem);
vkBindImageMemory(device, offscreenPass.color.image, offscreenPass.color.mem, 0);
```

然后创建 color image view：

```cpp
VkImageViewCreateInfo colorImageView =
    vks::initializers::imageViewCreateInfo();
colorImageView.viewType = VK_IMAGE_VIEW_TYPE_2D;
colorImageView.format = FB_COLOR_FORMAT;
colorImageView.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
colorImageView.subresourceRange.levelCount = 1;
colorImageView.subresourceRange.layerCount = 1;
colorImageView.image = offscreenPass.color.image;
vkCreateImageView(device, &colorImageView, nullptr, &offscreenPass.color.view);
```

## 8. 创建 Offscreen Sampler

离屏 color attachment 后面会被 `mirror.frag` 和 `quad.frag` 采样，所以需要 sampler：

```cpp
VkSamplerCreateInfo samplerInfo =
    vks::initializers::samplerCreateInfo();
samplerInfo.magFilter = VK_FILTER_LINEAR;
samplerInfo.minFilter = VK_FILTER_LINEAR;
samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE;
samplerInfo.addressModeV = samplerInfo.addressModeU;
samplerInfo.addressModeW = samplerInfo.addressModeU;
samplerInfo.mipLodBias = 0.0f;
samplerInfo.maxAnisotropy = 1.0f;
samplerInfo.minLod = 0.0f;
samplerInfo.maxLod = 1.0f;
samplerInfo.borderColor = VK_BORDER_COLOR_FLOAT_OPAQUE_WHITE;
vkCreateSampler(device, &samplerInfo, nullptr, &offscreenPass.sampler);
```

这里使用 clamp-to-edge，避免在投影采样或 debug 显示时从边缘外重复采样。

离屏 color attachment 没有 mip chain，所以：

```cpp
samplerInfo.maxLod = 1.0f;
```

## 9. 创建 Offscreen Depth Attachment

离屏 pass 也需要深度测试，所以先查询支持的 depth format：

```cpp
VkFormat fbDepthFormat;
VkBool32 validDepthFormat =
    vks::tools::getSupportedDepthFormat(physicalDevice, &fbDepthFormat);
assert(validDepthFormat);
```

然后复用 image create info，改成 depth usage：

```cpp
image.format = fbDepthFormat;
image.usage = VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT;
```

创建 depth image、分配 memory、绑定 memory：

```cpp
vkCreateImage(device, &image, nullptr, &offscreenPass.depth.image);
vkGetImageMemoryRequirements(device, offscreenPass.depth.image, &memReqs);
vkAllocateMemory(device, &memAlloc, nullptr, &offscreenPass.depth.mem);
vkBindImageMemory(device, offscreenPass.depth.image, offscreenPass.depth.mem, 0);
```

创建 depth image view 时 aspect mask 至少包含 depth：

```cpp
depthStencilView.subresourceRange.aspectMask = VK_IMAGE_ASPECT_DEPTH_BIT;
if (fbDepthFormat >= VK_FORMAT_D16_UNORM_S8_UINT) {
    depthStencilView.subresourceRange.aspectMask |= VK_IMAGE_ASPECT_STENCIL_BIT;
}
```

如果 format 带 stencil，就额外包含 stencil aspect。

## 10. Offscreen Render Pass：color 最终变成可采样布局

离屏 pass 有两个 attachment：

```cpp
std::array<VkAttachmentDescription, 2> attchmentDescriptions = {};
```

color attachment 的配置：

```cpp
attchmentDescriptions[0].format = FB_COLOR_FORMAT;
attchmentDescriptions[0].samples = VK_SAMPLE_COUNT_1_BIT;
attchmentDescriptions[0].loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
attchmentDescriptions[0].storeOp = VK_ATTACHMENT_STORE_OP_STORE;
attchmentDescriptions[0].initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
attchmentDescriptions[0].finalLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
```

这里有两个关键点：

```text
storeOp = STORE
    offscreen pass 结束后要保留颜色结果，给下一 pass 采样。

finalLayout = SHADER_READ_ONLY_OPTIMAL
    offscreen pass 结束时，color attachment 自动转成 shader 可读 layout。
```

depth attachment 的配置：

```cpp
attchmentDescriptions[1].format = fbDepthFormat;
attchmentDescriptions[1].samples = VK_SAMPLE_COUNT_1_BIT;
attchmentDescriptions[1].loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
attchmentDescriptions[1].storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
attchmentDescriptions[1].initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
attchmentDescriptions[1].finalLayout =
    VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```

depth 只在离屏 pass 内部用于深度测试，之后不需要采样，所以 `storeOp` 是 `DONT_CARE`。

## 11. Attachment Reference 和 Subpass

color attachment 在 subpass 中的 layout 是：

```cpp
VkAttachmentReference colorReference =
    { 0, VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL };
```

depth attachment 在 subpass 中的 layout 是：

```cpp
VkAttachmentReference depthReference =
    { 1, VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL };
```

subpass 描述如下：

```cpp
VkSubpassDescription subpassDescription = {};
subpassDescription.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
subpassDescription.colorAttachmentCount = 1;
subpassDescription.pColorAttachments = &colorReference;
subpassDescription.pDepthStencilAttachment = &depthReference;
```

这表示离屏 render pass 只有一个 graphics subpass，同时写 color 和 depth。

## 12. Subpass Dependency：连接写入和后续采样

离屏 render pass 创建了两个 dependency：

```cpp
std::array<VkSubpassDependency, 2> dependencies;
```

第一个 dependency 从外部进入 subpass：

```cpp
dependencies[0].srcSubpass = VK_SUBPASS_EXTERNAL;
dependencies[0].dstSubpass = 0;
dependencies[0].srcStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
dependencies[0].dstStageMask =
    VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT |
    VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT |
    VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT;
dependencies[0].srcAccessMask = VK_ACCESS_NONE_KHR;
dependencies[0].dstAccessMask =
    VK_ACCESS_COLOR_ATTACHMENT_READ_BIT |
    VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT |
    VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT |
    VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
```

第二个 dependency 从 subpass 退出到外部：

```cpp
dependencies[1].srcSubpass = 0;
dependencies[1].dstSubpass = VK_SUBPASS_EXTERNAL;
dependencies[1].srcStageMask =
    VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT |
    VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT |
    VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT;
dependencies[1].dstStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
dependencies[1].srcAccessMask =
    VK_ACCESS_COLOR_ATTACHMENT_READ_BIT |
    VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT |
    VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT |
    VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
dependencies[1].dstAccessMask = VK_ACCESS_MEMORY_READ_BIT;
```

它的作用可以理解为：

```text
离屏 pass 的 color/depth 写入完成后
    后续 fragment shader 读取 offscreen color image 才能开始。
```

所以 `buildCommandBuffer()` 里有注释：

```cpp
// Explicit synchronization is not required between the render pass,
// as this is done implicit via sub pass dependencies
```

这里不需要额外插入 `vkCmdPipelineBarrier()`，因为 offscreen render pass 的 finalLayout 和 dependency 已经完成了布局转换与可见性同步。

## 13. 创建 Offscreen Framebuffer

render pass 创建后，把 color view 和 depth view 放入 framebuffer：

```cpp
VkImageView attachments[2];
attachments[0] = offscreenPass.color.view;
attachments[1] = offscreenPass.depth.view;
```

创建 framebuffer：

```cpp
VkFramebufferCreateInfo fbufCreateInfo =
    vks::initializers::framebufferCreateInfo();
fbufCreateInfo.renderPass = offscreenPass.renderPass;
fbufCreateInfo.attachmentCount = 2;
fbufCreateInfo.pAttachments = attachments;
fbufCreateInfo.width = offscreenPass.width;
fbufCreateInfo.height = offscreenPass.height;
fbufCreateInfo.layers = 1;
vkCreateFramebuffer(device, &fbufCreateInfo, nullptr, &offscreenPass.frameBuffer);
```

这个 framebuffer 不属于 swapchain，不会被 present。它只是一个中间 render target。

## 14. VkDescriptorImageInfo：把离屏 color attachment 暴露给 shader

最后，`prepareOffscreen()` 填充一个 descriptor：

```cpp
offscreenPass.descriptor.imageLayout =
    VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
offscreenPass.descriptor.imageView = offscreenPass.color.view;
offscreenPass.descriptor.sampler = offscreenPass.sampler;
```

这个 descriptor 后面会写入 mirror descriptor set：

```cpp
vks::initializers::writeDescriptorSet(
    descriptorSets[i].mirror,
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
    1,
    &offscreenPass.descriptor)
```

这就是离屏结果进入主 pass shader 的入口。

## 15. Descriptor Layout：shaded 和 textured 两套接口

示例有两套 descriptor set layout。

第一套是 shaded layout：

```cpp
setLayoutBindings = {
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        VK_SHADER_STAGE_VERTEX_BIT,
        0),
};
vkCreateDescriptorSetLayout(..., &descriptorSetLayouts.shaded);
```

它只有一个 binding：

```text
binding 0
    vertex shader uniform buffer
```

用于正常龙模型和离屏龙模型。

第二套是 textured layout：

```cpp
setLayoutBindings = {
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        VK_SHADER_STAGE_VERTEX_BIT,
        0),
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        VK_SHADER_STAGE_FRAGMENT_BIT,
        1)
};
vkCreateDescriptorSetLayout(..., &descriptorSetLayouts.textured);
```

它包含 UBO 和 sampler：

```text
binding 0
    vertex shader uniform buffer

binding 1
    fragment shader sampler2D，采样 offscreen color attachment
```

用于 mirror 平面和 debug 全屏显示。

## 16. Descriptor Set：每帧三组用途

每个并发帧都有三类 descriptor set：

```cpp
struct DescriptorSets {
    VkDescriptorSet offscreen{ VK_NULL_HANDLE };
    VkDescriptorSet mirror{ VK_NULL_HANDLE };
    VkDescriptorSet model{ VK_NULL_HANDLE };
};

std::array<DescriptorSets, maxConcurrentFrames> descriptorSets;
```

`mirror` 使用 textured layout：

```cpp
writeDescriptorSets = {
    vks::initializers::writeDescriptorSet(
        descriptorSets[i].mirror,
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        0,
        &uniformBuffers[i].mirror.descriptor),
    vks::initializers::writeDescriptorSet(
        descriptorSets[i].mirror,
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        1,
        &offscreenPass.descriptor),
};
```

`model` 使用 shaded layout，只绑定正常模型 UBO：

```cpp
vks::initializers::writeDescriptorSet(
    descriptorSets[i].model,
    VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
    0,
    &uniformBuffers[i].model.descriptor)
```

`offscreen` 也使用 shaded layout，只绑定离屏镜像模型 UBO：

```cpp
vks::initializers::writeDescriptorSet(
    descriptorSets[i].offscreen,
    VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
    0,
    &uniformBuffers[i].offscreen.descriptor)
```

这样同一个龙模型可以在两个 pass 中使用不同 transform。

## 17. Pipeline Layout：textured 和 shaded

两套 descriptor layout 对应两套 pipeline layout：

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutInfo =
    vks::initializers::pipelineLayoutCreateInfo(
        &descriptorSetLayouts.shaded,
        1);
vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayouts.shaded);

pipelineLayoutInfo =
    vks::initializers::pipelineLayoutCreateInfo(
        &descriptorSetLayouts.textured,
        1);
vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayouts.textured);
```

使用规则是：

```text
pipelineLayouts.shaded
    给 shaded / shadedOffscreen pipeline 使用。

pipelineLayouts.textured
    给 mirror / debug pipeline 使用。
```

绑定 descriptor set 时，pipeline layout 必须和当前 pipeline 兼容。

## 18. 四条 Pipeline 的职责

示例创建了四条 pipeline：

```cpp
struct {
    VkPipeline debug{ VK_NULL_HANDLE };
    VkPipeline shaded{ VK_NULL_HANDLE };
    VkPipeline shadedOffscreen{ VK_NULL_HANDLE };
    VkPipeline mirror{ VK_NULL_HANDLE };
} pipelines;
```

它们的用途如下：

| Pipeline | Render pass | Layout | Shader | 用途 |
| --- | --- | --- | --- | --- |
| `debug` | 主 render pass | textured | `quad.vert/frag` | 全屏显示 offscreen color |
| `mirror` | 主 render pass | textured | `mirror.vert/frag` | 把 offscreen color 投影采样到平面 |
| `shaded` | 主 render pass | shaded | `phong.vert/frag` | 正常绘制龙模型 |
| `shadedOffscreen` | offscreen render pass | shaded | `phong.vert/frag` | 绘制翻转后的龙到离屏 framebuffer |

`debug` 和 `mirror` 都需要采样离屏结果，所以使用 textured descriptor layout。

`shaded` 和 `shadedOffscreen` 只需要 UBO，所以使用 shaded descriptor layout。

## 19. Graphics Pipeline：debug 和 mirror

公共 pipeline state 创建后，先把 cull mode 设置为 none：

```cpp
rasterizationState.cullMode = VK_CULL_MODE_NONE;
```

debug pipeline 使用全屏三角形 shader：

```cpp
shaderStages[0] = loadShader(
    getShadersPath() + "offscreen/quad.vert.spv",
    VK_SHADER_STAGE_VERTEX_BIT);
shaderStages[1] = loadShader(
    getShadersPath() + "offscreen/quad.frag.spv",
    VK_SHADER_STAGE_FRAGMENT_BIT);
vkCreateGraphicsPipelines(..., &pipelines.debug);
```

mirror pipeline 使用平面投影采样 shader：

```cpp
shaderStages[0] = loadShader(
    getShadersPath() + "offscreen/mirror.vert.spv",
    VK_SHADER_STAGE_VERTEX_BIT);
shaderStages[1] = loadShader(
    getShadersPath() + "offscreen/mirror.frag.spv",
    VK_SHADER_STAGE_FRAGMENT_BIT);
vkCreateGraphicsPipelines(..., &pipelines.mirror);
```

这两条 pipeline 都使用：

```cpp
pipelineCI.layout = pipelineLayouts.textured;
pipelineCI.renderPass = renderPass;
```

也就是主 swapchain render pass，并且 descriptor set 里包含 sampler2D。

## 20. Graphics Pipeline：shaded 和 shadedOffscreen

Phong shading pipeline 使用 shaded layout：

```cpp
pipelineCI.layout = pipelineLayouts.shaded;
shaderStages[0] = loadShader(
    getShadersPath() + "offscreen/phong.vert.spv",
    VK_SHADER_STAGE_VERTEX_BIT);
shaderStages[1] = loadShader(
    getShadersPath() + "offscreen/phong.frag.spv",
    VK_SHADER_STAGE_FRAGMENT_BIT);
vkCreateGraphicsPipelines(..., &pipelines.shaded);
```

这是主场景中的正常龙模型。

离屏 pipeline 复用同一组 shader，但改变 cull mode 和 render pass：

```cpp
rasterizationState.cullMode = VK_CULL_MODE_FRONT_BIT;
pipelineCI.renderPass = offscreenPass.renderPass;
vkCreateGraphicsPipelines(..., &pipelines.shadedOffscreen);
```

为什么改变 cull mode？

离屏模型的 transform 中有一项 Y 轴负缩放：

```cpp
uniformData.model = glm::scale(uniformData.model, glm::vec3(1.0f, -1.0f, 1.0f));
```

负缩放会翻转三角形绕序。为了让镜像模型仍然正确可见，需要把 cull mode 从 back 改成 front。

为什么要改变 render pass？

Vulkan graphics pipeline 和 render pass 兼容性相关。用于 offscreen framebuffer 的 pipeline 必须使用 `offscreenPass.renderPass` 创建。

## 21. Uniform Buffer：正常模型、镜面平面、离屏镜像模型

每帧三份 UBO 都是同一个 `UniformData` 大小：

```cpp
vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
    VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
    &buffer.model,
    sizeof(UniformData));
```

`updateUniformBuffers()` 中先更新公共投影和 view：

```cpp
uniformData.projection = camera.matrices.perspective;
uniformData.view = camera.matrices.view;
```

正常模型 UBO：

```cpp
uniformData.model = glm::mat4(1.0f);
uniformData.model = glm::rotate(
    uniformData.model,
    glm::radians(modelRotation.y),
    glm::vec3(0.0f, 1.0f, 0.0f));
uniformData.model = glm::translate(uniformData.model, modelPosition);
memcpy(uniformBuffers[currentBuffer].model.mapped, &uniformData, sizeof(UniformData));
```

镜面平面 UBO：

```cpp
uniformData.model = glm::mat4(1.0f);
memcpy(uniformBuffers[currentBuffer].mirror.mapped, &uniformData, sizeof(UniformData));
```

离屏镜像模型 UBO：

```cpp
uniformData.model = glm::mat4(1.0f);
uniformData.model = glm::rotate(
    uniformData.model,
    glm::radians(modelRotation.y),
    glm::vec3(0.0f, 1.0f, 0.0f));
uniformData.model = glm::scale(uniformData.model, glm::vec3(1.0f, -1.0f, 1.0f));
uniformData.model = glm::translate(uniformData.model, modelPosition);
memcpy(uniformBuffers[currentBuffer].offscreen.mapped, &uniformData, sizeof(UniformData));
```

这就是镜像效果的数学核心：在离屏 pass 中把龙沿 Y 轴翻转，再把结果贴到平面上。

## 22. 第一遍 Render Pass：渲染镜像场景到 offscreen framebuffer

`buildCommandBuffer()` 先开始 offscreen render pass：

```cpp
renderPassBeginInfo.renderPass = offscreenPass.renderPass;
renderPassBeginInfo.framebuffer = offscreenPass.frameBuffer;
renderPassBeginInfo.renderArea.extent.width = offscreenPass.width;
renderPassBeginInfo.renderArea.extent.height = offscreenPass.height;
```

清除 color 和 depth：

```cpp
clearValues[0].color = { { 0.0f, 0.0f, 0.0f, 0.0f } };
clearValues[1].depthStencil = { 1.0f, 0 };
```

设置 512 x 512 viewport/scissor：

```cpp
VkViewport viewport =
    vks::initializers::viewport(
        (float)offscreenPass.width,
        (float)offscreenPass.height,
        0.0f,
        1.0f);
vkCmdSetViewport(cmdBuffer, 0, 1, &viewport);

VkRect2D scissor =
    vks::initializers::rect2D(
        offscreenPass.width,
        offscreenPass.height,
        0,
        0);
vkCmdSetScissor(cmdBuffer, 0, 1, &scissor);
```

绑定离屏 UBO 和离屏 pipeline，绘制龙模型：

```cpp
vkCmdBindDescriptorSets(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayouts.shaded,
    0,
    1,
    &descriptorSets[currentBuffer].offscreen,
    0,
    nullptr);

vkCmdBindPipeline(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelines.shadedOffscreen);

models.example.draw(cmdBuffer);
```

这个 pass 结束后，offscreen color attachment 中保存的是镜像龙模型。

## 23. 两个 Render Pass 之间为什么不显式 barrier

offscreen pass 结束后，代码没有插入 `vkCmdPipelineBarrier()`。

原因是离屏 render pass 的 attachment 和 dependency 已经描述了所需转换：

```cpp
attchmentDescriptions[0].finalLayout =
    VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
```

以及：

```cpp
dependencies[1].dstStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
dependencies[1].dstAccessMask = VK_ACCESS_MEMORY_READ_BIT;
```

所以从概念上看：

```text
offscreen color attachment write
    -> render pass end
    -> layout becomes SHADER_READ_ONLY_OPTIMAL
    -> fragment shader sampling in main pass
```

这正是 render pass finalLayout + subpass dependency 的用途。

## 24. 第二遍 Render Pass：渲染主场景

第二个 pass 使用默认 swapchain render pass：

```cpp
renderPassBeginInfo.renderPass = renderPass;
renderPassBeginInfo.framebuffer = frameBuffers[currentImageIndex];
renderPassBeginInfo.renderArea.extent.width = width;
renderPassBeginInfo.renderArea.extent.height = height;
```

viewport/scissor 使用窗口尺寸：

```cpp
VkViewport viewport =
    vks::initializers::viewport((float)width, (float)height, 0.0f, 1.0f);
vkCmdSetViewport(cmdBuffer, 0, 1, &viewport);

VkRect2D scissor = vks::initializers::rect2D(width, height, 0, 0);
vkCmdSetScissor(cmdBuffer, 0, 1, &scissor);
```

如果 debug display 打开，就直接显示离屏纹理：

```cpp
vkCmdBindDescriptorSets(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayouts.textured,
    0,
    1,
    &descriptorSets[currentBuffer].mirror,
    0,
    nullptr);
vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.debug);
vkCmdDraw(cmdBuffer, 3, 1, 0, 0);
```

这里用的是全屏三角形，不需要 vertex buffer。

如果 debug display 关闭，就正常渲染镜面和平面上方的模型。

## 25. 主场景：先画镜面平面，再画正常模型

正常模式先绘制 reflection plane：

```cpp
vkCmdBindDescriptorSets(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayouts.textured,
    0,
    1,
    &descriptorSets[currentBuffer].mirror,
    0,
    nullptr);
vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.mirror);
models.plane.draw(cmdBuffer);
```

`descriptorSets[currentBuffer].mirror` 中包含：

```text
binding 0
    mirror 平面的 UBO

binding 1
    offscreen color attachment 的 sampler2D
```

然后绘制正常龙模型：

```cpp
vkCmdBindDescriptorSets(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayouts.shaded,
    0,
    1,
    &descriptorSets[currentBuffer].model,
    0,
    nullptr);
vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.shaded);
models.example.draw(cmdBuffer);
```

这次使用的是正常模型 UBO，没有 Y 轴翻转。

## 26. phong shader：普通模型光照

`phong.vert` 输入 position、color、normal：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inColor;
layout (location = 2) in vec3 inNormal;
```

UBO 包含 projection、view、model、lightPos：

```glsl
layout (binding = 0) uniform UBO
{
    mat4 projection;
    mat4 view;
    mat4 model;
    vec4 lightPos;
} ubo;
```

顶点位置变换：

```glsl
gl_Position = ubo.projection * ubo.view * ubo.model * vec4(inPos, 1.0);
```

输出 eye space position 和 light vector：

```glsl
outEyePos = vec3(ubo.view * ubo.model * vec4(inPos, 1.0));
outLightVec = normalize(ubo.lightPos.xyz - outEyePos);
```

fragment shader 做简单 Phong 光照：

```glsl
vec3 Eye = normalize(-inEyePos);
vec3 Reflected = normalize(reflect(-inLightVec, inNormal));

vec4 IAmbient = vec4(0.1, 0.1, 0.1, 1.0);
vec4 IDiffuse = vec4(max(dot(inNormal, inLightVec), 0.0));
...
outFragColor = vec4((IAmbient + IDiffuse) * vec4(inColor, 1.0) + ISpecular);
```

同一套 shader 同时用于主 pass 的正常龙和 offscreen pass 的镜像龙。

## 27. mirror shader：投影采样离屏纹理

`mirror.vert` 只接收 position：

```glsl
layout (location = 0) in vec3 inPos;
```

它用 UBO 计算裁剪空间位置，并把这个位置传给 fragment shader：

```glsl
outPos = ubo.projection * ubo.view * ubo.model * vec4(inPos.xyz, 1.0);
gl_Position = outPos;
```

`mirror.frag` 中绑定离屏纹理：

```glsl
layout (binding = 1) uniform sampler2D samplerColor;
```

它把 clip space 坐标转换到 0..1 的纹理坐标：

```glsl
vec4 tmp = vec4(1.0 / inPos.w);
vec4 projCoord = inPos * tmp;

projCoord += vec4(1.0);
projCoord *= vec4(0.5);
```

这就是常见的 projective texture lookup：

```text
clip space
    -> perspective divide
    -> NDC [-1, 1]
    -> texture coordinates [0, 1]
```

然后对离屏纹理做一个 7 x 7 的简单 blur：

```glsl
const float blurSize = 1.0 / 512.0;

for (int x = -3; x <= 3; x++)
{
    for (int y = -3; y <= 3; y++)
    {
        reflection += texture(
            samplerColor,
            vec2(projCoord.s + x * blurSize, projCoord.t + y * blurSize)) / 49.0;
    }
}
```

只在平面正面绘制反射：

```glsl
if (gl_FrontFacing)
{
    ...
}
```

这样平面的反面不会显示镜像结果。

## 28. debug shader：全屏显示 render target

debug pipeline 使用 `quad.vert` 和 `quad.frag`。

`quad.vert` 不需要 vertex buffer，而是用 `gl_VertexIndex` 生成一个全屏三角形：

```glsl
outUV = vec2((gl_VertexIndex << 1) & 2, gl_VertexIndex & 2);
gl_Position = vec4(outUV * 2.0f - 1.0f, 0.0f, 1.0f);
```

C++ 侧对应：

```cpp
vkCmdDraw(cmdBuffer, 3, 1, 0, 0);
```

`quad.frag` 直接采样 offscreen texture：

```glsl
layout (binding = 1) uniform sampler2D samplerColor;

void main()
{
    outFragColor = texture(samplerColor, inUV);
}
```

这个模式非常适合调试离屏 pass 输出是否正确。

## 29. UI Overlay：Display Render Target

UI 只有一个开关：

```cpp
virtual void OnUpdateUIOverlay(vks::UIOverlay *overlay)
{
    if (overlay->header("Settings")) {
        overlay->checkBox("Display render target", &debugDisplay);
    }
}
```

它控制 `buildCommandBuffer()` 中第二个 pass 的分支：

```cpp
if (debugDisplay)
{
    // Display the offscreen render target
    ...
}
else
{
    // Render the scene
    ...
}
```

调试 offscreen 渲染时，先打开这个开关直接看 render target，比只看最终镜面效果更容易定位问题。

## 30. 和普通纹理采样的关系

`offscreen.cpp` 和 `texture.cpp` 的后半段很像：都创建 `VkDescriptorImageInfo`，都在 fragment shader 中用 `sampler2D` 采样。

区别在于 image 数据来源不同：

| 示例 | sampled image 来源 | 何时写入 |
| --- | --- | --- |
| `texture.cpp` | KTX 文件上传到 `VkImage` | 初始化阶段 |
| `offscreen.cpp` | color attachment 渲染输出 | 每帧 offscreen pass |

因此 offscreen texture 必须额外关注：

- image usage 同时包含 color attachment 和 sampled。
- render pass finalLayout 必须转到 shader read layout。
- pass 之间要有正确同步。
- descriptor 指向的是 render target 的 color view。

## 31. 资源生命周期

析构函数释放离屏资源：

```cpp
vkDestroyImageView(device, offscreenPass.color.view, nullptr);
vkDestroyImage(device, offscreenPass.color.image, nullptr);
vkFreeMemory(device, offscreenPass.color.mem, nullptr);
vkDestroyImageView(device, offscreenPass.depth.view, nullptr);
vkDestroyImage(device, offscreenPass.depth.image, nullptr);
vkFreeMemory(device, offscreenPass.depth.mem, nullptr);
vkDestroyRenderPass(device, offscreenPass.renderPass, nullptr);
vkDestroySampler(device, offscreenPass.sampler, nullptr);
vkDestroyFramebuffer(device, offscreenPass.frameBuffer, nullptr);
```

还释放 pipeline、pipeline layout、descriptor set layout 和 uniform buffer：

```cpp
vkDestroyPipeline(device, pipelines.debug, nullptr);
vkDestroyPipeline(device, pipelines.shaded, nullptr);
vkDestroyPipeline(device, pipelines.shadedOffscreen, nullptr);
vkDestroyPipeline(device, pipelines.mirror, nullptr);
vkDestroyPipelineLayout(device, pipelineLayouts.textured, nullptr);
vkDestroyPipelineLayout(device, pipelineLayouts.shaded, nullptr);
vkDestroyDescriptorSetLayout(device, descriptorSetLayouts.shaded, nullptr);
vkDestroyDescriptorSetLayout(device, descriptorSetLayouts.textured, nullptr);
```

从依赖关系看：

```text
descriptor set
    引用 offscreenPass.color.view 和 offscreenPass.sampler。

framebuffer
    引用 color/depth image view。

image view
    引用 image。

image
    绑定 device memory。
```

程序退出时统一销毁这些对象。descriptor pool 由基类管理。

## 32. 常见错误和调试方向

第一，offscreen color image 的 usage 必须包含两种用途：

```cpp
VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT
```

少了 sampled usage，后续采样不合法。少了 color attachment usage，offscreen render pass 不能写它。

第二，color attachment 必须 `storeOp = VK_ATTACHMENT_STORE_OP_STORE`：

```cpp
attchmentDescriptions[0].storeOp = VK_ATTACHMENT_STORE_OP_STORE;
```

如果使用 `DONT_CARE`，render pass 结束后颜色结果可以被丢弃，后续采样内容不可靠。

第三，finalLayout 要和 descriptor imageLayout 匹配：

```cpp
attchmentDescriptions[0].finalLayout =
    VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;

offscreenPass.descriptor.imageLayout =
    VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
```

第四，offscreen pipeline 必须使用 offscreen render pass 创建：

```cpp
pipelineCI.renderPass = offscreenPass.renderPass;
vkCreateGraphicsPipelines(..., &pipelines.shadedOffscreen);
```

不能拿主 render pass 创建的 pipeline 直接用于不兼容的 offscreen render pass。

第五，镜像模型使用负缩放后要处理 cull mode：

```cpp
rasterizationState.cullMode = VK_CULL_MODE_FRONT_BIT;
```

否则可能因为绕序翻转导致模型被剔除。

第六，如果看不到镜面结果，先打开 `Display render target`。如果 debug 全屏图像正确，问题多半在 mirror shader 的投影采样或平面绘制；如果 debug 图像不正确，问题在 offscreen pass。

第七，修改 shader 后要重新编译 SPIR-V。运行时加载的是：

```text
shaders/glsl/offscreen/phong.vert.spv
shaders/glsl/offscreen/phong.frag.spv
shaders/glsl/offscreen/mirror.vert.spv
shaders/glsl/offscreen/mirror.frag.spv
shaders/glsl/offscreen/quad.vert.spv
shaders/glsl/offscreen/quad.frag.spv
```

不是直接加载 GLSL 文本。

## 33. 从这个示例提炼出的工程原则

第一，中间渲染结果要从创建 image 时就声明完整用途。

如果一张 image 既要作为 render target，又要作为 texture，就必须在 `usage` 中同时包含 attachment 和 sampled。

第二，render pass 的 finalLayout 很重要。

offscreen pass 写完 color attachment 后，下一 pass 要采样它，所以 final layout 应该是 shader read layout。

第三，pass 间同步最好由明确的 dependency 表达。

本示例通过 subpass dependency 连接 color attachment write 和 fragment shader read，避免手写额外 barrier。

第四，离屏 pipeline 通常需要自己的 render pass。

Vulkan pipeline 创建时绑定 render pass 兼容信息。即使 shader 相同，主 pass 和 offscreen pass 也可能需要不同 pipeline。

第五，调试离屏渲染时要提供直接显示 render target 的路径。

这个示例的 `debug` pipeline 很实用，能把问题快速定位到“离屏 pass 错”还是“后续采样错”。

## 34. 一个简化版 mental model

可以把 `offscreen.cpp` 理解成下面这条链路：

```text
prepareOffscreen()
    -> create color image (COLOR_ATTACHMENT | SAMPLED)
    -> create depth image
    -> create offscreen render pass
    -> finalLayout = SHADER_READ_ONLY_OPTIMAL
    -> create framebuffer
    -> create sampler
    -> fill VkDescriptorImageInfo

buildCommandBuffer()
    -> Pass 1: render mirrored dragon into offscreen framebuffer
    -> offscreen color becomes shader-readable
    -> Pass 2: sample offscreen color on mirror plane
    -> draw normal dragon
```

核心思想就是：

```text
render to texture
    先把场景渲染进 image
    再把这个 image 当纹理采样
```

## 35. 总结

`offscreen.cpp` 展示了 Vulkan 中 render-to-texture 的基础做法。它创建了一个独立的 512 x 512 framebuffer，包含 color 和 depth attachment；第一遍 render pass 把镜像模型写入 color attachment；第二遍 render pass 把这个 color attachment 作为 `sampler2D` 采样到平面上，形成镜面反射。

这个示例中：

- `prepareOffscreen()` 创建离屏 color/depth image、render pass、framebuffer、sampler 和 descriptor。
- `setupDescriptors()` 创建 shaded/textured 两套 descriptor layout，并为每帧分配 model、mirror、offscreen 三类 descriptor set。
- `preparePipelines()` 创建 debug、mirror、shaded、shadedOffscreen 四条 pipeline。
- `updateUniformBuffers()` 生成正常模型、镜面平面和 Y 轴翻转模型的三份 UBO。
- `buildCommandBuffer()` 在同一个 command buffer 中先执行 offscreen pass，再执行主场景 pass。
- `mirror.frag` 把离屏结果投影采样到平面，并做一个简单 blur。
- debug 模式可以直接显示 offscreen render target，便于调试。

如果后续要做阴影贴图、后处理、反射、G-buffer 或任意多 pass 渲染，这个示例里的 attachment usage、render pass finalLayout、subpass dependency、descriptor image 和 pass 间采样流程都是核心基础。
