---
title: "Vulkan-Sample-Note-Push-Constants"
date: 2023-09-13T15:26:24-07:00
draft: False
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---
# Vulkan Sample Note: Push constants
The sample is from [Sascha Williems](https://github.com/SaschaWillems/Vulkan/tree/master/examples/)

## TL;DR
* Using push constants to pass a small bit of static data to a shader
* Perfect for passing e.g. static per-object data or parameters without the need for descriptor sets
* The spec only requires a minimum of 128 bytes, so for passing larger blocks of data you'd use UBOs or SSBOs

## Notes
The sample renders 16 different color and position spheres via pushing costant data. 
```cpp
struct SpherePushConstantData {
	glm::vec4 color;
	glm::vec4 position;
};
std::array<SpherePushConstantData, 16> spheres;
```

### Resources
#### setupDescriptorSetLayout
We create a `VkPushConstantRange` for pushing constants
```cpp
void setupDescriptorSetLayout()
{
	std::vector<VkDescriptorSetLayoutBinding> setLayoutBindings = {
	    // Binding 0 : Vertex shader uniform buffer
		vks::initializers::descriptorSetLayoutBinding(
			VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
			VK_SHADER_STAGE_VERTEX_BIT,				0),
        };

	VkDescriptorSetLayoutCreateInfo descriptorLayout = vks::initializers::descriptorSetLayoutCreateInfo(setLayoutBindings);
	VK_CHECK_RESULT(vkCreateDescriptorSetLayout(device, &descriptorLayout, nullptr, &descriptorSetLayout));

	// Define the push constant range used by the pipeline layout
	// Note that the spec only requires a minimum of 128 bytes, so for passing larger blocks of data you'd use UBOs or SSBOs
	VkPushConstantRange pushConstantRange{};
	pushConstantRange.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
	pushConstantRange.offset = 0;
	pushConstantRange.size = sizeof(SpherePushConstantData);

	VkPipelineLayoutCreateInfo pipelineLayoutCreateInfo = vks::initializers::pipelineLayoutCreateInfo(&descriptorSetLayout, 1);
	pipelineLayoutCreateInfo.pushConstantRangeCount  = 1;
	pipelineLayoutCreateInfo.pPushConstantRanges = &pushConstantRange;
	VK_CHECK_RESULT(vkCreatePipelineLayout(device, &pipelineLayoutCreateInfo, nullptr, &pipelineLayout));
}
```
#### buildCommandBuffers

```cpp
void buildCommandBuffers()
{
    // Set target frame buffer
    ...
    
    // [POI] Render the spheres passing color and position via push constants
    uint32_t spherecount = static_cast<uint32_t>(spheres.size());
    for (uint32_t j = 0; j < spherecount; j++) {
        // [POI] Pass static sphere data as push constants
        vkCmdPushConstants(
            drawCmdBuffers[i],
            pipelineLayout,
	    VK_SHADER_STAGE_VERTEX_BIT,
	    0,
	    sizeof(SpherePushConstantData),
	    &spheres[j]);
        model.draw(drawCmdBuffers[i]);
    }
    
    ...
}
```
