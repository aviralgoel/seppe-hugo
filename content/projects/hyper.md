---
title: "Hybrid Ray Tracer"
date: 2022-11-04T18:15:02+01:00
draft: true
description: "A hybrid ray tracer in Vulkan to check out ray tracing in Vulkan"
cover: "/img/proj_Hyper/BistroExterior_01.png"
featured: true
tags: ["C++", "Vulkan", "Ray Tracing"]
weight: 1
---



I initially started this project as a spiritual successor to my [Vulkan renderer](/projects/vulkan-renderer), but eventually decided to keep the scope limited so that I could check out how ray tracing in Vulkan works.
Starting the project from scratch also gave me the opportunity to check out the latest features added in Vulkan 1.3 (mainly dynamic rendering), and improve on some abstractions over the Vulkan API.


# About the renderer
The renderer is a hybrid ray tracer: it first renders the base geometry in a rasterized pass first, then ray traces the scene for shadows, and finally combines the two outputs together in a combine pass.
[![NsightFrameStructure.png](/img/proj_Hyper/FrameStructure/NsightFrameStructure.png)](/img/proj_Hyper/FrameStructure/NsightFrameStructure.png)

## Geometry pass
All the geometry pass does is render the scene albedo color to a `R8G8B8A8_UNORM` render target:
[![Geometry_output_color.png](/img/proj_Hyper/FrameStructure/Geometry_output_color.png)](/img/proj_Hyper/FrameStructure/Geometry_output_color.png)

## Ray tracing pass
The ray tracing pass traces two rays:

1. One from the camera position into the scene, to get the world-position of the corresponding pixel in the scene.
2. One from that world-position towards the direction of the sun, with a small offset to simulate soft shadows.
    1. If this ray hits anything, the pixel is shadowed and a value of `0` is written to the output buffer.
    2. If the ray doesn't hit anything, the pixel is not shadowed and a value of `1` is written to the output buffer.

Currently this is simply done at 1 ray per pixel, without any denoising. I have been looking into some denoising techniques, but haven't gotten around to implementing one yet.

The result is written to a `R8_UNORM` render target:
[![Raytracing_output_color.png](/img/proj_Hyper/FrameStructure/Raytracing_output_color.png)](/img/proj_Hyper/FrameStructure/Raytracing_output_color.png)


Payload passed with the ray:
```c
struct HitInfo
{
    float rayT;
    bool isSecondaryRay;
};

struct Payload
{
    [[vk::location(0)]] HitInfo hitInfo;
};
```


Ray generation shader:
```c
[shader("raygeneration")]
void main()
{
    const float FLT_MAX = float(0x7F7FFFFF);
    
    uint3 launchId = DispatchRaysIndex();
    uint3 launchSize = DispatchRaysDimensions();
    
    const float2 pixelCenter = DispatchRaysIndex().xy + float2(0.5, 0.5);
    const float2 inUV = pixelCenter / DispatchRaysDimensions().xy;
    const float2 d = inUV * 2.0 - 1.0;
    
    float4 origin = mul(camera.viewInverse, float4(0, 0, 0, 1));
    float4 target = mul(camera.projInverse, float4(d.x, d.y, 1, 1));
    float4 direction = mul(camera.viewInverse, float4(normalize(target.xyz), 0));
    
    Payload payload = (Payload)0;
    payload.hitInfo.isSecondaryRay = false;
    
    RayDesc rayDesc;
    rayDesc.Origin = origin.xyz;
    rayDesc.Direction = direction.xyz;
    rayDesc.TMin = 0.001;
    rayDesc.TMax = 10000.0;
    TraceRay(accel, RAY_FLAG_FORCE_OPAQUE, 0xFF, 0, 0, 0, rayDesc, payload);
    
    image[int2(launchId.xy)] = float(payload.hitInfo.rayT == FLT_MAX) * 1.0;
}
```


Miss shader:
```c
[shader("miss")]
void main(inout Payload payload)
{
    const float FLT_MAX = float(0x7F7FFFFF);
    
    payload.hitInfo.rayT = FLT_MAX;
}
```


Closest Hit shader:
```c
[shader("closesthit")]
void main(inout Payload payload)
{
    if (payload.hitInfo.isSecondaryRay)
    {
        payload.hitInfo.rayT = RayTCurrent();
        return;
    }
    
    uint randSeed = InitRand(DispatchRaysIndex().x + DispatchRaysIndex().y * DispatchRaysDimensions().x, pushConstants.frameNr);
    
    float3 origin = WorldRayOrigin() + WorldRayDirection() * RayTCurrent();
    float3 randomOffset = GetRandomOnUnitSphere(randSeed);
    float3 direction = normalize(normalize(pushConstants.sunDir) * 10.0 + randomOffset * 0.1);
    
    RayDesc rayDesc;
    rayDesc.Origin = origin;
    rayDesc.Direction = direction;
    rayDesc.TMin = 0.001;
    rayDesc.TMax = 10000.0;
    payload.hitInfo.isSecondaryRay = true;
    TraceRay(accel, RAY_FLAG_FORCE_OPAQUE, 0xFF, 0, 0, 0, rayDesc, payload);
}
```



## Combine pass
The combine pass takes the output of the two previous passes and combines them into one output buffer. It's a very simple pass which:
- darkens the areas which are in shadows
- applies gamma correction

