# Vulkan 示例解析：gltfskinning.cpp 如何实现 glTF 骨骼蒙皮动画

本文分析的是 `examples/gltfskinning/gltfskinning.cpp`。它在 `gltfloading` 的基础上进一步展示了 glTF skinned animation，也就是常说的骨骼蒙皮动画：模型顶点不再只受单个 node/model matrix 影响，而是由多个 joint 矩阵按权重混合后变形。

这个示例加载的模型是：

```cpp
models/CesiumMan/glTF/CesiumMan.gltf
```

它的核心目标是讲清楚 glTF 里这些概念如何进入 Vulkan 渲染管线：

- `skin`
- `joint`
- `inverseBindMatrices`
- `JOINTS_0`
- `WEIGHTS_0`
- `animation sampler`
- `animation channel`
- shader storage buffer object, SSBO
- vertex shader skinning

如果 `gltfloading.cpp` 解决的是“如何从 glTF 文件中读取静态 mesh、材质、贴图并画出来”，那么 `gltfskinning.cpp` 解决的是下一层问题：如何根据动画更新骨骼姿态，并让每个顶点按 joint 权重在 GPU 上变形。

## 1. 示例定位：教学版 glTF skinning

文件开头说明：

```cpp
/*
 * Shows how to load and display an animated scene from a glTF file using vertex skinning
 */
```

这个示例仍然不是完整 glTF 引擎。它沿用了 `gltfloading` 的教学思路，把 glTF 的核心结构拆开写在示例内，而不是直接使用 `base/VulkanglTFModel` 的完整模型类。

它支持的重点是：

- glTF scene/node/mesh/primitive 基础结构。
- base color texture。
- 顶点 position/normal/uv/color。
- 顶点 joint indices 和 joint weights。
- skin 的 joint 列表。
- inverse bind matrices。
- LINEAR animation。
- translation / rotation / scale 动画通道。
- CPU 更新 joint matrix，GPU vertex shader 执行 skinning。

它没有完整支持：

- PBR metallic-roughness。
- normal map。
- morph target。
- 多动画混合。
- 多 skin 复杂绑定策略。
- cubic spline 动画插值。
- 非线性/step 插值。
- compute skinning。

因此它很适合学习 skinning 的主链路，而不是作为生产级 glTF runtime。

## 2. Skinning 的基本数学模型

普通静态 mesh 的 vertex shader 通常是：

```glsl
gl_Position = projection * view * model * vec4(position, 1.0);
```

也就是整个 mesh 使用同一个 model matrix。

骨骼蒙皮不一样。每个顶点会受多个 joint 影响。glTF 通常用 4 个 joint 和 4 个 weight 表示：

```text
vertex:
    JOINTS_0  = [j0, j1, j2, j3]
    WEIGHTS_0 = [w0, w1, w2, w3]
```

顶点的 skin matrix 大致是：

```text
skinMat =
    w0 * jointMatrix[j0] +
    w1 * jointMatrix[j1] +
    w2 * jointMatrix[j2] +
    w3 * jointMatrix[j3]
```

最终顶点位置：

```text
projection * view * model * skinMat * position
```

每个 `jointMatrix` 的含义是：把顶点从绑定姿态 bind pose 变换到当前动画姿态。通常可写成：

```text
jointMatrix = inverse(meshWorldMatrix) * jointWorldMatrix * inverseBindMatrix
```

这正是示例 `updateJoints()` 中做的事情。

## 3. 文件结构：头文件定义数据结构，cpp 实现流程

这个示例分成两个文件：

- `examples/gltfskinning/gltfskinning.h`
- `examples/gltfskinning/gltfskinning.cpp`

头文件中定义了两个核心类：

```cpp
class VulkanglTFModel
class VulkanExample : public VulkanExampleBase
```

`VulkanglTFModel` 是本示例自己的简化 glTF model runtime，负责：

- 加载图片、材质、纹理。
- 递归加载 node。
- 读取 vertex/index buffer。
- 读取 skin。
- 读取 animation。
- 每帧更新 joint matrices。
- 绘制 node 树。

`VulkanExample` 是示例应用层，负责：

- 创建 Vulkan pipeline。
- 创建 descriptor pool/layout/set。
- 创建 scene uniform buffer。
- 调用模型加载。
- 每帧推进动画、录制 command buffer。

## 4. VulkanglTFModel 的核心数据结构

### 4.1 Vertex：多了 jointIndices 和 jointWeights

相比 `gltfloading`，skinning 版顶点结构多了两个字段：

```cpp
struct Vertex
{
    glm::vec3 pos;
    glm::vec3 normal;
    glm::vec2 uv;
    glm::vec3 color;
    glm::vec4 jointIndices;
    glm::vec4 jointWeights;
};
```

