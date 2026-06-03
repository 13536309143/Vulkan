# Vulkan 示例解析：gltfloading.cpp 如何加载并渲染一个 glTF 场景

本文分析的是 `examples/gltfloading/gltfloading.cpp`。这个示例展示了如何用 tinyglTF 读取一个 glTF 2.0 模型，并把它转换成 Vulkan 可以渲染的资源：vertex buffer、index buffer、texture、descriptor set、graphics pipeline 和 draw call。

它加载的模型是：

```cpp
models/FlightHelmet/glTF/FlightHelmet.gltf
```

这是一个非常适合作为 glTF 入门案例的模型：有 scene/node/mesh/primitive/material/texture/image 这些核心结构，但示例没有引入动画、骨骼、复杂 PBR 材质等高级特性。

这份代码的价值不在于它实现了完整 glTF loader，而在于它把 glTF 到 Vulkan 的关键映射关系讲得很清楚：

- glTF 的 `image` 如何变成 Vulkan texture。
- glTF 的 `material` 如何找到 base color texture。
- glTF 的 `node` 如何形成场景树。
- glTF 的 `accessor + bufferView + buffer` 如何读出顶点和索引。
- 多个 primitive 如何合并到一份全局 vertex/index buffer。
- node transform 如何通过 push constants 传给 vertex shader。
- scene matrices 和 material textures 如何拆成两个 descriptor set。

## 1. 这个示例的定位

文件开头已经说明了它的边界：

```cpp
/*
 * Shows how to load and display a simple scene from a glTF file
 * Note that this isn't a complete glTF loader and only basic functions are shown here
 * This means no complex materials, no animations, no skins, etc.
 */
```

这句话很重要。`gltfloading.cpp` 不是完整引擎级 glTF loader。它是一个教学版实现，只保留了渲染 FlightHelmet 这个模型所需的最小核心：

- 位置、法线、UV、颜色。
- base color texture。
- node 层级 transform。
- mesh primitive。
- indexed draw。
- 一个简单的 diffuse + specular 光照 shader。

如果你要支持完整 glTF 2.0，需要继续处理：

- PBR metallic-roughness 材质。
- normal/occlusion/emissive texture。
- alpha mode。
- tangent。
- animation。
- skinning。
- morph target。
- sampler 参数。
- 多 UV set。
- sparse accessor。
- extension。

仓库里也有 `base/VulkanglTFModel.h`，那是更完整的 glTF helper。这个示例反而没有直接使用那个完整 loader，而是在 `gltfloading.cpp` 里定义了一个简化版 `VulkanglTFModel`，目的是让 glTF 数据结构和 Vulkan 资源创建过程更透明。

## 2. tinyGLTF：先把文件解析成内存模型

代码一开始定义了 tinyGLTF 的实现宏：

```cpp
#define TINYGLTF_IMPLEMENTATION
#define STB_IMAGE_IMPLEMENTATION
#define TINYGLTF_NO_STB_IMAGE_WRITE
#include "tiny_gltf.h"
```

tinyGLTF 负责把 `.gltf` 文件解析成 `tinygltf::Model`。这个对象里面保存了 glTF 的所有核心数组：

- `scenes`
- `nodes`
- `meshes`
- `materials`
- `textures`
- `images`
- `accessors`
- `bufferViews`
- `buffers`

示例自己的工作是把这些 glTF 结构转成更适合 Vulkan 渲染的结构。

加载入口在 `loadglTFFile()`：

```cpp
tinygltf::Model glTFInput;
tinygltf::TinyGLTF gltfContext;
std::string error, warning;

bool fileLoaded = gltfContext.LoadASCIIFromFile(&glTFInput, &error, &warning, filename);
```

这里使用的是 `LoadASCIIFromFile`，也就是加载 `.gltf` JSON 文本格式。FlightHelmet 示例模型的二进制数据和图片会由 glTF 文件引用或内嵌后经 tinyGLTF 解析出来。

Android 平台下会额外设置：

```cpp
tinygltf::asset_manager = androidApp->activity->assetManager;
```

因为 Android 资源在 APK 中，需要通过 asset manager 读取。

## 3. 简化版 VulkanglTFModel：保留 glTF 的基本结构

文件内部定义了一个 `VulkanglTFModel` 类：

```cpp
class VulkanglTFModel
```

它不是仓库 `base` 里的完整模型类，而是本示例专用的教学版 loader。

它持有 Vulkan 创建资源所需的上下文：

```cpp
vks::VulkanDevice* vulkanDevice;
VkQueue copyQueue;
```

`vulkanDevice` 用于创建 buffer、image、memory；`copyQueue` 用于上传 buffer 和 texture。

### 3.1 Vertex

```cpp
struct Vertex {
    glm::vec3 pos;
    glm::vec3 normal;
    glm::vec2 uv;
    glm::vec3 color;
};
```

