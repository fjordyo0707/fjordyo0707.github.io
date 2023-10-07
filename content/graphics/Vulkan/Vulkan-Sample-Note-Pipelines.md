---
title: "Vulkan-Sample-Note-Pipelines"
date: 2023-08-27T15:26:24-07:00
draft: False
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---
# Vulkan Sample Note: Pipeline
The sample is from [Sascha Williems](https://github.com/SaschaWillems/Vulkan/tree/master/examples/pipelines)
## TL;DR
* Understand `VulkanExampleBase` class
* Set up multiple pipelines for different rendering
* Learn glTF format
* Lean Basic UI

## Application Flow
Every vulkan example have a basic work flow to follow: 
![](/VK_Applicarion_Flow.png)


## `VulkanExampleBase` class
Vulkan is a pretty verbose API. To exploit every piece of performance, it is required to set up every single detail in your application. Thus, creating a base class to handle repeating setup and configuration will be extremely helpful. Let the base class get rid of the boring part, then we can enjoy some interesting rendering or performance tuning. 

### What does `VulkanExampleBase` do?

#### Prepare
Here is the prepare stage:
1. Initialize Vulkan
    * Create Instance
    * Enumerate physcial devices and select one
    * Store properties (including limits), features and memory properties of the physical device
    * Create a logical device from physical device
    * Get a graphics queue from the device
    * Create synchronization objects
        * A semaphore used to synchronize image presentation
            * Ensures that the image is displayed before we start submitting new commands to the queue
        * A semaphore used to synchronize command submission
            * Ensures that the image is not presented until all commands have been submitted and executed
3. Prepare
    * Initialize swapchains
    * Create command pool
    * Create command buffers
    * Create synchronization primitives
    * Set up depth stencil
    * Setup renderpass
    * Create pipeline cache
    * Set up frame buffer
    * Set up UI



### What should the application implement?

1. Prepare: application resources
    * Load shader
    * Load assets
    * Set up uniform buffer
    * Set up descriptor pool, descriptor set layout, descriptor set
    * Create pipeline and pipeline layout. 
2. Render: render command submit to device
    * Create pipeline, pipeline layout
    * Build up command buffers
3. (Optinal) Get physical device features, extensions
4. Change view


## Build Multiple pipelines
In this example, the application aims to render a glTF object with different shader with three pipeline, toon, phong and wireframe. The flow is mostly same as the regular rendering: Set up and binding resources, prepare pipelines, build up command buffers. However, during preparing pipelines, we will bind different pipelines with different shaders, and during building up command buffers, we will set up different viewport for each.

**Different shaders:**
```cpp
    // Textured pipeline
    // Phong shading pipeline
    shaderStages[0] = loadShader(getShadersPath() + "pipelines/phong.vert.spv", VK_SHADER_STAGE_VERTEX_BIT);
    shaderStages[1] = loadShader(getShadersPath() + "pipelines/phong.frag.spv", VK_SHADER_STAGE_FRAGMENT_BIT);
    VK_CHECK_RESULT(vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCI, nullptr, &pipelines.phong));

    // All pipelines created after the base pipeline will be derivatives
    pipelineCI.flags = VK_PIPELINE_CREATE_DERIVATIVE_BIT;
    // Base pipeline will be our first created pipeline
    pipelineCI.basePipelineHandle = pipelines.phong;
// It's only allowed to either use a handle or index for the base pipeline
    // As we use the handle, we must set the index to -1 (see section 9.5 of the specification)
    pipelineCI.basePipelineIndex = -1;

    // Toon shading pipeline
    shaderStages[0] = loadShader(getShadersPath() + "pipelines/toon.vert.spv", VK_SHADER_STAGE_VERTEX_BIT);
    shaderStages[1] = loadShader(getShadersPath() + "pipelines/toon.frag.spv", VK_SHADER_STAGE_FRAGMENT_BIT);
    VK_CHECK_RESULT(vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCI, nullptr, &pipelines.toon));

    // Pipeline for wire frame rendering
    // Non solid rendering is not a mandatory Vulkan feature
    if (enabledFeatures.fillModeNonSolid)
    {
    	rasterizationState.polygonMode = VK_POLYGON_MODE_LINE;
    	shaderStages[0] = loadShader(getShadersPath() + "pipelines/wireframe.vert.spv", VK_SHADER_STAGE_VERTEX_BIT);
	shaderStages[1] = loadShader(getShadersPath() + "pipelines/wireframe.frag.spv", VK_SHADER_STAGE_FRAGMENT_BIT);
	VK_CHECK_RESULT(vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCI, nullptr, &pipelines.wireframe));
    }
```
**Different viewport:**
```cpp
    for (int32_t i = 0; i < drawCmdBuffers.size(); ++i)
	{
		// Set target frame buffer
		renderPassBeginInfo.framebuffer = frameBuffers[i];

		VK_CHECK_RESULT(vkBeginCommandBuffer(drawCmdBuffers[i], &cmdBufInfo));

		vkCmdBeginRenderPass(drawCmdBuffers[i], &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);

		VkViewport viewport = vks::initializers::viewport((float)width, (float)height, 0.0f, 1.0f);
		vkCmdSetViewport(drawCmdBuffers[i], 0, 1, &viewport);

		VkRect2D scissor = vks::initializers::rect2D(width, height,	0, 0);
		vkCmdSetScissor(drawCmdBuffers[i], 0, 1, &scissor);

		vkCmdBindDescriptorSets(drawCmdBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout, 0, 1, &descriptorSet, 0, NULL);
		scene.bindBuffers(drawCmdBuffers[i]);

		// Left : Solid colored
		viewport.width = (float)width / 3.0;
		vkCmdSetViewport(drawCmdBuffers[i], 0, 1, &viewport);
		vkCmdBindPipeline(drawCmdBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.phong);
		vkCmdSetLineWidth(drawCmdBuffers[i], 1.0f);
		scene.draw(drawCmdBuffers[i]);

		// Center : Toon
		viewport.x = (float)width / 3.0;
		vkCmdSetViewport(drawCmdBuffers[i], 0, 1, &viewport);
		vkCmdBindPipeline(drawCmdBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.toon);
		// Line width > 1.0f only if wide lines feature is supported
		if (enabledFeatures.wideLines) {
			vkCmdSetLineWidth(drawCmdBuffers[i], 2.0f);
		}
		scene.draw(drawCmdBuffers[i]);

		if (enabledFeatures.fillModeNonSolid)
		{
			// Right : Wireframe
			viewport.x = (float)width / 3.0 + (float)width / 3.0;
			vkCmdSetViewport(drawCmdBuffers[i], 0, 1, &viewport);
				vkCmdBindPipeline(drawCmdBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, pipelines.wireframe);
			scene.draw(drawCmdBuffers[i]);
		}

		drawUI(drawCmdBuffers[i]);

		vkCmdEndRenderPass(drawCmdBuffers[i]);

		VK_CHECK_RESULT(vkEndCommandBuffer(drawCmdBuffers[i]));
	}
```

## glTF format
* [details](https://github.com/KhronosGroup/glTF-Tutorials/blob/master/gltfTutorial/README.md)

### Where does the vertex attribute set?
We let the `vkglTF` library take cares of vertex, texture, indices ... and let it create those buffers for us. However, in the shader code, we need to know the input variables.
```cpp
layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inNormal;
layout (location = 2) in vec3 inColor;
```
These locations need to be set manually to fit the shader input. We set it during pipeline creation
```cpp
VkGraphicsPipelineCreateInfo pipelineCI = vks::initializers::pipelineCreateInfo(pipelineLayout, renderPass);
pipelineCI.pVertexInputState  = vkglTF::Vertex::getPipelineVertexInputState({vkglTF::VertexComponent::Position, vkglTF::VertexComponent::Normal, vkglTF::VertexComponent::Color});
```