前四个字段是静态 glTF mesh 渲染所需数据：

- position
- normal
- uv
- color

后两个字段是 skinning 的关键：

- `jointIndices`：当前顶点受哪些 joint 影响。
- `jointWeights`：每个 joint 的影响权重。

它们对应 vertex shader 输入：

```glsl
layout (location = 4) in vec4 inJointIndices;
layout (location = 5) in vec4 inJointWeights;
```

如果没有这两个属性，shader 就无法知道每个顶点该用哪些 joint matrix 进行变形。

### 4.2 Node：增加 skin 和动画 TRS

node 结构：

```cpp
struct Node
{
    Node* parent;
    uint32_t index;
    std::vector<Node*> children;
    Mesh mesh;
    glm::vec3 translation{};
    glm::vec3 scale{1.0f};
    glm::quat rotation{};
    int32_t skin = -1;
    glm::mat4 matrix;
    glm::mat4 getLocalMatrix();
};
```

相比静态 glTF loading，skinning 版 node 需要保留可动画化的 TRS：

- translation
- rotation
- scale

动画更新时会直接修改这些字段。`getLocalMatrix()` 每次根据当前 TRS 计算 node 的本地矩阵：

```cpp
return glm::translate(glm::mat4(1.0f), translation)
    * glm::mat4(rotation)
    * glm::scale(glm::mat4(1.0f), scale)
    * matrix;
```

`skin` 是 glTF node 引用的 skin index：

```cpp
node->skin = inputNode.skin;
```

如果 node 有 mesh 且 `skin >= 0`，说明这个 mesh 会被对应 skin 的 joints 驱动。

### 4.3 Skin：joint 列表、inverse bind matrices、SSBO

skin 结构：

```cpp
struct Skin
{
    std::string name;
    Node* skeletonRoot = nullptr;
    std::vector<glm::mat4> inverseBindMatrices;
    std::vector<Node*> joints;
    std::array<vks::Buffer, maxConcurrentFrames> storageBuffers;
    std::array<VkDescriptorSet, maxConcurrentFrames> descriptorSets{};
};
```

这里每个字段都很重要：

- `skeletonRoot`：骨架根节点。
- `joints`：这个 skin 使用的 joint node 列表。
- `inverseBindMatrices`：每个 joint 的 inverse bind matrix。
- `storageBuffers`：每帧一份 SSBO，传当前 joint matrices 给 vertex shader。
- `descriptorSets`：每帧一份 descriptor set，绑定对应 SSBO。

为什么 joint matrix 用 SSBO 而不是 UBO？因为 joint matrix 数量可能比较多，SSBO 更适合数组型、大小相对灵活的数据。shader 里也明确使用了 storage buffer：

```glsl
layout(std430, set = 1, binding = 0) readonly buffer JointMatrices {
    mat4 jointMatrices[];
};
```

### 4.4 Animation：sampler 和 channel

glTF animation 的结构分成 sampler 和 channel：

```cpp
struct AnimationSampler
{
    std::string interpolation;
    std::vector<float> inputs;
    std::vector<glm::vec4> outputsVec4;
};

struct AnimationChannel
{
    std::string path;
    Node* node;
    uint32_t samplerIndex;
};
```

sampler 存关键帧数据：

- `inputs`：时间戳。
- `outputsVec4`：每个时间戳对应的值。
- `interpolation`：插值方式。

channel 说明 sampler 作用到哪个 node 的哪个属性：

- `translation`
- `rotation`
- `scale`

这正好对应 glTF 的动画模型：

```text
animation.channel -> target node + target path
animation.channel -> sampler
animation.sampler -> input times + output values
```

## 5. 加载流程：从 glTF 文件到可动画模型

加载入口：

```cpp
void VulkanExample::loadglTFFile(std::string filename)
```

它先通过 tinyGLTF 读取文件：

```cpp
tinygltf::Model glTFInput;
tinygltf::TinyGLTF gltfContext;
bool fileLoaded = gltfContext.LoadASCIIFromFile(&glTFInput, &error, &warning, filename);
```

然后把 Vulkan 上下文传给模型：

```cpp
glTFModel.vulkanDevice = vulkanDevice;
glTFModel.copyQueue = queue;
```

接着创建 CPU 侧临时数组：

```cpp
std::vector<uint32_t> indexBuffer;
std::vector<VulkanglTFModel::Vertex> vertexBuffer;
```

加载顺序是：

```cpp
glTFModel.loadImages(glTFInput);
glTFModel.loadMaterials(glTFInput);
glTFModel.loadTextures(glTFInput);
glTFModel.loadNode(...);
glTFModel.loadSkins(glTFInput);
glTFModel.loadAnimations(glTFInput);
```