这个 layout 对应 shader 输入：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inNormal;
layout (location = 2) in vec2 inUV;
layout (location = 3) in vec3 inColor;
```

也就是说，CPU 侧解析 glTF 后会把每个顶点整理成：

- 位置
- 法线
- 第一套 UV
- 顶点颜色

不过这个示例没有真正从 glTF 读取 vertex color，而是给每个顶点设置：

```cpp
.color = glm::vec3(1.0f)
```

所以 fragment shader 中的 `inColor` 实际上只是白色乘子，最终主要颜色来自 base color texture。

### 3.2 全局 vertex/index buffer

```cpp
struct {
    VkBuffer buffer;
    VkDeviceMemory memory;
} vertices;

struct {
    int count;
    VkBuffer buffer;
    VkDeviceMemory memory;
} indices;
```

这个示例没有为每个 primitive 创建独立 buffer，而是把整个 glTF scene 的所有顶点和索引合并到：

- 一份全局 vertex buffer。
- 一份全局 index buffer。

每个 primitive 只记录：

```cpp
struct Primitive {
    uint32_t firstIndex;
    uint32_t indexCount;
    int32_t materialIndex;
};
```

渲染时通过 `firstIndex` 和 `indexCount` 在全局 index buffer 中定位自己的 draw range：

```cpp
vkCmdDrawIndexed(commandBuffer, primitive.indexCount, 1, primitive.firstIndex, 0, 0);
```

这是实际渲染中很常见的组织方式。它减少了 buffer bind 次数，所有 mesh primitive 可以共享一组大 buffer。

### 3.3 Node / Mesh / Primitive

示例保留了 glTF 的场景树结构：

```cpp
struct Node {
    Node* parent;
    std::vector<Node*> children;
    Mesh mesh;
    glm::mat4 matrix;
};
```

一个 node 可以有：

- 父节点。
- 子节点。
- 本地 transform。
- 可选 mesh。

mesh 内部包含多个 primitive：

```cpp
struct Mesh {
    std::vector<Primitive> primitives;
};
```

这符合 glTF 的基本结构：一个 node 引用一个 mesh，一个 mesh 可以包含多个 primitive，每个 primitive 对应一次 draw call，通常也对应一个 material。

### 3.4 Material / Texture / Image

简化版材质：

```cpp
struct Material {
    glm::vec4 baseColorFactor = glm::vec4(1.0f);
    uint32_t baseColorTextureIndex;
};
```

简化版 texture：

```cpp
struct Texture {
    int32_t imageIndex;
};
```

简化版 image：

```cpp
struct Image {
    vks::Texture2D texture;
    VkDescriptorSet descriptorSet;
};
```

这三者对应 glTF 中的常见引用链：

```text
material.baseColorTexture.index
        |
        v
textures[index].source
        |
        v
images[source]
```

glTF 的 material 不直接保存图片数据，而是通过 texture 找到 image。示例也保留了这个两级引用关系：

- `Material::baseColorTextureIndex` 指向 `textures`。
- `Texture::imageIndex` 指向 `images`。
- `Image::texture` 是 Vulkan texture。
- `Image::descriptorSet` 用于 fragment shader 采样。

## 4. 加载图片：glTF image 到 Vulkan Texture2D

图片加载函数是：

```cpp
void loadImages(tinygltf::Model& input)
```

它遍历：

```cpp
input.images
```

每个 `tinygltf::Image` 中已经有 tinyGLTF 解码后的像素数据：

```cpp
tinygltf::Image& glTFImage = input.images[i];
```

### 4.1 RGB 转 RGBA

Vulkan 中常用纹理格式是 `VK_FORMAT_R8G8B8A8_UNORM`，但有些 glTF 图片可能只有 RGB 三通道：

```cpp
if (glTFImage.component == 3) {
    bufferSize = glTFImage.width * glTFImage.height * 4;
    buffer = new unsigned char[bufferSize];
    ...
}
```

代码会手动把 RGB 扩展成 RGBA：

```cpp
memcpy(rgba, rgb, sizeof(unsigned char) * 3);
rgba += 4;
rgb += 3;
```

这里没有显式写 alpha，严格来说新分配 buffer 的第 4 通道没有被赋值。这个示例模型和渲染路径通常不会暴露明显问题，但如果要写成健壮 loader，应该把 alpha 写成 `255`。

### 4.2 上传到 Vulkan texture

最终调用：

```cpp
images[i].texture.fromBuffer(
    buffer,
    bufferSize,
    VK_FORMAT_R8G8B8A8_UNORM,
    glTFImage.width,
    glTFImage.height,
    vulkanDevice,
    copyQueue);
