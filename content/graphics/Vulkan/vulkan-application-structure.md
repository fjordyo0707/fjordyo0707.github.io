---
title: "Vulkan Application Structure"
date: 2023-01-24T14:40:29-07:00
draft: false
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---
# Vulkan Application Structure

The source code and content is mostly based on [Vulkan-Samples](https://github.com/KhronosGroup/Vulkan-Samples/tree/main). The blog is merely for knowledge sharing what I learned from it.

## Flow for an application
1. Prepare
    * Basic: could be pre-defined for most applications.
        * Initialize the swapchain
        * Create command pool
        * Set up the swapchain
        * Create command buffers
        * Create synchronization primitives
        * Setup depth stencil
        * Setup render pass
        * Create pipeline cache
        * Setup frameBuffer
        * Setup UI
    * Case: defined by application scenario
        * <span style="color:blue ">Load assets</span>
        * Initialize lights
        * <span style="color:red ">Prepare uniform buffers</span>
        * <span style="color:red ">Setup descriptor set layout</span>
        * <span style="color:red ">Prepare pipelines</span>
        * <span style="color:red ">Setup descriptor pool</span>
        * <span style="color:red ">Setup descriptor set</span>
        * <span style="color:red ">Build command buffers</span>
2. Update:
    * Render the scene
    * update the parameter

## Application
The base class `Application` is like a empty class telling what virtual method should override when you derive it. It still includes some implementations, but mostly related to system level parameters, like time, window, input event. It hasn't inludes any graphics API yet.
![](/vk_application_structure_app.png)


## VulkanSample
The `VulkanSample` class inheirt `Application` class and further add more Vulkan details. Few virtual methods is mandatory to implement, which will be called in the further overridden method. Here is the lists of the method need to be override if inheirt `VulkanSample`. For some forgot the inheritance rule in C++, here is the breif recap: 
* **public inheritance** makes public members of the base class public in the derived class, and the protected members of the base class remain protected in the derived class.
* **protected inheritanc**e makes the public and protected members of the base class protected in the derived class.
* **private inheritance** makes the public and protected members of the base class private in the derived class.

**Public:**
* `virtual void create_device();`
* `virtual void create_instance();`
* `virtual ~VulkanSample();`

**Protected:**
* `virtual void draw(CommandBuffer &command_buffer, RenderTarget &render_target);`
* `virtual void draw_renderpass(CommandBuffer &command_buffer, RenderTarget &render_target);`
* `virtual void render(CommandBuffer &command_buffer);`
* `virtual const std::vector<const char *> get_validation_layers();`
* `virtual void request_gpu_features(PhysicalDevice &gpu);`
* `virtual void create_render_context();`
* `virtual void prepare_render_context();`
* `virtual void reset_statsvirtual void draw_gui();_view(){};`
* `virtual void draw_gui();`
* `virtual void update_debug_window();`
![](/static/vk_application_structure_sample.png)

### RenderContext
**RenderContext acts as a frame manager for the sample**, with a lifetime that is the same as that of the Application itself. It acts as a container for RenderFrame objects, swapping between them (begin_frame, end_frame) and forwarding requests for Vulkan resources to the active frame.

#### RenderFrame
**RenderFrame is a container for per-frame data**, including BufferPool objects, synchronization primitives (semaphores, fences) and the swapchain RenderTarget. **The RenderFrame is responsible for creating a RenderTarget** using RenderTarget::CreateFunc. A RenderFrame cannot be destroyed individually since frames are managed by the RenderContext, the whole context must be destroyed. This is because each RenderFrame holds Vulkan objects such as the swapchain image.

#### RenderTarget
RenderTarget contains three vectors for: `core::Image`, `core::ImageView` and `Attachment`. The first two are Vulkan images and corresponding image views respectively.
![](/vk_application_structure_render_target.png)




## ApiVulkanSample
`ApiVulkanSample` is still a virtual class, but contains more specific object creations to get rid of verbose graphic pipeline object creation and provide helper function to create resources. Thus, it provide useful template but sacrifice the flexibility, which may affect the performance
![](/vk_application_structure_api_sample.png)