注意 `loadSkins()` 在 `loadNode()` 之后调用。这是必要的，因为 skin 中的 joints 是通过 node index 找 node：

```cpp
Node* node = nodeFromIndex(jointIndex);
```

只有 node 树先建立好，skin 才能把 glTF joint index 解析成 `Node*`。

## 6. loadNode：读取普通属性和 skinning 属性

`loadNode()` 保留了 glTF scene graph，同时读取 mesh primitive 的顶点和索引。

普通属性：

```cpp
POSITION
NORMAL
TEXCOORD_0
```

skinning 属性：

```cpp
JOINTS_0
WEIGHTS_0
```

读取 joint indices：

```cpp
if (glTFPrimitive.attributes.find("JOINTS_0") != glTFPrimitive.attributes.end())
{
    const tinygltf::Accessor& accessor =
        input.accessors[glTFPrimitive.attributes.find("JOINTS_0")->second];
    const tinygltf::BufferView& view = input.bufferViews[accessor.bufferView];
    jointIndicesBuffer =
        reinterpret_cast<const uint16_t*>(&(input.buffers[view.buffer].data[accessor.byteOffset + view.byteOffset]));
}
```

读取 joint weights：

```cpp
if (glTFPrimitive.attributes.find("WEIGHTS_0") != glTFPrimitive.attributes.end())
{
    const tinygltf::Accessor& accessor =
        input.accessors[glTFPrimitive.attributes.find("WEIGHTS_0")->second];
    const tinygltf::BufferView& view = input.bufferViews[accessor.bufferView];
    jointWeightsBuffer =
        reinterpret_cast<const float*>(&(input.buffers[view.buffer].data[accessor.byteOffset + view.byteOffset]));
}
```

然后组装顶点：

```cpp
vert.jointIndices = hasSkin ? glm::vec4(glm::make_vec4(&jointIndicesBuffer[v * 4])) : glm::vec4(0.0f);
vert.jointWeights = hasSkin ? glm::make_vec4(&jointWeightsBuffer[v * 4]) : glm::vec4(0.0f);
```

这一步把 glTF 的 per-vertex skinning 信息打包进最终 Vulkan vertex buffer。

有一个值得注意的工程细节：代码假设 `JOINTS_0` 是 `uint16_t`，shader 输入则用 float vec4。对于这个示例模型没问题，但完整 glTF loader 需要根据 accessor 的 component type 处理 `UNSIGNED_BYTE` 和 `UNSIGNED_SHORT` 等不同格式。

## 7. loadSkins：把 glTF skin 转成 joint matrix 数据源

核心函数：

```cpp
void VulkanglTFModel::loadSkins(tinygltf::Model& input)
```

它遍历：

```cpp
input.skins
```

### 7.1 找 skeleton root

```cpp
skins[i].skeletonRoot = nodeFromIndex(glTFSkin.skeleton);
```

glTF skin 可以指定 skeleton root。示例保存了它，但后续 joint 计算主要依赖 joints 列表和 node 层级。

### 7.2 找 joint nodes

```cpp
for (int jointIndex : glTFSkin.joints)
{
    Node* node = nodeFromIndex(jointIndex);
    if (node) {
        skins[i].joints.push_back(node);
    }
}
```

glTF skin 的 `joints` 是 node index 数组。示例把这些 index 解析成 `Node*`。

这一步很关键：动画通道更新的是 node 的 TRS；joint matrix 计算也需要 joint node 的当前 world matrix。所以 skin 不能只保存 joint index，它必须能找到实际 node。

### 7.3 读取 inverse bind matrices

```cpp
const tinygltf::Accessor& accessor = input.accessors[glTFSkin.inverseBindMatrices];
const tinygltf::BufferView& bufferView = input.bufferViews[accessor.bufferView];
const tinygltf::Buffer& buffer = input.buffers[bufferView.buffer];

skins[i].inverseBindMatrices.resize(accessor.count);
memcpy(
    skins[i].inverseBindMatrices.data(),
    &buffer.data[accessor.byteOffset + bufferView.byteOffset],
    accessor.count * sizeof(glm::mat4));
```

inverse bind matrix 的作用是把顶点从 mesh bind pose 空间带到 joint bind pose 的逆空间。实际运行时用：

```text
jointWorldMatrix * inverseBindMatrix
```

得到“从绑定姿态到当前姿态”的变换。

### 7.4 为每个 frame 创建 joint matrix SSBO

```cpp
for (auto& buffer : skins[i].storageBuffers) {
    vulkanDevice->createBuffer(
        VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
        VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
        &buffer,
        sizeof(glm::mat4) * skins[i].inverseBindMatrices.size(),
        skins[i].inverseBindMatrices.data());
    buffer.map();
}
```