```

`vks::Texture2D::fromBuffer` 会负责：

- 创建 staging buffer。
- 创建 Vulkan image。
- 拷贝像素数据。
- 创建 image view。
- 创建 sampler。
- 填充 `VkDescriptorImageInfo`。

所以图片上传部分没有在 `gltfloading.cpp` 中完全展开，而是复用了框架 texture helper。

## 5. 加载 texture 和 material

texture 加载非常轻：

```cpp
void loadTextures(tinygltf::Model& input)
{
    textures.resize(input.textures.size());
    for (size_t i = 0; i < input.textures.size(); i++) {
        textures[i].imageIndex = input.textures[i].source;
    }
}
```

glTF texture 本来还可以包含 sampler 信息，但这个示例只关心 texture 引用的 image：

```text
texture.source -> image index
```

material 加载：

```cpp
void loadMaterials(tinygltf::Model& input)
```

它只读取两个基础属性：

```cpp
baseColorFactor
baseColorTexture
```

代码：

```cpp
if (glTFMaterial.values.find("baseColorFactor") != glTFMaterial.values.end()) {
    materials[i].baseColorFactor =
        glm::make_vec4(glTFMaterial.values["baseColorFactor"].ColorFactor().data());
}

if (glTFMaterial.values.find("baseColorTexture") != glTFMaterial.values.end()) {
    materials[i].baseColorTextureIndex =
        glTFMaterial.values["baseColorTexture"].TextureIndex();
}
```

不过后续 shader 并没有使用 `baseColorFactor`，只使用 base color texture 和顶点颜色。这再次说明它是教学版 loader，而不是完整 PBR 材质实现。

## 6. 加载 Node：glTF 场景树如何变成可绘制结构

核心函数是：

```cpp
void loadNode(
    const tinygltf::Node& inputNode,
    const tinygltf::Model& input,
    VulkanglTFModel::Node* parent,
    std::vector<uint32_t>& indexBuffer,
    std::vector<VulkanglTFModel::Vertex>& vertexBuffer)
```

它做三件事：

1. 创建本示例自己的 `Node`。
2. 解析 node transform 和 child node。
3. 如果 node 有 mesh，就解析 mesh primitive 的顶点和索引。

### 6.1 解析 node transform

glTF node 的 transform 有两种表达方式：

- TRS：translation、rotation、scale。
- matrix：一个 4x4 矩阵。

示例都处理了：

```cpp
node->matrix = glm::mat4(1.0f);

if (inputNode.translation.size() == 3) {
    node->matrix = glm::translate(node->matrix, glm::vec3(glm::make_vec3(inputNode.translation.data())));
}
if (inputNode.rotation.size() == 4) {
    glm::quat q = glm::make_quat(inputNode.rotation.data());
    node->matrix *= glm::mat4(q);
}
if (inputNode.scale.size() == 3) {
    node->matrix = glm::scale(node->matrix, glm::vec3(glm::make_vec3(inputNode.scale.data())));
}
if (inputNode.matrix.size() == 16) {
    node->matrix = glm::make_mat4x4(inputNode.matrix.data());
}
```

这就是 node 的本地矩阵。真正绘制时还会沿 parent 链向上累乘，得到最终 node transform。

### 6.2 递归加载子节点

```cpp
for (size_t i = 0; i < inputNode.children.size(); i++) {
    loadNode(input.nodes[inputNode.children[i]], input, node, indexBuffer, vertexBuffer);
}
```

这保留了 glTF 的场景层级。

为什么不能只把所有 mesh 平铺？因为 node 层级会影响 transform。一个子节点的最终矩阵是：

```text
root matrix * parent matrix * local matrix
```

渲染时必须恢复这个层级关系。

## 7. 从 accessor 读取顶点属性

glTF 的顶点数据不是直接放在 mesh primitive 里，而是通过三层结构间接访问：

```text
primitive.attributes["POSITION"] -> accessor index
accessor.bufferView -> bufferView index
bufferView.buffer -> buffer index
buffer.data + bufferView.byteOffset + accessor.byteOffset
```

示例读取 position 的代码：

```cpp
const tinygltf::Accessor& accessor =
    input.accessors[glTFPrimitive.attributes.find("POSITION")->second];
const tinygltf::BufferView& view = input.bufferViews[accessor.bufferView];
positionBuffer =
    reinterpret_cast<const float*>(&(input.buffers[view.buffer].data[accessor.byteOffset + view.byteOffset]));