The result of the combine pass looks like this:
[![Combine_output_color.png](/img/proj_Hyper/FrameStructure/Combine_output_color.png)](/img/proj_Hyper/FrameStructure/Combine_output_color.png)


# Shaders
The previous project already had a shader hot-reloading system in place, but it required you to compile the shaders before-hand, and then hot-reload the compiled shader binary code. I wanted to change this so that I could just press one button and have everything just work.
I also wanted to play around with shader reflection, to automatically generate descriptor sets.

## Shader compilation
I started with GLSL shaders which were compiled through [shaderc](https://github.com/google/shaderc), but eventually decided to switch over to HLSL shaders with the [DirectX Shader Compiler](https://github.com/microsoft/DirectXShaderCompiler), as I am going to need it in the future if I want to add some library which only comes in HLSL form.
After shaders are compiled they get reflected with [spirv_cross](https://github.com/KhronosGroup/SPIRV-Cross). From this reflection data I generate descriptor sets automatically, because setting up descriptor sets automatically can become a lot of effort.

Currently the reflection data isn't shared with all the parts of the renderer yet, as I have yet to figure out a clean way to do this. The Vulkan pipelines are created with manually defining the descriptor sets, but luckily the validation layers do warn you when these descriptor sets of the pipeline and shaders don't match up, so it's not too big of a problem right now.

## Shader hot-reloading
If you want to reload a shader, you also have to re-create the pipeline it belongs to. To handle this, I created the `ShaderLibrary` class, which keeps track of all the shaders and their dependencies.
```cpp
struct ShaderDependencies
{
    std::vector<VulkanGraphicsPipeline*> graphicsPipelines;
    std::vector<VulkanRayTracingPipeline*> rayTracingPipelines;
};

class ShaderLibrary
{
public:
    // Functions for registring shaders and their dependencies...
private:
    std::unordered_map<UUID, ShaderDependencies> m_ShaderDependencies;
};
```

With this system in place, it's rather easy to reload a shader and re-create their respective dependencies:
```cpp
// Reload the shader from the latest version on disk.
pShader->Reload();

// Recreate all dependent graphics pipelines
for (VulkanGraphicsPipeline* graphicsPipeline : m_ShaderDependencies[pShader->GetId()].graphicsPipelines)
{
    graphicsPipeline->Recreate();
}

// Recreate all dependent ray tracing pipelines
for (VulkanRayTracingPipeline* rtPipeline : m_ShaderDependencies[pShader->GetId()].rayTracingPipelines)
{
    rtPipeline->Recreate();
}
```

## Shader cache
Because recompiling all shaders on each startup was starting to take some time, I decided to implement a system to cache the compiled SPIR-V shaders. How this works is quite simple:
- First, I check when the shader file was last edited, and see if that shader has been cached with that timestamp.
- If it matches, I can simply load the contents of that binary shader file into memory and create a shader module with it.
- If it doesn't match, or that shader has never been cached before, load the shader source code into a string, pass it to the HLSL or GLSL compiler, write the binary code to the cached file on disk and create a shader module with the shader code.


# Improvements for the future
Of course nothing is ever finished, and I have a couple of things I want to improve on in the future:
- Share the shader reflection data with all parts that might need it, to avoid having to set them up manually and making sure they match everywhere.
- Implement PBR shading. For this I will likely have to move the ray tracing pass before the geometry pass so that I have the shadow information in time for the PBR shading. Alternatively I could work with a G-buffer that holds the albedo, normal and depth information, which I could then use to calculate the world-position in the ray tracing pass, to avoid tracing that first ray. However I will have to measure the performance and memory tradeoffs between the two ways to see which one is better.
- Add a denoiser for the ray traced shadows. Currently I'm thinking of implementing the [FidelityFX Shadows Denoiser](https://gpuopen.com/fidelityfx-denoiser/#shadow), as it seems to be doing everything I need.
- Implement more ray traced techniques, such as:
	- Ambient Occlusion
	- Reflections
	- Global Illumination

# Media
Enjoy some screenshots from the renderer, in a variety of different scenes:

(click on an image to view it in full-size)

| | |
|--|--|
| [![BistroExterior_01.png](/img/proj_Hyper/BistroExterior_01.png)](/img/proj_Hyper/BistroExterior_01.png) | [![BistroExterior_02.png](/img/proj_Hyper/BistroExterior_02.png)](/img/proj_Hyper/BistroExterior_02.png) |
| [![BistroInterior_01.png](/img/proj_Hyper/BistroInterior_01.png)](/img/proj_Hyper/BistroInterior_01.png) | [![NewSponza_01.png](/img/proj_Hyper/NewSponza_01.png)](/img/proj_Hyper/NewSponza_01.png) |
| [![SanMiguel_01.png](/img/proj_Hyper/SanMiguel_01.png)](/img/proj_Hyper/SanMiguel_01.png) | [![SanMiguel_02.png](/img/proj_Hyper/SanMiguel_02.png)](/img/proj_Hyper/SanMiguel_02.png) |
| [![TrainStation_01.png](/img/proj_Hyper/TrainStation_01.png)](/img/proj_Hyper/TrainStation_01.png) | [![TrainStation_02.png](/img/proj_Hyper/TrainStation_02.png)](/img/proj_Hyper/TrainStation_02.png) |

---
Project repository: https://github.com/SeppahBaws/Hyper