每个 skin 有 `maxConcurrentFrames` 份 storage buffer。原因和 per-frame UBO 一样：CPU 每帧会更新 joint matrices，GPU 可能还在读取上一帧的数据，所以要避免覆盖正在使用的 buffer。

初始内容写的是 inverse bind matrices，之后每帧会被当前动画姿态下的 joint matrices 覆盖。

## 8. loadAnimations：读取 glTF 动画采样数据

函数：

```cpp
void VulkanglTFModel::loadAnimations(tinygltf::Model& input)
```

它读取 glTF animation 的两个部分：sampler 和 channel。

### 8.1 Sampler：时间和输出值

读取 input 时间戳：

```cpp
const tinygltf::Accessor& accessor = input.accessors[glTFSampler.input];
const float* buf = static_cast<const float*>(dataPtr);
for (size_t index = 0; index < accessor.count; index++) {
    dstSampler.inputs.push_back(buf[index]);
}
```

同时统计 animation 的起止时间：

```cpp
if (input < animations[i].start) animations[i].start = input;
if (input > animations[i].end) animations[i].end = input;
```

读取 output：

```cpp
switch (accessor.type)
{
    case TINYGLTF_TYPE_VEC3:
        outputsVec4.push_back(glm::vec4(buf[index], 0.0f));
        break;
    case TINYGLTF_TYPE_VEC4:
        outputsVec4.push_back(buf[index]);
        break;
}
```

translation 和 scale 通常是 `vec3`，rotation 是 quaternion `vec4`。为了统一存储，示例全部放进 `glm::vec4`。

### 8.2 Channel：sampler 作用到哪个 node 的哪个 path

```cpp
dstChannel.path = glTFChannel.target_path;
dstChannel.samplerIndex = glTFChannel.sampler;
dstChannel.node = nodeFromIndex(glTFChannel.target_node);
```

channel 把 sampler 和 node 连接起来：

```text
sampler: 关键帧时间和值
channel: 这些值用于哪个 node 的 translation/rotation/scale
```

这就是动画系统的核心关系。

## 9. updateAnimation：每帧推进动画

每帧在 `VulkanExample::render()` 中调用：

```cpp
if (!paused) {
    glTFModel.updateAnimation(frameTimer, currentBuffer);
}
```

`updateAnimation()` 先更新当前 frame index：

```cpp
this->currentBuffer = currentBuffer;
```

然后推进当前动画时间：

```cpp
Animation& animation = animations[activeAnimation];
animation.currentTime += deltaTime;
if (animation.currentTime > animation.end) {
    animation.currentTime -= animation.end;
}
```

这里实现了循环播放。更严谨的写法通常会考虑 `animation.start`，但这个示例模型的动画起点适配当前写法。

### 9.1 查找当前关键帧区间

对每个 channel：

```cpp
AnimationSampler& sampler = animation.samplers[channel.samplerIndex];
for (size_t i = 0; i < sampler.inputs.size() - 1; i++)
```

如果当前时间落在两个 keyframe 之间：

```cpp
if ((animation.currentTime >= sampler.inputs[i]) &&
    (animation.currentTime <= sampler.inputs[i + 1]))
```

计算插值因子：

```cpp
float a = (animation.currentTime - sampler.inputs[i]) /
          (sampler.inputs[i + 1] - sampler.inputs[i]);
```

### 9.2 根据 channel path 更新 node TRS

translation：

```cpp
channel.node->translation =
    glm::mix(sampler.outputsVec4[i], sampler.outputsVec4[i + 1], a);
```

rotation：

```cpp
glm::quat q1(...);
glm::quat q2(...);
channel.node->rotation = glm::normalize(glm::slerp(q1, q2, a));
```

scale：

```cpp
channel.node->scale =
    glm::mix(sampler.outputsVec4[i], sampler.outputsVec4[i + 1], a);
```

rotation 使用 `slerp` 是正确选择。四元数旋转不能简单线性插值，否则可能出现非匀速或不稳定旋转。

示例只支持：

```cpp
sampler.interpolation == "LINEAR"
```

遇到其他插值方式会输出提示并跳过。

### 9.3 更新 joint matrices

动画更新完 node TRS 后：

```cpp
for (auto& node : nodes) {
    updateJoints(node);
}
```

这一步把“node 当前姿态”转换成“shader 可用的 joint matrix 数组”。

## 10. updateJoints：CPU 计算每个 joint matrix

核心代码：

