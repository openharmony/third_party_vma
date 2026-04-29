# VMA (Vulkan Memory Allocator)

Vulkan Memory Allocator (VMA)是一个易于集成的Vulkan显存分配库。它是一个单头文件的C++库，为Vulkan显存管理和资源创建提供高级接口，简化了Vulkan中复杂的显存分配操作。

## VMA使用场景

Vulkan中的显存分配和资源（缓冲区和图像）创建相比旧版图形API更加复杂，原因包括：
- 需要大量样板代码
- 需要查询设备支持的内存堆和内存类型
- 建议分配较大的内存块并在其中分配资源，因为存在内存块数量限制

OpenHarmony引入VMA，主要为图形渲染引擎提供高效的显存管理能力，降低Vulkan显存分配的复杂度。

## 目录结构

```
docs/            # 文档目录，包含详细的API文档
include/         # 头文件目录
    vk_mem_alloc.h    # VMA单头文件库
oh_src/             # 源代码目录
    vma_impl.cpp      # OpenHarmony适配-用于编译成动态库
tools/           # 工具目录
    GpuMemDumpVis/    # 显存可视化工具
CMakeLists.txt   # CMake编译描述文件
bundle.json      # OpenHarmony适配-部件配置文件
BUILD.gn         # OpenHarmony适配-编译描述文件
LICENSE.txt      # 许可证文件
README.md        # 软件说明
```

## OpenHarmony对VMA的适配

OpenHarmony新增以下适配文件，将VMA改造为共享库：

| 文件 | 说明 |
|------|------|
| `oh_src/vma_impl.cpp` | 定义`VMA_IMPLEMENTATION`宏，编译实现代码为共享库 |
| `BUILD.gn` | OpenHarmony编译描述文件 |
| `bundle.json` | OpenHarmony部件配置文件 |

原始VMA是单头文件库，需要在使用处定义`VMA_IMPLEMENTATION`宏来编译实现代码。OpenHarmony通过`oh_src/vma_impl.cpp`将VMA改造为共享库，方便其他部件动态链接使用，避免每个使用者都需要重复编译实现代码。

## 相关参考

https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator

## 许可证

本项目遵从LICENSE.txt中所描述的MIT许可证。

## 相关仓

