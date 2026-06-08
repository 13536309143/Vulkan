# Vulkan 示例解析：texture.cpp 如何加载 KTX 纹理并在 Shader 中采样

本文分析的是 `examples/texture/texture.cpp`。这个示例展示了 Vulkan 中最基础、也最重要的纹理使用流程：**从磁盘加载一张 2D 纹理，把它上传到 GPU 的 `VkImage`，创建 `VkImageView` 和 `VkSampler`，再通过 descriptor set 交给 fragment shader 采样。**

这个示例最终绘制一个带 UV 的四边形：

- 四边形由两个三角形组成。
- 顶点包含 position、uv、normal。
- 纹理文件是 `assets/textures/metalplate01_rgba.ktx`。
- 纹理数据先进入 host visible staging buffer。
- staging buffer 再复制到 device local optimal tiled image。
- fragment shader 通过 `sampler2D` 采样纹理。
- UI 中可以调整 `LOD bias`，观察 mip level 选择的变化。

这篇文章会按代码执行顺序拆解：KTX 读取、staging 上传、image layout transition、mipmap copy region、sampler、image view、descriptor、pipeline、command buffer 和 shader 采样。

## 1. 示例要解决的问题

Vulkan 里的纹理不是一个单独对象。要让 shader 采样一张图，通常至少需要这些对象配合：

```text
VkImage
    保存实际图像数据。

VkDeviceMemory
    绑定到 VkImage 的显存。

VkImageView
    描述 shader 能看到 VkImage 的哪些 mip level、array layer 和格式视图。

VkSampler
    描述如何采样图像，例如过滤、寻址模式、mipmap 模式、各向异性过滤。

VkDescriptorSet
    把 image view 和 sampler 绑定到 shader 的 binding 上。
```

`texture.cpp` 的开头注释已经说明了主题：

```cpp
/*
* Vulkan Example - Texture loading (and display) example (including mip maps)
*
* This sample shows how to upload a 2D texture to the device and how to display it.
* In Vulkan this is done using images, views and samplers.
*/
```

它不是展示复杂材质系统，而是把 Vulkan 使用 2D 纹理时最关键的一条路径讲清楚。

## 2. 程序整体结构

示例继承自框架基类：

```cpp
class VulkanExample : public VulkanExampleBase
```

`VulkanExampleBase` 负责 Vulkan instance、device、swapchain、render pass、framebuffer、command buffer、同步、camera 和 UI overlay 等基础设施。

这个示例自己维护的核心资源如下。

首先是纹理对象：

```cpp
struct Texture {
    VkSampler sampler{ VK_NULL_HANDLE };
    VkImage image{ VK_NULL_HANDLE };
    VkImageLayout imageLayout;
    VkDeviceMemory deviceMemory{ VK_NULL_HANDLE };
    VkImageView view{ VK_NULL_HANDLE };
    uint32_t width{ 0 };
    uint32_t height{ 0 };
    uint32_t mipLevels{ 0 };
} texture;
```

这个结构体不是 Vulkan 原生对象，而是示例自己定义的聚合结构。它把一张可采样纹理所需的 Vulkan 对象放在一起。

然后是四边形的 vertex buffer 和 index buffer：

```cpp
vks::Buffer vertexBuffer;
vks::Buffer indexBuffer;
uint32_t indexCount{ 0 };
```

再是每帧更新的 uniform 数据：

```cpp
struct UniformData {
    glm::mat4 projection;
    glm::mat4 modelView;
    glm::vec4 viewPos;
    float lodBias = 0.0f;
} uniformData;

std::array<vks::Buffer, maxConcurrentFrames> uniformBuffers;
```

`lodBias` 用来控制 fragment shader 调用 `texture()` 时的 mip level bias。

最后是 pipeline 和 descriptor 相关对象：

```cpp
VkPipeline pipeline{ VK_NULL_HANDLE };
VkPipelineLayout pipelineLayout{ VK_NULL_HANDLE };
VkDescriptorSetLayout descriptorSetLayout{ VK_NULL_HANDLE };
std::array<VkDescriptorSet, maxConcurrentFrames> descriptorSets{};
```

这个示例只需要一条 graphics pipeline，因为它只渲染一个四边形。

## 3. 程序运行流程

初始化入口是 `prepare()`：

```cpp
void prepare()
{
    VulkanExampleBase::prepare();
    loadTexture();
    generateQuad();
    prepareUniformBuffers();
    setupDescriptors();
    preparePipelines();
    prepared = true;
}
```

这几步可以理解为：

1. `VulkanExampleBase::prepare()`：创建 swapchain、render pass、framebuffer 等基础 Vulkan 对象。
2. `loadTexture()`：读取 KTX 文件，创建 `VkImage`、`VkImageView` 和 `VkSampler`。
3. `generateQuad()`：创建带 UV 的四边形 vertex/index buffer。
4. `prepareUniformBuffers()`：为每个并发帧创建 uniform buffer。
5. `setupDescriptors()`：把 UBO 和纹理写入 descriptor set。
6. `preparePipelines()`：创建 pipeline layout 和 graphics pipeline。

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