```cpp
if (node->skin > -1)
{
    glm::mat4 inverseTransform = glm::inverse(getNodeMatrix(node));
    Skin skin = skins[node->skin];
    size_t numJoints = (uint32_t) skin.joints.size();
    std::vector<glm::mat4> jointMatrices(numJoints);
    for (size_t i = 0; i < numJoints; i++)
    {
        jointMatrices[i] = getNodeMatrix(skin.joints[i]) * skin.inverseBindMatrices[i];
        jointMatrices[i] = inverseTransform * jointMatrices[i];
    }
    skin.storageBuffers[currentBuffer].copyTo(
        jointMatrices.data(),
        jointMatrices.size() * sizeof(glm::mat4));
}
```

可以拆成三层理解。

### 10.1 getNodeMatrix：计算 node world matrix

```cpp
glm::mat4 nodeMatrix = node->getLocalMatrix();
Node* currentParent = node->parent;
while (currentParent) {
    nodeMatrix = currentParent->getLocalMatrix() * nodeMatrix;
    currentParent = currentParent->parent;
}
return nodeMatrix;
```

它从当前 node 开始，不断乘上父节点本地矩阵，得到当前 node 的世界矩阵。

### 10.2 joint matrix 公式

对每个 joint：

```cpp
jointMatrices[i] =
    getNodeMatrix(skin.joints[i]) *
    skin.inverseBindMatrices[i];
```

这表示 joint 当前世界矩阵乘 inverse bind matrix。

然后：

```cpp
jointMatrices[i] = inverseTransform * jointMatrices[i];
```

其中：

```cpp
inverseTransform = inverse(getNodeMatrix(node));
```

`node` 是引用 skin 的 mesh node。乘上它的 inverse world matrix，可以把 joint 变换转换到 mesh node 的局部空间，和后续 shader 中的 `primitive.model` 组合起来。

最终可理解为：

```text
jointMatrix =
    inverse(meshNodeWorld) *
    jointWorld *
    inverseBindMatrix
```

### 10.3 写入当前 frame 的 SSBO

```cpp
skin.storageBuffers[currentBuffer].copyTo(...)
```

这里每帧把当前动画姿态下的 joint matrix 数组写入当前 frame 的 storage buffer。shader 通过 set 1 读取这个数组。

一个细节：代码中 `Skin skin = skins[node->skin];` 是拷贝而不是引用。由于 `vks::Buffer` 包含的是 Vulkan handle 和 mapped pointer，这里拷贝后调用 `copyTo` 仍然写向同一个底层 buffer；但从代码表达上，用 `Skin& skin = skins[node->skin];` 会更清晰，也避免误导读者以为修改的是临时对象。

## 11. Descriptor 设计：三套 set

skinning 版 pipeline layout 使用三套 descriptor set：

```cpp
Set 0 = Scene matrices (VS)
Set 1 = Joint matrices (VS)
Set 2 = Material texture (FS)
```

代码：

```cpp
std::array<VkDescriptorSetLayout, 3> setLayouts = {
    descriptorSetLayouts.matrices,
    descriptorSetLayouts.jointMatrices,
    descriptorSetLayouts.textures
};
```

这和 shader 完全对应。

### 11.1 set 0：Scene UBO

vertex shader：

```glsl
layout (set = 0, binding = 0) uniform UBOScene
{
    mat4 projection;
    mat4 view;
    vec4 lightPos;
} uboScene;
```

C++：

```cpp
struct UniformData
{
    glm::mat4 projection;
    glm::mat4 model;
    glm::vec4 lightPos = glm::vec4(5.0f, 5.0f, 5.0f, 1.0f);
} uniformData;
```

字段名 `model` 实际上传的是 view matrix：

```cpp
uniformData.model = camera.matrices.view;
```

字段名不影响 GPU，内存顺序才重要。shader 的 `view` 对应 C++ 中第二个 `mat4`。

### 11.2 set 1：Joint Matrices SSBO

descriptor set layout：

```cpp
setLayoutBinding =
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_STORAGE_BUFFER,
        VK_SHADER_STAGE_VERTEX_BIT,
        0);
```

每个 skin、每个 frame 都分配一个 descriptor set：

```cpp
for (auto& skin : glTFModel.skins) {
    vkAllocateDescriptorSets(..., &skin.descriptorSets[i]);
    VkWriteDescriptorSet writeDescriptorSet =
        vks::initializers::writeDescriptorSet(
            skin.descriptorSets[i],
            VK_DESCRIPTOR_TYPE_STORAGE_BUFFER,
            0,
            &skin.storageBuffers[i].descriptor);
    vkUpdateDescriptorSets(...);
}
```

为什么要 per-frame？因为 joint matrices 每帧变化，而且 GPU 可能还在使用上一帧。

为什么要 per-skin？因为不同 mesh node 可能引用不同 skin，不同 skin 有不同 joint 列表和 matrix 数组。

### 11.3 set 2：Material Texture

fragment shader：

```glsl
layout (set = 2, binding = 0) uniform sampler2D samplerColorMap;
```

