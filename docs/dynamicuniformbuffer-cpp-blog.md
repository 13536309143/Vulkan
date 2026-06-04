# Vulkan 示例解析：dynamicuniformbuffer.cpp 如何用 Dynamic Uniform Buffer 批量绘制对象

本文分析的是 `examples/dynamicuniformbuffer/dynamicuniformbuffer.cpp`[dynamicuniformbuffer.cpp](https://github.com/13536309143/Vulkan/blob/main/examples/dynamicuniformbuffer/dynamicuniformbuffer.cpp)。这个示例展示了 Vulkan 中一个非常实用的资源绑定技巧：**Dynamic Uniform Buffer**。

它要解决的问题很典型：如果场景里有很多对象，每个对象都有自己的 model matrix，该怎么把这些矩阵高效传给 shader？

最直观的做法是：

```text
每个对象一个 uniform buffer
每个对象一个 descriptor set
每次 draw 前绑定对应 descriptor set
```

这当然能工作，但当对象数量变多时，buffer 数量、descriptor set 数量、descriptor 更新和绑定管理都会变得繁琐。

`dynamicuniformbuffer.cpp` 展示了另一种方式：

```text
创建一个大 uniform buffer
把所有对象的 model matrix 按固定间距写进去
descriptor set 只绑定这个大 buffer
每次 draw 前通过 dynamic offset 选择当前对象的数据段
```

示例中一共绘制：

```cpp
constexpr auto OBJECT_INSTANCES = 125;
```

也就是 125 个立方体。它们共享同一个 vertex/index buffer、同一条 pipeline、同一个 descriptor set，但每次 draw 时传入不同的 dynamic offset，所以 vertex shader 读到不同的 model matrix，最终形成一个 5 x 5 x 5 的立方体阵列。

这篇文章会从 Vulkan 资源绑定模型、内存对齐、descriptor layout、dynamic offset、command buffer 绘制流程几个角度完整拆解这个示例。

## 1. 示例目标：用一个 UBO 存放多个对象的矩阵

源码开头的注释已经把主题说明得很清楚：

```cpp
/*
* Demonstrates the use of dynamic uniform buffers.
*
* Instead of using one uniform buffer per-object, this example allocates one big uniform buffer
* with respect to the alignment reported by the device via minUniformBufferOffsetAlignment that
* contains all matrices for the objects in the scene.
*
* The used descriptor type VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC then allows to set a dynamic
* offset used to pass data from the single uniform buffer to the connected shader binding point.
*/
```

这段话里有三个关键词。

第一，`one big uniform buffer`：不是每个对象一个 UBO，而是所有对象共享一个大 UBO。

第二，`minUniformBufferOffsetAlignment`：每个对象的数据段不能随便紧密排列，必须满足 GPU 的 uniform buffer offset 对齐要求。

第三，`VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC`：descriptor 类型不是普通 `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER`，而是 dynamic uniform buffer。它允许在绑定 descriptor set 时额外传入一个 offset。

最终每个对象的 draw call 类似这样：

```cpp
uint32_t dynamicOffset = j * static_cast<uint32_t>(dynamicAlignment);

vkCmdBindDescriptorSets(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    0,
    1,
    &descriptorSet,
    1,
    &dynamicOffset);

vkCmdDrawIndexed(cmdBuffer, indexCount, 1, 0, 0, 0);
```

这里 `descriptorSet` 是同一个，但 `dynamicOffset` 每次不同。Vulkan 会把这个 offset 应用到 dynamic uniform buffer binding 上，使 shader 看到大 buffer 中不同位置的数据。

## 2. 普通 Uniform Buffer 和 Dynamic Uniform Buffer 的区别

先看普通 uniform buffer 的使用方式。

如果 shader 里有：

```glsl
layout (binding = 0) uniform UBO {
    mat4 model;
} ubo;
```

C++ 侧通常创建一个 `VkDescriptorBufferInfo`：

```cpp
VkDescriptorBufferInfo info{};
info.buffer = buffer;
info.offset = 0;
info.range = sizeof(ModelData);
```

然后通过 `vkUpdateDescriptorSets()` 把这个 buffer range 写入 descriptor set。

普通 uniform buffer 的 offset 和 range 基本在 descriptor 更新时确定。绘制时绑定 descriptor set，shader 读取的就是这个固定 range。

Dynamic uniform buffer 的思路不同。

descriptor 里仍然写入一个 buffer 和一个 range，但这个 range 的起始位置可以在绑定 descriptor set 时再额外偏移。

可以理解成：

```text
普通 UBO:
    shader 看到 descriptor 中固定的 buffer offset

Dynamic UBO:
    shader 看到 descriptor 中的 base offset + bind 时传入的 dynamic offset
```

因此 dynamic uniform buffer 适合这种场景：

```text
同一种数据结构重复出现很多次
每次 draw 只需要选择其中一份
每份数据大小较小，例如一个 model matrix 或一组对象参数
```

这个示例中，每个对象的数据就是一个：

```cpp
glm::mat4
```

也就是 model matrix。

## 3. 程序整体结构

示例继承自：

```cpp
class VulkanExample : public VulkanExampleBase
```

核心成员如下：

```cpp
vks::Buffer vertexBuffer;
vks::Buffer indexBuffer;
uint32_t indexCount{ 0 };
```

这三个负责 cube 的几何数据。

uniform buffer 被分成两类：

```cpp
struct UniformBuffers {
    vks::Buffer view;
    vks::Buffer dynamic;
};

std::array<UniformBuffers, maxConcurrentFrames> uniformBuffers;
```

`view` 是普通 uniform buffer，用来存放 projection 和 view 矩阵。

`dynamic` 是 dynamic uniform buffer，用来存放 125 个对象的 model matrix。

普通 UBO 的 CPU 侧数据是：

```cpp
struct {
    glm::mat4 projection;
    glm::mat4 view;
} uboVS;
```

dynamic UBO 的 CPU 侧数据是：

```cpp
struct UboDataDynamic {
    glm::mat4* model{ nullptr };
} uboDataDynamic;
```

这里没有使用 `std::vector<glm::mat4>`，而是手动 aligned allocation。原因后面会详细讲：dynamic uniform buffer 每个对象的数据起始地址必须满足 `minUniformBufferOffsetAlignment`。

每个对象还有随机旋转数据：

```cpp
glm::vec3 rotations[OBJECT_INSTANCES];
glm::vec3 rotationSpeeds[OBJECT_INSTANCES];
```

Vulkan 对象包括：

```cpp
VkPipeline pipeline{ VK_NULL_HANDLE };
VkPipelineLayout pipelineLayout{ VK_NULL_HANDLE };
VkDescriptorSet descriptorSet{ VK_NULL_HANDLE };
VkDescriptorSetLayout descriptorSetLayout{ VK_NULL_HANDLE };

size_t dynamicAlignment{ 0 };
```

`dynamicAlignment` 是这个示例的关键变量。它表示 dynamic UBO 中相邻两个对象数据的间隔。

## 4. 渲染目标：125 个立方体组成 5 x 5 x 5 阵列

示例定义：

```cpp
constexpr auto OBJECT_INSTANCES = 125;
```

后续更新动态矩阵时，会计算：

```cpp
uint32_t dim =
    static_cast<uint32_t>(pow(OBJECT_INSTANCES, (1.0f / 3.0f)));
```

125 的立方根是 5，因此：

```text
dim = 5
```

三层循环：

```cpp
for (uint32_t x = 0; x < dim; x++) {
    for (uint32_t y = 0; y < dim; y++) {
        for (uint32_t z = 0; z < dim; z++) {
            const uint32_t index = x * dim * dim + y * dim + z;
            ...
        }
    }
}
```

最终生成 5 x 5 x 5 个对象的 model matrix。

这些对象共享同一个 cube mesh，但每个对象的位置和旋转不同。

## 5. 生成 Cube 几何数据

示例没有使用 glTF 模型，而是在 `generateCube()` 中手写了一个彩色 cube。

顶点结构：

```cpp
struct Vertex {
    float pos[3];
    float color[3];
};
```

每个顶点包含：

- position
- color

顶点数组有 8 个顶点，index 数组组成 12 个三角形：

```cpp
std::vector<uint32_t> indices = {
    0,1,2, 2,3,0,
    1,5,6, 6,2,1,
    7,6,5, 5,4,7,
    4,0,3, 3,7,4,
    4,5,1, 1,0,4,
    3,2,6, 6,7,3,
};
```

然后创建 vertex buffer 和 index buffer：

```cpp
vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_VERTEX_BUFFER_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
    VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
    &vertexBuffer,
    vertices.size() * sizeof(Vertex),
    vertices.data());

vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_INDEX_BUFFER_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
    VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
    &indexBuffer,
    indices.size() * sizeof(uint32_t),
    indices.data());
```

源码注释也说明了：为了简单，这里没有 staging 到 device local memory。真实项目中静态 vertex/index buffer 通常会先上传到 staging buffer，再 copy 到 `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT` 的 GPU 本地内存。

这个示例的重点不是几何上传性能，而是 uniform buffer 的动态寻址。

## 6. Shader 接口：一个普通 UBO，一个 dynamic UBO

vertex shader 位于：

```text
shaders/glsl/dynamicuniformbuffer/base.vert
```

输入属性：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inColor;
```

普通 view UBO：

```glsl
layout (binding = 0) uniform UboView
{
    mat4 projection;
    mat4 view;
} uboView;
```

对象级 UBO：

```glsl
layout (binding = 1) uniform UboInstance
{
    mat4 model;
} uboInstance;
```

注意 shader 本身并不知道 binding 1 是普通 uniform buffer 还是 dynamic uniform buffer。GLSL 里声明仍然是普通 uniform block：

```glsl
layout (binding = 1) uniform UboInstance
```

“dynamic” 这个行为发生在 Vulkan API 侧：

- descriptor set layout 把 binding 1 声明为 `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC`
- 绑定 descriptor set 时传入 dynamic offset

shader 中正常读取：

```glsl
mat4 modelView = uboView.view * uboInstance.model;
gl_Position = uboView.projection * modelView * vec4(inPos.xyz, 1.0);
```

fragment shader 非常简单：

```glsl
layout (location = 0) in vec3 inColor;
layout (location = 0) out vec4 outFragColor;

void main()
{
    outFragColor = vec4(inColor, 1.0);
}
```

这个示例不使用纹理、不使用光照，就是为了让所有注意力集中在 dynamic uniform buffer 上。

## 7. 为什么必须关心 minUniformBufferOffsetAlignment

Dynamic uniform buffer 最大的坑是 offset 对齐。

Vulkan 规范要求：用于 dynamic uniform buffer 的 dynamic offset 必须是设备限制：

```cpp
minUniformBufferOffsetAlignment
```

的整数倍。

这个限制来自：

```cpp
vulkanDevice->properties.limits.minUniformBufferOffsetAlignment
```

不同 GPU 上这个值可能不同。常见值可能是 64、128、256 等。

而一个 `glm::mat4` 的大小通常是：

```text
64 bytes
```

如果某个设备要求 alignment 是 256，那么虽然每个对象只需要 64 bytes，实际布局也必须变成：

```text
object 0: offset 0
object 1: offset 256
object 2: offset 512
object 3: offset 768
...
```

中间的空间是 padding。不能简单按：

```text
offset = index * sizeof(glm::mat4)
```

否则当 alignment 大于 64 时，dynamic offset 不合法，validation layer 会报错，GPU 读取也可能出问题。

示例中对齐计算如下：

```cpp
size_t minUboAlignment =
    vulkanDevice->properties.limits.minUniformBufferOffsetAlignment;

dynamicAlignment = sizeof(glm::mat4);

if (minUboAlignment > 0) {
    dynamicAlignment =
        (dynamicAlignment + minUboAlignment - 1) &
        ~(minUboAlignment - 1);
}
```

这段代码的作用是：把 `sizeof(glm::mat4)` 向上取整到 `minUboAlignment` 的倍数。

例如：

```text
sizeof(glm::mat4) = 64
minUniformBufferOffsetAlignment = 256

dynamicAlignment = 256
```

如果 alignment 是 64：

```text
dynamicAlignment = 64
```

如果 alignment 是 128：

```text
dynamicAlignment = 128
```

后续每个对象的 offset 都按 `dynamicAlignment` 计算，而不是按 `sizeof(glm::mat4)` 计算。

这是 dynamic uniform buffer 的核心工程细节。

## 8. 为什么要手动 alignedAlloc

示例定义了两个跨平台辅助函数：

```cpp
void* alignedAlloc(size_t size, size_t alignment)
{
    void *data = nullptr;
#if defined(_MSC_VER) || defined(__MINGW32__)
    data = _aligned_malloc(size, alignment);
#else
    int res = posix_memalign(&data, alignment, size);
    if (res != 0)
        data = nullptr;
#endif
    return data;
}

void alignedFree(void* data)
{
#if defined(_MSC_VER) || defined(__MINGW32__)
    _aligned_free(data);
#else
    free(data);
#endif
}
```

CPU 侧也要按 `dynamicAlignment` 来组织数据。因为代码会通过指针偏移写入每个对象的 matrix：

```cpp
glm::mat4* modelMat =
    (glm::mat4*)(((uint64_t)uboDataDynamic.model +
    (index * dynamicAlignment)));
```

如果 CPU 侧分配的内存没有满足对齐要求，虽然最终 copy 到 GPU buffer 时仍然是字节拷贝，但本地写入和调试都更容易出问题。示例选择 aligned allocation，保证 CPU 侧临时数据布局和 GPU 侧 dynamic buffer 布局一致。

分配大小是：

```cpp
size_t bufferSize = OBJECT_INSTANCES * dynamicAlignment;

uboDataDynamic.model =
    (glm::mat4*)alignedAlloc(bufferSize, dynamicAlignment);
```

也就是说，如果 125 个对象、每个对象对齐后占 256 bytes，那么 dynamic buffer 总大小是：

```text
125 * 256 = 32000 bytes
```

其中真正的矩阵数据只有：

```text
125 * 64 = 8000 bytes
```

剩下的是为了满足 uniform buffer dynamic offset alignment 产生的 padding。

这是一种典型的 GPU 数据布局成本：为了换取合法访问和更好的硬件对齐，需要牺牲一些内存空间。

## 9. 创建 Uniform Buffers

`prepareUniformBuffers()` 中为每个并发帧创建两类 buffer。

第一类是普通 view uniform buffer：

```cpp
vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
    VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
    &buffer.view,
    sizeof(uboVS));
