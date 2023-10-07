---
title: "Vulkan-Sample-Note-Dynamic-Uniform-Buffers"
date: 2023-09-04T15:26:24-07:00
draft: False
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---

# Vulkan Sample Note: Dynamic uniform buffers
The sample is from [Sascha Williems](https://github.com/SaschaWillems/Vulkan/tree/master/examples/)

## TL;DR
* Allocate aligned large uniform buffer for multiple objects, and access each uniform buffer by indexing
* Avoid creating multiple same uniform buffer to reduce overhead 

## Notes

### Uniform buffer dynamic
When set up descriptor set layout, need to use special descriptor type `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC` during binding.

Declare bug chunk of dynamic uniform buffer 
```cpp
// One big uniform buffer that contains all matrices
// Note that we need to manually allocate the data to cope for GPU-specific uniform buffer offset alignments
struct UboDataDynamic {
	glm::mat4 *model = nullptr;
} uboDataDynamic;
```
```cpp

    // Allocate data for the dynamic uniform buffer object
    // We allocate this manually as the alignment of the offset differs between GPUs

    // Calculate required alignment based on minimum device offset alignment
    size_t minUboAlignment = vulkanDevice->properties.limits.minUniformBufferOffsetAlignment;
    dynamicAlignment = sizeof(glm::mat4);
    if (minUboAlignment > 0) {
	    dynamicAlignment = (dynamicAlignment + minUboAlignment - 1) & ~(minUboAlignment - 1);
    }

    size_t bufferSize = OBJECT_INSTANCES * dynamicAlignment;

    uboDataDynamic.model = (glm::mat4*)alignedAlloc(bufferSize, dynamicAlignment);
    assert(uboDataDynamic.model);

```

### Aligned memory allocation
Normally, we can use `std::aligned_alloc` in C++17.
Allocate size bytes of uninitialized storage whose alignment is specified by alignment
In Windows, it could be used `_aligned_malloc`, otherwise  `posix_memalign`.

### Calculate the correct alignment
```cpp
// Calculate required alignment based on minimum device offset alignment
size_t minUboAlignment = vulkanDevice->properties.limits.minUniformBufferOffsetAlignment;
dynamicAlignment = sizeof(glm::mat4);
if (minUboAlignment > 0) {
	dynamicAlignment = (dynamicAlignment + minUboAlignment - 1) & ~(minUboAlignment - 1);
		}
```
Since the `minUboAlignment` `dynamicAlignment` will be power of 2, the operation `dynamicAlignment = (dynamicAlignment + minUboAlignment - 1) & ~(minUboAlignment - 1);` acts like `max(dynamicAlignment, minUboAlignment )`.