C++ 为每张 image 创建 descriptor set：

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

纹理是静态的，所以不需要 per-frame 复制。

## 12. Pipeline：顶点输入增加 JOINTS_0 / WEIGHTS_0

pipeline 的 vertex input 定义：

```cpp
const std::vector<VkVertexInputAttributeDescription> vertexInputAttributes = {
    {0, 0, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, pos)},
    {1, 0, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, normal)},
    {2, 0, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, uv)},
    {3, 0, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, color)},
    {4, 0, VK_FORMAT_R32G32B32A32_SFLOAT, offsetof(Vertex, jointIndices)},
    {5, 0, VK_FORMAT_R32G32B32A32_SFLOAT, offsetof(Vertex, jointWeights)},
};
```

这里意图是把属性绑定到 shader location 0 到 5。按照 Vulkan 结构体字段顺序，`VkVertexInputAttributeDescription` 是：

```cpp
location, binding, format, offset
```

所以 `{4, 0, ...}` 表示 location 4、binding 0。它对应：

```glsl
layout (location = 4) in vec4 inJointIndices;
layout (location = 5) in vec4 inJointWeights;
```

一个小问题是 UV 字段是 `glm::vec2`，但这里给的是：

```cpp
VK_FORMAT_R32G32B32_SFLOAT
```

更严格的格式应该是 `VK_FORMAT_R32G32_SFLOAT`。当前写法会让 GPU 对 location 2 读取 3 个 float，而 `uv` 后面紧跟 `color`，因此第三个分量会读到 color 的第一个 float。shader 只声明 `vec2 inUV`，实际只使用前两个分量，所以这个问题通常不会显现；但从工程质量上讲，UV attribute 应该改成 `R32G32`。

## 13. Vertex Shader：真正执行 skinning 的地方

shader：`shaders/glsl/gltfskinning/skinnedmodel.vert`

关键输入：

```glsl
layout (location = 4) in vec4 inJointIndices;
layout (location = 5) in vec4 inJointWeights;
```

joint matrices SSBO：

```glsl
layout(std430, set = 1, binding = 0) readonly buffer JointMatrices {
    mat4 jointMatrices[];
};
```

核心 skinning 代码：

```glsl
mat4 skinMat =
    inJointWeights.x * jointMatrices[int(inJointIndices.x)] +
    inJointWeights.y * jointMatrices[int(inJointIndices.y)] +
    inJointWeights.z * jointMatrices[int(inJointIndices.z)] +
    inJointWeights.w * jointMatrices[int(inJointIndices.w)];
```

顶点位置：

```glsl
gl_Position =
    uboScene.projection *
    uboScene.view *
    primitive.model *
    skinMat *
    vec4(inPos.xyz, 1.0);
```

这就是 GPU vertex skinning。CPU 只负责每帧更新 jointMatrices，真正的逐顶点变形在 vertex shader 中完成。

法线也使用 skinMat：

```glsl
outNormal =
    normalize(transpose(inverse(mat3(uboScene.view * primitive.model * skinMat))) * inNormal);
```

相比 `gltfloading` 示例，这里对 normal 的处理更合理，因为顶点已经被 skinMat 变形，normal 也应跟随变换。

## 14. Fragment Shader：简单贴图和光照

fragment shader 使用 set 2 的 base color texture：

```glsl
layout (set = 2, binding = 0) uniform sampler2D samplerColorMap;
```

核心逻辑：

```glsl
vec4 color = texture(samplerColorMap, inUV) * vec4(inColor, 1.0);

vec3 N = normalize(inNormal);
vec3 L = normalize(inLightVec);
vec3 V = normalize(inViewVec);
vec3 R = reflect(-L, N);
vec3 diffuse = max(dot(N, L), 0.5) * inColor;
vec3 specular = pow(max(dot(R, V), 0.0), 16.0) * vec3(0.75);
outFragColor = vec4(diffuse * color.rgb + specular, 1.0);
```

这不是 glTF PBR，而是一个教学用的简单 diffuse + specular 模型。对于 CesiumMan 这种示例资产，足够展示动画变形和基本材质。

## 15. drawNode：绑定 skin SSBO 和 material texture

绘制入口：

```cpp
void VulkanglTFModel::draw(VkCommandBuffer commandBuffer, VkPipelineLayout pipelineLayout)
```

先绑定全局 vertex/index buffer：

```cpp
vkCmdBindVertexBuffers(commandBuffer, 0, 1, &vertices.buffer, offsets);
vkCmdBindIndexBuffer(commandBuffer, indices.buffer, 0, VK_INDEX_TYPE_UINT32);
```

然后递归绘制 node：

```cpp
drawNode(commandBuffer, pipelineLayout, *node);
```

### 15.1 Push constant：primitive model matrix

