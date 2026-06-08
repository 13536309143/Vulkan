# Vulkan 示例解析：texturecubemap.cpp 如何加载 Cubemap 并实现 Skybox 和反射

本文分析的是 `examples/texturecubemap/texturecubemap.cpp`。这个示例展示了 Vulkan 中如何加载和使用 **cubemap texture**：把一张包含 6 个面的 KTX 纹理上传到 GPU，创建 cube-compatible `VkImage` 和 `VK_IMAGE_VIEW_TYPE_CUBE` 视图，再在 shader 中通过 `samplerCube` 采样。

这个示例最终绘制两类内容：

- 一个作为背景的 skybox。
- 一个可切换模型的反射物体。

两者都使用同一张 cubemap：

- skybox shader 直接用方向向量采样 cubemap。
- reflection shader 根据视线方向和法线计算反射方向，再用反射方向采样 cubemap。
- UI 可以切换显示对象、开关 skybox，并调整 `LOD bias`。

这篇文章会按代码执行顺序拆解：cubemap 的 Vulkan 表示、KTX 六个面的上传、6 个 array layer、mipmap copy region、image layout transition、cube image view、sampler、descriptor、两条 pipeline、skybox shader、reflection shader 和 command buffer 绘制流程。

## 1. Cubemap 解决什么问题

普通 2D 纹理使用二维 UV 坐标采样：

```glsl
texture(sampler2D, vec2(u, v))
```

Cubemap 使用三维方向向量采样：

```glsl
texture(samplerCube, vec3(x, y, z))
```

可以把 cubemap 理解成包围观察者的六张 2D 纹理：

```text
+X / -X
+Y / -Y
+Z / -Z
```

GPU 根据传入的方向向量自动选择对应的 cube face，并计算该 face 内部的 2D 采样坐标。

Cubemap 常见用途包括：

- skybox / environment background。
- 近似环境反射。
- image based lighting 的环境贴图。
- 点光源阴影 cubemap。

`texturecubemap.cpp` 展示的是最基础的两种用途：把 cubemap 画成背景，并把它作为反射贴到一个物体上。

## 2. 程序整体结构

示例继承自框架基类：

```cpp
class VulkanExample : public VulkanExampleBase
```

核心成员如下。

首先是开关和 cubemap 纹理：

```cpp
bool displaySkybox = true;

vks::Texture cubeMap;
```

`cubeMap` 是框架里的纹理结构，包含 image、memory、sampler、image view、尺寸、mip level 和当前 layout 等信息。

然后是模型：

```cpp
struct Models {
    vkglTF::Model skybox;
    std::vector<vkglTF::Model> objects;
    int32_t objectIndex = 0;
} models;
```

`skybox` 使用一个 cube 模型。`objects` 中放了多个可切换的反射物体：

```cpp
std::vector<std::string> filenames = {
    "sphere.gltf",
    "teapot.gltf",
    "torusknot.gltf",
    "venus.gltf"
};
```

uniform 数据如下：

```cpp
struct UniformData {
    glm::mat4 projection;
    glm::mat4 modelView;
    glm::mat4 inverseModelview;
    float lodBias = 0.0f;
} uniformData;

std::array<vks::Buffer, maxConcurrentFrames> uniformBuffers;
```

`modelView` 给 skybox 和反射对象使用，`inverseModelview` 给反射 shader 把反射向量转换回 cubemap 采样空间，`lodBias` 用于影响 cubemap mip level 选择。

最后是两条 pipeline：

```cpp
struct {
    VkPipeline skybox{ VK_NULL_HANDLE };
    VkPipeline reflect{ VK_NULL_HANDLE };
} pipelines;
```

这两条 pipeline 共用 descriptor layout 和 pipeline layout，但使用不同 shader，也有不同的 depth/cull 状态。

## 3. 程序运行流程

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

执行顺序可以拆成：

1. `VulkanExampleBase::prepare()`：创建 swapchain、render pass、framebuffer 等基础对象。
2. `loadAssets()`：加载 skybox cube、可反射物体和 cubemap KTX 纹理。
3. `prepareUniformBuffers()`：为每个并发帧创建 uniform buffer。
4. `setupDescriptors()`：把 UBO 和 cubemap sampler 写入 descriptor set。
5. `preparePipelines()`：创建 skybox pipeline 和 reflect pipeline。

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

