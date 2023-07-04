---
title: "3D Software Rendering"
date: 2022-11-04T18:15:02+01:00
draft: false
description: "A 3D CPU based software renderer written completely from scratch in C and capable of performing all the major graphics pipeline technique."
cover: "/pictures/avigl.png"
featured: true
tags: ["C", "SDL", "Rasterization"]
weight: 1
---

{{< rawhtml >}}

<details>
<summary>Table of Contents</summary>
{{</ rawhtml >}}

- [About the renderer](#about-the-renderer)
- [Shading](#about-the-renderer)
- [Texture Mapping](#about-the-renderer)
- [Camera](#about-the-renderer)
- [Culling](#about-the-renderer)
- [Demo](#demo)

{{< rawhtml >}}

</details>
{{</ rawhtml >}}

# About the renderer

The rederer performs rasterization of 3D models. It uses the SDL library for creating window and polling user input. The entirety of the renderer is written from scratch in C.

It implements fundamental graphics programming techniques like loading meshes, backface fulling, frustum clipping, texture mapping, shading etc.

## Texture Mapping

{{< rawhtml >}}
<div style="text-align: center; width: 100%">
        <img src="../images/texture_mapping.gif">
</div>
{{< /rawhtml >}}

Perpective correct texture mapping loaded from png files.

## Gouraud Shading

{{< rawhtml >}}
<div style="text-align: center; width: 100%">
        <img src="../images/gouraud_shading.gif">
</div>
{{< /rawhtml >}}
![](C:\Hugo\aviral-hugo\content\pictures\gouraud_shading.gif)

Gouraud Shading by calculating per vertex normal and interpolating the color for each fragment of the model. Based on Directional lighting model.

## View Frustum Clipping & Culling

{{< rawhtml >}}
<div style="text-align: center; width: 100%">
        <img src="../images/frustum_clipping.gif">
</div>
{{< /rawhtml >}}
![](C:\Hugo\aviral-hugo\content\pictures\frustum_clipping.gif)

View Frsuting clipping and culling based on the view frustum planes. Model triangles are clipped, and new triangles are formed if needed.

## Demo

Project repository: [GitHub - 3d renderer in C](https://github.com/aviralgoel/AviGL)

{{< rawhtml >}}
<div style="text-align: center; width: 100%">
        <img src="../images/final_demo.gif">
</div>
{{< /rawhtml >}}

![](C:\Hugo\aviral-hugo\content\pictures\final_demo.gif)
