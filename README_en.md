# VMA (Vulkan Memory Allocator)

Vulkan Memory Allocator (VMA) is an easy-to-integrate Vulkan memory allocation library. It is a single-header C++ library that provides high-level interfaces for Vulkan memory management and resource creation, simplifying the complex memory allocation operations in Vulkan.

## Use Cases of VMA

Memory allocation and resource (buffer and image) creation in Vulkan is more complex compared to older graphics APIs (such as D3D11 or OpenGL) for several reasons:
- It requires a lot of boilerplate code
- Driver must be queried for supported memory heaps and memory types
- It is recommended to allocate larger memory chunks and assign parts of them to particular resources, as there is a limit on maximum number of memory blocks

The introduction of VMA in OpenHarmony provides efficient memory management capabilities for graphics rendering engines (such as graphic_3d), reducing the complexity of Vulkan memory allocation.

## Directory Structure

```
docs/                # Documentation directory, containing detailed API documentation
include/             # Header file directory
    vk_mem_alloc.h   # VMA single-header library
oh_src/                 # Source code directory
    vma_impl.cpp     # OpenHarmony adaptation file for building as shared library
tools/               # Tools directory
    GpuMemDumpVis/   # GPU memory visualization tool
CMakeLists.txt       # CMake build description file
bundle.json          # OpenHarmony component configuration file
BUILD.gn             # OpenHarmony build description file
LICENSE.txt          # License file
README.md            # Software description
```

## Adaptation of VMA for OpenHarmony

OpenHarmony adds the following adaptation files to convert VMA into a shared library:

| File | Description |
|------|-------------|
| `oh_src/vma_impl.cpp` | Defines `VMA_IMPLEMENTATION` macro, compiles implementation as shared library |
| `BUILD.gn` | OpenHarmony build description file |
| `bundle.json` | OpenHarmony component configuration file |

The original VMA is a single-header library that requires defining `VMA_IMPLEMENTATION` macro at the usage point to compile the implementation code. OpenHarmony converts VMA into a shared library via `oh_src/vma_impl.cpp`, enabling dynamic linking by other components and avoiding repeated compilation of implementation code by each user.

## Relevant References

https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator

## License

This project is subject to the MIT License described in LICENSE.txt.

## Related Warehouse