```cpp
vkCmdPushConstants(
    commandBuffer,
    pipelineLayout,
    VK_SHADER_STAGE_VERTEX_BIT,
    0,
    sizeof(glm::mat4),
    &nodeMatrix);
```

push constant 对应 shader：

```glsl
layout(push_constant) uniform PushConsts {
    mat4 model;
} primitive;
```

这里有一个区别于 `updateJoints()` 的点：`drawNode()` 中计算 nodeMatrix 使用的是 `node.matrix` 和 parent 的 `matrix`，而不是 `getLocalMatrix()`。对 CesiumMan 这个 skinned mesh 来说，主要动画形变来自 joint SSBO；mesh node 自身通常不依赖每帧 TRS 动画。若要让普通 animated node transform 也体现在绘制 model matrix 上，应考虑使用 `getNodeMatrix()`。

### 15.2 set 1：绑定当前 skin 的 joint matrices

```cpp
vkCmdBindDescriptorSets(
    commandBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    1,
    1,
    &skins[node.skin].descriptorSets[currentBuffer],
    0,
    nullptr);
```

这里绑定 descriptor set 1，也就是当前 node 所引用 skin 的 SSBO。

这一步决定 vertex shader 中：

```glsl
jointMatrices[int(inJointIndices.x)]
```

读到的是哪个 skin 的 joint matrix 数组。

### 15.3 set 2：绑定 primitive 材质纹理

```cpp
VulkanglTFModel::Texture texture =
    textures[materials[primitive.materialIndex].baseColorTextureIndex];

vkCmdBindDescriptorSets(
    commandBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    2,
    1,
    &images[texture.imageIndex].descriptorSet,
    0,
    nullptr);
```

然后绘制：

```cpp
vkCmdDrawIndexed(commandBuffer, primitive.indexCount, 1, primitive.firstIndex, 0, 0);
```

所以每个 primitive 的 draw call 绑定了：

- set 0：当前 frame 的 scene UBO。
- set 1：当前 skin 的 joint matrix SSBO。
- set 2：当前 material 的 texture。

## 16. render：每帧动画和绘制顺序

主循环：

```cpp
void VulkanExample::render()
{
    if (!prepared)
        return;
    VulkanExampleBase::prepareFrame();
    updateUniformBuffers();
    if (!paused) {
        glTFModel.updateAnimation(frameTimer, currentBuffer);
    }
    buildCommandBuffer();
    VulkanExampleBase::submitFrame();
    updateUniformBuffers();
}
```

关键顺序是：

1. `prepareFrame()`：获取当前 swapchain image，选择 currentBuffer。
2. `updateUniformBuffers()`：更新 projection/view/light。
3. `updateAnimation()`：推进动画，更新 node TRS，计算 joint matrices，写入当前 frame SSBO。
4. `buildCommandBuffer()`：绑定 descriptor、pipeline、vertex/index buffer，录制 draw。
5. `submitFrame()`：提交并 present。

最后又调用一次 `updateUniformBuffers()`，对当前逻辑来说不是必要步骤。它会在 `submitFrame()` 推进 `currentBuffer` 后更新下一帧 UBO，但下一帧开头仍然会再更新一次。因此可以视为冗余调用，不影响正确性。

## 17. 数据流总览

可以把整个 skinning 示例总结为：

```text
CesiumMan.gltf
        |
        v
tinyGLTF
        |
        +--> mesh primitive attributes
        |       POSITION / NORMAL / TEXCOORD_0 / JOINTS_0 / WEIGHTS_0
        |       |
        |       v
        |   global vertex buffer + index buffer
        |
        +--> skins
        |       joints -> Node*
        |       inverseBindMatrices -> CPU array
        |       |
        |       v
        |   per-frame SSBO for joint matrices
        |
        +--> animations
        |       samplers: keyframe times and values
        |       channels: target node and path
        |
        v
每帧 render
        |
        +--> updateAnimation(frameTimer)
        |       interpolate TRS
        |       update node transforms
        |       compute joint matrices
        |       copy joint matrices to SSBO(set 1)
        |
        +--> update scene UBO(set 0)
        |
        +--> drawNode
                bind skin SSBO(set 1)
                bind material texture(set 2)
                vkCmdDrawIndexed
        |
        v
vertex shader
        |
        +--> skinMat = weighted sum of joint matrices
        +--> position = projection * view * model * skinMat * pos
```

这条链路就是 glTF skinning 在 Vulkan 中的核心实现。

## 18. 这个示例的高价值设计点

### 18.1 动画系统和渲染系统解耦

动画更新只修改 node TRS 和 joint matrix SSBO。绘制阶段只绑定 SSBO 并 draw。这种分层很清晰：