每帧更新的是相机矩阵、inverse matrix 和 `lodBias`。cubemap 本身只在初始化时上传一次。

## 4. 设备特性：可选启用各向异性过滤

和普通 2D 纹理示例一样，这个示例也尝试启用 sampler anisotropy：

```cpp
virtual void getEnabledFeatures()
{
    if (deviceFeatures.samplerAnisotropy) {
        enabledFeatures.samplerAnisotropy = VK_TRUE;
    }
}
```

创建 sampler 时再次检查：

```cpp
sampler.maxAnisotropy = 1.0f;
if (vulkanDevice->features.samplerAnisotropy)
{
    sampler.maxAnisotropy =
        vulkanDevice->properties.limits.maxSamplerAnisotropy;
    sampler.anisotropyEnable = VK_TRUE;
}
```

Vulkan 中可选 feature 的使用路径通常都是：

```text
查询 physical device feature
    -> 创建 logical device 时启用
    -> 创建具体对象时使用
```

## 5. 加载资源：skybox、反射物体和 cubemap

`loadAssets()` 先设置 glTF 加载标志：

```cpp
uint32_t glTFLoadingFlags =
    vkglTF::FileLoadingFlags::PreTransformVertices |
    vkglTF::FileLoadingFlags::FlipY;
```

然后加载 skybox cube：

```cpp
models.skybox.loadFromFile(
    getAssetPath() + "models/cube.gltf",
    vulkanDevice,
    queue,
    glTFLoadingFlags);
```

反射物体有四个：

```cpp
std::vector<std::string> filenames = {
    "sphere.gltf",
    "teapot.gltf",
    "torusknot.gltf",
    "venus.gltf"
};

objectNames = { "Sphere", "Teapot", "Torusknot", "Venus" };
```

最后加载 cubemap：

```cpp
loadCubemap(
    getAssetPath() + "textures/cubemap_yokohama_rgba.ktx",
    VK_FORMAT_R8G8B8A8_UNORM);
```

这个 KTX 文件包含 6 个 face，并且包含 mip chain。

## 6. Vulkan 中 cubemap 的表示方式

Vulkan 没有单独的 `VkCubeMap` 对象。Cubemap 本质上是一个特殊的 2D image array：

```text
imageType = VK_IMAGE_TYPE_2D
arrayLayers = 6
flags = VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT
viewType = VK_IMAGE_VIEW_TYPE_CUBE
```

代码中创建 image 时设置：

```cpp
imageCreateInfo.imageType = VK_IMAGE_TYPE_2D;
imageCreateInfo.arrayLayers = 6;
imageCreateInfo.flags = VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT;
```

创建 image view 时设置：

```cpp
view.viewType = VK_IMAGE_VIEW_TYPE_CUBE;
view.subresourceRange.layerCount = 6;
```

也就是说：

```text
VkImage
    是一个 6 层的 2D image array。

VkImageView
    把这 6 层解释成一个 cubemap。

samplerCube
    在 shader 里用方向向量采样这个 cube view。
```

## 7. KTX 数据读取

非 Android 平台直接从文件创建 KTX texture：

```cpp
result = ktxTexture_CreateFromNamedFile(
    filename.c_str(),
    KTX_TEXTURE_CREATE_LOAD_IMAGE_DATA_BIT,
    &ktxTexture);
```

Android 平台则通过 asset manager 读取 APK 内的资源，再从内存创建 KTX texture。

读取成功后，示例提取 cubemap 的基础信息：

```cpp
cubeMap.width = ktxTexture->baseWidth;
cubeMap.height = ktxTexture->baseHeight;
cubeMap.mipLevels = ktxTexture->numLevels;
ktx_uint8_t *ktxTextureData = ktxTexture_GetData(ktxTexture);
ktx_size_t ktxTextureSize = ktxTexture_GetSize(ktxTexture);
```

注意这里的 `width` 和 `height` 是单个 cube face 的尺寸，不是 6 个 face 拼接后的总尺寸。

## 8. Staging Buffer：上传 cubemap 原始数据

和普通 2D 纹理一样，cubemap 也先把 KTX 数据复制到 host visible staging buffer：

```cpp
VkBufferCreateInfo bufferCreateInfo = vks::initializers::bufferCreateInfo();
bufferCreateInfo.size = ktxTextureSize;
bufferCreateInfo.usage = VK_BUFFER_USAGE_TRANSFER_SRC_BIT;
bufferCreateInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

vkCreateBuffer(device, &bufferCreateInfo, nullptr, &stagingBuffer);
```

