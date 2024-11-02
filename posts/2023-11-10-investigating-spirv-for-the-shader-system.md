---
title: "Investigating SPIR-V for the shader system - Part 1"
author: youness
popular: false
image: /images/Spir.png
tags: ['.NET', 'Shaders']
---

In this first part of a new series of blog posts, we will learn more about Stride's shader system, its limitations and how to make it better thanks to a very useful shader language called SPIR-V. This will be the first step in implementing a new and better shader system.

---

Table of Contents:

[[TOC]]

The Stride engine has a powerful shader system, helping users build very complex shaders thanks to its language features like composition and inheritance, and its C# integration through the very powerful material system.

## What are shaders?

Simply put, they are typically snippets of code that are run on the GPU, mostly written in C-like languages like GLSL and HLSL.

When a game has to render on the screen, it starts by preparing the data on the GPU and then sends shaders with commands. The GPU then executes those commands and if everything goes well, the game ends with a lovely picture drawn on the screen. This single call of a GPU is called a `draw call`.

{% img-click 'Shader Overview' '/images/blog/2023-11/shaders-explanation.png' %}

It can be simplified to this view. There is a first stage called `Vertex stage` which is meant to compute vertices and make sure they are transformed into data that can be rasterized by your GPU, and a final stage called `Pixel stage` or `Fragment stage` that takes the rasterized output of the rasterizer and computes color for each pixel on the screen based on which triangles they belong to.

In both stages, you can provide textures/images and data through buffers, this can help give more details to your rendering, or help you compute more complicated things.

## Rendering in game engines

Drawing a single mesh is simple, one buffer of vertices, maybe some textures, do a draw call and ✨ marvel at the screen ✨. Repeat it many times and you get a video, very useful to make video games 👾.

In game engines, objects being drawn the same ways are grouped into `materials`. A material can be seen as a recipe for drawing a mesh with ingredients like shader code, textures, vertices and other informations.

Having many different materials implies creating many shader source code, and depending if you use [forward or deferred shading](https://learnopengl.com/Advanced-Lighting/Deferred-Shading), you might need to reuse code from one file to another leading to lots of duplicate code 👩‍💻👨‍💻.

To solve this problem, all game engines have developed shader systems that help create permutations of shader code depending on the material you're using. It helps reuse code you've already written, which speeds up development and gives you the power to organize your code in more manageable ways.

A good example of that are [shader graphs from Unity](https://unity.com/features/shader-graph) or [the material editor in Unreal](https://docs.unrealengine.com/5.0/en-US/unreal-engine-material-editor-ui/), whenever you add a block/function to the graph, it reuses shader code already written, to create a new permutation.

## How does it work in Stride?

Stride has its own shader language called SDSL which stands for **S**tri**D**e **S**hader **L**anguage.

SDSL uses a very smart system of composition and inheritance, similar to object oriented programming. You can learn more about it [through the manual](https://doc.stride3d.net/latest/en/manual/graphics/effects-and-shaders/index.html).

## The many problems shader systems are facing

To create permutations you need to be aware of the many limitations you will face while writing code for GPU.

### GPUs are all different

As obvious as it sounds, it needs to be reminded. Whenever you write code for your GPU you have to be aware that it will not work for all other GPUs.

10 years ago, two languages were the most used for writing shader code: HLSL and GLSL. In the case of OpenGL, GPU vendors would provide you with their own GLSL compilers to work for their GPUs through different drivers. A valid code for one compiler would be a syntax error for another. It was a big mess 💩, but a manageable one!

Stride was created during this period and it was designed to run on a wide variety of devices 🎮 🕹️ 💻 🖥️.

The solution chosen for creating code permutations that could work on many different machine was transpilation. Text would be parsed in abstract syntax trees and the text would be translated from one language to another by manipulating those trees. [To have a better idea of how abstract syntax tree work, this playlist of videos will definitely help](https://www.youtube.com/watch?v=cxNlb2GTKIc&list=PLTd6ceoshpreZuklA7RBMubSmhE0OHWh_&pp=iAQB).

### Drivers are not all the same

Drivers can be seen as a front-end API to use the GPU. They have different paradigm and logic which may result in your shader code needing to be different across different drivers.

Here's a table of which language is supported by which machine depending on the driver used. I chose the most used machines for gaming.

Note that some APIs are tied to the machine you're developing for, adding more complexity to the shader system, needing to handle GPUs that have very specific features. Vulkan and OpenGL can be used on Mac and iOS but OpenGL is deprecated and Vulkan run on MoltenVK, a Vulkan implementation on top of Metal.

<div class="table-responsive">
<table class="table table-striped table-sm">
  <thead>
  <tr>
    <th></th>
    <th>Direct3D</th>
    <th>Vulkan</th>
    <th>OpenGL</th>
    <th>Metal</th>
    <th>PSGL</th>
    <th>NVN</th>
  </tr></thead>
  <tr>
    <td>Windows/XBox</td>
    <td>HLSL</td>
    <td>HLSL/GLSL</td>
    <td>GLSL</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>Linux</td>
    <td></td>
    <td>HLSL/GLSL</td>
    <td>GLSL</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>Mac/iOS</td>
    <td></td>
    <td>HLSL/GLSL</td>
    <td>GLSL</td>
    <td>MSL</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>Android</td>
    <td></td>
    <td>HLSL/GLSL</td>
    <td>GLSL</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>Playstation 4/5</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>PSSL</td>
    <td></td>
  </tr>
  <tr>
    <td>Nintendo Switch</td>
    <td></td>
    <td>HLSL/GLSL</td>
    <td>GLSL</td>
    <td></td>
    <td></td>
    <td>?</td>
  </tr>
</table>
</div>

### Performance Cost

Managed languages like C# are well known to be slower with string objects since they treat them as immutable. On top of that, tree structures, especially the one used for shader compilation in Stride, tend to perform very slowly and are prone to be slow and work against the GC.

## Why investigating SPIR-V?

Three things happened in the past 10 years:

* Vulkan was released with a new kind of bytecode shader language called SPIR-V
* .NET Core introduced a new high-performance feature for operating on slices of array (something that C# is way better at than processing string objects), namely `Span<T>`
* Most importantly, tools like [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) and [Naga](https://github.com/gfx-rs/wgpu/tree/trunk/naga) were created as a means to translate shaders through SPIR-V

It was clear for everyone that we had to investigate how we could compile SDSL to SPIR-V and use all those very performant tools to transpile SDSL to other languages.

SDSL is a very peculiar language and shifting from processing tree structures to processing segments of byte code is a great endeavor, especially for an engine.

The following parts of this series of blog posts will focus on how such a system can be created, thanks to the many new C# and .NET features that came out since the release of .NET Core.

## Summary

Embracing SPIR-V for Stride's shader system will hopefully empower us with a better control over the shader language but also improve performance for shader compilation 💪💪.

Thank you for reading!