每帧变化的数据只有 uniform buffer，纹理本身不会每帧重新上传。

## 4. 设备特性：可选启用各向异性过滤

示例重写了 `getEnabledFeatures()`：

```cpp
virtual void getEnabledFeatures()
{
    if (deviceFeatures.samplerAnisotropy) {
        enabledFeatures.samplerAnisotropy = VK_TRUE;
    };
}
```

`samplerAnisotropy` 是可选设备特性。只有 physical device 支持时，logical device 创建时才能启用。

后面创建 sampler 时也会再次检查：

```cpp
if (vulkanDevice->features.samplerAnisotropy) {
    sampler.maxAnisotropy =
        vulkanDevice->properties.limits.maxSamplerAnisotropy;
    sampler.anisotropyEnable = VK_TRUE;
} else {
    sampler.maxAnisotropy = 1.0;
    sampler.anisotropyEnable = VK_FALSE;
}
```

这体现了 Vulkan 的常见模式：**特性需要先查询，再启用，再使用。**

## 5. 纹理文件：为什么使用 KTX

`loadTexture()` 中加载的是 KTX 文件：

```cpp
std::string filename =
    getAssetPath() + "textures/metalplate01_rgba.ktx";

VkFormat format = VK_FORMAT_R8G8B8A8_UNORM;
```

KTX 是 Khronos 的纹理容器格式，可以保存纹理尺寸、mipmap 数据、格式信息等。这个示例使用的纹理是 RGBA 8-bit UNORM，对应 Vulkan 格式：

```cpp
VK_FORMAT_R8G8B8A8_UNORM
```

非 Android 平台直接从文件创建 KTX texture：

```cpp
result = ktxTexture_CreateFromNamedFile(
    filename.c_str(),
    KTX_TEXTURE_CREATE_LOAD_IMAGE_DATA_BIT,
    &ktxTexture);
```

Android 上资源在 APK 内部，所以需要先通过 asset manager 读出字节，再从内存创建：

```cpp
result = ktxTexture_CreateFromMemory(
    textureData,
    size,
    KTX_TEXTURE_CREATE_LOAD_IMAGE_DATA_BIT,
    &ktxTexture);
```

加载完成后，示例从 KTX 对象读取宽、高、mip level 数量和原始像素数据：

```cpp
texture.width = ktxTexture->baseWidth;
texture.height = ktxTexture->baseHeight;
texture.mipLevels = ktxTexture->numLevels;
ktx_uint8_t *ktxTextureData = ktxTexture_GetData(ktxTexture);
ktx_size_t ktxTextureSize = ktxTexture_GetSize(ktxTexture);
```

## 6. Linear Tiling 和 Optimal Tiling 的区别

代码注释中详细解释了 Vulkan image tiling：

```cpp
VkBool32 useStaging = true;
```

本示例默认使用 staging 路径，也就是：

```text
KTX data
    -> host visible staging buffer
    -> device local optimal tiled VkImage
```

Vulkan image 有两种常见 tiling：

```text
VK_IMAGE_TILING_LINEAR
    内存布局相对线性，CPU 可以直接映射写入。
    但格式和采样支持有限，通常不适合作为最终纹理。

VK_IMAGE_TILING_OPTIMAL
    GPU 实现自定义布局，适合采样和渲染。
    通常位于 device local memory，CPU 不能直接写。
```

示例保留了 linear tiling 分支用于学习：

```cpp
bool forceLinearTiling = false;
```

但实际路径总是推荐 optimal tiling。注释也直接给出结论：

```text
In Short: Always use optimal tiled images for rendering.
```

## 7. Staging Buffer：把 KTX 数据先放到 CPU 可写内存

optimal tiled image 不能直接从 CPU 写入，所以示例先创建 staging buffer：

```cpp
VkBufferCreateInfo bufferCreateInfo = vks::initializers::bufferCreateInfo();
bufferCreateInfo.size = ktxTextureSize;
bufferCreateInfo.usage = VK_BUFFER_USAGE_TRANSFER_SRC_BIT;
bufferCreateInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
VK_CHECK_RESULT(vkCreateBuffer(device, &bufferCreateInfo, nullptr, &stagingBuffer));
```

这个 buffer 的用途是 transfer source：

```cpp
VK_BUFFER_USAGE_TRANSFER_SRC_BIT
```

然后查询内存需求：

```cpp
vkGetBufferMemoryRequirements(device, stagingBuffer, &memReqs);
```

分配 host visible、host coherent 内存：

```cpp
memAllocInfo.memoryTypeIndex =
    vulkanDevice->getMemoryType(
        memReqs.memoryTypeBits,
        VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
        VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
```

绑定 buffer 和 memory：

