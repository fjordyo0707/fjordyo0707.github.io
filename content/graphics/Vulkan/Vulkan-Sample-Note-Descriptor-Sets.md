---
title: "Vulkan-Sample-Note-Descriptor-Sets"
date: 2023-09-01T15:26:24-07:00
draft: False
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---

# Vulkan Sample Note: Descriptor sets
The sample is from [Sascha Williems](https://github.com/SaschaWillems/Vulkan/tree/master/examples/descriptorsets)

## TL;DR
* Demonstration of binding descriptor sets, 1 uniform buffer and 1 image smapler

## Note
### Descriptor Set Layout
The layout describes the shader bindings and types used for a certain descriptor layout and as such must match the shader bindings

### Descriptor pool
Actual descriptors are allocated from a descriptor pool telling the driver what types and how many descriptors this application will use

#### Why there is a maxSets during descriptor pool creation?
Individual descriptors take up some form of resource, whether CPU, GPU, or both. But bundling them into descriptor sets can also takes up resources, depending on the implementation. As such, if an implementation can pre-allocate some number of set resources, that would be good for minimizing runtime allocations. [source](https://stackoverflow.com/a/65541669)

In this example, we limit our `maxSet` to 2, and each set will have 1 uniform buffer and 1 image sampler. Also, in uniform buffer pool the `descriptorCount` will be 2, and image sampler as well. 

![](/VK_Descriptor_Sets.png)