[vma](https://gitcode.com/openharmony-sig/third_party_vma)

## OpenHarmony中的使用

### 面向对象

OpenHarmony系统内部的图形渲染引擎开发者。

### 动态库引用方式

在`BUILD.gn`中添加依赖：

```gni
external_deps = [
  "vma:vma",
]
```

在`bundle.json`中声明依赖：

```json
"deps": {
    "third_party": [
        "vma"
    ]
}
```

在源代码中引入头文件：

```cpp
#include "vk_mem_alloc.h"
```

### VMA_IMPLEMENTATION宏原理说明

**原始VMA的单头文件模式：**

VMA采用单头文件（single-header）库设计模式。头文件`vk_mem_alloc.h`同时包含声明和实现代码，通过条件编译控制：

```cpp
// 仅声明（相当于头文件）
#include "vk_mem_alloc.h"

// 声明 + 实现（相当于源文件）
#define VMA_IMPLEMENTATION
#include "vk_mem_alloc.h"
```

在`vk_mem_alloc.h`内部，实现代码被条件编译包裹：

```cpp
#ifdef VMA_IMPLEMENTATION
// 所有函数实现代码...
#endif
```

这种模式要求：
- 整个程序中只能有一个编译单元定义`VMA_IMPLEMENTATION`
- 如果多个`.cpp`文件都定义此宏，会导致重复定义链接错误

**OpenHarmony的共享库改造：**

OpenHarmony通过`oh_src/vma_impl.cpp`集中定义`VMA_IMPLEMENTATION`并编译为共享库：

```cpp
// oh_src/vma_impl.cpp
#define VMA_IMPLEMENTATION
#include "vk_mem_alloc.h"
```

这种设计的优势：
1. **避免重复编译**：实现代码只需编译一次，链接共享库即可
2. **避免链接冲突**：无需担心多编译单元重复定义问题

**重要：使用时切勿定义VMA_IMPLEMENTATION**

使用OpenHarmony的VMA共享库时，只需`#include "vk_mem_alloc.h"`获取声明，**切勿**在自己的代码中定义`VMA_IMPLEMENTATION`，否则会导致：

```cpp
// 错误示例 - 会导致链接冲突
#define VMA_IMPLEMENTATION  // 不要这样做！
#include "vk_mem_alloc.h"
```

```
error: multiple definition of 'vmaCreateAllocator'
```

### BUILD.gn共享库编译实现

```gni
ohos_shared_library("vma") {
  sources = [ "oh_src/vma_impl.cpp" ]  # 含VMA_IMPLEMENTATION的源文件
  public_configs = [ ":vma_public_config" ]  # 导出include目录给依赖方
  external_deps = [ "vulkan-headers:vulkan_headers" ]
  part_name = "vma"
  subsystem_name = "thirdparty"
}
```

关键点：`sources`指定含`VMA_IMPLEMENTATION`的源文件，`public_configs`让依赖方自动获得头文件路径。

**依赖链路：**

```
vma_impl.cpp (含VMA_IMPLEMENTATION) → libvma.z.so → 使用者模块
```

### 条件编译控制

VMA依赖Vulkan运行，当目标平台不支持Vulkan或不需要Vulkan渲染时，可通过编译选项控制VMA是否参与编译。

**示例：graphic_3d的条件编译实现**

graphic_3d通过`RENDER_BUILD_VULKAN`参数控制Vulkan后端，启用时依赖VMA：

```gni
# foundation/graphic/graphic_3d/lume/lume_config.gni
declare_args() {
  RENDER_BUILD_VULKAN = true  # 控制是否编译Vulkan后端
}

# foundation/graphic/graphic_3d/lume/LumeRender/BUILD.gn
if (RENDER_BUILD_VULKAN) {
  sources += [ "src/vulkan/gpu_memory_allocator_vk.cpp", ... ]
  external_deps += [
    "vulkan-headers:vulkan_headers",
    "vulkan-loader:vulkan_loader",
    "vma:vma"  # Vulkan后端启用时依赖VMA
  ]
}
```

**示例：Skia的条件编译实现**

Skia通过`skia_use_vulkan`参数控制Vulkan支持，进而控制VMA依赖：

```gni
# third_party/skia/m133/gn/skia.gni
declare_args() {
  if (is_fuchsia) {
    skia_use_vulkan = true
  } else {
    skia_use_vulkan = false
    if (skia_feature_product == "phone" || 
        skia_feature_product == "tablet" ||
        skia_feature_use_vulkan) {
      skia_use_vulkan = true
    }
  }
}

# VMA依赖Vulkan，Vulkan禁用时自动禁用VMA
skia_use_vma = skia_use_vulkan
```

在BUILD.gn中通过条件判断控制依赖：

```gni
# third_party/skia/m133/BUILD.gn
config("skia_private") {
  if (skia_use_vma) {
    defines += [ "SK_USE_VMA" ]  # 代码中通过#ifdef SK_USE_VMA条件编译
  }
}

# third_party/skia/m133/BUILD.gn (line 2973)
source_set("gpu") {
  if (skia_use_vma) {
    external_deps += [ "vma:vma" ]  # 仅在启用时链接VMA
  }
}
```

```gni
# third_party/skia/m133/src/gpu/vk/vulkanmemoryallocator/BUILD.gn
import("../../../../gn/skia.gni")  # 导入skia_use_vma参数

source_set("vulkanmemoryallocator") {
  external_deps = [ "vma:vma" ]  # 依赖VMA共享库
  # ...
}
```

### 代码示例

```cpp
#include "vk_mem_alloc.h"

// 创建分配器
VmaAllocator allocator;
VmaAllocatorCreateInfo allocatorInfo = {};
// 需要设置Vulkan实例和设备
allocatorInfo.instance = instance;
allocatorInfo.physicalDevice = physicalDevice;
allocatorInfo.device = device;
allocatorInfo.vulkanApiVersion = VK_API_VERSION_1_0;

vmaCreateAllocator(&allocatorInfo, &allocator);

// 创建缓冲区
VkBufferCreateInfo bufferInfo = { VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO };
bufferInfo.size = 65536;
bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;

VmaAllocationCreateInfo allocInfo = {};
allocInfo.usage = VMA_MEMORY_USAGE_AUTO;

VkBuffer buffer;
VmaAllocation allocation;
vmaCreateBuffer(allocator, &bufferInfo, &allocInfo, &buffer, &allocation, nullptr);

// 使用缓冲区...

// 销毁缓冲区
vmaDestroyBuffer(allocator, buffer, allocation);

// 销毁分配器
vmaDestroyAllocator(allocator);
```

更多API用法参考：https://gpuopen-librariesandsdks.github.io/VulkanMemoryAllocator/html/