```cpp
vkBindBufferMemory(device, stagingBuffer, stagingMemory, 0);
```

最后把 KTX 像素数据复制进去：

```cpp
uint8_t *data;
vkMapMemory(device, stagingMemory, 0, memReqs.size, 0, (void **)&data);
memcpy(data, ktxTextureData, ktxTextureSize);
vkUnmapMemory(device, stagingMemory);
```

因为内存带有 `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`，这里不需要手动调用 `vkFlushMappedMemoryRanges()`。

## 8. VkBufferImageCopy：为每个 mip level 准备拷贝区域

KTX 文件中已经包含 mipmap 数据。示例需要把每个 mip level 从 staging buffer 拷贝到目标 `VkImage` 的对应 mip level。

代码为每个 mip level 创建一个 `VkBufferImageCopy`：

```cpp
std::vector<VkBufferImageCopy> bufferCopyRegions;

for (uint32_t i = 0; i < texture.mipLevels; i++) {
    ktx_size_t offset;
    KTX_error_code ret =
        ktxTexture_GetImageOffset(ktxTexture, i, 0, 0, &offset);
    assert(ret == KTX_SUCCESS);

    VkBufferImageCopy bufferCopyRegion = {};
    bufferCopyRegion.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    bufferCopyRegion.imageSubresource.mipLevel = i;
    bufferCopyRegion.imageSubresource.baseArrayLayer = 0;
    bufferCopyRegion.imageSubresource.layerCount = 1;
    bufferCopyRegion.imageExtent.width = ktxTexture->baseWidth >> i;
    bufferCopyRegion.imageExtent.height = ktxTexture->baseHeight >> i;
    bufferCopyRegion.imageExtent.depth = 1;
    bufferCopyRegion.bufferOffset = offset;
    bufferCopyRegions.push_back(bufferCopyRegion);
}
```

关键字段含义是：

```text
bufferOffset
    当前 mip level 数据在 staging buffer 中的起始字节偏移。

imageSubresource.mipLevel
    要写入 VkImage 的哪个 mip level。

imageExtent
    当前 mip level 的宽、高、深度。
```

这里没有手动计算每个 mip 的字节偏移，而是使用 `ktxTexture_GetImageOffset()` 从 KTX 数据结构中查询。这比手算更稳妥。

## 9. 创建 Device Local Optimal Image

接下来创建真正用于 shader 采样的 `VkImage`：

```cpp
VkImageCreateInfo imageCreateInfo = vks::initializers::imageCreateInfo();
imageCreateInfo.imageType = VK_IMAGE_TYPE_2D;
imageCreateInfo.format = format;
imageCreateInfo.mipLevels = texture.mipLevels;
imageCreateInfo.arrayLayers = 1;
imageCreateInfo.samples = VK_SAMPLE_COUNT_1_BIT;
imageCreateInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
imageCreateInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
imageCreateInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
imageCreateInfo.extent = { texture.width, texture.height, 1 };
imageCreateInfo.usage =
    VK_IMAGE_USAGE_TRANSFER_DST_BIT |
    VK_IMAGE_USAGE_SAMPLED_BIT;
VK_CHECK_RESULT(vkCreateImage(device, &imageCreateInfo, nullptr, &texture.image));
```

几个关键点：

- `imageType = VK_IMAGE_TYPE_2D`：普通 2D 纹理。
- `mipLevels = texture.mipLevels`：保留 KTX 文件中的 mip chain。
- `tiling = VK_IMAGE_TILING_OPTIMAL`：使用 GPU 友好的布局。
- `initialLayout = VK_IMAGE_LAYOUT_UNDEFINED`：初始内容不需要保留。
- `usage = TRANSFER_DST | SAMPLED`：先作为拷贝目标，再作为 shader 采样资源。

然后分配 device local memory：

```cpp
vkGetImageMemoryRequirements(device, texture.image, &memReqs);
memAllocInfo.allocationSize = memReqs.size;
memAllocInfo.memoryTypeIndex =
    vulkanDevice->getMemoryType(
        memReqs.memoryTypeBits,
        VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);
vkAllocateMemory(device, &memAllocInfo, nullptr, &texture.deviceMemory);
vkBindImageMemory(device, texture.image, texture.deviceMemory, 0);
```

device local memory 通常不能被 CPU 直接 map，但适合 GPU 采样。

## 10. Image Layout Transition：UNDEFINED 到 TRANSFER_DST

在 Vulkan 中，image 必须处于正确 layout 才能执行对应操作。刚创建的 image 是：

```cpp
VK_IMAGE_LAYOUT_UNDEFINED
```

要把 staging buffer 拷贝进去，需要先转成：

```cpp
VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL
```

示例创建 subresource range，覆盖所有 mip level：

```cpp
VkImageSubresourceRange subresourceRange = {};
subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
subresourceRange.baseMipLevel = 0;
subresourceRange.levelCount = texture.mipLevels;
subresourceRange.layerCount = 1;
```

