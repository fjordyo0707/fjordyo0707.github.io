---
title: "Vulkan Learning RoadMap"
date: 2023-07-07T15:26:24-07:00
draft: False
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---

# Vulkan Learning RoadMap

## Fullfill list
- [x] https://vulkan-tutorial.com/
- [x] [My Vulkan Introduction](https://hackmd.io/ZC6AKFNMTnuZ_Vcq3gEklA)
- [ ] https://github.com/SaschaWillems/Vulkan#Examples
    * **Basic**
        * [x] [First triangle](/graphics/vulkan/vulkan-sample-note-first-triangle/)
        * [x] [**Pipelines**](/graphics/vulkan/vulkan-sample-note-pipelines)
        * [x] [**Descriptor sets**](/graphics/vulkan/vulkan-sample-note-descriptor-sets)
        * [x] [**Dynamic uniform buffers**](/graphics/vulkan/vulkan-sample-note-dynamic-uniform-buffers)
        * [X] [**Push constants**](/graphics/vulkan/vulkan-sample-note-push-constants)
        * [x] [Specialization constants](/graphics/vulkan/vulkan-sample-note-specialization-constants)
        * [X] [Texture mapping](/graphics/vulkan/vulkan-sample-note-texture-mapping)
        * [X] [Texture arrays](/graphics/vulkan/vulkan-sample-note-texture-arrays)
        * [X] [**Cube map textures**](/graphics/vulkan/vulkan-sample-note-cube-map-textures)
        * [ ] Cube map arrays
        * [ ] 3D texture
        * [ ] **Input attachments**
        * [ ] **Sub passes**
        * [ ] Offscreen rendering
        * [ ] CPU particle system
        * [ ] Stencil buffer
        * [ ] Vertex attributes
    * glTF
        * [ ] glTF model loading and rendering
        * [ ] glTF vertex skinning
        * [ ] glTF scene rendering
    * **Advanced**
        * [ ] Multi sampling
        * [ ] High dynamic range
        * [ ] **Shadow mapping**
        * [ ] Cascaded shadow mapping
        * [ ] Omnidirectional shadow mapping
        * [ ] Run-time mip-map generation
        * [ ] Capturing screenshots
        * [ ] Order Independent Transparency
    * **Performance**
        * [ ] Multi threaded command buffer generation
        * [ ] Instancing
        * [ ] Indirect drawing
        * [ ] Occlusion queries
        * [ ] Pipeline statistics
    * Physically Based Rendering
        * [ ] PBR basics
        * [ ] PBR image based lighting
        * [ ] Textured PBR with IBL
    * Deferred
        * [ ] Deferred shading basics
        * [ ] Deferred multi sampling
        * [ ] Deferred shading shadow mapping
        * [ ] Screen space ambient occlusion
    * Compute Shader
    * Geometry Shader
    * Tessellation Shader
    * Hardware accelerated ray tracing
    * Headless
    * **User Interface**
        * [ ] Text rendering
        * [ ] Distance field fonts
        * [ ] ImGui overlay




## RoadMap
Learning Vulkan, a low-level graphics and compute API, can be challenging but rewarding. Vulkan offers high performance and control over GPU resources, making it a popular choice for graphics and parallel computing applications. Here's a step-by-step guide on how to learn Vulkan:

1. ~~**Prerequisites:**~~
   Before diving into Vulkan, ensure you have a solid understanding of computer graphics concepts, linear algebra, and basic programming skills (C++ is recommended).

2. ~~**Official Documentation:**~~
   Start by reading the official Vulkan documentation provided by the Khronos Group. The documentation includes specifications, guides, tutorials, and code examples that cover all aspects of Vulkan programming.

   - [Vulkan Overview](https://www.khronos.org/vulkan/)
   - [Vulkan Documentation](https://www.khronos.org/registry/vulkan/)

3. ~~**Programming Environment Setup:**~~
   Set up your development environment with the necessary tools and libraries. You'll need a compatible GPU, a Vulkan-capable driver, and an SDK. The Vulkan SDK provides headers, libraries, and tools for development.

   - [Vulkan SDK](https://vulkan.lunarg.com/sdk/home)

4. ~~**Tutorials and Books:**~~
   There are several online tutorials and books available to help you learn Vulkan. Some recommended resources include:
   
   - ~~[Vulkan Tutorial](https://vulkan-tutorial.com/): A comprehensive tutorial that covers Vulkan fundamentals with code examples.~~
   - 🤩[LunarG: Introduction to Vulkan](https://vulkan.lunarg.com/doc/view/latest/windows/tutorial/html/00-intro.html)
   - [Vulkan Guide](https://vkguide.dev/)
   - "Introduction to Computer Graphics and the Vulkan API" by Kenwright and Shreiner: A book that offers an in-depth introduction to Vulkan.
   - "Vulkan Programming Guide: The Official Guide to Learning Vulkan" by Munshi, et al.: A comprehensive guide for learning Vulkan programming.

5. **Sample Code and Demos:**
   Study existing Vulkan sample code and demos to understand how different features are implemented. Many SDKs and tutorials provide sample projects that you can explore.
   * [SaschaWillems](https://github.com/SaschaWillems/Vulkan#glTF) 
   * [VulkanSamples](https://github.com/KhronosGroup/Vulkan-Samples/tree/main)

6. **Practice Projects:**
   Start with small projects to implement basic graphics rendering using Vulkan. Gradually increase the complexity of your projects as you become more comfortable with the API.

7. **API Reference:**
   Familiarize yourself with the Vulkan API reference to understand available functions, structures, and enumerations. This knowledge will be crucial for creating your own applications.

   - [Vulkan API Reference](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/)

8. **Debugging and Profiling:**
   Learn how to use debugging and profiling tools provided by the SDK to identify and fix issues in your Vulkan applications.

9. **Online Communities and Forums:**
   Join online communities and forums dedicated to Vulkan development. Engage with other developers, ask questions, and share your experiences. Some platforms include:

   - [Vulkan Subreddit](https://www.reddit.com/r/vulkan/)
   - [Khronos Vulkan Forums](https://community.khronos.org/c/vulkan/7)

10. **Continuous Learning:**
    Keep up with the latest developments in Vulkan and graphics programming. As technology evolves, new features and extensions may be introduced.

Remember that learning Vulkan takes time and practice. Be patient and persistent, and don't hesitate to ask for help when you encounter challenges. Building a strong foundation in Vulkan will open up exciting opportunities in graphics and parallel computing.

[Good GPU Hardware Introduction](https://fgiesen.wordpress.com/2011/07/09/a-trip-through-the-graphics-pipeline-2011-index/)