vertexCount = accessor.count;
```

normal 和 UV 也是同样路径：

```cpp
"NORMAL"
"TEXCOORD_0"
```

### 7.1 为什么 accessor 这么重要

在 glTF 中，accessor 描述的是“如何解释 buffer 中的一段数据”：

- 数据数量：`count`
- 元素类型：如 `VEC3`
- component 类型：如 float、uint16
- 起始偏移：`byteOffset`
- 所属 bufferView

bufferView 描述的是 buffer 中一段连续区域，buffer 才是真正的字节数组。

这个结构看起来绕，但它允许 glTF 复用 buffer，并用统一方式描述 position、normal、UV、index、animation keyframe 等各种数据。

### 7.2 组装本示例的 Vertex

读取到各属性指针后，示例把它们整理成自己的 `Vertex`：

```cpp
Vertex vert{
    .pos = glm::vec4(glm::make_vec3(&positionBuffer[v * 3]), 1.0f),
    .normal = glm::normalize(glm::vec3(normalsBuffer ? glm::make_vec3(&normalsBuffer[v * 3]) : glm::vec3(0.0f))),
    .uv = texCoordsBuffer ? glm::make_vec2(&texCoordsBuffer[v * 2]) : glm::vec3(0.0f),
    .color = glm::vec3(1.0f),
};
vertexBuffer.push_back(vert);
```

有几个细节：

- position 必须存在，否则无法确定 `vertexCount`。
- normal 可选；没有 normal 时给零向量。
- UV 可选；没有 UV 时给零。
- color 固定为白色。
- normal 会 normalize。

这一步完成后，glTF 中分散的 attribute 数据就变成了 Vulkan pipeline 直接可读的 interleaved vertex layout。

## 8. 从 accessor 读取索引

索引读取也通过 accessor：

```cpp
const tinygltf::Accessor& accessor = input.accessors[glTFPrimitive.indices];
const tinygltf::BufferView& bufferView = input.bufferViews[accessor.bufferView];
const tinygltf::Buffer& buffer = input.buffers[bufferView.buffer];
```

glTF 支持不同 index component type，所以代码处理了三种：

```cpp
TINYGLTF_PARAMETER_TYPE_UNSIGNED_INT
TINYGLTF_PARAMETER_TYPE_UNSIGNED_SHORT
TINYGLTF_PARAMETER_TYPE_UNSIGNED_BYTE
```

无论 glTF 原始索引是 32 位、16 位还是 8 位，最终都会写入：

```cpp
std::vector<uint32_t> indexBuffer;
```

并且每个索引都会加上：

```cpp
vertexStart
```

例如：

```cpp
indexBuffer.push_back(buf[index] + vertexStart);
```

这是因为示例把所有 primitive 的顶点都追加到同一个全局 `vertexBuffer`。每个 primitive 自己的索引通常从 0 开始，追加到全局 buffer 后必须加上当前 primitive 的顶点起始偏移。

同时记录：

```cpp
uint32_t firstIndex = static_cast<uint32_t>(indexBuffer.size());
uint32_t indexCount = 0;
```

最终生成 primitive：

```cpp
Primitive primitive{
    .firstIndex = firstIndex,
    .indexCount = indexCount,
    .materialIndex = glTFPrimitive.material
};
node->mesh.primitives.push_back(primitive);
```

这就是后续 `vkCmdDrawIndexed` 能定位每个 primitive 的原因。

## 9. loadglTFFile：从 glTF 文件到 GPU buffer

`loadglTFFile()` 是整个模型加载的总入口。

加载文件后，先把 Vulkan 上下文交给模型对象：

```cpp
glTFModel.vulkanDevice = vulkanDevice;
glTFModel.copyQueue = queue;
```

然后创建临时 CPU 侧数组：

```cpp
std::vector<uint32_t> indexBuffer;
std::vector<VulkanglTFModel::Vertex> vertexBuffer;
```

如果 glTF 文件加载成功：

```cpp
glTFModel.loadImages(glTFInput);
glTFModel.loadMaterials(glTFInput);
glTFModel.loadTextures(glTFInput);
```

然后读取默认 scene：

```cpp
const tinygltf::Scene& scene = glTFInput.scenes[0];
for (size_t i = 0; i < scene.nodes.size(); i++) {
    const tinygltf::Node node = glTFInput.nodes[scene.nodes[i]];
    glTFModel.loadNode(node, glTFInput, nullptr, indexBuffer, vertexBuffer);
}
```

到这里，CPU 侧已经有：

- `glTFModel.images`
- `glTFModel.textures`
- `glTFModel.materials`
- `glTFModel.nodes`
- 临时 `vertexBuffer`
- 临时 `indexBuffer`

接下来把顶点和索引上传到 GPU。

## 10. Staging Buffer：把顶点和索引放到 device local memory

和 `triangle_light.cpp` 不同，`gltfloading.cpp` 使用 staging buffer 上传几何数据。

先计算大小：

```cpp
size_t vertexBufferSize = vertexBuffer.size() * sizeof(VulkanglTFModel::Vertex);
size_t indexBufferSize = indexBuffer.size() * sizeof(uint32_t);
glTFModel.indices.count = static_cast<uint32_t>(indexBuffer.size());
```

创建 host-visible staging buffer：

```cpp
vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_TRANSFER_SRC_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
    &vertexStaging,
    vertexBufferSize,
    vertexBuffer.data());