然后设置 image memory barrier：

```cpp
VkImageMemoryBarrier imageMemoryBarrier =
    vks::initializers::imageMemoryBarrier();
imageMemoryBarrier.image = texture.image;
imageMemoryBarrier.subresourceRange = subresourceRange;
imageMemoryBarrier.srcAccessMask = 0;
imageMemoryBarrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
imageMemoryBarrier.oldLayout = VK_IMAGE_LAYOUT_UNDEFINED;
imageMemoryBarrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
```

通过 `vkCmdPipelineBarrier()` 提交 layout transition：

```cpp
vkCmdPipelineBarrier(
    copyCmd,
    VK_PIPELINE_STAGE_HOST_BIT,
    VK_PIPELINE_STAGE_TRANSFER_BIT,
    0,
    0, nullptr,
    0, nullptr,
    1, &imageMemoryBarrier);
```

这一步保证后续 transfer 写入 image 时，image 已经处于合法 layout。

## 11. vkCmdCopyBufferToImage：把 mip chain 拷贝到 VkImage

layout 转成 transfer destination 后，就可以执行 buffer 到 image 的拷贝：

```cpp
vkCmdCopyBufferToImage(
    copyCmd,
    stagingBuffer,
    texture.image,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    static_cast<uint32_t>(bufferCopyRegions.size()),
    bufferCopyRegions.data());
```

这里一次调用传入了所有 mip level 的 `VkBufferImageCopy`。

这一步完成后，image 中已经有纹理数据，但它还不能被 fragment shader 采样。因为当前 layout 仍然是：

```cpp
VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL
```

还需要转到 shader read layout。

## 12. Image Layout Transition：TRANSFER_DST 到 SHADER_READ_ONLY

拷贝完成后，示例复用同一个 barrier，修改访问 mask 和 layout：

```cpp
imageMemoryBarrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
imageMemoryBarrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;
imageMemoryBarrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
imageMemoryBarrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
```

然后插入第二个 pipeline barrier：

```cpp
vkCmdPipelineBarrier(
    copyCmd,
    VK_PIPELINE_STAGE_TRANSFER_BIT,
    VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT,
    0,
    0, nullptr,
    0, nullptr,
    1, &imageMemoryBarrier);
```

这一步建立了两个保证：

```text
TRANSFER 写入完成后
    fragment shader 才能读取。

image layout 从 TRANSFER_DST_OPTIMAL
    转成 SHADER_READ_ONLY_OPTIMAL。
```

示例把当前 layout 存下来：

```cpp
texture.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
```

后面写 descriptor 时会用这个 layout。

## 13. 提交一次性拷贝命令并释放 staging 资源

上传命令录制在一个临时 command buffer 中：

```cpp
VkCommandBuffer copyCmd =
    vulkanDevice->createCommandBuffer(VK_COMMAND_BUFFER_LEVEL_PRIMARY, true);
```

录制完成后调用：

```cpp
vulkanDevice->flushCommandBuffer(copyCmd, queue, true);
```

这里的 `true` 表示等待拷贝命令执行完成。完成后 staging buffer 和 staging memory 就不再需要：

```cpp
vkFreeMemory(device, stagingMemory, nullptr);
vkDestroyBuffer(device, stagingBuffer, nullptr);
```

这是一种典型资源流转：

```text
staging buffer
    只用于上传阶段，可以释放。

device local image
    保存最终纹理，渲染期间持续存在。
```

## 14. Linear Tiling 分支为什么不推荐

代码中还有一个 `else` 分支，用 linear tiled image 直接映射内存：

```cpp
imageCreateInfo.tiling = VK_IMAGE_TILING_LINEAR;
imageCreateInfo.usage = VK_IMAGE_USAGE_SAMPLED_BIT;
imageCreateInfo.initialLayout = VK_IMAGE_LAYOUT_PREINITIALIZED;
```

然后直接 map image memory：

```cpp
vkMapMemory(device, mappableMemory, 0, memReqs.size, 0, &data);
memcpy(data, ktxTextureData, memReqs.size);
vkUnmapMemory(device, mappableMemory);
```

这个路径更直观，但限制很多：

- linear tiling 支持的格式和功能有限。
- 通常不支持完整 mipmap 使用。
- GPU 采样性能不如 optimal tiling。
- 很多真实设备上不适合作为最终采样纹理。

所以示例保留它主要是为了学习 Vulkan image tiling 的差异，实际渲染应优先使用 staging + optimal image。

## 15. 创建 Sampler：描述如何采样纹理

`VkImage` 保存图像数据，`VkSampler` 描述采样方式：