```

它保存 projection 和 view 矩阵，数据量很小，并且每帧更新。这里使用 host coherent memory，因此写入后不需要手动 flush。

第二类是 dynamic uniform buffer：

```cpp
vulkanDevice->createBuffer(
    VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT,
    &buffer.dynamic,
    bufferSize);
```

注意这里没有使用：

```cpp
VK_MEMORY_PROPERTY_HOST_COHERENT_BIT
```

因此更新后必须手动 flush：

```cpp
vkFlushMappedMemoryRanges(device, 1, &memoryRange);
```

为什么 dynamic buffer 不使用 host coherent？

示例的 README 解释了工程动机：真实项目中通常只更新 dynamic buffer 的一部分，例如只更新移动过的对象，然后手动 flush 对应范围。手动 flush 可以更精确地控制可见性范围，避免 coherent memory 在某些平台上的潜在开销。

这个示例为了简单，每帧更新整个 dynamic buffer，然后 flush 整个 range。

## 10. descriptor.range 为什么设为 dynamicAlignment

创建 dynamic buffer 后，示例做了一个重要设置：

```cpp
buffer.dynamic.descriptor.range = dynamicAlignment;
```

这表示 descriptor 绑定的可见 range 是一个对象的数据段大小。

对于 dynamic uniform buffer，最终 shader 看到的 range 可以理解为：

```text
buffer.dynamic.descriptor.offset + dynamicOffset
到
buffer.dynamic.descriptor.offset + dynamicOffset + descriptor.range
```

示例中 base offset 通常是 0，range 是 `dynamicAlignment`。

每次 draw 时传入：

```cpp
dynamicOffset = objectIndex * dynamicAlignment;
```

所以 shader 的 binding 1 每次看到的是：

```text
第 objectIndex 个对象的数据段
```

虽然 shader 只读取 `mat4 model`，也就是 64 bytes，但 descriptor range 设为 `dynamicAlignment` 是合理的，因为每个对象在 buffer 中占据的是一个对齐后的槽位。

也可以把它理解为：

```text
dynamicAlignment 是对象槽位大小
sizeof(glm::mat4) 是槽位中真正有意义的数据大小
```

## 11. 初始化随机旋转数据

Uniform buffer 创建完成后，示例为每个对象准备随机旋转和旋转速度：

```cpp
std::default_random_engine rndEngine(
    benchmark.active ? 0 : (unsigned)time(nullptr));

