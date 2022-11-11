---
title: "Geometry Wars"
date: 2022-04-30T21:42:57+02:00
draft: false
featured: false
tags: ["C++", "DirectX 11"]
description: "A small replica of Geometry Wars 3 in the Overlord game engine."
cover: "/img/proj_GW-Overlord/Screenshot_3.png"
weight: 4
---

Geometry Wars 3 remade within the Overlord Engine.



Playthrough video demonstrating the gameplay and highlighting all the features:

<iframe width="560" height="315" src="https://www.youtube.com/embed/szag3OTdOTk" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


## Custom HDR pipeline

The Overlord engine didn't have a HDR pipeline, so I edited the engine to make use of a HDR pipeline. Although it was my first time doing anything with HDR, it was still fairly easy to get started with once I had a basic understanding of how it works. [LearnOpenGL.com](https://learnopengl.com) proved to be a very useful resource to understand the "theory" behind it.
The main reason why I wanted to get HDR up and running was so that I could add bloom to the game. What would Geometry Wars be without any bloom, right?

> Quick note: even though I'm utilizing an HDR pipeline, the renderer is by no means a PBR one. The game only uses simple shapes and colors, so it wasn't even necessary at all.


### Tonemapping

To bring the rendered HDR result back to a 0-1 range, I utilized a simple exposure/gamma-based tonemapping pass, mainly to keep everything simple but also so that I could easily adjust the exposure at runtime, instead of having to re-launch the game every time I adjusted it.

You can see the pixel shader that does the tonemapping below:

```c
float4 PS(PS_INPUT input) : SV_Target
{
    const float gamma = 2.2f;
    float3 hdrColor = gTexture.Sample(samPoint, input.TexCoord).rgb;
    float3 bloomColor = gBloomTexture.Sample(samPoint, input.TexCoord).rgb;
    hdrColor += bloomColor;

    // exposure tone mapping
    float3 mapped = float3(1.0f, 1.0f, 1.0f) - exp(-hdrColor * gExposure);
    mapped = pow(mapped, float3(1.0f / gamma, 1.0f / gamma, 1.0f / gamma));

    return float4(mapped, 1.0f);
}
```


### Bloom

After the scene is done rendering, the post-process effects kick off. First of all, the bloom cutout pass is started, which outputs a full-screen texture that only hold the pixels that have a brightness of higher than 1.
This is then passed on to a multi-pass Gaussian blur to get a nice blur effect. At first I tried blurring with a simple single-pass blur, but the result didn't quite look that great so it quickly made me switch to the Gaussian blur. The performance impact was barely noticeable as well so it was a bit of a no-brainer.

After the blurring is done, the result gets sent to the final pass of the bloom post-process, which blends the blurred image with the tonemapped image.