```cpp
VkSamplerCreateInfo sampler = vks::initializers::samplerCreateInfo();
sampler.magFilter = VK_FILTER_LINEAR;
sampler.minFilter = VK_FILTER_LINEAR;
sampler.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
sampler.addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT;
sampler.addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT;
sampler.addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT;
sampler.mipLodBias = 0.0f;
sampler.compareOp = VK_COMPARE_OP_NEVER;
sampler.minLod = 0.0f;
sampler.maxLod = (useStaging) ? (float)texture.mipLevels : 0.0f;
```

这里设置了：

- 放大过滤：linear。
- 缩小过滤：linear。
- mipmap 过滤：linear。
- UVW 寻址模式：repeat。
- 最小 LOD：0。
- 最大 LOD：纹理 mip level 数量。

如果设备支持各向异性过滤，就启用最大各向异性：

```cpp
sampler.maxAnisotropy =
    vulkanDevice->properties.limits.maxSamplerAnisotropy;
sampler.anisotropyEnable = VK_TRUE;
```

最后创建 sampler：

```cpp
VK_CHECK_RESULT(vkCreateSampler(device, &sampler, nullptr, &texture.sampler));
```

同一个 image 可以搭配不同 sampler。例如一个 sampler 使用 repeat，一个使用 clamp；一个使用 linear，一个使用 nearest。Vulkan 把 image 数据和采样状态明确拆开。

## 16. 创建 Image View：描述 shader 看到哪部分 Image

shader 不能直接访问 `VkImage`，需要通过 `VkImageView`：

```cpp
VkImageViewCreateInfo view = vks::initializers::imageViewCreateInfo();
view.viewType = VK_IMAGE_VIEW_TYPE_2D;
view.format = format;
view.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
view.subresourceRange.baseMipLevel = 0;
view.subresourceRange.baseArrayLayer = 0;
view.subresourceRange.layerCount = 1;
view.subresourceRange.levelCount = (useStaging) ? texture.mipLevels : 1;
view.image = texture.image;
VK_CHECK_RESULT(vkCreateImageView(device, &view, nullptr, &texture.view));
```

image view 主要定义：

```text
viewType
    这是 2D、3D、cube 还是 array view。

format
    shader 看到的格式。

subresourceRange
    shader 可以访问哪些 mip level 和 array layer。
```

本示例的 view 覆盖所有 mip level 和一个 array layer。

## 17. 四边形几何：position、uv、normal

示例不是加载模型，而是在代码中生成一个四边形：

```cpp
struct Vertex {
    float pos[3];
    float uv[2];
    float normal[3];
};
```

四个顶点组成一个 UV 映射平面：

```cpp
std::vector<Vertex> vertices =
{
    { {  1.0f,  1.0f, 0.0f }, { 1.0f, 1.0f },{ 0.0f, 0.0f, 1.0f } },
    { { -1.0f,  1.0f, 0.0f }, { 0.0f, 1.0f },{ 0.0f, 0.0f, 1.0f } },
    { { -1.0f, -1.0f, 0.0f }, { 0.0f, 0.0f },{ 0.0f, 0.0f, 1.0f } },
    { {  1.0f, -1.0f, 0.0f }, { 1.0f, 0.0f },{ 0.0f, 0.0f, 1.0f } }
};
```

索引把四边形拆成两个三角形：

```cpp
std::vector<uint32_t> indices = { 0, 1, 2, 2, 3, 0 };
indexCount = static_cast<uint32_t>(indices.size());
```

这里的 normal 用于 fragment shader 做简单光照，而 UV 用于纹理采样。

## 18. Vertex / Index Buffer 也使用 staging 上传

`generateQuad()` 对 vertex buffer 和 index buffer 也使用 staging 上传。

先创建 host visible source buffer：

```cpp
vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_TRANSFER_SRC_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
    VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
    &stagingBuffers.vertices,
    vertices.size() * sizeof(Vertex),
    vertices.data());
```

再创建 device local destination buffer：

```cpp
vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_VERTEX_BUFFER_BIT |
    VK_BUFFER_USAGE_TRANSFER_DST_BIT,
    VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
    &vertexBuffer,
    vertices.size() * sizeof(Vertex));
```

然后执行 buffer copy：

```cpp
vulkanDevice->copyBuffer(&stagingBuffers.vertices, &vertexBuffer, queue);
vulkanDevice->copyBuffer(&stagingBuffers.indices, &indexBuffer, queue);
```

最后释放 staging buffer：

```cpp
stagingBuffers.vertices.destroy();
stagingBuffers.indices.destroy();
```

这和纹理上传的思想一致：

```text
CPU 写 staging resource
    -> GPU copy 到 device local resource
    -> 渲染时只使用 device local resource
```

## 19. Uniform Buffer：矩阵、相机位置和 LOD Bias

uniform buffer 每个并发帧一份：

```cpp
for (auto& buffer : uniformBuffers) {
    vulkanDevice->createBuffer(
        VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
        VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
        VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
        &buffer,
        sizeof(UniformData),
        &uniformData);
    buffer.map();
}
```

每帧更新：