然后分配 host visible、host coherent 内存：

```cpp
memAllocInfo.memoryTypeIndex =
    vulkanDevice->getMemoryType(
        memReqs.memoryTypeBits,
        VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
        VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
```

把 KTX 数据复制进去：

```cpp
uint8_t *data;
vkMapMemory(device, stagingMemory, 0, memReqs.size, 0, (void **)&data);
memcpy(data, ktxTextureData, ktxTextureSize);
vkUnmapMemory(device, stagingMemory);
```

staging buffer 只是上传中转资源。真正渲染时使用的是后面创建的 device local `VkImage`。

## 9. 创建 Cube-Compatible Optimal Image

目标 image 的创建参数如下：

```cpp
VkImageCreateInfo imageCreateInfo = vks::initializers::imageCreateInfo();
imageCreateInfo.imageType = VK_IMAGE_TYPE_2D;
imageCreateInfo.format = format;
imageCreateInfo.mipLevels = cubeMap.mipLevels;
imageCreateInfo.samples = VK_SAMPLE_COUNT_1_BIT;
imageCreateInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
imageCreateInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
imageCreateInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
imageCreateInfo.extent = { cubeMap.width, cubeMap.height, 1 };
imageCreateInfo.usage =
    VK_IMAGE_USAGE_TRANSFER_DST_BIT |
    VK_IMAGE_USAGE_SAMPLED_BIT;
imageCreateInfo.arrayLayers = 6;
imageCreateInfo.flags = VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT;
```

几个关键点：

- `arrayLayers = 6`：cubemap 的 6 个面在 Vulkan 中对应 6 个 array layer。
- `VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT`：告诉 Vulkan 这个 image 可以创建 cube view。
- `VK_IMAGE_USAGE_TRANSFER_DST_BIT`：允许从 staging buffer 拷贝数据。
- `VK_IMAGE_USAGE_SAMPLED_BIT`：允许 shader 采样。
- `VK_IMAGE_TILING_OPTIMAL`：使用 GPU 友好的 image layout。

然后分配 device local memory：

```cpp
vkGetImageMemoryRequirements(device, cubeMap.image, &memReqs);
memAllocInfo.allocationSize = memReqs.size;
memAllocInfo.memoryTypeIndex =
    vulkanDevice->getMemoryType(
        memReqs.memoryTypeBits,
        VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);
vkAllocateMemory(device, &memAllocInfo, nullptr, &cubeMap.deviceMemory);
vkBindImageMemory(device, cubeMap.image, cubeMap.deviceMemory, 0);
```

## 10. VkBufferImageCopy：6 个 face x 多个 mip level

普通 2D 纹理只需要遍历 mip level。Cubemap 还要遍历 6 个 face：

```cpp
for (uint32_t face = 0; face < 6; face++)
{
    for (uint32_t level = 0; level < cubeMap.mipLevels; level++)
    {
        ...
    }
}
```

每个 face 的每个 mip level 都对应一个 `VkBufferImageCopy`：

```cpp
ktx_size_t offset;
KTX_error_code ret =
    ktxTexture_GetImageOffset(ktxTexture, level, 0, face, &offset);
assert(ret == KTX_SUCCESS);

VkBufferImageCopy bufferCopyRegion = {};
bufferCopyRegion.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
bufferCopyRegion.imageSubresource.mipLevel = level;
bufferCopyRegion.imageSubresource.baseArrayLayer = face;
bufferCopyRegion.imageSubresource.layerCount = 1;
bufferCopyRegion.imageExtent.width = ktxTexture->baseWidth >> level;
bufferCopyRegion.imageExtent.height = ktxTexture->baseHeight >> level;
bufferCopyRegion.imageExtent.depth = 1;
bufferCopyRegion.bufferOffset = offset;
```

关键字段是：

```text
imageSubresource.baseArrayLayer
    当前写入哪一个 cube face。

imageSubresource.mipLevel
    当前写入该 face 的哪个 mip level。

bufferOffset
    当前 face + mip level 在 staging buffer 中的起始偏移。
```

最终 copy region 数量是：

```text
6 * cubeMap.mipLevels
```