std::normal_distribution<float> rndDist(-1.0f, 1.0f);

for (uint32_t i = 0; i < OBJECT_INSTANCES; i++) {
    rotations[i] =
        glm::vec3(
            rndDist(rndEngine),
            rndDist(rndEngine),
            rndDist(rndEngine)) *
        2.0f * (float)M_PI;

    rotationSpeeds[i] =
        glm::vec3(
            rndDist(rndEngine),
            rndDist(rndEngine),
            rndDist(rndEngine));
}
```

如果 benchmark 模式开启，随机种子固定为 0，保证每次运行结果可复现。否则使用当前时间作为随机种子。

这些旋转数据每帧会参与 model matrix 更新。

## 12. Descriptor Pool：普通 UBO 和 Dynamic UBO 各需要容量

descriptor setup 在 `setupDescriptors()` 中。

首先创建 descriptor pool：

```cpp
std::vector<VkDescriptorPoolSize> poolSizes = {
    vks::initializers::descriptorPoolSize(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        maxConcurrentFrames),

    vks::initializers::descriptorPoolSize(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC,
        maxConcurrentFrames)
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

pool 中有两类 descriptor：

- `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER`
- `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC`

这两个是不同 descriptor type。dynamic uniform buffer 不能用普通 uniform buffer descriptor 替代。

示例按 `maxConcurrentFrames` 预留容量，因为 uniform buffers 是按 frame-in-flight 复制的。

## 13. Descriptor Set Layout：binding 1 是 Dynamic UBO

descriptor set layout 有两个 binding：

```cpp
std::vector<VkDescriptorSetLayoutBinding> setLayoutBindings = {
    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        VK_SHADER_STAGE_VERTEX_BIT,
        0),

    vks::initializers::descriptorSetLayoutBinding(
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC,
        VK_SHADER_STAGE_VERTEX_BIT,
        1)
};
```

binding 0：

```text
type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER
stage = vertex
binding = 0
```

对应 shader：

```glsl
layout (binding = 0) uniform UboView
{
    mat4 projection;
    mat4 view;
} uboView;
```

binding 1：

```text
type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC
stage = vertex
binding = 1
```

对应 shader：

```glsl
layout (binding = 1) uniform UboInstance
{
    mat4 model;
} uboInstance;
```

这就是 dynamic uniform buffer 最关键的声明：shader 看起来仍然是普通 uniform block，但 descriptor set layout 告诉 Vulkan binding 1 是 dynamic 类型。

创建 layout：

```cpp
VkDescriptorSetLayoutCreateInfo descriptorLayout =
    vks::initializers::descriptorSetLayoutCreateInfo(setLayoutBindings);

VK_CHECK_RESULT(vkCreateDescriptorSetLayout(
    device,
    &descriptorLayout,
    nullptr,
    &descriptorSetLayout));
```

## 14. Descriptor Set：绑定 view buffer 和 dynamic buffer

示例分配 descriptor set，并写入两个 binding：

```cpp
VK_CHECK_RESULT(vkAllocateDescriptorSets(
    device,
    &allocInfo,
    &descriptorSet));

std::vector<VkWriteDescriptorSet> writeDescriptorSets = {
    vks::initializers::writeDescriptorSet(
        descriptorSet,
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        0,
        &uniformBuffers[currentBuffer].view.descriptor),

    vks::initializers::writeDescriptorSet(
        descriptorSet,
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC,
        1,
        &uniformBuffers[currentBuffer].dynamic.descriptor),
};

vkUpdateDescriptorSets(
    device,
    static_cast<uint32_t>(writeDescriptorSets.size()),
    writeDescriptorSets.data(),
    0,
    nullptr);
```

从概念上看，descriptor set 中的资源表是：

```text
set 0
    binding 0 -> 当前帧 view uniform buffer
    binding 1 -> 当前帧 dynamic uniform buffer
```

后续 draw 时，binding 0 固定读取 projection/view，binding 1 根据 dynamic offset 读取不同对象的 model matrix。

### 代码阅读注意

当前文件中 `descriptorSet` 是单个句柄：

```cpp
VkDescriptorSet descriptorSet{ VK_NULL_HANDLE };
```

但 `setupDescriptors()` 中又按 `uniformBuffers.size()` 循环分配 descriptor set。循环里写入时使用的是 `uniformBuffers[currentBuffer]`，并且每次分配都会覆盖同一个 `descriptorSet` 变量。

从示例意图和 README 来看，核心思想是“一个 descriptor set + dynamic offset 绘制多个对象”。而在多帧并发框架中，更严谨的组织通常会是：

```text
每个 frame-in-flight 一个 descriptor set
每个 descriptor set 指向对应 frame 的 view buffer 和 dynamic buffer
绘制当前帧时绑定 descriptorSets[currentBuffer]
```

也就是类似：

```cpp
std::array<VkDescriptorSet, maxConcurrentFrames> descriptorSets;
```

这不影响理解 dynamic uniform buffer 的核心机制，但写博客或做工程迁移时值得特别说明：**dynamic offset 解决的是“一个 buffer 内选择哪个对象数据”的问题，不解决 frame-in-flight 资源隔离问题。**

frame-in-flight 仍然需要用正确的 per-frame buffer 和 descriptor set 管理。

## 15. Pipeline Layout：descriptor layout 接入 pipeline

pipeline layout 创建很直接：

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutCreateInfo =
    vks::initializers::pipelineLayoutCreateInfo(
        &descriptorSetLayout,
        1);

VK_CHECK_RESULT(vkCreatePipelineLayout(
    device,
    &pipelineLayoutCreateInfo,
    nullptr,
    &pipelineLayout));
```

pipeline layout 表示这条 graphics pipeline 会使用一个 descriptor set layout。

shader 中 binding 0 和 binding 1 都没有显式写 `set = 0`，在 Vulkan GLSL 中默认就是 set 0。因此 pipeline layout 只需要一个 set layout。

如果要扩展成更真实的 renderer，可以设计成：

```text
set 0: camera / frame data
set 1: material textures
set 2: object dynamic uniform buffer
```

当前示例为了教学，把 view 和 instance 数据放在同一个 set 里。

## 16. Graphics Pipeline：顶点输入和 shader

示例创建了一条普通 graphics pipeline。

顶点输入 binding：

```cpp
VkVertexInputBindingDescription vertexInputBinding =
    vks::initializers::vertexInputBindingDescription(
        0,
        sizeof(Vertex),
        VK_VERTEX_INPUT_RATE_VERTEX);
```

两个 vertex attribute：

```cpp
std::vector<VkVertexInputAttributeDescription> vertexInputAttributes = {
    vks::initializers::vertexInputAttributeDescription(
        0,
        0,
        VK_FORMAT_R32G32B32_SFLOAT,
        offsetof(Vertex, pos)),

    vks::initializers::vertexInputAttributeDescription(
        0,
        1,
        VK_FORMAT_R32G32B32_SFLOAT,
        offsetof(Vertex, color)),
};
```

对应 shader：

```glsl
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inColor;
```

pipeline 的其他状态比较标准：

```cpp
VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST
VK_POLYGON_MODE_FILL
VK_CULL_MODE_NONE
VK_FRONT_FACE_COUNTER_CLOCKWISE
VK_COMPARE_OP_LESS_OR_EQUAL
VK_SAMPLE_COUNT_1_BIT
```

动态状态：

```cpp
VK_DYNAMIC_STATE_VIEWPORT
VK_DYNAMIC_STATE_SCISSOR
```

shader：

```cpp
shaderStages[0] = loadShader(
    getShadersPath() + "dynamicuniformbuffer/base.vert.spv",
    VK_SHADER_STAGE_VERTEX_BIT);

shaderStages[1] = loadShader(
    getShadersPath() + "dynamicuniformbuffer/base.frag.spv",
    VK_SHADER_STAGE_FRAGMENT_BIT);
```

最后创建 pipeline：

```cpp
vkCreateGraphicsPipelines(
    device,
    pipelineCache,
    1,
    &pipelineCreateInfo,
    nullptr,
    &pipeline);
```

这一部分没有特殊 magic。Dynamic uniform buffer 的特殊性不在 pipeline state，而在 descriptor type 和 bind-time offset。

## 17. 更新普通 View UBO

每帧先更新 projection 和 view：

```cpp
void updateUniformBuffers()
{
    uboVS.projection = camera.matrices.perspective;
    uboVS.view = camera.matrices.view;

    memcpy(
        uniformBuffers[currentBuffer].view.mapped,
        &uboVS,
        sizeof(uboVS));
}
```

这个 buffer 使用 host coherent memory，所以不需要手动 flush。

shader 中对应：

```glsl
uboView.projection
uboView.view
```

这部分数据是所有对象共享的。无论绘制第几个 cube，projection 和 view 都一样。

## 18. 更新 Dynamic UBO：按 dynamicAlignment 写入每个对象

dynamic UBO 的更新在 `updateDynamicUniformBuffer()` 中。

首先计算 3D 网格维度：

```cpp
uint32_t dim =
    static_cast<uint32_t>(
        pow(OBJECT_INSTANCES, (1.0f / 3.0f)));

glm::vec3 offset(5.0f);
```

然后遍历 5 x 5 x 5 个对象：

```cpp
for (uint32_t x = 0; x < dim; x++) {
    for (uint32_t y = 0; y < dim; y++) {
        for (uint32_t z = 0; z < dim; z++) {
            const uint32_t index =
                x * dim * dim + y * dim + z;
            ...
        }
    }
}
```

最关键的是通过对齐 offset 找到当前对象的 CPU 侧写入地址：

```cpp
glm::mat4* modelMat =
    (glm::mat4*)(((uint64_t)uboDataDynamic.model +
    (index * dynamicAlignment)));
```

注意这里不是：

```cpp
&uboDataDynamic.model[index]
```

因为 `uboDataDynamic.model[index]` 会按 `sizeof(glm::mat4)` 递增，而不是按 `dynamicAlignment` 递增。

正确布局是：

```text
base + 0 * dynamicAlignment
base + 1 * dynamicAlignment
base + 2 * dynamicAlignment
...
```

然后更新矩阵：

```cpp
rotations[index] += frameTimer * rotationSpeeds[index];

glm::vec3 pos = glm::vec3(
    -((dim * offset.x) / 2.0f) + offset.x / 2.0f + x * offset.x,
    -((dim * offset.y) / 2.0f) + offset.y / 2.0f + y * offset.y,
    -((dim * offset.z) / 2.0f) + offset.z / 2.0f + z * offset.z);

*modelMat = glm::translate(glm::mat4(1.0f), pos);
*modelMat = glm::rotate(*modelMat, rotations[index].x, glm::vec3(1.0f, 1.0f, 0.0f));
*modelMat = glm::rotate(*modelMat, rotations[index].y, glm::vec3(0.0f, 1.0f, 0.0f));
*modelMat = glm::rotate(*modelMat, rotations[index].z, glm::vec3(0.0f, 0.0f, 1.0f));
```

更新完 CPU 侧 aligned memory 后，一次性拷贝到当前帧的 dynamic uniform buffer：

```cpp
memcpy(
    uniformBuffers[currentBuffer].dynamic.mapped,
    uboDataDynamic.model,
    uniformBuffers[currentBuffer].dynamic.size);
```

因为 dynamic buffer 不是 host coherent，必须 flush：

```cpp
VkMappedMemoryRange memoryRange =
    vks::initializers::mappedMemoryRange();

memoryRange.memory =
    uniformBuffers[currentBuffer].dynamic.memory;

memoryRange.size =
    uniformBuffers[currentBuffer].dynamic.size;

vkFlushMappedMemoryRanges(device, 1, &memoryRange);
```

源码注释写的是：

```cpp
// Flush to make changes visible to the host
```

更准确地说，这里是让 CPU 写入的 mapped memory 对 device 可见。也就是让 GPU 能看到 CPU 刚写进去的数据。

## 19. Command Buffer：同一个 descriptor set，不同 dynamic offset

`buildCommandBuffer()` 是 dynamic uniform buffer 真正发挥作用的地方。

先开始 render pass、设置 viewport/scissor、绑定 pipeline：

```cpp
vkCmdBindPipeline(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipeline);
```

绑定 vertex/index buffer：

```cpp
VkDeviceSize offsets[1] = { 0 };

vkCmdBindVertexBuffers(
    cmdBuffer,
    0,
    1,
    &vertexBuffer.buffer,
    offsets);

vkCmdBindIndexBuffer(
    cmdBuffer,
    indexBuffer.buffer,
    0,
    VK_INDEX_TYPE_UINT32);
```

然后循环绘制 125 个对象：

```cpp
for (uint32_t j = 0; j < OBJECT_INSTANCES; j++) {
    uint32_t dynamicOffset =
        j * static_cast<uint32_t>(dynamicAlignment);

    vkCmdBindDescriptorSets(
        cmdBuffer,
        VK_PIPELINE_BIND_POINT_GRAPHICS,
        pipelineLayout,
        0,
        1,
        &descriptorSet,
        1,
        &dynamicOffset);

    vkCmdDrawIndexed(
        cmdBuffer,
        indexCount,
        1,
        0,
        0,
        0);
}
```

这段代码可以翻译成：

```text
第 0 个对象:
    dynamicOffset = 0 * dynamicAlignment
    shader binding 1 读取第 0 个 model matrix
    draw cube

第 1 个对象:
    dynamicOffset = 1 * dynamicAlignment
    shader binding 1 读取第 1 个 model matrix
    draw cube

第 2 个对象:
    dynamicOffset = 2 * dynamicAlignment
    shader binding 1 读取第 2 个 model matrix
    draw cube

...
```

所有 draw call 都使用同一个 descriptor set：

```cpp
&descriptorSet
```

但因为 `pDynamicOffsets` 不同，binding 1 对应的 buffer range 不同。

这就是 dynamic uniform buffer 的核心。

## 20. vkCmdBindDescriptorSets 的 dynamic offset 参数

再单独看这行：

```cpp
vkCmdBindDescriptorSets(
    cmdBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    0,
    1,
    &descriptorSet,
    1,
    &dynamicOffset);
```

前面的参数和普通 descriptor set 绑定一样：

- `cmdBuffer`：当前 command buffer。
- `VK_PIPELINE_BIND_POINT_GRAPHICS`：绑定到 graphics pipeline。
- `pipelineLayout`：当前 pipeline 使用的 layout。
- `0`：从 set 0 开始绑定。
- `1`：绑定一个 descriptor set。
- `&descriptorSet`：descriptor set 指针。

最后两个参数是 dynamic uniform buffer 的关键：

```cpp
1,
&dynamicOffset
```

它们分别是：

```cpp
dynamicOffsetCount
pDynamicOffsets
```

如果当前绑定的 descriptor set 中有多个 dynamic descriptor，那么 `pDynamicOffsets` 要按 descriptor set 和 binding 顺序提供多个 offset。

这个示例中只有一个 dynamic descriptor：

```text
set 0 binding 1
```

所以只需要传一个 offset。

如果 layout 是：

```text
binding 0 = uniform buffer
binding 1 = dynamic uniform buffer
binding 2 = dynamic uniform buffer
```

那么绑定时就需要传两个 dynamic offset，顺序对应 binding 1 和 binding 2。

## 21. Dynamic offset 和 descriptor offset/range 的关系

理解 dynamic uniform buffer 时，必须区分三个概念：

```text
descriptor buffer info offset
descriptor buffer info range
bind-time dynamic offset
```

descriptor 写入时，`VkDescriptorBufferInfo` 里有：

```cpp
buffer
offset
range
```

dynamic offset 是绑定 descriptor set 时传入的：

```cpp
pDynamicOffsets
```

最终 shader 看到的 buffer 区间可以理解为：

```text
buffer:
    descriptor.offset + dynamicOffset

range:
    descriptor.range
```

这个示例中：

```text
descriptor.offset = 0
descriptor.range = dynamicAlignment
dynamicOffset = objectIndex * dynamicAlignment
```

所以第 j 个对象看到：

```text
buffer offset = j * dynamicAlignment
range = dynamicAlignment
```

shader 在这个 range 内读取：

```glsl
mat4 model
```

也就是当前对象的 model matrix。

## 22. 为什么这里没有用 instancing

这个示例绘制 125 个 cube，但它没有使用 Vulkan instancing：

```cpp
vkCmdDrawIndexed(cmdBuffer, indexCount, 1, 0, 0, 0);
```

其中 instance count 是：

```cpp
1
```

它是循环 125 次 draw call，而不是一次 draw 125 个 instance。

这是有意为之。这个示例的主题是 dynamic uniform buffer，而 dynamic offset 是在绑定 descriptor set 时传入的。每个对象要使用不同 dynamic offset，就自然写成：

```text
bind descriptor set with offset
draw one object
```

如果使用 instancing，更常见的做法是：

- instance attributes
- storage buffer
- shader 根据 `gl_InstanceIndex` 索引数组
- dynamic uniform buffer 只作为较粗粒度资源绑定

也就是说，dynamic uniform buffer 适合“每个 draw 选择一份小型 uniform 数据”。如果要一次 draw 大量实例，storage buffer 或 instance buffer 往往更合适。

这也是工程上需要区分的点：

```text
Dynamic UBO:
    减少 descriptor set 数量
    仍然可能有多个 draw call
    适合每 draw 一份小 uniform 数据

Instancing + SSBO/vertex instance data:
    减少 draw call 数量
    适合大量实例化对象
```

## 23. Dynamic Uniform Buffer 的优点

第一，减少 descriptor set 数量。

如果 125 个对象每个对象一个 descriptor set，就要管理 125 份对象级 descriptor。使用 dynamic UBO 后，可以用同一个 descriptor set 加不同 offset 绘制多个对象。

第二，减少 descriptor 更新。

descriptor set 绑定的是同一个大 buffer，不需要每个对象都更新 descriptor。每帧只需要更新 buffer 内容。

第三，数据连续，便于批量写入。

所有 model matrix 存在一块连续内存里，可以一次性 memcpy 到 mapped buffer，也可以只更新变化的子范围。

第四，draw loop 很明确。

每个 draw 前只计算：

```cpp
dynamicOffset = objectIndex * dynamicAlignment;
```

然后绑定 descriptor set 时传进去。

第五，适合小型 per-object uniform 数据。

例如：

- model matrix
- normal matrix
- object color
- material scalar parameters
- object id

只要每个对象的数据量不大，并且符合 uniform buffer 限制，就可以考虑 dynamic UBO。

## 24. Dynamic Uniform Buffer 的限制

第一，受 uniform buffer range 限制。

设备有：

```cpp
maxUniformBufferRange
```

如果单个对象数据太大，或者希望一个 binding 暴露很大数组，uniform buffer 未必合适。

第二，受 dynamic uniform buffer 数量限制。

README 提到应该关注：

```cpp
maxDescriptorSetUniformBuffersDynamic
```

Vulkan 规范要求至少支持一定数量的 dynamic uniform buffer，但真实项目中如果一个 descriptor set 里放很多 dynamic UBO，需要检查设备 limit。

第三，offset 必须满足 alignment。

这是最常见的坑：

```cpp
dynamicOffset % minUniformBufferOffsetAlignment == 0
```

如果忘记 padding，很容易在某些 GPU 上出错。

第四，不适合大量数据随机访问。

Dynamic UBO 的模型是“绑定时选择一个 range，shader 读取这个 range”。如果 shader 需要随机访问大量对象数据，storage buffer 更合适。

第五，不能替代 frame-in-flight 资源管理。

dynamic offset 解决的是“同一个 buffer 里选哪段对象数据”。如果 CPU/GPU 多帧并发，仍然需要避免当前帧写入 GPU 正在读取的数据。示例通过每个 frame 一份 `uniformBuffers` 来处理这个问题。

## 25. 和 descriptorsets.cpp 的关系

前一类 descriptor set 示例通常会这样组织：

```text
cube 0 descriptor set
    binding 0 -> cube 0 uniform buffer
    binding 1 -> cube 0 texture

cube 1 descriptor set
    binding 0 -> cube 1 uniform buffer
    binding 1 -> cube 1 texture
```

绘制时：

```text
bind cube 0 descriptor set
draw
bind cube 1 descriptor set
draw
```

`dynamicuniformbuffer.cpp` 的思路是：

```text
one descriptor set
    binding 0 -> view uniform buffer
    binding 1 -> big dynamic uniform buffer

draw object 0 with dynamicOffset 0
draw object 1 with dynamicOffset dynamicAlignment
draw object 2 with dynamicOffset 2 * dynamicAlignment
```

也就是说，它把“对象差异”从 descriptor set 数量转移到了 dynamic offset 上。

可以把两者对比成：

```text
普通 per-object descriptor:
    不同对象绑定不同 descriptor set

dynamic uniform buffer:
    不同对象绑定同一 descriptor set，但传不同 dynamic offset
```

这就是 dynamic UBO 的价值。

## 26. 和 Push Constants 的区别

有人可能会问：每个对象一个 model matrix，为什么不用 push constants？

Push constants 确实适合小数据，而且更新非常方便：

```cpp
vkCmdPushConstants(..., &modelMatrix);
draw
```

但 push constants 有几个限制。

第一，大小限制更严格。很多设备至少支持 128 bytes，虽然一个 mat4 是 64 bytes，可以放下，但如果对象参数更多就不够了。

第二，push constants 更适合极小、频繁变化的数据。大量对象数据如果都通过 push constants 管理，虽然可行，但不一定是最清晰的数据组织方式。

第三，dynamic uniform buffer 可以把所有对象数据预先写入一个大 buffer，draw loop 只传 offset，更适合和 descriptor 体系、buffer streaming、partial update 结合。

工程上可以这样选择：

```text
少量标量或一个矩阵:
    push constants 很方便

很多对象的一组 uniform 数据:
    dynamic uniform buffer 更结构化

大量实例数据或随机访问数据:
    storage buffer / instance buffer 更合适
```

## 27. 和 Storage Buffer 的区别

Storage buffer 也可以存很多对象的 model matrix，例如：

```glsl
layout (binding = 1) readonly buffer Objects {
    mat4 model[];
} objects;
```

然后 shader 里通过 object index 或 instance index 访问：

```glsl
mat4 model = objects.model[gl_InstanceIndex];
```

这比 dynamic uniform buffer 更灵活，也适合大量对象和实例化渲染。

但 dynamic uniform buffer 的优势是：

- 语义简单。
- 对小型 uniform 数据访问路径更传统。
- 每次 draw 只暴露一个对象的数据段。
- shader 端不需要数组索引逻辑。

所以 dynamic UBO 更像是“传统 per-object UBO 的批量打包版本”，而 storage buffer 更像是“shader 可索引的大数据表”。

## 28. 真实项目中如何组织

在真实 Vulkan renderer 中，一个常见设计是：

```text
set 0: per-frame data
    projection
    view
    camera position
    lights

set 1: material data
    textures
    samplers
    material factors

set 2: per-object dynamic UBO
    model matrix
    normal matrix
    object parameters
```

然后 draw loop 可能是：

```text
bind pipeline
bind frame descriptor set once

for each material batch:
    bind material descriptor set

    for each object:
        bind object descriptor set with dynamic offset
        draw
```

如果对象数量很大，还会进一步考虑：

- instancing
- indirect draw
- storage buffer
- bindless descriptor
- GPU driven culling

但 dynamic uniform buffer 仍然是理解 Vulkan 资源绑定优化的一块重要基石。

## 29. 这个示例的完整执行流程

初始化：

```text
prepare()
    VulkanExampleBase::prepare()
    generateCube()
    prepareUniformBuffers()
        计算 dynamicAlignment
        分配 CPU aligned memory
        创建 view UBO
        创建 dynamic UBO
        map buffers
        初始化随机旋转
    setupDescriptors()
        创建 descriptor pool
        创建 descriptor set layout
        写入 binding 0 view UBO
        写入 binding 1 dynamic UBO
    preparePipelines()
        创建 pipeline layout
        创建 graphics pipeline
    prepared = true
```

每帧：

```text
render()
    prepareFrame()
    updateUniformBuffers()
        更新 projection/view
    updateDynamicUniformBuffer()
        为 125 个对象计算 model matrix
        按 dynamicAlignment 写入 CPU aligned memory
        memcpy 到当前帧 dynamic UBO
        flush mapped memory
    buildCommandBuffer()
        bind pipeline
        bind vertex/index buffer
        for each object:
            dynamicOffset = objectIndex * dynamicAlignment
            bind descriptor set with dynamicOffset
            draw indexed cube
    submitFrame()
```

最核心的链路是：

```text
CPU 按 alignment 写入大 buffer
descriptor set 绑定这个大 buffer
draw 前传入 dynamic offset
shader 从 binding 1 读取当前对象 model matrix
```

## 30. 常见错误和调试方向

第一，dynamic offset 没有按 `minUniformBufferOffsetAlignment` 对齐。

症状通常是 validation layer 报错，或者某些 GPU 上矩阵读取错乱。

第二，buffer size 按 `sizeof(T) * count` 分配，而不是按 `dynamicAlignment * count` 分配。

如果 alignment 大于数据大小，后面的对象会写越界或读错位置。

第三，CPU 侧数组按普通数组索引写入。

错误写法：

```cpp
uboDataDynamic.model[index] = model;
```

正确思路：

```cpp
glm::mat4* modelMat =
    (glm::mat4*)((uint64_t)base + index * dynamicAlignment);
```

第四，descriptor type 写成普通 uniform buffer。

binding 1 必须是：

```cpp
VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC
```

否则 `vkCmdBindDescriptorSets` 传入 dynamic offset 没有对应 dynamic binding。

第五，非 coherent memory 写完后忘记 flush。

如果 dynamic buffer 没有 `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`，写入 mapped memory 后必须调用：

```cpp
vkFlushMappedMemoryRanges
```

第六，多帧并发时绑定了错误 frame 的 descriptor 或 buffer。

dynamic offset 只选择对象数据段，不负责选择当前帧 buffer。frame-in-flight 仍然要用 `currentBuffer` 管理。

## 31. 总结

`dynamicuniformbuffer.cpp` 是理解 Vulkan dynamic uniform buffer 的经典示例。

它展示了：

- 如何用一个大 uniform buffer 存放多个对象的 model matrix。
- 如何查询 `minUniformBufferOffsetAlignment` 并计算 `dynamicAlignment`。
- 为什么 CPU 侧临时数据也要按 `dynamicAlignment` 写入。
- 如何创建 `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC` binding。
- 如何通过 `vkCmdBindDescriptorSets` 的 `pDynamicOffsets` 参数选择当前对象数据。
- 为什么 dynamic UBO 可以减少 per-object descriptor set 数量。
- 为什么它仍然需要正确处理 frame-in-flight 和 memory flush。

它的核心思想可以浓缩成一句话：

```text
Descriptor set 绑定一整块对象数据，dynamic offset 决定本次 draw 读取其中哪一段。
```

再换一种 Vulkan renderer 的表达：

```text
pipeline 决定绘制状态
descriptor set 决定绑定哪组资源
dynamic offset 决定这组资源中的当前对象数据切片
```

理解这个示例之后，再看更复杂的对象批处理、材质系统、实例渲染、storage buffer、GPU driven rendering，会更容易判断每种数据应该放在哪里，以及为什么 Vulkan 要把资源绑定和 offset 选择设计得如此显式。

