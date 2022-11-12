---
title: "Vulkan Renderer"
date: 2022-04-30T21:42:46+02:00
draft: false
description: "A Vulkan renderer I worked on to improve my knowledge of the Vulkan API."
cover: "https://cdn.seppedekeyser.be/img/proj_VulkanRenderer/Pelican_Banner.png"
featured: true
tags: ["C++", "Vulkan"]
weight: 3
---

This is a 3D Vulkan renderer that I built, to improve my knowledge of the Vulkan API.

The base organization was heavily inspired by [TheCherno's Hazel tutorial engine](https://github.com/TheCherno/Hazel), with some personal twists on top.

This project is available on GitHub: [https://github.com/SeppahBaws/PelicanEngine]().


Features include:
- Dynamic fly camera
- Simple shading using albedo and AO textures
- ImGui debug UI
- ECS system
- Scene file serialization and deserialization
- Shader hot-reloading
- Asset manager which controls the lifetime of assets
- Pipeline builder to aid in setting up different pipelines

Currently in-progress:
- PBR renderer

# Implementation

Of course the project itself is too big to talk about everything in here, so I selected some of the most interesting bits.

Here you can see how the objects in the scene get rendered, utilizing entt's easy view iterator.
I also insert a debug marker, so that I can easily spot this part in a graphics debugger.

```cpp
void Scene::Draw()
{
    // Retrieve the current command buffer
    vk::CommandBuffer cmd = VulkanRenderer::GetCurrentBuffer();

    // Insert a debug marker for graphics debuggers
    VkDebugMarker::BeginRegion(cmd, "Scene Render");

    // Record all objects draw calls to the command buffer
    auto view = m_Registry.view<TransformComponent, ModelComponent>();
    for (auto [entity, transformComp, modelComp] : view.each())
    {
        modelComp.pModel->Draw();
    }

    VkDebugMarker::EndRegion(cmd);
}
```

This is an example of how the PipelineBuilder can be used to easily create multiple pipelines:

```cpp
void VulkanRenderer::CreateGraphicsPipeline()
{
    // Load our shader
    VulkanShader* pShader = new VulkanShader();
    pShader->AddShader(ShaderType::Vertex, "res/shaders/vert.spv");
    pShader->AddShader(ShaderType::Fragment, "res/shaders/frag.spv");

    // Setup all our parameters in the pipeline
    PipelineBuilder builder{ m_pDevice->GetDevice() };
    builder.SetShader(pShader);
    builder.SetInputAssembly(vk::PrimitiveTopology::eTriangleList, false);
    builder.SetViewport(0.0f, 0.0f, swapChainWidth, swapChainHeight, 0.0f, 1.0f);
    builder.SetScissor(vk::Offset2D(0, 0), m_pSwapChain->GetExtent());
    builder.SetRasterizer(vk::PolygonMode::eFill, vk::CullModeFlagBits::eBack);
    builder.SetMultisampling();
    builder.SetDepthStencil(true, true, vk::CompareOp::eLess);
    builder.SetColorBlend(true, vk::BlendOp::eAdd, vk::BlendOp::eAdd, false, vk::LogicOp::eCopy);
    builder.SetDescriptorSetLayout(1, &m_DescriptorSetLayout);

    // Build the filled rendering pipeline
    m_Pipelines[static_cast<int>(RenderMode::Filled)] = builder.BuildGraphics(m_RenderPass);

    // Build the line rendering pipeline
    builder.SetRasterizer(vk::PolygonMode::eLine, vk::CullModeFlagBits::eBack);
    m_Pipelines[static_cast<int>(RenderMode::Lines)] = builder.BuildGraphics(m_RenderPass);

    // Build the point rendering pipeline
    builder.SetRasterizer(vk::PolygonMode::ePoint, vk::CullModeFlagBits::eBack);
    m_Pipelines[static_cast<int>(RenderMode::Points)] = builder.BuildGraphics(m_RenderPass);

    // Clean up shader, we don't need it anymore
    delete pShader;
    pShader = nullptr;
}
```

Of course there is so much more going on in this project, please visit the [GitHub repository](https://github.com/SeppahBaws/PelicanEngine) to get the full picture.

# Media
Here you can see the shader hot-reloading system in action:

{{< rawhtml >}}
<video width="100%" controls muted src="https://cdn.seppedekeyser.be/img/proj_VulkanRenderer/vid/Pelican_ShaderReload.mp4">
</video>
{{< /rawhtml >}}

Here is the Dear ImGui debug UI doing its job:

{{< rawhtml >}}
<video width="100%" controls muted src="https://cdn.seppedekeyser.be/img/proj_VulkanRenderer/vid/Pelican_ImGui.mp4">
</video>
{{< /rawhtml >}}

And finally, a sneak preview of the PBR renderer (currently, only simple lighting is implemented)

{{< rawhtml >}}
<video width="100%" controls muted src="https://cdn.seppedekeyser.be/img/proj_VulkanRenderer/vid/Pelican_PBR.mp4">
</video>
{{< /rawhtml >}}

used libraries:
- [glfw](https://github.com/glfw/glfw) (windowing)
- [glm](https://github.com/g-truc/glm) (maths library)
- [Dear ImGui](https://github.com/ocornut/imgui) (debug immediate-mode UI)
- [logtools](https://github.com/SeppahBaws/logtools) (small logging library)
- [tinygltf](https://github.com/syoyo/tinygltf) (loading gltf model files)
- [stb_image](https://github.com/nothings/stb) (reading/writing image files)
- [json](https://github.com/nlohmann/json) (reading/writing json files)
- [entt](https://github.com/skypjack/entt) (ECS system)

[Project Repository](https://github.com/SeppahBaws/PelicanEngine)