## 11. Image Layout Transition：覆盖 6 个 face

layout transition 使用的 subresource range 覆盖所有 mip level 和 6 个 layer：

```cpp
VkImageSubresourceRange subresourceRange = {};
subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
subresourceRange.baseMipLevel = 0;
subresourceRange.levelCount = cubeMap.mipLevels;
subresourceRange.layerCount = 6;
```

先从 undefined 转到 transfer destination：

```cpp
vks::tools::setImageLayout(
    copyCmd,
    cubeMap.image,
    VK_IMAGE_LAYOUT_UNDEFINED,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    subresourceRange);
```

然后拷贝所有 face 和 mip level：

```cpp
vkCmdCopyBufferToImage(
    copyCmd,
    stagingBuffer,
    cubeMap.image,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    static_cast<uint32_t>(bufferCopyRegions.size()),
    bufferCopyRegions.data());
```

拷贝完成后转到 shader read layout：

```cpp
cubeMap.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;

vks::tools::setImageLayout(
    copyCmd,
    cubeMap.image,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    cubeMap.imageLayout,
    subresourceRange);
```

这一步让 cubemap 可以在 fragment shader 中被 `samplerCube` 采样。

## 12. 提交上传命令并释放 staging 资源

上传命令使用一次性 command buffer：

```cpp
VkCommandBuffer copyCmd =
    vulkanDevice->createCommandBuffer(VK_COMMAND_BUFFER_LEVEL_PRIMARY, true);
```

录制完成后提交并等待：

```cpp
vulkanDevice->flushCommandBuffer(copyCmd, queue, true);
```

因为已经等待上传完成，staging 资源可以释放：

```cpp
vkFreeMemory(device, stagingMemory, nullptr);
vkDestroyBuffer(device, stagingBuffer, nullptr);
ktxTexture_Destroy(ktxTexture);
```

渲染期间只保留 device local `cubeMap.image`、`cubeMap.deviceMemory`、`cubeMap.view` 和 `cubeMap.sampler`。

## 13. 创建 Cubemap Sampler

sampler 描述 cubemap 的采样方式：

```cpp
VkSamplerCreateInfo sampler = vks::initializers::samplerCreateInfo();
sampler.magFilter = VK_FILTER_LINEAR;
sampler.minFilter = VK_FILTER_LINEAR;
sampler.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
sampler.addressModeU = VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE;
sampler.addressModeV = sampler.addressModeU;
sampler.addressModeW = sampler.addressModeU;
sampler.mipLodBias = 0.0f;
sampler.compareOp = VK_COMPARE_OP_NEVER;
sampler.minLod = 0.0f;
sampler.maxLod = static_cast<float>(cubeMap.mipLevels);
sampler.borderColor = VK_BORDER_COLOR_FLOAT_OPAQUE_WHITE;
```

Cubemap 通常使用 `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE`，避免 cube face 边缘采样时出现不希望的重复边界。

如果支持各向异性过滤，则启用：

```cpp
sampler.maxAnisotropy =
    vulkanDevice->properties.limits.maxSamplerAnisotropy;
sampler.anisotropyEnable = VK_TRUE;
```

最后创建 sampler：

```cpp
vkCreateSampler(device, &sampler, nullptr, &cubeMap.sampler);
```

## 14. 创建 Cube Image View

image view 是 cubemap 的关键：

```cpp
VkImageViewCreateInfo view = vks::initializers::imageViewCreateInfo();
view.viewType = VK_IMAGE_VIEW_TYPE_CUBE;
view.format = format;
view.subresourceRange = { VK_IMAGE_ASPECT_COLOR_BIT, 0, 1, 0, 1 };
view.subresourceRange.layerCount = 6;
view.subresourceRange.levelCount = cubeMap.mipLevels;
view.image = cubeMap.image;
vkCreateImageView(device, &view, nullptr, &cubeMap.view);
```

如果这里使用普通 2D view，shader 中的 `samplerCube` 就无法正确采样。Vulkan 需要：

```text
VkImageCreateInfo::flags
    包含 VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT

VkImageViewCreateInfo::viewType
    设置为 VK_IMAGE_VIEW_TYPE_CUBE

subresourceRange.layerCount
    设置为 6
```

这三者共同决定这张 image 能作为 cubemap 使用。

## 15. Descriptor Set：UBO + samplerCube

descriptor pool 包含两类 descriptor：

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

