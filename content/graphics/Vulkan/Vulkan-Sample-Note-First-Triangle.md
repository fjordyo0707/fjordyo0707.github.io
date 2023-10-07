---
title: "Vulkan-Sample-Note-First-Triangle"
date: 2023-08-25T15:26:24-07:00
draft: False
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---
# Vulkan Sample Note: First triangle
The sample is from [Sascha Williems](https://github.com/SaschaWillems/Vulkan/tree/master/examples/triangle)

## TL;DR
* A basic template of simple Vulkan example
* Details of each Vulkan object, which will be abstract in the future example
* Easy explanation of the rendering flow

## Notes
The example is pretty much the same as my [Vulkan Introduction](/ZC6AKFNMTnuZ_Vcq3gEklA?both). I have appended their comment in my terms' explaination with the quoting and bullet point. Thus, I won't put too much words in this note.

### When to write in Command Buffer?
In [My Vulkan Introduction](https://hackmd.io/ZC6AKFNMTnuZ_Vcq3gEklA) code, which is based on [the online tutorial from Alexander Overvoorde](https://vulkan-tutorial.com/Introduction), it records the command buffer and begin render pass in the draw loop. However, it will cause a performance issue to do so. In the Vulkan example, we build up and write the command buffer ahead, including render pass, binding to buffer, to reduce unecessary operation. During render loop, we select the command buffer with the swapchain index, which the operation is already stored inside.
