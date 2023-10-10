---
title: "Vulkan-Sample-Note-Texture-Arrays"
date: 2023-10-06T15:26:24-07:00
draft: False
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---
# Vulkan Sample Note: Texture arrays
The sample is from [Sascha Williems](https://github.com/SaschaWillems/Vulkan/tree/master/examples/)

> Loads a 2D texture array containing multiple 2D texture slices (each with its own mip chain) and renders multiple meshes each sampling from a different layer of the texture. 2D texture arrays don't do any interpolation between the slices.

## TL;DR
* How to correctly map the ubo memory from host to buffer
* Learn instance rendering
* Learn loading texutre layer in texture array (useful for skybox)
![](/VK_texture_array.png)


## Notes

### What does `vkMapMemory` do?
```cpp
		// Update instanced part of the uniform buffer
		uint8_t *pData;
		uint32_t dataOffset = sizeof(uboVS.matrices);
		uint32_t dataSize = layerCount * sizeof(UboInstanceData);
		VK_CHECK_RESULT(vkMapMemory(device, uniformBufferVS.memory, dataOffset, dataSize, 0, (void **)&pData));
		memcpy(pData, uboVS.instance, dataSize);
		vkUnmapMemory(device, uniformBufferVS.memory);
```
The code above tried to copy the host data to the device. The host doesn't know the device address, so it requires `vkMapMemory` to let the cpu pointer get the pointer in the device. In short,
> To "map memory" in Vulkan means to obtain a CPU pointer to VkDeviceMemory, to be able to read from it or write to it in CPU code.

#### What does `vkUnmapMemory` do?
The CPU pointer doesn't need to know the device address anymore.

#### How can you keep map persistent?
```cpp
		// Update instanced part of the uniform buffer
		uint8_t *pData;
		uint32_t dataOffset = sizeof(uboVS.matrices);
		uint32_t dataSize = layerCount * sizeof(UboInstanceData);
		VK_CHECK_RESULT(vkMapMemory(device, uniformBufferVS.memory, dataOffset, dataSize, 0, (void **)&pData));
		memcpy(pData, uboVS.instance, dataSize);
		vkUnmapMemory(device, uniformBufferVS.memory);

		// Map persistent
		VK_CHECK_RESULT(uniformBufferVS.map());

		updateUniformBuffersCamera();
	}

	void updateUniformBuffersCamera()
	{
		uboVS.matrices.projection = camera.matrices.perspective;
		uboVS.matrices.view = camera.matrices.view;
		memcpy(uniformBufferVS.mapped, &uboVS.matrices, sizeof(uboVS.matrices));
	}
```
The `vks::Buffer uniformBufferVS;` has a built in function `map()` to keep the buffer memory map, also meanwhile it records the offset of the current buffer already used

```cpp
	/** 
	* Map a memory range of this buffer. If successful, mapped points to the specified buffer range.
	* 
	* @param size (Optional) Size of the memory range to map. Pass VK_WHOLE_SIZE to map the complete buffer range.
	* @param offset (Optional) Byte offset from beginning
	* 
	* @return VkResult of the buffer mapping call
	*/
	VkResult Buffer::map(VkDeviceSize size, VkDeviceSize offset)
	{
		return vkMapMemory(device, memory, offset, size, 0, &mapped);
	}
```
We could see after that in `updateUniformBuffersCamera()` function, it just requires `memcpy(...)`

### Instancing rendering
We could call instance rendering via `vkCmdDrawIndexed` 
```cpp
vkCmdDrawIndexed(command_buffer, indices_size, instance_count, 0, 0, instance_first_index);
```
* `instance_count`: number of instances to draw
* `instance_first_index`: first instance will have this id
Inside shader, use variable `gl_InstanceIndex` to access `instance_first_index` instance id

#### Example
Application Code:
```cpp
...
	struct UboInstanceData {
		// Model matrix
		glm::mat4 model;
		// Texture array index
		// Vec4 due to padding
		glm::vec4 arrayIndex;
	};

	struct {
		// Global matrices
		struct {
			glm::mat4 projection;
			glm::mat4 view;
		} matrices;
		// Separate data for each instance
		UboInstanceData *instance;
	} uboVS;
...

```
**Vertex Shader:**
```cpp
#version 450

layout (location = 0) in vec3 inPos;
layout (location = 1) in vec2 inUV;

struct Instance
{
	mat4 model;
	vec4 arrayIndex;
};

layout (binding = 0) uniform UBO 
{
	mat4 projection;
	mat4 view;
	Instance instance[8];
} ubo;

layout (location = 0) out vec3 outUV;

void main() 
{
	outUV = vec3(inUV, ubo.instance[gl_InstanceIndex].arrayIndex.x);
	mat4 modelView = ubo.view * ubo.instance[gl_InstanceIndex].model;
	gl_Position = ubo.projection * modelView * vec4(inPos, 1.0);
}

```

Fragment Shader:
```cpp
#version 450

layout (binding = 1) uniform sampler2DArray samplerArray;

layout (location = 0) in vec3 inUV;

layout (location = 0) out vec4 outFragColor;

void main() 
{
	outFragColor = texture(samplerArray, inUV);
}
```
In fragment shader, we need to specify the uniform with the keyword *sampler2DArray* to achieve the right sampling input, which is `(U, V, ArrayIndex)`.

### Texture Array
Thanks for the reply [Iburinoc on Reddit](https://www.reddit.com/r/vulkan/comments/a4r32s/whats_the_aspectlevellayer_in_a_vulkan_image/ebgycha/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button)
> As a rough overview:
> 
> Aspect: Images can hold various types of data, in Vulkan either color (the type of image you'd normally think of), depth (where the image represents the distance from the camera to objects in the scene), and stencil (a bitmap that can be used for any purpose). The aspect indicates how it's meant to be used and what type of data it contains.
> 
> Level: Googling mipmaps will give you a good overview here, but the idea is that if you're sampling a texture that's far away, to improve performance and visuals, the GPU can sample a down-scaled version of the original image. Often you'll have many levels of mipmap, with level 0 being the original image, level 1 being the half the width and half the height, and so on.
> 
> Layer: Images in Vulkan can actually be an array of images, called layers. I'm not too familiar with these but I assume they're used for e.g. animations and that kind of thing.
> 
> VkImageSubresourceLayers is a parameter struct used to select a specific subset of the resources of an image, in this case, which aspect and mipmap level, as well as a range of array layers.

```cpp
...
		// Setup buffer copy regions for array layers
		std::vector<VkBufferImageCopy> bufferCopyRegions;

		// To keep this simple, we will only load layers and no mip level
		for (uint32_t layer = 0; layer < layerCount; layer++)
		{
			// Calculate offset into staging buffer for the current array layer
			ktx_size_t offset;
			KTX_error_code ret = ktxTexture_GetImageOffset(ktxTexture, 0, layer, 0, &offset);
			assert(ret == KTX_SUCCESS);
			// Setup a buffer image copy structure for the current array layer
			VkBufferImageCopy bufferCopyRegion = {};
			bufferCopyRegion.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
			bufferCopyRegion.imageSubresource.mipLevel = 0;
			bufferCopyRegion.imageSubresource.baseArrayLayer = layer;
			bufferCopyRegion.imageSubresource.layerCount = 1;
			bufferCopyRegion.imageExtent.width = ktxTexture->baseWidth;
			bufferCopyRegion.imageExtent.height = ktxTexture->baseHeight;
			bufferCopyRegion.imageExtent.depth = 1;
			bufferCopyRegion.bufferOffset = offset;
			bufferCopyRegions.push_back(bufferCopyRegion);
		}

		// Create optimal tiled target image
		VkImageCreateInfo imageCreateInfo = vks::initializers::imageCreateInfo();
		imageCreateInfo.imageType = VK_IMAGE_TYPE_2D;
		imageCreateInfo.format = format;
		imageCreateInfo.mipLevels = 1;
		imageCreateInfo.samples = VK_SAMPLE_COUNT_1_BIT;
		imageCreateInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
		imageCreateInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
		imageCreateInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
		imageCreateInfo.extent = { textureArray.width, textureArray.height, 1 };
		imageCreateInfo.usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
		imageCreateInfo.arrayLayers = layerCount;
...
		// Create image view
		VkImageViewCreateInfo view = vks::initializers::imageViewCreateInfo();
		view.viewType = VK_IMAGE_VIEW_TYPE_2D_ARRAY;
		view.format = format;
		view.subresourceRange = { VK_IMAGE_ASPECT_COLOR_BIT, 0, 1, 0, 1 };
		view.subresourceRange.layerCount = layerCount;
		view.subresourceRange.levelCount = 1;
		view.image = textureArray.image;
		VK_CHECK_RESULT(vkCreateImageView(device, &view, nullptr, &textureArray.view));  
```

View type need to be set during creating the image view, `view.viewType = VK_IMAGE_VIEW_TYPE_2D_ARRAY;`

### Different between 2D array texture and 3D texture
> A 2D array texture is a 2D texture where each mipmap level contains an array of 2D images. A 3D texture is a texture where each mipmap level contains a single three-dimensional image.
> 
> The purpose of a 2D array texture is to be able to have multiple 2D textures in a single object. That way, the shader itself can select which image to use for each particular object. Texture coordinates mean the same thing as 2D textures: representing a location in 2D space. It’s just that you also have a selector that says which 2D image in the array to use.
> 
> 3D textures are all about sampling from within a volume of data. That is, you have texture coordinates which are three-dimensional in nature, perhaps representing a position within the texture’s volumetric space.

[source: khronos community](https://community.khronos.org/t/2d-texture-array-vs-3d-texture/73003/2)
