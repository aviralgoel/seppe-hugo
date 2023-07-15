---
title: "Vulkan Renderer"
date: 2022-04-30T21:42:46+02:00
draft: false
description: "A Vulkan renderer I worked on to improve my knowledge of the Vulkan API."
cover: "https://cdn.seppedekeyser.be/img/proj_VulkanRenderer/Pelican_Banner.png"
featured: false
tags: ["C++", "Vulkan"]
weight: 3
---

After finishing the [Vulkan Tutorial](https://vulkan-tutorial.com), I understood how the main concepts of the API work, but couldn't quite wrap my head around how they would work together in a more complex project.

In the end, I ended up going on a long and deep dive into more of the details of the Vulkan API and how to write a game engine of sorts.
I absolutely loved working on this project, and loved learning a lot from all the little challenges that stood in my way.

Below the overview video I go into more details on some of the technical aspects of the project.

{{< youtube zj0sBo4uxG4 >}}

# Abstraction layer over Vulkan

Because Vulkan is a very verbose API, I knew that building an abstraction layer on top of Vulkan would be vital.

The main pain point I identified was setting up the pipelines, as they require a lot of boilerplate structs to be filled out.
With my PipelineBuilder abstraction, it was super easy to set up a new pipeline:

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

# Shader hot-reloading

Implementing shader hot-reloading was a fun little side-quest to figure out, because going into this I had no clue where to start.
In the end I realized that all I had to do was simply destroy the current render pass and graphics pipeline, reload the shader files and recreate the render pass and graphics pipeline.

Because of my small abstraction layer, this was fairly easy to implement, and the end code resulted in something like this:

```cpp
void VulkanRenderer::ReloadShaders()
{
    m_pDevice->WaitIdle();

    m_Pipeline.Cleanup(m_pDevice->GetDevice());
    m_pDevice->GetDevice().destroyRenderPass(m_RenderPass);

    CreateRenderPass();
    CreateGraphicsPipeline();
}
```

# ECS scene system

After hearing a lot of positive voices around using an ECS system, I decided to check out [EnTT](https://github.com/skypjack/entt) and implement it in my project.

Its implementation in the project ended up being very trivial, as it mostly just needs to call EnTT functions:

```cpp
class Entity
{
public:
    Entity() = delete;

    template<typename T, typename... Args>
    T& AddComponent(Args&&... args)
    {
        ASSERT_MSG(!HasComponent<T>(), "Entity already has component!");
        T& component = m_pScene->m_Registry.emplace<T>(m_Entity, std::forward<Args>(args)...);
        return component;
    }

    template<typename T>
    void RemoveComponent()
    {
        ASSERT_MSG(HasComponent<T>(), "Entity does not have component!");
        m_pScene->m_Registry.remove<T>(m_Entity);
    }

    template<typename T>
    bool HasComponent()
    {
        return m_pScene->m_Registry.all_of<T>(m_Entity);
    }

private:
    // Scene holds the entt::registry, from which entities are created.
    // Only the scene is allowed to create entities
    friend class Scene;
    Entity(entt::entity entity);

    entt::entity m_Entity{ entt::null };
    Scene* m_pScene;
};
```

EnTT makes it really easy to query entities which have certain components attached to them:

```cpp
void Scene::Draw()
{
    // Insert a debug marker for graphics debuggers
    vk::CommandBuffer cmd = VulkanRenderer::GetCurrentBuffer();
    VkDebugMarker::BeginRegion(cmd, "Scene Render");

    // Get all entities that have a transform and a model component,
    // and record their draw calls.
    auto view = m_Registry.view<TransformComponent, ModelComponent>();
    for (auto [entity, transformComp, modelComp] : view.each())
    {
        modelComp.pModel->Draw();
    }

    VkDebugMarker::EndRegion(cmd);
}
```

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
- [EnTT](https://github.com/skypjack/entt) (ECS system)

[Project Repository](https://github.com/SeppahBaws/PelicanEngine)