descriptor set layout 有两个 binding：

```cpp
std::vector<VkDescriptorSetLayoutBinding> setLayoutBindings = {
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT,
        0),
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        VK_SHADER_STAGE_FRAGMENT_BIT,
        1)
};
```

binding 0 是 UBO，vertex shader 和 fragment shader 都可能访问。binding 1 是 cubemap sampler，只给 fragment shader 使用。

创建 cubemap descriptor：

```cpp
VkDescriptorImageInfo textureDescriptor =
    vks::initializers::descriptorImageInfo(
        cubeMap.sampler,
        cubeMap.view,
        cubeMap.imageLayout);
```

每个并发帧一个 descriptor set：

```cpp
for (auto i = 0; i < uniformBuffers.size(); i++) {
    vkAllocateDescriptorSets(device, &allocInfo, &descriptorSets[i]);
    ...
}
```

写入 UBO 和同一张 cubemap：

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
        &textureDescriptor),
};
```

UBO 每帧不同，cubemap 是静态资源，所以每个 descriptor set 都引用同一个 `cubeMap.view` 和 `cubeMap.sampler`。

## 16. Uniform Buffer：矩阵、逆矩阵和 LOD Bias

uniform buffer 每帧一份：

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
uniformData.inverseModelview = glm::inverse(camera.matrices.view);
memcpy(uniformBuffers[currentBuffer].mapped, &uniformData, sizeof(UniformData));
```

这里有一个重要注释：

```cpp
// Both the object and skybox use the same uniform data,
// the translation part of the skybox is removed in the shader
```

skybox 和反射物体都使用同一个 UBO。skybox shader 会主动去掉 view matrix 的平移部分，让背景只随相机旋转，不随相机位置移动。

## 17. Pipeline Layout：两条 pipeline 共用同一套资源接口

pipeline layout 只包含一个 descriptor set layout：

```cpp
const VkPipelineLayoutCreateInfo pipelineLayoutCI =
    vks::initializers::pipelineLayoutCreateInfo(&descriptorSetLayout, 1);

vkCreatePipelineLayout(device, &pipelineLayoutCI, nullptr, &pipelineLayout);
```

skybox pipeline 和 reflect pipeline 都使用这个 `pipelineLayout`。原因是它们的资源接口相同：

```text
set 0 binding 0
    UBO

set 0 binding 1
    cubemap sampler
```

虽然两个 pipeline 的 shader 不同，但它们都能用这套 descriptor layout。

## 18. Graphics Pipeline：先创建 skybox，再创建 reflect

`preparePipelines()` 先准备公共的 pipeline 状态：

```cpp
VkPipelineInputAssemblyStateCreateInfo inputAssemblyState =
    vks::initializers::pipelineInputAssemblyStateCreateInfo(
        VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST,
        0,
        VK_FALSE);

VkPipelineRasterizationStateCreateInfo rasterizationState =
    vks::initializers::pipelineRasterizationStateCreateInfo(
        VK_POLYGON_MODE_FILL,
        VK_CULL_MODE_BACK_BIT,
        VK_FRONT_FACE_COUNTER_CLOCKWISE,
        0);
```

一开始 depth test 和 depth write 都关闭：

```cpp
VkPipelineDepthStencilStateCreateInfo depthStencilState =
    vks::initializers::pipelineDepthStencilStateCreateInfo(
        VK_FALSE,
        VK_FALSE,
        VK_COMPARE_OP_LESS_OR_EQUAL);
```

顶点输入使用 glTF 的 position 和 normal：

```cpp
pipelineCI.pVertexInputState =
    vkglTF::Vertex::getPipelineVertexInputState({
        vkglTF::VertexComponent::Position,
        vkglTF::VertexComponent::Normal
    });
```

创建 skybox pipeline 时使用 skybox shader：

```cpp
shaderStages[0] = loadShader(
    getShadersPath() + "texturecubemap/skybox.vert.spv",
    VK_SHADER_STAGE_VERTEX_BIT);
shaderStages[1] = loadShader(
    getShadersPath() + "texturecubemap/skybox.frag.spv",
    VK_SHADER_STAGE_FRAGMENT_BIT);
rasterizationState.cullMode = VK_CULL_MODE_FRONT_BIT;
vkCreateGraphicsPipelines(..., &pipelines.skybox);
```

