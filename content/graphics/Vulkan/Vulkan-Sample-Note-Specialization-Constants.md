---
title: "Vulkan-Sample-Note-Specialization-Constants"
date: 2023-09-20T15:26:24-07:00
draft: False
author: Cheng-Yu Fan
tags: ["Vulkan"]
categories: ["Graphics"]
---
# Vulkan Sample Note: Specialization constants
The sample is from [Sascha Williems](https://github.com/SaschaWillems/Vulkan/tree/master/examples/)

## TL;DR
* Create specialization constants for control variables in shader code
* Specialization constants could be seen as per shader or per pipeline constants


## Notes

### preparePipelines
Specialization info is assigned is part of the shader stage (modul) and must be set after creating the module and before creating the pipeline
```cpp
void preparePipelines()
{
    ...
    // Prepare specialization data

    // Host data to take specialization constants from
    struct SpecializationData {
    // Sets the lighting model used in the fragment "uber" shader
    	uint32_t lightingModel;
    	// Parameter for the toon shading part of the fragment shader
    	float toonDesaturationFactor = 0.5f;
    } specializationData;
    
    // Each shader constant of a shader stage corresponds to one map entry
    std::array<VkSpecializationMapEntry, 2> specializationMapEntries;
    // Shader bindings based on specialization constants are marked by the new "constant_id" layout qualifier:
    //	layout (constant_id = 0) const int LIGHTING_MODEL = 0;
    //	layout (constant_id = 1) const float PARAM_TOON_DESATURATION = 0.0f;
    
    // Map entry for the lighting model to be used by the fragment shader
    specializationMapEntries[0].constantID = 0;
    specializationMapEntries[0].size = sizeof(specializationData.lightingModel);
    specializationMapEntries[0].offset = 0;
    
    // Map entry for the toon shader parameter
    specializationMapEntries[1].constantID = 1;
    specializationMapEntries[1].size = sizeof(specializationData.toonDesaturationFactor);
    specializationMapEntries[1].offset = offsetof(SpecializationData, toonDesaturationFactor);
    
    // Prepare specialization info block for the shader stage
    VkSpecializationInfo specializationInfo{};
    specializationInfo.dataSize = sizeof(specializationData);
    specializationInfo.mapEntryCount = static_cast<uint32_t>(specializationMapEntries.size());
    specializationInfo.pMapEntries = specializationMapEntries.data();
    specializationInfo.pData = &specializationData; 

    // Create pipelines
    // All pipelines will use the same "uber" shader and specialization constants to change branching and parameters of that shader
    shaderStages[0] = loadShader(getShadersPath() + "specializationconstants/uber.vert.spv", VK_SHADER_STAGE_VERTEX_BIT);
    shaderStages[1] = loadShader(getShadersPath() + "specializationconstants/uber.frag.spv", VK_SHADER_STAGE_FRAGMENT_BIT);
    // Specialization info is assigned is part of the shader stage (modul) and must be set after creating the module and before creating the pipeline
    shaderStages[1].pSpecializationInfo = &specializationInfo;

    // Solid phong shading
    specializationData.lightingModel = 0;
    VK_CHECK_RESULT(vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCI, nullptr, &pipelines.phong));

    // Phong and textured
    specializationData.lightingModel = 1;
    VK_CHECK_RESULT(vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCI, nullptr, &pipelines.toon));
    
    // Textured discard
    specializationData.lightingModel = 2;
    VK_CHECK_RESULT(vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCI, nullptr, &pipelines.textured));
}
```

### fragment shader

```cpp
#version 450
...
// We use this constant to control the flow of the shader depending on the 
// lighting model selected at pipeline creation time
layout (constant_id = 0) const int LIGHTING_MODEL = 0;
// Parameter for the toon shading part of the shader
layout (constant_id = 1) const float PARAM_TOON_DESATURATION = 0.0f; 

void main()
{
    switch (LIGHTING_MODEL) {
        case 0: // Phong		
            {
                ...
            }
        case 1: // Toon		
            {
                ...
            }
        case 2: // Textured
            {
               ... 
            }
    }
}
```