```

index staging 同理。

再创建 GPU device local buffer：

```cpp
vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT,
    VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
    vertexBufferSize,
    &glTFModel.vertices.buffer,
    &glTFModel.vertices.memory);
```

index buffer 同理。

然后录制 copy command：

```cpp
VkCommandBuffer copyCmd = vulkanDevice->createCommandBuffer(VK_COMMAND_BUFFER_LEVEL_PRIMARY, true);

copyRegion.size = vertexBufferSize;
vkCmdCopyBuffer(copyCmd, vertexStaging.buffer, glTFModel.vertices.buffer, 1, &copyRegion);

copyRegion.size = indexBufferSize;
vkCmdCopyBuffer(copyCmd, indexStaging.buffer, glTFModel.indices.buffer, 1, &copyRegion);

vulkanDevice->flushCommandBuffer(copyCmd, queue, true);
```

最后销毁 staging buffer：

```cpp
vertexStaging.destroy();
indexStaging.destroy();
```

这是更接近真实项目的写法：静态几何数据最终放在 device local memory，GPU 访问效率更高。

## 11. Descriptor 设计：set 0 放矩阵，set 1 放纹理

这个示例使用两个 descriptor set layout：

```cpp
struct DescriptorSetLayouts {
    VkDescriptorSetLayout matrices;
    VkDescriptorSetLayout textures;
} descriptorSetLayouts;
```

对应 shader：

```glsl
layout (set = 0, binding = 0) uniform UBOScene
```

和：

```glsl
layout (set = 1, binding = 0) uniform sampler2D samplerColorMap;
```

这种拆法很有代表性：

- set 0：每帧变化的 scene/global 数据。
- set 1：每个 material/texture 变化的数据。

### 11.1 Descriptor Pool

```cpp
std::vector<VkDescriptorPoolSize> poolSizes = {
    vks::initializers::descriptorPoolSize(VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, maxConcurrentFrames),
    vks::initializers::descriptorPoolSize(
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        static_cast<uint32_t>(glTFModel.images.size()) * maxConcurrentFrames),
};
```

pool 支持两类 descriptor：

- uniform buffer。
- combined image sampler。

代码中 image sampler 的数量乘了 `maxConcurrentFrames`，但后续实际只为每张 image 分配一份 descriptor set，没有按 frame 复制。这个 pool 配额偏宽松，不影响正确性。

### 11.2 set 0：Scene Matrices

矩阵 layout：

```cpp
VkDescriptorSetLayoutBinding setLayoutBinding =
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        VK_SHADER_STAGE_VERTEX_BIT,
        0);
```

然后为每个 frame 分配一个 descriptor set：

```cpp
for (auto i = 0; i < uniformBuffers.size(); i++) {
    vkAllocateDescriptorSets(..., &descriptorSets[i]);
    VkWriteDescriptorSet writeDescriptorSet =
        vks::initializers::writeDescriptorSet(
            descriptorSets[i],
            VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
            0,
            &uniformBuffers[i].descriptor);
    vkUpdateDescriptorSets(...);
}
```

原因是 scene matrices 每帧会更新，并且有多个 in-flight frame，所以每帧需要自己的 UBO。

### 11.3 set 1：Material Texture

纹理 layout：

```cpp
setLayoutBinding =
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        VK_SHADER_STAGE_FRAGMENT_BIT,
        0);
```

然后每张 image 分配一个 descriptor set：

```cpp
for (auto& image : glTFModel.images) {
    vkAllocateDescriptorSets(..., &image.descriptorSet);
    VkWriteDescriptorSet writeDescriptorSet =
        vks::initializers::writeDescriptorSet(
            image.descriptorSet,
            VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
            0,
            &image.texture.descriptor);
    vkUpdateDescriptorSets(...);
}
```

纹理是静态资源，不需要每个 frame 复制一份 descriptor set。绘制 primitive 时，根据 material 找到对应 image descriptor set 并绑定。

## 12. Pipeline Layout：descriptor set + push constants

pipeline layout 使用两个 descriptor set layout：

```cpp
std::array<VkDescriptorSetLayout, 2> setLayouts = {
    descriptorSetLayouts.matrices,
    descriptorSetLayouts.textures
};
```

还定义了一个 push constant：

```cpp
VkPushConstantRange pushConstantRange =
    vks::initializers::pushConstantRange(VK_SHADER_STAGE_VERTEX_BIT, sizeof(glm::mat4), 0);
