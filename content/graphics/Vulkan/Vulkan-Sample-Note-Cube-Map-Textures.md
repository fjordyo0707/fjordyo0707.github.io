---
title: "Vulkan-Sample-Note-Cube-Map-Textures"
date: 2023-10-07T15:26:24-07:00
draft: False
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---
# Vulkan Sample Note: Cube map textures
The sample is from [Sascha Williems](https://github.com/SaschaWillems/Vulkan/tree/master/examples/)

Loads a cube map texture from disk containing six different faces. All faces and mip levels are uploaded into video memory, and the cubemap is displayed on a skybox as a backdrop and on a 3D model as a reflection.

## TL;DR
* Load skybox texture, which including 6 faces
* Learn rendering with 2 pipeline and 2 descriptor set
* Understand reflection effect in shader




## Notes

### loadCubemap()
```cpp
        void loadCubemap(std::string filename, VkFormat format, bool forceLinearTiling)
        {
            ...

	// Create optimal tiled target image
	VkImageCreateInfo imageCreateInfo = vks::initializers::imageCreateInfo();
	imageCreateInfo.imageType = VK_IMAGE_TYPE_2D;
	imageCreateInfo.format = format;
	imageCreateInfo.mipLevels = cubeMap.mipLevels;
	imageCreateInfo.samples = VK_SAMPLE_COUNT_1_BIT;
	imageCreateInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
	imageCreateInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
	imageCreateInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
	imageCreateInfo.extent = { cubeMap.width, cubeMap.height, 1 };
	imageCreateInfo.usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
	// Cube faces count as array layers in Vulkan
	imageCreateInfo.arrayLayers = 6;
	// This flag is required for cube map images
	imageCreateInfo.flags = VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT;
 
        ...

	// Setup buffer copy regions for each face including all of its miplevels
	std::vector<VkBufferImageCopy> bufferCopyRegions;
	uint32_t offset = 0;

	for (uint32_t face = 0; face < 6; face++)
	{
		for (uint32_t level = 0; level < cubeMap.mipLevels; level++)
		{
			// Calculate offset into staging buffer for the current mip level and face
			ktx_size_t offset;
			KTX_error_code ret = ktxTexture_GetImageOffset(ktxTexture, level, 0, face, &offset);
			assert(ret == KTX_SUCCESS);
			VkBufferImageCopy bufferCopyRegion = {};
			bufferCopyRegion.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
			bufferCopyRegion.imageSubresource.mipLevel = level;
			bufferCopyRegion.imageSubresource.baseArrayLayer = face;
			bufferCopyRegion.imageSubresource.layerCount = 1;
			bufferCopyRegion.imageExtent.width = ktxTexture->baseWidth >> level;
			bufferCopyRegion.imageExtent.height = ktxTexture->baseHeight >> level;
			bufferCopyRegion.imageExtent.depth = 1;
			bufferCopyRegion.bufferOffset = offset;
			bufferCopyRegions.push_back(bufferCopyRegion);
		}
	}
            
        ...
            
	// Create image view
		VkImageViewCreateInfo view = vks::initializers::imageViewCreateInfo();
	// Cube map view type
	view.viewType = VK_IMAGE_VIEW_TYPE_CUBE;
	view.format = format;
	view.subresourceRange = { VK_IMAGE_ASPECT_COLOR_BIT, 0, 1, 0, 1 };
	// 6 array layers (faces)
	view.subresourceRange.layerCount = 6;
	// Set number of mip levels
	view.subresourceRange.levelCount = cubeMap.mipLevels;
	view.image = cubeMap.image;
	VK_CHECK_RESULT(vkCreateImageView(device, &view, nullptr, &cubeMap.view));
        }
```
The process is pretty much the same as other texture creation. However, 
there are two attributes need to be specified for the cube map image resources
* `imageCreateInfo.flags = VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT;`
* `view.viewType = VK_IMAGE_VIEW_TYPE_CUBE;`

### Shader Code
#### Skybox
**Fragment Shader:**
```cpp
#version 450

layout (binding = 1) uniform samplerCube samplerCubeMap;

layout (location = 0) in vec3 inUVW;

layout (location = 0) out vec4 outFragColor;

void main() 
{
	outFragColor = texture(samplerCubeMap, inUVW);
}
```
keyword `samplerCube` is set for the cube map texture.

### Two Pipeline and descriptor Set
![](/Vk_Structure_Cubemap.png)

### Reflection
Let's take a look at the object fragment code.
```cpp
#version 450

layout (binding = 1) uniform samplerCube samplerColor;

layout (binding = 0) uniform UBO
{
	mat4 projection;
	mat4 model;
	mat4 invModel;
	float lodBias;
} ubo;

layout (location = 0) in vec3 inPos;
layout (location = 1) in vec3 inNormal;
layout (location = 2) in vec3 inViewVec;
layout (location = 3) in vec3 inLightVec;

layout (location = 0) out vec4 outFragColor;

void main()
{
	vec3 cI = normalize (inPos);
	vec3 cR = reflect (cI, normalize(inNormal));

	cR = vec3(ubo.invModel * vec4(cR, 0.0));
	// Convert cubemap coordinates into Vulkan coordinate space
	cR.xy *= -1.0;

	vec4 color = texture(samplerColor, cR, ubo.lodBias);

	vec3 N = normalize(inNormal);
	vec3 L = normalize(inLightVec);
	vec3 V = normalize(inViewVec);
	vec3 R = reflect(-L, N);
	vec3 ambient = vec3(0.5) * color.rgb;
	vec3 diffuse = max(dot(N, L), 0.0) * vec3(1.0);
	vec3 specular = pow(max(dot(R, V), 0.0), 16.0) * vec3(0.5);
	outFragColor = vec4(ambient + diffuse * color.rgb + specular, 1.0);
}
```
* `inPos` is the object primitive in world space
* `inNormal` is the object primitive normal in world space
* `inViewVec` is the vector from camera to  object
* `inLightVec` is the vector from light to  object

### Reflection Calculation
```cpp
	vec3 cI = normalize (inPos);
	vec3 cR = reflect (cI, normalize(inNormal));

	cR = vec3(ubo.invModel * vec4(cR, 0.0));
	// Convert cubemap coordinates into Vulkan coordinate space
	cR.xy *= -1.0;

	vec4 color = texture(samplerColor, cR, ubo.lodBias);
```
Our utlimate goal is to find the mapping from object to the unit cube. First, we will find the mapping to unit sphere and the unit sphere could be sampled throught texture cubemap. (Please correct me if I m wrong) 
1. Treat `inPos` as a vector (from object to camera).  Normalize it as `cI`.
2. Caluclate the reflection `cR` from `inNormal` and `cI`.
3. Transform the reflection `cR` back to model space
4. Convert cubemap coordinates into Vulkan coordinate space
5. Sample from the cube map