skybox 使用 front-face culling，因为相机在 cube 内部，需要渲染 cube 的内侧面。

创建 reflect pipeline 时切换 shader，并开启 depth：

```cpp
shaderStages[0] = loadShader(
    getShadersPath() + "texturecubemap/reflect.vert.spv",
    VK_SHADER_STAGE_VERTEX_BIT);
shaderStages[1] = loadShader(
    getShadersPath() + "texturecubemap/reflect.frag.spv",
    VK_SHADER_STAGE_FRAGMENT_BIT);

depthStencilState.depthWriteEnable = VK_TRUE;
depthStencilState.depthTestEnable = VK_TRUE;
rasterizationState.cullMode = VK_CULL_MODE_BACK_BIT;
vkCreateGraphicsPipelines(..., &pipelines.reflect);
```

两条 pipeline 共用大部分状态，但 shader、cull mode 和 depth 状态不同。

## 19. Skybox Vertex Shader：去掉相机平移

`skybox.vert` 输入 cube 顶点位置：

```glsl
layout (location = 0) in vec3 inPos;
```

它把顶点位置直接作为 cubemap 采样方向：

```glsl
outUVW = inPos;
```

然后做坐标转换：

```glsl
outUVW.xy *= -1.0;
```

注释说明这是为了转换到 Vulkan cubemap 坐标空间。

skybox 最关键的是去掉 view matrix 的平移部分：

```glsl
mat4 viewMat = mat4(mat3(ubo.model));
gl_Position = ubo.projection * viewMat * vec4(inPos.xyz, 1.0);
```

`mat3(ubo.model)` 只保留旋转和缩放部分，再转回 `mat4`。这样 skybox 会跟随相机旋转，但不会因为相机位置变化而移动。

这正是 skybox 的常见做法：背景应该看起来无限远。

## 20. Skybox Fragment Shader：直接采样 samplerCube

`skybox.frag` 很短：

```glsl
layout (binding = 1) uniform samplerCube samplerCubeMap;

layout (location = 0) in vec3 inUVW;

layout (location = 0) out vec4 outFragColor;

void main()
{
    outFragColor = texture(samplerCubeMap, inUVW);
}
```

它接收 vertex shader 输出的方向向量 `inUVW`，直接采样 cubemap。

这里不需要手动判断采样哪个 face，`samplerCube` 会根据方向向量自动完成 face 选择和面内坐标计算。

## 21. Reflection Vertex Shader：输出位置、法线和视线方向

`reflect.vert` 输入 position 和 normal：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inNormal;
```

UBO 包含 projection、model、invModel 和 lodBias：

```glsl
layout (binding = 0) uniform UBO
{
    mat4 projection;
    mat4 model;
    mat4 invModel;
    float lodBias;
} ubo;
```

代码计算裁剪空间位置：

```glsl
gl_Position = ubo.projection * ubo.model * vec4(inPos.xyz, 1.0);
```

同时输出：

```glsl
outPos = vec3(ubo.model * vec4(inPos, 1.0));
outNormal = mat3(ubo.model) * inNormal;

vec3 lightPos = vec3(0.0f, -5.0f, 5.0f);
outLightVec = lightPos.xyz - outPos.xyz;
outViewVec = -outPos.xyz;
```

这些数据会在 fragment shader 中用于计算反射方向和简单光照。

## 22. Reflection Fragment Shader：反射方向采样 cubemap

`reflect.frag` 使用 `samplerCube`：

```glsl
layout (binding = 1) uniform samplerCube samplerColor;
```

先计算入射方向和反射方向：

```glsl
vec3 cI = normalize(inPos);
vec3 cR = reflect(cI, normalize(inNormal));
```

然后把反射方向转换回 cubemap 采样空间：

```glsl
cR = vec3(ubo.invModel * vec4(cR, 0.0));
cR.xy *= -1.0;
```

最后带 LOD bias 采样 cubemap：

```glsl
vec4 color = texture(samplerColor, cR, ubo.lodBias);
```

这个 `color` 不是来自普通 2D 材质贴图，而是来自环境 cubemap。随后 shader 把它和简单光照组合：

```glsl
vec3 ambient = vec3(0.5) * color.rgb;
vec3 diffuse = max(dot(N, L), 0.0) * vec3(1.0);
vec3 specular = pow(max(dot(R, V), 0.0), 16.0) * vec3(0.5);
outFragColor = vec4(ambient + diffuse * color.rgb + specular, 1.0);
```

这是一种简化环境反射，不是完整 PBR，但足够展示 cubemap 反射的基本机制。

## 23. Command Buffer：先画 skybox，再画反射物体

`buildCommandBuffer()` 开始 render pass 后，设置 viewport 和 scissor：

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

这个 descriptor set 同时提供当前帧 UBO 和 cubemap sampler。

如果 UI 中开启 skybox，就先绘制背景：

```cpp
if (displaySkybox)
{
    vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.skybox);
    models.skybox.draw(cmdBuffer);
}
```

然后绘制反射物体：

```cpp
vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.reflect);
models.objects[models.objectIndex].draw(cmdBuffer);
```

绘制顺序是合理的：

```text
skybox
    depth test/write 关闭，作为背景绘制。

