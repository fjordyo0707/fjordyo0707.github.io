---
title: "Vulkan-Sample-Note-Texture-Mapping"
date: 2023-09-30T15:26:24-07:00
draft: False
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---
# Vulkan Sample Note: Texture Mapping
The sample is from [Sascha Williems](https://github.com/SaschaWillems/Vulkan/tree/master/examples/)

## TL;DR
* Understand essential object needs for a texture object
* Understand how to create texture resources
* Understand how to use the texutre inside shader via image sampler
* Understand the step for texture creation and copy during creating the graphic pipeline
* Learn to control the imgui to change texture Level of Details


## Notes

### What does a texutre object need?
```cpp
	// Contains all Vulkan objects that are required to store and use a texture
	// Note that this repository contains a texture class (VulkanTexture.hpp) that encapsulates texture loading functionality in a class that is used in subsequent demos
	struct Texture {
		VkSampler sampler;
		VkImage image;
		VkImageLayout imageLayout;
		VkDeviceMemory deviceMemory;
		VkImageView view;
		uint32_t width, height;
		uint32_t mipLevels;
	} texture;
```

### Tiling
Tiling could also referred to **memory layout**.
There are two types of image tiling in Vulkan
* Linear tiled images:
    * These are stored as is and can be copied directly to. But due to the linear nature they're not a good match for GPUs and format and feature support is very limited.
    * It's not advised to use linear tiled images for anything else than copying from host to GPU if buffer copies are not an option.
    * Linear tiling is thus only implemented for learning purposes, one should always prefer optimal tiled image.
* Optimal tiled images:
    * These are stored in an implementation specific layout matching the capability of the hardware. They usually support more formats and features and are much faster.
    * Optimal tiled images are stored on the device and not accessible by the host. So they can't be written directly to (like liner tiled images) and always require
			some sort of data copy, either from a buffer or	a linear tiled image.
            
**Always use optimal tiled images for rendering**

### loadTexture()

#### What is KTX format
KTX (Khronos Texture) is an efficient, lightweight container format for reliably distributing GPU textures to diverse platforms and applications.
![](/Vk_Texture_KTX.png)
[source](https://www.khronos.org/ktx/)

Loading texture contains multiple steps, which may be a bit verbose, but it is nice to understand the details. Generally, we can split it to the below step:
1. Load asset to KTX object
2. Copy data to an optimal tiled image: <span style="color:blue">related to staging buffer</span>, <span style="color:green">related to device(GPU) </span>,  <span style="color:red">related to synchronization</span>, <span style="color:orange"> staging buffer to device</span> 
    1. <span style="color:blue">Create a host-visible staging buffer</span>
    2.  <span style="color:blue">Copy texture data into host local staging buffer</span>
    3.  <span style="color:blue">Setup buffer copy regions for each mip level</span>
    4. <span style="color:green">Create optimal tiled target image on the device (GPU). Set initial layout of the image to undefined</span>
    5. <span style="color:green">Create command buffer for image layout transition</span>
    6.  <span style="color:orange">Set up image memory barriers for the texture image, including the sub resource, old new layout.</span>
    7. <span style="color:red">Insert a memory dependency at the proper pipeline stages that will execute the image layout transition</span>
    8. <span style="color:orange">Copy mip levels from staging buffer</span>
    9. <span style="color:green">Once the data has been uploaded we transfer to the texture image to the shader read layout, so it can be sampled</span>
    10. <span style="color:red">Insert a memory dependency at the proper pipeline stages that will execute the image layout transition</span>
    11. Store current layout for later reuse
3. Create a texture sampler
    * Textures are accessed by samplers
    * Have multiple sampler objects for the same texture with different settings
4. Create image view
    * Textures are not directly accessed by the shaders
	* Textures are **abstracted by image views containing additional information and sub resource ranges**

### Resrouces

#### Descriptor Pool
```cpp
	void setupDescriptorPool()
	{
		// Example uses one ubo and one image sampler
		std::vector<VkDescriptorPoolSize> poolSizes =
		{
			vks::initializers::descriptorPoolSize(VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, 1),
			vks::initializers::descriptorPoolSize(VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, 1)
		};

		VkDescriptorPoolCreateInfo descriptorPoolInfo =
			vks::initializers::descriptorPoolCreateInfo(
				static_cast<uint32_t>(poolSizes.size()),
				poolSizes.data(),
				2);

		VK_CHECK_RESULT(vkCreateDescriptorPool(device, &descriptorPoolInfo, nullptr, &descriptorPool));
	}
```

#### Descriptor Set Layout
```cpp
	void setupDescriptorSetLayout()
	{
		std::vector<VkDescriptorSetLayoutBinding> setLayoutBindings =
		{
			// Binding 0 : Vertex shader uniform buffer
			vks::initializers::descriptorSetLayoutBinding(
				VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
				VK_SHADER_STAGE_VERTEX_BIT,
				0),
			// Binding 1 : Fragment shader image sampler
			vks::initializers::descriptorSetLayoutBinding(
				VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
				VK_SHADER_STAGE_FRAGMENT_BIT,
				1)
		};

		VkDescriptorSetLayoutCreateInfo descriptorLayout =
			vks::initializers::descriptorSetLayoutCreateInfo(
				setLayoutBindings.data(),
				static_cast<uint32_t>(setLayoutBindings.size()));

		VK_CHECK_RESULT(vkCreateDescriptorSetLayout(device, &descriptorLayout, nullptr, &descriptorSetLayout));

		VkPipelineLayoutCreateInfo pPipelineLayoutCreateInfo =
			vks::initializers::pipelineLayoutCreateInfo(
				&descriptorSetLayout,
				1);

		VK_CHECK_RESULT(vkCreatePipelineLayout(device, &pPipelineLayoutCreateInfo, nullptr, &pipelineLayout));
	}
```

#### Descriptor Set
```cpp
	void setupDescriptorSet()
	{
		VkDescriptorSetAllocateInfo allocInfo =
			vks::initializers::descriptorSetAllocateInfo(
				descriptorPool,
				&descriptorSetLayout,
				1);

		VK_CHECK_RESULT(vkAllocateDescriptorSets(device, &allocInfo, &descriptorSet));

		// Setup a descriptor image info for the current texture to be used as a combined image sampler
		VkDescriptorImageInfo textureDescriptor;
		textureDescriptor.imageView = texture.view;				// The image's view (images are never directly accessed by the shader, but rather through views defining subresources)
		textureDescriptor.sampler = texture.sampler;			// The sampler (Telling the pipeline how to sample the texture, including repeat, border, etc.)
		textureDescriptor.imageLayout = texture.imageLayout;	// The current layout of the image (Note: Should always fit the actual use, e.g. shader read)

		std::vector<VkWriteDescriptorSet> writeDescriptorSets =
		{
			// Binding 0 : Vertex shader uniform buffer
			vks::initializers::writeDescriptorSet(
				descriptorSet,
				VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
				0,
				&uniformBufferVS.descriptor),
			// Binding 1 : Fragment shader texture sampler
			//	Fragment shader: layout (binding = 1) uniform sampler2D samplerColor;
			vks::initializers::writeDescriptorSet(
				descriptorSet,
				VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,		// The descriptor set will use a combined image sampler (sampler and image could be split)
				1,												// Shader binding point 1
				&textureDescriptor)								// Pointer to the descriptor image for our texture
		};

		vkUpdateDescriptorSets(device, static_cast<uint32_t>(writeDescriptorSets.size()), writeDescriptorSets.data(), 0, NULL);
	}
```
Some interesting comment here:
* The image's view (images are never directly accessed by the shader, but rather through views defining subresources)
* The sampler (Telling the pipeline how to sample the texture, including repeat, border, etc.)
* The current layout of the image (Note: Should always fit the actual use, e.g. shader read)

### Imgui Usage
```cpp
	virtual void OnUpdateUIOverlay(vks::UIOverlay *overlay)
	{
		if (overlay->header("Settings")) {
			if (overlay->sliderFloat("LOD bias", &uboVS.lodBias, 0.0f, (float)texture.mipLevels)) {
				updateUniformBuffers();
			}
		}
	}
```
The GUI usage on the windows is pretty straight-forward.

```cpp
overlay->sliderFloat("LOD bias", &uboVS.lodBias, 0.0f, (float)texture.mipLevels)
```
are declared as: 
```cpp
bool vks::UIOverlay::sliderFloat(const char *caption, float *value, float min, float max);
```

### Further study 
* [Pipeline Barriers](https://arm-software.github.io/vulkan_best_practice_for_mobile_developers/samples/performance/pipeline_barriers/pipeline_barriers_tutorial.html)