```cpp
uniformData.projection = camera.matrices.perspective;
uniformData.modelView = camera.matrices.view;
uniformData.viewPos = camera.viewPos;
memcpy(uniformBuffers[currentBuffer].mapped, &uniformData, sizeof(uniformData));
```

`lodBias` 不在这里直接修改，而是由 UI overlay 控制：

```cpp
overlay->sliderFloat(
    "LOD bias",
    &uniformData.lodBias,
    0.0f,
    (float)texture.mipLevels);
```

下一帧 `updateUniformBuffers()` 会把新的 `lodBias` 写入当前帧 UBO。

## 20. Descriptor Set Layout：binding 0 是 UBO，binding 1 是纹理

descriptor pool 包含两种 descriptor：

```cpp
std::vector<VkDescriptorPoolSize> poolSizes = {
    vks::initializers::descriptorPoolSize(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        maxConcurrentFrames),
    vks::initializers::descriptorPoolSize(
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        maxConcurrentFrames)
};
```

descriptor set layout 声明两个 binding：

```cpp
std::vector<VkDescriptorSetLayoutBinding> setLayoutBindings = {
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        VK_SHADER_STAGE_VERTEX_BIT,
        0),
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        VK_SHADER_STAGE_FRAGMENT_BIT,
        1)
};
```

对应 shader 中的：

```glsl
layout (binding = 0) uniform UBO
{
    mat4 projection;
    mat4 model;
    vec4 viewPos;
    float lodBias;
} ubo;

layout (binding = 1) uniform sampler2D samplerColor;
```

binding 0 给 vertex shader 使用，binding 1 给 fragment shader 使用。

## 21. VkDescriptorImageInfo：把 Image View、Sampler 和 Layout 组合起来

写入 combined image sampler descriptor 前，需要准备 `VkDescriptorImageInfo`：

```cpp
VkDescriptorImageInfo textureDescriptor{};
textureDescriptor.imageView = texture.view;
textureDescriptor.sampler = texture.sampler;
textureDescriptor.imageLayout = texture.imageLayout;
```

这三个字段分别代表：

```text
imageView
    shader 看到的 image 子资源范围。

sampler
    shader 采样时使用的过滤和寻址状态。

imageLayout
    image 当前用于采样的 layout，本示例是 SHADER_READ_ONLY_OPTIMAL。
```

然后每帧 descriptor set 都写入 UBO 和同一张纹理：

```cpp
std::vector<VkWriteDescriptorSet> writeDescriptorSets = {
    vks::initializers::writeDescriptorSet(
        descriptorSets[i],
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        0,
        &uniformBuffers[i].descriptor),

    vks::initializers::writeDescriptorSet(
        descriptorSets[i],
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        1,
        &textureDescriptor)
};
```

这里 texture image 只有一份，但 descriptor set 有多份。原因是这个示例按 frame-in-flight 分配 descriptor set，每个 descriptor set 绑定不同 UBO，同时复用同一个 static texture。

## 22. Pipeline Layout 和 Graphics Pipeline

pipeline layout 只接入一个 descriptor set layout：

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutCreateInfo =
    vks::initializers::pipelineLayoutCreateInfo(&descriptorSetLayout, 1);

vkCreatePipelineLayout(
    device,
    &pipelineLayoutCreateInfo,
    nullptr,
    &pipelineLayout);
```

graphics pipeline 使用两个 shader：

```cpp
shaderStages[0] = loadShader(
    getShadersPath() + "texture/texture.vert.spv",
    VK_SHADER_STAGE_VERTEX_BIT);

shaderStages[1] = loadShader(
    getShadersPath() + "texture/texture.frag.spv",
    VK_SHADER_STAGE_FRAGMENT_BIT);
```

固定函数状态比较常规：

- triangle list。
- fill polygon mode。
- 不剔除面。
- counter-clockwise front face。
- 深度测试和深度写入开启。
- viewport 和 scissor 是动态状态。
- 不启用 color blending。

最终调用：

```cpp
vkCreateGraphicsPipelines(
    device,
    pipelineCache,
    1,
    &pipelineCreateInfo,
    nullptr,
    &pipeline);
```

## 23. Vertex Input：C++ Vertex 如何映射到 Shader location

顶点 buffer 只有一个 binding：

```cpp
std::vector<VkVertexInputBindingDescription> vertexInputBindings = {
    vks::initializers::vertexInputBindingDescription(
        0,
        sizeof(Vertex),
        VK_VERTEX_INPUT_RATE_VERTEX)
};
```

三个 attribute 分别映射到 shader location：

```cpp
std::vector<VkVertexInputAttributeDescription> vertexInputAttributes = {
    vks::initializers::vertexInputAttributeDescription(
        0, 0, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, pos)),
    vks::initializers::vertexInputAttributeDescription(
        0, 1, VK_FORMAT_R32G32_SFLOAT, offsetof(Vertex, uv)),
    vks::initializers::vertexInputAttributeDescription(
        0, 2, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, normal)),
};
```

对应 vertex shader：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec2 inUV;
layout (location = 2) in vec3 inNormal;
```