```

这个 push constant 用来传每个 node 的 model matrix。

为什么不把 node matrix 放到 uniform buffer？因为每个 node/primitive 绘制时都可能不同。push constants 很适合这种“小而频繁变化”的 draw-level 数据：

- 不需要创建额外 buffer。
- 不需要更新 descriptor。
- 直接编码进 command buffer。

最终 pipeline layout：

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutCI{
    .sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO,
    .setLayoutCount = static_cast<uint32_t>(setLayouts.size()),
    .pSetLayouts = setLayouts.data(),
    .pushConstantRangeCount = 1,
    .pPushConstantRanges = &pushConstantRange
};
```

这对应 shader 中：

```glsl
layout(push_constant) uniform PushConsts {
    mat4 model;
} primitive;
```

## 13. Graphics Pipeline：把 glTF Vertex layout 接到 shader

pipeline 的固定状态和大多数 3D 示例类似：

```cpp
VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST
VK_POLYGON_MODE_FILL
VK_CULL_MODE_BACK_BIT
VK_FRONT_FACE_COUNTER_CLOCKWISE
VK_COMPARE_OP_LESS_OR_EQUAL
```

它启用了：

- 三角形列表。
- 背面剔除。
- 深度测试。
- 深度写入。
- 动态 viewport/scissor。

### 13.1 Vertex Input

顶点输入 binding：

```cpp
vertexInputBindingDescription(
    0,
    sizeof(VulkanglTFModel::Vertex),
    VK_VERTEX_INPUT_RATE_VERTEX)
```

attribute：

```cpp
location 0: pos    -> VK_FORMAT_R32G32B32_SFLOAT
location 1: normal -> VK_FORMAT_R32G32B32_SFLOAT
location 2: uv     -> VK_FORMAT_R32G32_SFLOAT
location 3: color  -> VK_FORMAT_R32G32B32_SFLOAT
```

这和 `mesh.vert` 完全对应：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inNormal;
layout (location = 2) in vec2 inUV;
layout (location = 3) in vec3 inColor;
```

### 13.2 Shader

pipeline 使用：

```cpp
gltfloading/mesh.vert.spv
gltfloading/mesh.frag.spv
```

vertex shader 做了三件事：

1. 用 `projection * view * model` 计算 `gl_Position`。
2. 把 normal、UV、color 传给 fragment shader。
3. 计算 view vector 和 light vector，用于简单光照。

fragment shader：

```glsl
vec4 color = texture(samplerColorMap, inUV) * vec4(inColor, 1.0);

vec3 N = normalize(inNormal);
vec3 L = normalize(inLightVec);
vec3 V = normalize(inViewVec);
vec3 R = reflect(L, N);
vec3 diffuse = max(dot(N, L), 0.15) * inColor;
vec3 specular = pow(max(dot(R, V), 0.0), 16.0) * vec3(0.75);
outFragColor = vec4(diffuse * color.rgb + specular, 1.0);
```

这不是 glTF 标准 PBR，只是一个教学用的简单 diffuse + specular 光照模型。

还有一个值得注意的细节：vertex shader 里计算了：

```glsl
vec3 lPos = mat3(uboScene.view) * uboScene.lightPos.xyz;
```

但后面没有使用 `lPos`。同时 normal 和 lighting vector 的空间处理也比较简化，没有严格使用完整 model-view normal matrix。FlightHelmet 这个示例能正常展示，但如果模型有复杂 node transform 或非均匀缩放，这个 shader 就不是严谨的光照实现。

### 13.3 Wireframe Pipeline

除了 solid pipeline，还可以创建 wireframe pipeline：

```cpp
if (deviceFeatures.fillModeNonSolid) {
    rasterizationStateCI.polygonMode = VK_POLYGON_MODE_LINE;
    rasterizationStateCI.lineWidth = 1.0f;
    vkCreateGraphicsPipelines(..., &pipelines.wireframe);
}
```

启用 wireframe 需要设备支持：

```cpp
deviceFeatures.fillModeNonSolid
```

所以示例在 `getEnabledFeatures()` 中请求：

```cpp
if (deviceFeatures.fillModeNonSolid) {
    enabledFeatures.fillModeNonSolid = VK_TRUE;
}
```

UI overlay 中也只在设备支持时显示 wireframe 开关。

## 14. Uniform Buffer：每帧更新 scene 数据

scene uniform：

```cpp
struct UniformData {
    glm::mat4 projection;
    glm::mat4 model;
    glm::vec4 lightPos = glm::vec4(5.0f, 5.0f, -5.0f, 1.0f);
    glm::vec4 viewPos;
} uniformData;
```

这个结构对应 shader：

```glsl
layout (set = 0, binding = 0) uniform UBOScene
{
    mat4 projection;
    mat4 view;
    vec4 lightPos;
    vec4 viewPos;
} uboScene;
```

注意 C++ 里的字段名叫 `model`，shader 里叫 `view`。字段名不重要，内存顺序才重要。这里第二个 `mat4` 实际上传的是：

```cpp
uniformData.model = camera.matrices.view;
```

所以 shader 的 `uboScene.view` 拿到的是相机 view matrix。

每帧更新：

```cpp
uniformData.projection = camera.matrices.perspective;
uniformData.model = camera.matrices.view;
uniformData.viewPos = camera.viewPos;
memcpy(uniformBuffers[currentBuffer].mapped, &uniformData, sizeof(UniformData));
```

和其他示例一样，uniform buffer 是 per-frame 的：

```cpp
std::array<vks::Buffer, maxConcurrentFrames> uniformBuffers;
```

这样可以避免 CPU 写入当前帧 UBO 时覆盖 GPU 上一帧仍在读取的数据。

## 15. Draw：遍历 node 树，按 primitive 发出 draw call

模型绘制入口：

```cpp
void draw(VkCommandBuffer commandBuffer, VkPipelineLayout pipelineLayout)
```

先绑定全局 vertex/index buffer：

```cpp
VkDeviceSize offsets[1] = { 0 };
vkCmdBindVertexBuffers(commandBuffer, 0, 1, &vertices.buffer, offsets);
vkCmdBindIndexBuffer(commandBuffer, indices.buffer, 0, VK_INDEX_TYPE_UINT32);
```

然后遍历顶层 nodes：

```cpp
for (auto& node : nodes) {
    drawNode(commandBuffer, pipelineLayout, node);
}
```

### 15.1 drawNode：计算最终 node matrix

```cpp
glm::mat4 nodeMatrix = node->matrix;
VulkanglTFModel::Node* currentParent = node->parent;
while (currentParent) {
    nodeMatrix = currentParent->matrix * nodeMatrix;
    currentParent = currentParent->parent;
}
```

这一步把 node 的本地矩阵沿父节点链一路乘上去，得到最终 model matrix。

然后通过 push constants 传给 vertex shader：

```cpp
vkCmdPushConstants(
    commandBuffer,
    pipelineLayout,
    VK_SHADER_STAGE_VERTEX_BIT,
    0,
    sizeof(glm::mat4),
    &nodeMatrix);
