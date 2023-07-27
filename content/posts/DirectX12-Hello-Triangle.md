---
title: "DirectX12 Hello Triangle"
date: 2023-07-27T15:39:41-07:00
draft: false
---
# DirectX12 Hello Triangle
The online resources for starting DirectX12  to beccoming a intermediate level has a pretty steep learning curve. At least for me, it tooks me a while to find the next step after drawing my first triangle. Thus, I would like to start from the github repo [Microsoft DirectX-Graphics-Sample](https://github.com/microsoft/DirectX-Graphics-Samples/tree/master).

This code in this post is located at the repo `DirectX-Graphics-Samples/tree/master/Samples/Desktop/D3D12HelloWorld/src/HelloTriangle`. Feel free to build and test it by yourself. 

> Before you start reading, I would recommend you read [my previous post: DirectX12 Intoduction](https://fjordyo0707.github.io/posts/directx12-intoduction/) to have better understanding

## Headers
### Public
Let's take a look at `D3D12HelloTriangle.h`. The public method consists one constructor and four virtual overwrite functions, `OnInit`, `OnUpdate`, `OnRender`, `OnDestroy`.
* **OnInit**: Load pipeline.and assets 
    * Declare object memory, like enable debug layer, create device adapter, command queue, swap chain, desciptor heap, frame resources.
    * Create root signature. Compiling and loading shaders. Create a pipeline state object, command lists, a vertex buffer, sync objects. Upload assets to GPU. 
* **OnUpdate**: Update the resources and send to GPU. 
* **OnRender**: Record all the commands we need to render the scene into the command list. Execute the command list. Presnet the frame from swap chain. Wait for the frame to complete
* **OnDestroy**: Clean the reources

### Private
In private object parts, the private functions are all helper functions to organize the code. For attributes, I liked the example seperate them into three different kinds of category, pipeline objects, app resources, synchronization objects.
#### Pipeline objects
```C++
    // Pipeline objects.
    CD3DX12_VIEWPORT m_viewport;
    CD3DX12_RECT m_scissorRect;
    ComPtr<IDXGISwapChain3> m_swapChain;
    ComPtr<ID3D12Device> m_device;
    ComPtr<ID3D12Resource> m_renderTargets[FrameCount];
    ComPtr<ID3D12CommandAllocator> m_commandAllocator;
    ComPtr<ID3D12CommandQueue> m_commandQueue;
    ComPtr<ID3D12RootSignature> m_rootSignature;
    ComPtr<ID3D12DescriptorHeap> m_rtvHeap;
    ComPtr<ID3D12PipelineState> m_pipelineState;
    ComPtr<ID3D12GraphicsCommandList> m_commandList;
    UINT m_rtvDescriptorSize;
```

#### App resources
```
    ComPtr<ID3D12Resource> m_vertexBuffer;
    D3D12_VERTEX_BUFFER_VIEW m_vertexBufferView;
```

#### Synchronization objects
```
    UINT m_frameIndex;
    HANDLE m_fenceEvent;
    ComPtr<ID3D12Fence> m_fence;
    UINT64 m_fenceValue;
```

## CPP Files
### Public
I would explain it in the sequenctial way in the code. I wouldn't dig in every section details. Just provide a big pictures what is going on right now

#### OnInit
1. Enable the debug layer. Here is the statement from [Microsoft Doc: Using the debug layer to debug apps](https://learn.microsoft.com/en-us/windows/win32/direct3d11/using-the-debug-layer-to-test-apps)
> The debug layer helps you write Direct3D code. In addition, your productivity can increase when you use the debug layer because you can immediately see the causes of obscure rendering errors or even black screens at their source. Forgot to set a texture but read from it in your pixel shader
Output depth but have no depth-stencil state bound
Texture creation failed with INVALIDARG
2. Get the GPU device with `IDXGIAdapter `. 
> IDXGIAdapter is an interface used to represent an adapter, which is a device that handles the rendering workload in a graphics subsystem. This interface is part of the DirectX Graphics Infrastructure (DXGI) library, which is a collection of APIs used to manage graphics and multimedia resources in Windows-based systems.

3. Create a command queue. For more details check [Microsoft Doc: Design Philosophy of Command Queues and Command Lists](https://learn.microsoft.com/en-us/windows/win32/direct3d12/design-philosophy-of-command-queues-and-command-lists)
> To execute work on the GPU, an app must explicitly submit a command list to a command queue associated with the Direct3D device. A direct command list can be submitted for execution multiple times, but the app is responsible for ensuring that the direct command list has finished executing on the GPU before submitting it again. 

4. Create swap chain
5. Create descriptor heaps. For details Check [Microsoft Doc: Creating Descriptor Heaps](https://learn.microsoft.com/en-us/windows/win32/direct3d12/creating-descriptor-heaps)
6. Create frame resources
7. Create root signatures
8. Create a pipeline state object
9. Create a command list
10. Create Vertex buffer and move to resources
    * For moving data to the vertex buffer, please check [Microsoft Doc: ID3D12Resource::Map method](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12resource-map)
12. Create sync object and wait until assets been uploaded to the GPU
13. Wait for the command list to execute; we are reusing the same command list in our main loop but for now, we just want to wait for setup to complete before continuing.

#### OnUpdate
Nothing ...

#### OnRender
1. Reset command allocators
2. Reset command lists
3. Set command list states
5. Record commands to command lists
6. Close command lists
7. Execute command lists
8. Present the frame
9. Move to the next frame


## MISC
### Constant Buffer
Declare an constant buffer `SceneConstantBuffer m_constantBufferData;` struct for shader usage. Also, we need a `ComPtr<ID3D12Resource> m_constantBuffer;` to hold the constant buffer in GPU. To use it inside the pipeline, we need a `ComPtr<ID3D12DescriptorHeap> m_cbvHeap;` to store the resource. For CPU side, `UINT8* m_pCbvDataBegin;` is the raw pointer mapping to the GPU resource `m_constantBuffer`. For moving resouces from CPU to GPU, check the [Microsoft Doc: Uploading different types of resources](https://learn.microsoft.com/en-us/windows/win32/direct3d12/uploading-resources)
```cpp
struct SceneConstantBuffer
    {
        XMFLOAT4 offset;
        float padding[60]; // Padding so the constant buffer is 256-byte aligned.
    };
    static_assert((sizeof(SceneConstantBuffer) % 256) == 0, "Constant Buffer size must be 256-byte aligned");

// Pipeline objects.
ComPtr<ID3D12DescriptorHeap> m_cbvHeap;

// App resources.
ComPtr<ID3D12Resource> m_constantBuffer;
SceneConstantBuffer m_constantBufferData;
UINT8* m_pCbvDataBegin;
```
We can create it during `OnInit`. Declare the memory section in `RootSignature`
```cpp
// Create a root signature consisting of a descriptor table with a single CBV.
    {
        D3D12_FEATURE_DATA_ROOT_SIGNATURE featureData = {};

        // This is the highest version the sample supports. If CheckFeatureSupport succeeds, the HighestVersion returned will not be greater than this.
        featureData.HighestVersion = D3D_ROOT_SIGNATURE_VERSION_1_1;

        if (FAILED(m_device->CheckFeatureSupport(D3D12_FEATURE_ROOT_SIGNATURE, &featureData, sizeof(featureData))))
        {
            featureData.HighestVersion = D3D_ROOT_SIGNATURE_VERSION_1_0;
        }

        CD3DX12_DESCRIPTOR_RANGE1 ranges[1];
        CD3DX12_ROOT_PARAMETER1 rootParameters[1];

        ranges[0].Init(D3D12_DESCRIPTOR_RANGE_TYPE_CBV, 1, 0, 0, D3D12_DESCRIPTOR_RANGE_FLAG_DATA_STATIC);
        rootParameters[0].InitAsDescriptorTable(1, &ranges[0], D3D12_SHADER_VISIBILITY_VERTEX);

        // Allow input layout and deny uneccessary access to certain pipeline stages.
        D3D12_ROOT_SIGNATURE_FLAGS rootSignatureFlags =
            D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT |
            D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS |
            D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS |
            D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS |
            D3D12_ROOT_SIGNATURE_FLAG_DENY_PIXEL_SHADER_ROOT_ACCESS;

        CD3DX12_VERSIONED_ROOT_SIGNATURE_DESC rootSignatureDesc;
        rootSignatureDesc.Init_1_1(_countof(rootParameters), rootParameters, 0, nullptr, rootSignatureFlags);

        ComPtr<ID3DBlob> signature;
        ComPtr<ID3DBlob> error;
        ThrowIfFailed(D3DX12SerializeVersionedRootSignature(&rootSignatureDesc, featureData.HighestVersion, &signature, &error));
        ThrowIfFailed(m_device->CreateRootSignature(0, signature->GetBufferPointer(), signature->GetBufferSize(), IID_PPV_ARGS(&m_rootSignature)));
    }
```
Create the constant buffer data and move to the GPU
```cpp
// Create the constant buffer.
    {
        const UINT constantBufferSize = sizeof(SceneConstantBuffer);    // CB size is required to be 256-byte aligned.

        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(constantBufferSize),
            D3D12_RESOURCE_STATE_GENERIC_READ,
            nullptr,
            IID_PPV_ARGS(&m_constantBuffer)));

        // Describe and create a constant buffer view.
        D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc = {};
        cbvDesc.BufferLocation = m_constantBuffer->GetGPUVirtualAddress();
        cbvDesc.SizeInBytes = constantBufferSize;
        m_device->CreateConstantBufferView(&cbvDesc, m_cbvHeap->GetCPUDescriptorHandleForHeapStart());

        // Map and initialize the constant buffer. We don't unmap this until the
        // app closes. Keeping things mapped for the lifetime of the resource is okay.
        CD3DX12_RANGE readRange(0, 0);        // We do not intend to read from this resource on the CPU.
        ThrowIfFailed(m_constantBuffer->Map(0, &readRange, reinterpret_cast<void**>(&m_pCbvDataBegin)));
        memcpy(m_pCbvDataBegin, &m_constantBufferData, sizeof(m_constantBufferData));
    }
```
Do the update in `OnUpdate`. The update is applied to `m_constantBufferData` and then copied back to `m_pCbvDataBegin`. Since the `m_pCbvDataBegin` is mapped to `m_constantBuffer`. The change will be applied in the next frame.
```cpp
// Update frame-based values.
void D3D12HelloConstBuffers::OnUpdate()
{
    const float translationSpeed = 0.005f;
    const float offsetBounds = 1.25f;

    m_constantBufferData.offset.x += translationSpeed;
    if (m_constantBufferData.offset.x > offsetBounds)
    {
        m_constantBufferData.offset.x = -offsetBounds;
    }
    memcpy(m_pCbvDataBegin, &m_constantBufferData, sizeof(m_constantBufferData));
}
```

### Texture
We need a `ComPtr<ID3D12Resource> m_texture;` to hold the texture in the GPU. To use it inside the pipeline, we need a `ComPtr<ID3D12DescriptorHeap> m_srvHeap;` to store the resource.
```cpp
// Pipeline objects
ComPtr<ID3D12DescriptorHeap> m_srvHeap;

// App resources.
ComPtr<ID3D12Resource> m_texture;
```

To create the texture, we need to know how to describe the texture object(like most of the object in Direct3D 12). After that, genetate the cpu side data, and move it to GPU resources object. Finally, create a SRV to store the GPU resources.
```cpp
// Note: ComPtr's are CPU objects but this resource needs to stay in scope until
// the command list that references it has finished executing on the GPU.
// We will flush the GPU at the end of this method to ensure the resource is not
// prematurely destroyed.
    ComPtr<ID3D12Resource> textureUploadHeap;

// Create the texture.
    {
        // Describe and create a Texture2D.
        D3D12_RESOURCE_DESC textureDesc = {};
        textureDesc.MipLevels = 1;
        textureDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
        textureDesc.Width = TextureWidth;
        textureDesc.Height = TextureHeight;
        textureDesc.Flags = D3D12_RESOURCE_FLAG_NONE;
        textureDesc.DepthOrArraySize = 1;
        textureDesc.SampleDesc.Count = 1;
        textureDesc.SampleDesc.Quality = 0;
        textureDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;

        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
            D3D12_HEAP_FLAG_NONE,
            &textureDesc,
            D3D12_RESOURCE_STATE_COPY_DEST,
            nullptr,
            IID_PPV_ARGS(&m_texture)));

        const UINT64 uploadBufferSize = GetRequiredIntermediateSize(m_texture.Get(), 0, 1);

        // Create the GPU upload buffer.
        ThrowIfFailed(m_device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(uploadBufferSize),
            D3D12_RESOURCE_STATE_GENERIC_READ,
            nullptr,
            IID_PPV_ARGS(&textureUploadHeap)));

        // Copy data to the intermediate upload heap and then schedule a copy 
        // from the upload heap to the Texture2D.
        std::vector<UINT8> texture = GenerateTextureData();

        D3D12_SUBRESOURCE_DATA textureData = {};
        textureData.pData = &texture[0];
        textureData.RowPitch = TextureWidth * TexturePixelSize;
        textureData.SlicePitch = textureData.RowPitch * TextureHeight;

        UpdateSubresources(m_commandList.Get(), m_texture.Get(), textureUploadHeap.Get(), 0, 0, 1, &textureData);
        m_commandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(m_texture.Get(), D3D12_RESOURCE_STATE_COPY_DEST, D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE));

        // Describe and create a SRV for the texture.
        D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
        srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
        srvDesc.Format = textureDesc.Format;
        srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
        srvDesc.Texture2D.MipLevels = 1;
        m_device->CreateShaderResourceView(m_texture.Get(), &srvDesc, m_srvHeap->GetCPUDescriptorHandleForHeapStart());
    }
```
We need to applied the corresponding change when creating `RootSignature`. You could notice that the `D2D12_DESCRIPTOR_RANGE_TYPE` is different from the above constant buffer example. In constant buffer, it is `D3D12_DESCRIPTOR_RANGE_TYPE_CBV` and here we have `D3D12_DESCRIPTOR_RANGE_TYPE_SRV`, as it is different kind of resources. Also, we need to declare the sampler to sample the value in the texture. 

```cpp
// Create the root signature.
    {
        D3D12_FEATURE_DATA_ROOT_SIGNATURE featureData = {};

        // This is the highest version the sample supports. If CheckFeatureSupport succeeds, the HighestVersion returned will not be greater than this.
        featureData.HighestVersion = D3D_ROOT_SIGNATURE_VERSION_1_1;

        if (FAILED(m_device->CheckFeatureSupport(D3D12_FEATURE_ROOT_SIGNATURE, &featureData, sizeof(featureData))))
        {
            featureData.HighestVersion = D3D_ROOT_SIGNATURE_VERSION_1_0;
        }

        CD3DX12_DESCRIPTOR_RANGE1 ranges[1];
        ranges[0].Init(D3D12_DESCRIPTOR_RANGE_TYPE_SRV, 1, 0, 0, D3D12_DESCRIPTOR_RANGE_FLAG_DATA_STATIC);

        CD3DX12_ROOT_PARAMETER1 rootParameters[1];
        rootParameters[0].InitAsDescriptorTable(1, &ranges[0], D3D12_SHADER_VISIBILITY_PIXEL);

        D3D12_STATIC_SAMPLER_DESC sampler = {};
        sampler.Filter = D3D12_FILTER_MIN_MAG_MIP_POINT;
        sampler.AddressU = D3D12_TEXTURE_ADDRESS_MODE_BORDER;
        sampler.AddressV = D3D12_TEXTURE_ADDRESS_MODE_BORDER;
        sampler.AddressW = D3D12_TEXTURE_ADDRESS_MODE_BORDER;
        sampler.MipLODBias = 0;
        sampler.MaxAnisotropy = 0;
        sampler.ComparisonFunc = D3D12_COMPARISON_FUNC_NEVER;
        sampler.BorderColor = D3D12_STATIC_BORDER_COLOR_TRANSPARENT_BLACK;
        sampler.MinLOD = 0.0f;
        sampler.MaxLOD = D3D12_FLOAT32_MAX;
        sampler.ShaderRegister = 0;
        sampler.RegisterSpace = 0;
        sampler.ShaderVisibility = D3D12_SHADER_VISIBILITY_PIXEL;

        CD3DX12_VERSIONED_ROOT_SIGNATURE_DESC rootSignatureDesc;
        rootSignatureDesc.Init_1_1(_countof(rootParameters), rootParameters, 1, &sampler, D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);

        ComPtr<ID3DBlob> signature;
        ComPtr<ID3DBlob> error;
        ThrowIfFailed(D3DX12SerializeVersionedRootSignature(&rootSignatureDesc, featureData.HighestVersion, &signature, &error));
        ThrowIfFailed(m_device->CreateRootSignature(0, signature->GetBufferPointer(), signature->GetBufferSize(), IID_PPV_ARGS(&m_rootSignature)));
    }
```



### FrameBuffering
In the primitive example, we have a helper function `WaitForPreviousFrame()`
```cpp
void D3D12HelloConstBuffers::WaitForPreviousFrame()
{
    // WAITING FOR THE FRAME TO COMPLETE BEFORE CONTINUING IS NOT BEST PRACTICE.
    // This is code implemented as such for simplicity. The D3D12HelloFrameBuffering
    // sample illustrates how to use fences for efficient resource usage and to
    // maximize GPU utilization.

    // Signal and increment the fence value.
    const UINT64 fence = m_fenceValue;
    ThrowIfFailed(m_commandQueue->Signal(m_fence.Get(), fence));
    m_fenceValue++;

    // Wait until the previous frame is finished.
    if (m_fence->GetCompletedValue() < fence)
    {
        ThrowIfFailed(m_fence->SetEventOnCompletion(fence, m_fenceEvent));
        WaitForSingleObject(m_fenceEvent, INFINITE);
    }

    m_frameIndex = m_swapChain->GetCurrentBackBufferIndex();
}
```
Instead of waiting for the previous frame finished, we could have a function `MoveToNextFrame`, start processing the back buffer and swap the frame index to the back buffer to fully utilize the pipeline.
```cpp
// Prepare to render the next frame.
void D3D12HelloFrameBuffering::MoveToNextFrame()
{
    // Schedule a Signal command in the queue.
    const UINT64 currentFenceValue = m_fenceValues[m_frameIndex];
    ThrowIfFailed(m_commandQueue->Signal(m_fence.Get(), currentFenceValue));

    // Update the frame index.
    m_frameIndex = m_swapChain->GetCurrentBackBufferIndex();

    // If the next frame is not ready to be rendered yet, wait until it is ready.
    if (m_fence->GetCompletedValue() < m_fenceValues[m_frameIndex])
    {
        ThrowIfFailed(m_fence->SetEventOnCompletion(m_fenceValues[m_frameIndex], m_fenceEvent));
        WaitForSingleObjectEx(m_fenceEvent, INFINITE, FALSE);
    }

    // Set the fence value for the next frame.
    m_fenceValues[m_frameIndex] = currentFenceValue + 1;
}
```