这里有两个编号概念：

```text
binding
    顶点 buffer 绑定槽位。

location
    shader 输入变量位置。
```

本示例所有顶点属性都来自 binding 0，但分别进入 location 0、1、2。

## 24. Vertex Shader：传递 UV、LOD Bias 和光照向量

`texture.vert` 首先把 UV 和 LOD bias 传给 fragment shader：

```glsl
outUV = inUV;
outLodBias = ubo.lodBias;
```

然后计算裁剪空间位置：

```glsl
gl_Position = ubo.projection * ubo.model * vec4(inPos.xyz, 1.0);
```

它还计算了 normal、light vector 和 view vector：

```glsl
outNormal = mat3(inverse(transpose(ubo.model))) * inNormal;
vec3 lightPos = vec3(0.0);
vec3 lPos = mat3(ubo.model) * lightPos.xyz;
outLightVec = lPos - pos.xyz;
outViewVec = ubo.viewPos.xyz - pos.xyz;
```

所以 fragment shader 不只是把纹理颜色直接输出，还会做一个简单的 diffuse + specular 光照。

## 25. Fragment Shader：带 LOD Bias 的 texture 采样

`texture.frag` 中的纹理 binding 是：

```glsl
layout (binding = 1) uniform sampler2D samplerColor;
```

核心采样代码是：

```glsl
vec4 color = texture(samplerColor, inUV, inLodBias);
```

第三个参数 `inLodBias` 会影响 mip level 选择。UI slider 改变 `uniformData.lodBias`，vertex shader 把它传给 fragment shader，fragment shader 用它采样纹理。

采样后，shader 做简单光照：

```glsl
vec3 diffuse = max(dot(N, L), 0.0) * vec3(1.0);
float specular = pow(max(dot(R, V), 0.0), 16.0) * color.a;

outFragColor = vec4(diffuse * color.rgb + specular, 1.0);
```

这里 `color.rgb` 是纹理颜色，`color.a` 用于控制 specular 强度。

## 26. Command Buffer：绑定 Descriptor、Pipeline、Vertex/Index Buffer 并绘制

`buildCommandBuffer()` 中先开始 render pass，设置 viewport 和 scissor：

```cpp
VkViewport viewport =
    vks::initializers::viewport((float)width, (float)height, 0.0f, 1.0f);
vkCmdSetViewport(cmdBuffer, 0, 1, &viewport);

VkRect2D scissor = vks::initializers::rect2D(width, height, 0, 0);
vkCmdSetScissor(cmdBuffer, 0, 1, &scissor);
```

然后绑定 descriptor set：

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

这一步让 shader 能访问当前帧 UBO 和 texture sampler。

接着绑定 pipeline：

```cpp
vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
```

再绑定 vertex buffer 和 index buffer：

```cpp
VkDeviceSize offsets[1] = { 0 };
vkCmdBindVertexBuffers(cmdBuffer, 0, 1, &vertexBuffer.buffer, offsets);
vkCmdBindIndexBuffer(cmdBuffer, indexBuffer.buffer, 0, VK_INDEX_TYPE_UINT32);
```

最后发出 indexed draw：

```cpp
vkCmdDrawIndexed(cmdBuffer, indexCount, 1, 0, 0, 0);
```

`indexCount` 是 6，也就是两个三角形。

## 27. UI Overlay：LOD Bias 如何实时影响采样

示例重写了 `OnUpdateUIOverlay()`：

```cpp
virtual void OnUpdateUIOverlay(vks::UIOverlay *overlay)
{
    if (overlay->header("Settings")) {
        overlay->sliderFloat(
            "LOD bias",
            &uniformData.lodBias,
            0.0f,
            (float)texture.mipLevels);
    }
}
```

这个 slider 修改的是 CPU 侧的 `uniformData.lodBias`。下一帧：

```cpp
memcpy(uniformBuffers[currentBuffer].mapped, &uniformData, sizeof(uniformData));
```

新的 bias 被写入 UBO，然后 vertex shader 输出给 fragment shader。

数据流是：

```text
UI slider
    -> uniformData.lodBias
    -> current frame uniform buffer
    -> vertex shader outLodBias
    -> fragment shader texture(..., bias)
    -> mip level 选择改变
```

## 28. 资源生命周期

析构函数释放了本示例创建的 Vulkan 对象：

```cpp
destroyTextureImage(texture);
vkDestroyPipeline(device, pipeline, nullptr);
vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);
vertexBuffer.destroy();
indexBuffer.destroy();
for (auto& buffer : uniformBuffers) {
    buffer.destroy();
}
```

纹理释放函数是：