```

### 15.2 每个 primitive 绑定自己的 texture

每个 primitive 有自己的 material：

```cpp
primitive.materialIndex
```

代码根据 material 找到 texture，再找到 image：

```cpp
VulkanglTFModel::Texture texture =
    textures[materials[primitive.materialIndex].baseColorTextureIndex];
```

然后绑定 set 1：

```cpp
vkCmdBindDescriptorSets(
    commandBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    1,
    1,
    &images[texture.imageIndex].descriptorSet,
    0,
    nullptr);
```

注意参数里的 `1`，表示绑定到 descriptor set index 1，也就是 material texture set。

最后 draw：

```cpp
vkCmdDrawIndexed(commandBuffer, primitive.indexCount, 1, primitive.firstIndex, 0, 0);
```

这样一个 glTF scene 就被拆成了多次 indexed draw call。

## 16. buildCommandBuffer：一帧内完整渲染命令

主示例类的 `buildCommandBuffer()` 使用基类的 command buffer：

```cpp
VkCommandBuffer cmdBuffer = drawCmdBuffers[currentBuffer];
```

然后开始 render pass：

```cpp
VkClearValue clearValues[2]{};
clearValues[0].color = { { 0.25f, 0.25f, 0.25f, 1.0f } };
clearValues[1].depthStencil = { 1.0f, 0 };
```

设置 viewport/scissor：

```cpp
vkCmdSetViewport(cmdBuffer, 0, 1, &viewport);
vkCmdSetScissor(cmdBuffer, 0, 1, &scissor);
```

绑定 set 0，也就是当前 frame 的 scene matrices：

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

绑定 solid 或 wireframe pipeline：

```cpp
vkCmdBindPipeline(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    wireframe ? pipelines.wireframe : pipelines.solid);
```

绘制 glTF 模型：

```cpp
glTFModel.draw(cmdBuffer, pipelineLayout);
```

最后绘制 UI：

```cpp
drawUI(cmdBuffer);
```

这就是一帧中所有图形命令的录制。

## 17. render：基类负责 acquire/submit/present

每帧渲染函数：

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

这和很多 Sascha Willems 示例一致：

- `prepareFrame()`：等待 fence，acquire swapchain image。
- `updateUniformBuffers()`：更新当前 frame 的 scene UBO。
- `buildCommandBuffer()`：录制当前 frame 的 draw commands。
- `submitFrame()`：submit command buffer 并 present。

这个示例的重点不在 swapchain 同步细节，而在 glTF 数据如何变成 Vulkan draw call，所以 frame lifecycle 交给基类是合理的。

## 18. 数据流总览

可以把整个流程总结成：

```text
FlightHelmet.gltf
        |
        v
tinyGLTF LoadASCIIFromFile
        |
        v