reflect object
    depth test/write 开启，正常绘制前景物体。
```

## 24. UI Overlay：LOD Bias、模型选择和 Skybox 开关

UI 逻辑在 `OnUpdateUIOverlay()`：

```cpp
if (overlay->header("Settings")) {
    overlay->sliderFloat(
        "LOD bias",
        &uniformData.lodBias,
        0.0f,
        (float)cubeMap.mipLevels);
    overlay->comboBox("Object type", &models.objectIndex, objectNames);
    overlay->checkBox("Skybox", &displaySkybox);
}
```

三个控件分别影响：

```text
LOD bias
    写入 uniformData.lodBias，影响 reflect.frag 中 texture(..., lodBias) 的 mip 选择。

Object type
    修改 models.objectIndex，决定绘制 sphere / teapot / torusknot / venus。

Skybox
    修改 displaySkybox，决定是否绘制背景 cube。
```

`lodBias` 的数据流是：

```text
UI slider
    -> uniformData.lodBias
    -> current frame uniform buffer
    -> reflect.frag UBO
    -> texture(samplerColor, cR, ubo.lodBias)
```

## 25. 和普通 2D 纹理示例的区别

`texture.cpp` 和 `texturecubemap.cpp` 都使用 KTX、staging buffer、optimal image、sampler、image view 和 descriptor。关键差异在这里：

| 项目 | 2D texture | Cubemap |
| --- | --- | --- |
| shader sampler | `sampler2D` | `samplerCube` |
| 采样坐标 | `vec2 uv` | `vec3 direction` |
| image layers | `1` | `6` |
| image flag | 无 cube flag | `VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT` |
| image view type | `VK_IMAGE_VIEW_TYPE_2D` | `VK_IMAGE_VIEW_TYPE_CUBE` |
| copy regions | 每个 mip level 一项 | 每个 face 的每个 mip level 一项 |
| 常见用途 | 材质贴图 | skybox、环境反射 |

可以把 cubemap 看作普通 texture 示例的扩展：

```text
2D texture
    一个 image layer，UV 采样。

Cubemap
    六个 image layer，方向向量采样。
```

## 26. 资源生命周期

析构函数释放 cubemap 和 pipeline 相关对象：

```cpp
vkDestroyImageView(device, cubeMap.view, nullptr);
vkDestroyImage(device, cubeMap.image, nullptr);
vkDestroySampler(device, cubeMap.sampler, nullptr);
vkFreeMemory(device, cubeMap.deviceMemory, nullptr);
vkDestroyPipeline(device, pipelines.skybox, nullptr);
vkDestroyPipeline(device, pipelines.reflect, nullptr);
vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);
for (auto& buffer : uniformBuffers) {
    buffer.destroy();
}
```

从依赖关系看：

```text
descriptor set
    引用 cubeMap.view 和 cubeMap.sampler。

cubeMap.view
    引用 cubeMap.image。

cubeMap.image
    绑定 cubeMap.deviceMemory。
