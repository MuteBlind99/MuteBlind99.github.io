---
layout: post
title:  "Building a Modern 3D Rendering Engine with OpenGL (C++)"
date:   2026-02-04 14:12:26 +0100
categories: Blog Post Graphic Engine
---
## 1. Introduction & Motivation

This project was developed as part of my **Computer Graphics curriculum at SAE Institute Geneva** (2025–2026).
The main goal was to dive deep into the **modern real-time rendering pipeline**, using **OpenGL** as a learning tool to fully understand how contemporary 3D engines work under the hood.

### Why OpenGL?

OpenGL remains one of the best APIs to truly learn graphics programming:
- Cross-platform
- Explicit GPU control
- Minimal abstraction compared to Vulkan or DirectX 12

Rather than stopping at a simple *“Hello Triangle”*, I wanted to build a **complete rendering engine** featuring:
- Deferred rendering
- Dynamic lighting and shadows
- Massive instancing
- Screen-space effects (SSAO, post-processing)

### Timeline & Context

- **Started:** September 2025  
- **Finished:** Early 2026  
- **Location:** Geneva, Switzerland  
- **Platforms:** Windows (primary), Linux (Ubuntu)  
- **Development time:** ~4–5 months (alongside coursework)

### What was achieved

The result is a **cross-platform 3D renderer** capable of:
- Loading complex 3D models
- Handling dozens of dynamic lights
- Applying SSAO, skybox rendering, and shadow mapping (directional + point lights)
- Rendering thousands of instanced objects while maintaining stable performance

### Downloads & Links

- **Windows build:**  
  https://github.com/MuteBlind99/Graphic-Engine/releases/download/1.0/Graphic_Engine_win.zip  
- **Source code:**  
  https://github.com/MuteBlind99/Graphic-Engine  

## 2. Project Architecture

The engine is designed around a **clear separation of responsibilities**, following a modular and data-oriented approach.
The goal is to keep each system focused, predictable, and easy to extend without introducing tight coupling.

At a high level, the project is divided into three major layers:

- **Core & Resource Management**
- **Scene & Camera Logic**
- **Rendering Pipeline**

This architecture allows new rendering features (SSAO, shadow maps, instancing, post-processing) to be integrated incrementally without rewriting existing systems.

---

### Core & Resource Management

GPU resources are expensive and must be handled carefully.  
To avoid memory leaks, redundant allocations, and undefined behavior, all GPU-related objects are managed centrally.

This includes:

- **Shader objects** (vertex / fragment)
- **Pipeline programs** (linked shader stages)
- **Textures**
- **Buffers (VBO, EBO, UBO, SSBO)**

Each resource follows a strict lifecycle:
1. Load data from disk or memory
2. Upload to GPU
3. Bind only when required
4. Release explicitly at shutdown

Shaders are loaded from source files, compiled per stage, and linked into reusable pipeline programs.  
This abstraction keeps OpenGL calls localized and makes debugging significantly easier.

---

### Meshes, Models, and Instancing

The engine relies exclusively on **modern OpenGL objects**:

- Vertex Array Objects (VAO)
- Vertex Buffer Objects (VBO)
- Element Buffer Objects (EBO)

Meshes store raw geometry data (positions, normals, UVs, tangents), while **models** act as containers grouping multiple meshes and materials.

To scale efficiently with large object counts, the engine makes extensive use of **GPU instancing**:

- A single mesh is shared by many instances
- Per-instance transformation matrices are uploaded once
- Rendering is performed using `glDrawElementsInstanced`

This approach drastically reduces draw calls and CPU overhead, making it possible to render thousands of objects with minimal performance cost.

Instancing is used for:
- Repeated environment assets
- Debug geometry
- Stress-testing the deferred pipeline

---

### Scene Representation

The scene layer acts as the bridge between game logic and rendering.

It is responsible for:
- Tracking visible objects
- Storing transforms (position, rotation, scale)
- Providing bounding data for culling
- Submitting renderable instances to the renderer

Scene objects themselves remain lightweight and mostly data-driven.
All heavy logic (lighting, shading, post-processing) is deferred to the rendering pipeline.

This separation ensures that scene complexity does not directly impact rendering performance.

---

### Camera System

The engine implements a **real-time FPS-style camera** designed for debugging and scene exploration.

Key characteristics:
- Euler angles (Yaw / Pitch)
- Free-look rotation
- Frame-rate independent movement
- Dynamic view and projection matrix updates

The camera system provides:
- View matrix
- Projection matrix
- Camera position (world-space)

These values are updated every frame and shared across all rendering passes, ensuring consistency between geometry, lighting, and post-processing.

---

### Rendering-Oriented Design

One important design decision was to make the **renderer stateless whenever possible**.

Instead of storing scene data internally, the renderer:
- Receives already-prepared draw information
- Executes rendering passes in a fixed order
- Outputs results to framebuffers or textures

This design keeps rendering deterministic and makes advanced features such as:
- Deferred shading
- Multi-pass lighting
- Screen-space effects

much easier to reason about and debug.

---

### Summary

The architecture prioritizes:
- Clear ownership of data
- Minimal coupling between systems
- Explicit GPU resource control
- Scalability with scene complexity

This foundation proved robust enough to support advanced techniques such as deferred rendering, shadow mapping, SSAO, and large-scale instancing, which are detailed in the following sections.