tinygltf::Model
        |
        +--> images   -> Texture2D -> image descriptor set(set 1)
        |
        +--> materials -> baseColorTextureIndex
        |
        +--> textures -> imageIndex
        |
        +--> scenes/nodes
                 |
                 v
              loadNode recursive traversal
                 |
                 +--> accessors/bufferViews/buffers -> CPU vertex/index arrays
                 |
                 v
              global GPU vertex/index buffers
        |
        v
render frame
        |
        +--> set 0: scene UBO(projection/view/light/viewPos)
        |
        +--> push constant: node model matrix
        |
        +--> set 1: primitive base color texture
        |
        v
vkCmdDrawIndexed per primitive
```

这张图其实就是 glTF 到 Vulkan 的核心转换路径。

## 19. 这个示例的高价值设计点

### 19.1 保留 glTF 的 scene graph

它没有把所有 primitive 简单平铺，而是保留 node parent/children。这让 push constant model matrix 能正确表达层级 transform。

### 19.2 全局 vertex/index buffer

所有 primitive 合并到一套 buffer，通过 `firstIndex/indexCount` 区分 draw range。这是高效且常见的渲染组织方式。

### 19.3 per-frame UBO + per-image texture descriptor

scene matrices 是动态数据，按 frame 复制；texture 是静态数据，不按 frame 复制。这个 descriptor 设计体现了 Vulkan 资源更新频率分层的思想。

### 19.4 push constants 用于 node matrix

node matrix 是每个 node/draw 变化的小块数据，用 push constants 比单独建 buffer 或频繁改 descriptor 更直接。

### 19.5 staging 上传几何数据

顶点和索引最终进入 device local buffer，符合真实渲染项目的性能思路。

## 20. 这个示例的局限和改进方向

这个示例刻意简化了很多 glTF 特性。写博客或继续扩展时，最好明确这些边界。

### 20.1 材质不是 PBR

glTF 2.0 的标准材质是 metallic-roughness PBR。这个示例只用了 base color texture，并用简单 diffuse/specular shader 做显示。

### 20.2 baseColorFactor 读取了但没有实际参与 shader

代码读取了：

```cpp
baseColorFactor
```

但 fragment shader 没有使用它。如果要更接近 glTF，应把 material factor 也传入 shader。

### 20.3 没有 normal map / occlusion / emissive

FlightHelmet 这类模型通常有多张 PBR texture。此示例只采样 base color map。

### 20.4 没有动画和蒙皮

代码没有处理：

- animation sampler/channel。
- skin。
- joints。
- weights。

如果加载带骨骼动画的模型，只会得到静态几何或无法正确显示。

### 20.5 shader 的光照空间处理比较简化

vertex shader 中 lighting vector 和 normal 的计算没有完整考虑 node model transform 和 normal matrix。它能用于演示，但不是严谨的生产级光照。

### 20.6 RGB 转 RGBA 时 alpha 应显式填充

`loadImages()` 中 RGB 转 RGBA 时复制了 RGB，但没有写 alpha 通道。更健壮的写法应该设置：

```cpp
rgba[3] = 255;
```

## 21. 如果继续扩展，建议怎么做

可以按下面路线逐步增强：

1. 把 `baseColorFactor` 加入 material UBO 或 push constants。
2. 支持 normal map，并在 vertex layout 中加入 tangent。
3. 支持 metallic-roughness texture，替换当前简单光照为 PBR shader。
4. 按 material 批处理 draw call，减少 descriptor set 切换。
5. 支持 alpha mask 和 alpha blend。
6. 支持多个 UV set。
7. 使用完整 `base/VulkanglTFModel`，对比教学版 loader 和工程版 loader 的差异。
8. 加入 animation 和 skinning。
9. 为每个 node 缓存 world matrix，避免每次 draw 都向上遍历 parent。
10. 用 indirect draw 或 meshlet/cluster 方案进一步优化大场景渲染。

## 22. 总结

`gltfloading.cpp` 是一个很好的“glTF 到 Vulkan”桥梁示例。它没有试图实现完整 glTF 2.0，而是把核心路径拆得足够清楚：

```text
glTF 文件
  -> tinygltf::Model
  -> image/material/texture/node/mesh/primitive
  -> CPU vertex/index arrays
  -> GPU vertex/index buffers and textures
  -> descriptor sets
  -> pipeline
  -> recursive node draw
  -> vkCmdDrawIndexed
```

如果你已经理解了 triangle 示例，那么这个文件就是下一步：从“手写三个顶点”过渡到“从真实资产文件中读取几何、材质和贴图，并组织成可渲染场景”。

它最值得学习的不是某个单独 API，而是资源组织方式：全局 geometry buffer、descriptor set 分层、push constants 传 per-node 矩阵、primitive 级 texture binding，以及 glTF scene graph 到 Vulkan draw call 的转换。