```

示例退出时统一释放这些对象。descriptor pool 由基类管理。

## 27. 常见错误和调试方向

第一，创建 cubemap image 时必须设置：

```cpp
imageCreateInfo.arrayLayers = 6;
imageCreateInfo.flags = VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT;
```

少了 cube-compatible flag，就不能创建 `VK_IMAGE_VIEW_TYPE_CUBE`。

第二，image view 必须是 cube view：

```cpp
view.viewType = VK_IMAGE_VIEW_TYPE_CUBE;
view.subresourceRange.layerCount = 6;
```

如果 view type 错误，shader 中的 `samplerCube` 不能按 cubemap 方式采样。

第三，copy region 的 `baseArrayLayer` 要对应 face：

```cpp
bufferCopyRegion.imageSubresource.baseArrayLayer = face;
```

否则 6 个 face 的数据会被写到错误 layer，表现为天空盒方向错乱或反射错误。

第四，layout transition 的 subresource range 要覆盖 6 个 layer：

```cpp
subresourceRange.layerCount = 6;
```

只 transition 一个 layer 会导致其他 face 仍处于错误 layout。

第五，cubemap 坐标空间要匹配。示例中 skybox 和 reflection shader 都做了：

```glsl
*.xy *= -1.0;
```

如果换了 cubemap 文件或坐标约定，方向可能需要调整。

第六，修改 shader 后要重新编译 SPIR-V。运行时加载的是：

```text
shaders/glsl/texturecubemap/skybox.vert.spv
shaders/glsl/texturecubemap/skybox.frag.spv
shaders/glsl/texturecubemap/reflect.vert.spv
shaders/glsl/texturecubemap/reflect.frag.spv
```

不是直接加载 GLSL 文本文件。

## 28. 从这个示例提炼出的工程原则

第一，cubemap 在 Vulkan 中是 6 层 2D image array 加 cube view。

不要把 cubemap 当成 6 张完全独立的 texture。Vulkan 的正确表达是一个 image，6 个 array layer，一个 cube-compatible view。

第二，上传 cubemap 时要同时关心 face 和 mip level。

普通 2D 纹理只遍历 mip，cubemap 需要：

```text
for face in 0..5
    for mip in 0..mipLevels-1
```

第三，skybox shader 应该移除 view matrix 的平移。

否则 skybox 会随着相机移动，看起来像一个有限大小的盒子，而不是无限远环境。

第四，skybox 和反射可以共享同一张 cubemap 和同一个 descriptor set。

它们只是采样方向不同：skybox 用 cube 顶点方向，反射物体用 `reflect()` 计算出的反射方向。

第五，pipeline 状态要根据用途分开。

skybox 和反射对象使用不同 shader、不同 cull mode、不同 depth 设置，所以用两条 pipeline 是合理的。

## 29. 一个简化版 mental model

可以把这个示例理解成下面的链路：

```text
cubemap_yokohama_rgba.ktx
    -> ktxTexture
    -> staging VkBuffer
    -> VkImage(2D, arrayLayers=6, CUBE_COMPATIBLE)
    -> copy face 0..5, mip 0..N
    -> layout: SHADER_READ_ONLY_OPTIMAL
    -> VkImageView(VK_IMAGE_VIEW_TYPE_CUBE)
    -> VkSampler(CLAMP_TO_EDGE)
    -> VkDescriptorSet binding 1
    -> samplerCube
    -> skybox: texture(cubeMap, direction)
    -> reflection: texture(cubeMap, reflect(view, normal), lodBias)
```

理解这条链路后，Vulkan 中 cubemap 的创建、上传和 shader 采样逻辑就清楚了。

## 30. 总结

`texturecubemap.cpp` 展示了 Vulkan cubemap 的完整基础用法：从 KTX 文件读取 6 个 face 和 mip chain，通过 staging buffer 上传到一个 6 层、cube-compatible、optimal tiled `VkImage`，创建 cube image view 和 sampler，再通过 combined image sampler descriptor 交给 shader。

这个示例中：

- `loadCubemap()` 负责 KTX 读取、staging 上传、image 创建、layout transition、sampler 和 cube view 创建。
- `loadAssets()` 加载 skybox cube、多个可反射 glTF 模型和 cubemap 纹理。
- `setupDescriptors()` 把 UBO 和 cubemap sampler 写入每帧 descriptor set。
- `preparePipelines()` 创建 skybox 和 reflect 两条 pipeline。
- `skybox.vert/frag` 用方向向量直接采样 cubemap。
- `reflect.vert/frag` 根据视线和法线计算反射方向，再采样 cubemap。
- `buildCommandBuffer()` 先绘制 skybox，再绘制当前选中的反射物体。

如果后续要做环境光照、PBR IBL、动态反射或点光源阴影，cubemap 的这套 image array、cube view、samplerCube 和 face/mip 上传流程是必须掌握的基础。