[vma](https://gitcode.com/openharmony-sig/third_party_vma)

## Usage in OpenHarmony

### Target Audience

Graphics rendering engine developers within the OpenHarmony system.

### Dynamic Library Reference

Add dependency in `BUILD.gn`:

```gni
external_deps = [
  "vma:vma",
]
```

Declare dependency in `bundle.json`:

```json
"deps": {
    "third_party": [
        "vma"
    ]
}
```

Include the header file in source code:

```cpp
#include "vk_mem_alloc.h"
```

### VMA_IMPLEMENTATION Macro Explained

**Original VMA Single-Header Pattern:**

VMA adopts a single-header library design pattern. The header file `vk_mem_alloc.h` contains both declarations and implementations, controlled by conditional compilation:

```cpp
// Declarations only (like a normal header)
#include "vk_mem_alloc.h"

// Declarations + Implementations (like a source file)
#define VMA_IMPLEMENTATION
#include "vk_mem_alloc.h"
```

Inside `vk_mem_alloc.h`, implementation code is wrapped in conditional compilation:

```cpp
#ifdef VMA_IMPLEMENTATION
// All function implementations...
#endif
```

This pattern requires:
- Only one compilation unit in the entire program can define `VMA_IMPLEMENTATION`
- If multiple `.cpp` files define this macro, it causes multiple definition linker errors

**OpenHarmony Shared Library Adaptation:**

OpenHarmony centralizes `VMA_IMPLEMENTATION` definition in `oh_src/vma_impl.cpp` and compiles it as a shared library:

```cpp
// oh_src/vma_impl.cpp
#define VMA_IMPLEMENTATION
#include "vk_mem_alloc.h"
```

Advantages of this design:
1. **Avoid repeated compilation**: Implementation code is compiled once; just link the shared library
2. **Avoid linker conflicts**: No need to worry about multiple compilation units defining the macro

**Important: Do NOT define VMA_IMPLEMENTATION when using the shared library**

When using OpenHarmony's VMA shared library, simply `#include "vk_mem_alloc.h"` for declarations. **Do NOT** define `VMA_IMPLEMENTATION` in your own code, otherwise:

```cpp
// Wrong - will cause linker conflict
#define VMA_IMPLEMENTATION  // Do NOT do this!
#include "vk_mem_alloc.h"
```

```
error: multiple definition of 'vmaCreateAllocator'
```

### BUILD.gn Shared Library Compilation

```gni
ohos_shared_library("vma") {
  sources = [ "oh_src/vma_impl.cpp" ]  # Source file containing VMA_IMPLEMENTATION
  public_configs = [ ":vma_public_config" ]  # Export include directory to dependents
  external_deps = [ "vulkan-headers:vulkan_headers" ]
  part_name = "vma"
  subsystem_name = "thirdparty"
}
```

Key points: `sources` specifies the file with `VMA_IMPLEMENTATION`, `public_configs` provides header path to dependents.

**Dependency Chain:**

```
vma_impl.cpp (contains VMA_IMPLEMENTATION) → libvma.z.so → Consumer module
```

### Conditional Compilation Control

VMA depends on Vulkan runtime. When the target platform doesn't support Vulkan or doesn't require Vulkan rendering, compilation options can control whether VMA is included in the build.

**Example: graphic_3d Conditional Compilation Implementation**

graphic_3d uses `RENDER_BUILD_VULKAN` parameter to control Vulkan backend, depends on VMA when enabled:

```gni
# foundation/graphic/graphic_3d/lume/lume_config.gni
declare_args() {
  RENDER_BUILD_VULKAN = true  # Control whether to build Vulkan backend
}

# foundation/graphic/graphic_3d/lume/LumeRender/BUILD.gn
if (RENDER_BUILD_VULKAN) {
  sources += [ "src/vulkan/gpu_memory_allocator_vk.cpp", ... ]
  external_deps += [
    "vulkan-headers:vulkan_headers",
    "vulkan-loader:vulkan_loader",
    "vma:vma"  # Depend on VMA when Vulkan backend is enabled
  ]
}
```

**Example: Skia Conditional Compilation Implementation**

Skia uses the `skia_use_vulkan` parameter to control Vulkan support, which in turn controls VMA dependency:

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

# VMA depends on Vulkan; auto-disable when Vulkan is disabled
skia_use_vma = skia_use_vulkan
```

In BUILD.gn, use conditional statements to control dependencies:

```gni
# third_party/skia/m133/BUILD.gn
config("skia_private") {
  if (skia_use_vma) {
    defines += [ "SK_USE_VMA" ]  // Code uses #ifdef SK_USE_VMA for conditional compilation
  }
}

# third_party/skia/m133/BUILD.gn (line 2973)
source_set("gpu") {
  if (skia_use_vma) {
    external_deps += [ "vma:vma" ]  // Only link VMA when enabled
  }
}
```

```gni
# third_party/skia/m133/src/gpu/vk/vulkanmemoryallocator/BUILD.gn
import("../../../../gn/skia.gni")  // Import skia_use_vma parameter

source_set("vulkanmemoryallocator") {
  external_deps = [ "vma:vma" ]  // Depend on VMA shared library
  # ...
}
```

### Code Example

```cpp
#include "vk_mem_alloc.h"

// Create allocator
VmaAllocator allocator;
VmaAllocatorCreateInfo allocatorInfo = {};
// Set Vulkan instance and device
allocatorInfo.instance = instance;
allocatorInfo.physicalDevice = physicalDevice;
allocatorInfo.device = device;
allocatorInfo.vulkanApiVersion = VK_API_VERSION_1_0;

vmaCreateAllocator(&allocatorInfo, &allocator);

// Create buffer
VkBufferCreateInfo bufferInfo = { VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO };
bufferInfo.size = 65536;
bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;

VmaAllocationCreateInfo allocInfo = {};
allocInfo.usage = VMA_MEMORY_USAGE_AUTO;

VkBuffer buffer;
VmaAllocation allocation;
vmaCreateBuffer(allocator, &bufferInfo, &allocInfo, &buffer, &allocation, nullptr);

// Use buffer...

// Destroy buffer
vmaDestroyBuffer(allocator, buffer, allocation);

// Destroy allocator
vmaDestroyAllocator(allocator);
```

For more API usage, refer to: https://gpuopen-librariesandsdks.github.io/VulkanMemoryAllocator/html/