- CPU animation update。
- GPU vertex skinning。

### 18.2 Joint matrices 使用 SSBO

SSBO 比 UBO 更适合 joint matrix 数组，特别是 joint 数量增加时。shader 中用 runtime array：

```glsl
mat4 jointMatrices[];
```

这比固定大小 uniform array 更灵活。

### 18.3 Per-frame storage buffer 避免同步问题

每个 skin 都有 `maxConcurrentFrames` 份 storage buffer。CPU 更新当前 frame 的 joint matrices 时，不会覆盖 GPU 正在读取的上一帧数据。

### 18.4 Shader 侧执行 vertex skinning

CPU 不修改 vertex buffer，只更新 joint matrices。每个顶点的变形在 vertex shader 中按 joint weights 计算。这是实时渲染中最常见的 skinning 路径。

### 18.5 Descriptor set 按更新频率拆分

三套 descriptor set 的职责非常清楚：

- set 0：每帧变化的 scene UBO。
- set 1：每帧变化、每个 skin 不同的 joint matrices。
- set 2：静态 material texture。

这种拆分方式比把所有资源塞进一个 descriptor set 更利于扩展和更新。

## 19. 局限和可以改进的地方

### 19.1 只支持 LINEAR 插值

代码遇到非 LINEAR 插值会跳过：

```cpp
if (sampler.interpolation != "LINEAR") {
    std::cout << "This sample only supports linear interpolations\n";
    continue;
}
```

完整 glTF 需要支持 `STEP` 和 `CUBICSPLINE`。

### 19.2 JOINTS_0 component type 假设较强

示例把 joint indices 当作 `uint16_t` 读取。完整 loader 应该根据 accessor.componentType 分别处理 `UNSIGNED_BYTE` 和 `UNSIGNED_SHORT`。

### 19.3 UV attribute 格式应为 R32G32

pipeline 中 UV 使用了 `VK_FORMAT_R32G32B32_SFLOAT`，但 vertex 里 `uv` 是 `glm::vec2`。更严谨应改为 `VK_FORMAT_R32G32_SFLOAT`。

### 19.4 drawNode 对 animated node matrix 的处理较简化

`updateJoints()` 用 `getLocalMatrix()`，但 `drawNode()` push constant 使用 `node.matrix`。对这个示例的 skinned model 没有明显问题，但如果 mesh node 本身也有动画，绘制 model matrix 应该统一走 `getNodeMatrix()`。

### 19.5 没有动画选择和混合

只有：

```cpp
uint32_t activeAnimation = 0;
```

没有多动画切换 UI，也没有 cross-fade / blend tree。

### 19.6 没有 compute skinning

当前是 vertex shader skinning。更复杂场景可能会改成 compute shader 预先写 skinned vertex buffer，以便后续 shadow、velocity、culling、meshlet 等 pass 复用。

## 20. 如果继续扩展，建议路线

1. 修正 UV vertex attribute format 为 `VK_FORMAT_R32G32_SFLOAT`。
2. 支持 `JOINTS_0` 的 `UNSIGNED_BYTE` 和 `UNSIGNED_SHORT` 两种 component type。
3. 支持 `STEP` 和 `CUBICSPLINE` animation interpolation。
4. 把 `Skin skin = skins[node->skin]` 改成引用，提升代码表达准确性。
5. 统一 drawNode 的 node matrix 计算，支持 mesh node 自身动画。
6. 加入 animation selection UI。
7. 支持多个动画混合。
8. 支持 tangent 和 normal map。
9. 用 PBR shader 替换简单 diffuse/specular。
10. 尝试 compute skinning，并对比 vertex shader skinning 的优缺点。

## 21. 总结

`gltfskinning.cpp` 的核心是把 glTF 的骨骼动画数据拆成两条流：

```text
动画流：
    animation sampler/channel
    -> 更新 node translation/rotation/scale
    -> 计算 joint world matrix
    -> jointMatrix = inverse(meshWorld) * jointWorld * inverseBindMatrix
    -> 写入 SSBO

渲染流：
    vertex buffer 中包含 JOINTS_0 / WEIGHTS_0
    -> draw 时绑定 scene UBO、skin SSBO、material texture
    -> vertex shader 按权重混合 joint matrices
    -> 输出变形后的顶点
```

理解这个示例后，glTF skinning 不再是黑盒。它本质上就是：CPU 根据动画更新骨骼矩阵，GPU 根据每个顶点的 joint indices 和 weights 做矩阵加权变形。

如果你已经读过 `gltfloading.cpp`，那么 `gltfskinning.cpp` 是非常自然的下一步：它在静态 glTF 渲染的基础上增加了“时间维度”和“骨骼变形维度”，展示了真实角色动画进入 Vulkan 管线的关键路径。