```cpp
void destroyTextureImage(Texture texture)
{
    vkDestroyImageView(device, texture.view, nullptr);
    vkDestroyImage(device, texture.image, nullptr);
    vkDestroySampler(device, texture.sampler, nullptr);
    vkFreeMemory(device, texture.deviceMemory, nullptr);
}
```

从依赖关系看，通常可以理解为：

```text
descriptor 引用 image view 和 sampler
image view 引用 image
image 绑定 device memory
```

程序退出时统一释放这些对象。descriptor pool 由基类管理，所以这里没有显式销毁。

## 29. 常见错误和调试方向

第一，image layout 和 descriptor 中声明的 layout 必须匹配。

本示例最终保存：

```cpp
texture.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
```

descriptor 写入：

```cpp
textureDescriptor.imageLayout = texture.imageLayout;
```

如果 image 实际 layout 没有 transition 到 shader read，fragment shader 采样就是错误用法。

第二，`VkImageCreateInfo::usage` 必须包含实际用途。

本示例需要先 copy，再 sample：

```cpp
VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT
```

缺任何一个都会导致后续操作不合法。

第三，mip level 数量要在 image、image view、sampler 中保持一致。

相关位置包括：

```cpp
imageCreateInfo.mipLevels = texture.mipLevels;
view.subresourceRange.levelCount = texture.mipLevels;
sampler.maxLod = (float)texture.mipLevels;
```

第四，descriptor binding 必须和 shader 一致。

C++ 中 binding 1 是 combined image sampler：

```cpp
VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, 1
```

shader 中也必须是 binding 1：

```glsl
layout (binding = 1) uniform sampler2D samplerColor;
```

第五，修改 GLSL 后要重新编译 SPIR-V。示例运行时加载的是：

```text
shaders/glsl/texture/texture.vert.spv
shaders/glsl/texture/texture.frag.spv
```

不是直接加载 `.vert` 和 `.frag` 文本。

## 30. 从这个示例提炼出的工程原则

第一，静态纹理资源适合放在 device local image 中。

CPU 不需要每帧修改纹理，所以用 staging 上传一次，然后释放 staging 资源。

第二，image layout transition 是 Vulkan 纹理使用的核心步骤。

同一张 image 在不同阶段需要不同 layout：

```text
UNDEFINED
    初始状态，不保留内容。

TRANSFER_DST_OPTIMAL
    接收 staging buffer copy。

SHADER_READ_ONLY_OPTIMAL
    被 fragment shader 采样。
```

第三，`VkImage`、`VkImageView`、`VkSampler` 的职责要分清。

```text
VkImage
    数据本体。

VkImageView
    shader 看到的数据视图。

VkSampler
    采样规则。
```

第四，每帧 descriptor set 可以复用静态纹理。

本示例每帧 descriptor set 不同，是因为 UBO 每帧不同。纹理本身是静态资源，所以所有 descriptor set 写入同一个 image view 和 sampler。

第五，mipmap 可以离线存储，也可以运行时生成。

这个示例使用 KTX 中已有的 mip chain。如果想学习运行时生成 mipmap，可以看 `examples/texturemipmapgen/texturemipmapgen.cpp`。

## 31. 一个简化版 mental model

可以把 `texture.cpp` 的纹理路径理解成下面这条链：

```text
metalplate01_rgba.ktx
    -> ktxTexture
    -> staging VkBuffer
    -> VkImage(VK_IMAGE_TILING_OPTIMAL, DEVICE_LOCAL)
    -> layout: TRANSFER_DST_OPTIMAL
    -> vkCmdCopyBufferToImage
    -> layout: SHADER_READ_ONLY_OPTIMAL
    -> VkImageView
    -> VkSampler
    -> VkDescriptorImageInfo
    -> VkDescriptorSet binding 1
    -> sampler2D samplerColor
    -> texture(samplerColor, inUV, inLodBias)
```

理解这条链路后，Vulkan 中最基础的 2D 纹理采样流程就清楚了。

## 32. 总结

`texture.cpp` 展示的是 Vulkan 纹理使用的完整基础路径：加载 KTX 数据，使用 staging buffer 上传到 optimal tiled device local image，执行必要的 image layout transition，创建 sampler 和 image view，再通过 combined image sampler descriptor 交给 fragment shader。

这个示例中：

- `loadTexture()` 负责纹理加载、上传、sampler 和 image view 创建。
- `generateQuad()` 创建带 UV 的四边形。
- `setupDescriptors()` 把 UBO 和纹理绑定到 shader。
- `preparePipelines()` 创建使用 texture shader 的 graphics pipeline。
- `buildCommandBuffer()` 绑定 descriptor、pipeline、vertex/index buffer 并绘制。
- UI slider 修改 `lodBias`，实时影响 mipmap 采样。

如果后续要实现材质系统、PBR 纹理、贴图数组、cubemap 或 render-to-texture，这个示例中的 `VkImage`、`VkImageView`、`VkSampler`、descriptor 和 layout transition 是必须先掌握的基础。
