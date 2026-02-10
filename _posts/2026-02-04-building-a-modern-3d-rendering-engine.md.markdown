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

## 3. Asset Loading (Models & Textures)

Efficient asset loading is a critical part of the engine, especially in a **deferred rendering architecture** where all geometric and material data must be prepared upfront during the geometry pass.

The asset system is designed with three main goals in mind:

- Reliability (no missing or partially loaded assets)
- Performance (minimal runtime overhead)
- Scalability (support for complex scenes and large datasets)

---

### Model Loading Pipeline

3D models are imported using a dedicated loading layer built on top of **Assimp** (Open Asset Import Library).

This provides native support for a wide range of formats:
- `.obj`
- `.fbx`
- `.gltf / .glb`
- `.dae`
- and many others

During the loading process, each model is decomposed into:
- One or more meshes
- Associated materials
- Texture references
- Bounding information (used later for culling and instancing)

The loading pipeline follows these steps:

1. Import the model file via Assimp
2. Traverse the scene hierarchy recursively
3. Extract mesh data (vertices, indices, materials)
4. Upload geometry data to GPU buffers
5. Cache textures to avoid duplicate loads

All GPU buffers (VAO, VBO, EBO) are created **once at load time**, ensuring zero allocation cost during rendering.

---

### Mesh Data Layout

Each mesh stores a tightly packed vertex layout optimized for GPU consumption:

- Position (`vec3`)
- Normal (`vec3`)
- Texture coordinates (`vec2`)
- Tangent / Bitangent (for normal mapping)

This layout supports:
- Standard lighting
- Normal mapping
- Deferred shading compatibility

Meshes are immutable once uploaded, which simplifies lifetime management and avoids synchronization issues between CPU and GPU.

---

### Texture Management

Textures are loaded from disk using **stb_image** and uploaded to the GPU as standard OpenGL textures.

The engine supports:
- Diffuse / Albedo maps
- Normal maps
- Specular maps
- Optional ambient occlusion maps

Each texture is configured with:
- Proper color format detection (RGB / RGBA)
- Mipmap generation
- Linear filtering
- Repeat or clamp wrapping modes

To prevent redundant GPU uploads, textures are cached internally.
If multiple meshes reference the same texture file, it is only loaded and uploaded once.

This caching strategy significantly reduces memory usage and loading times for complex models.

---

### Material Binding Strategy

Materials in the engine are lightweight and data-driven.

Each mesh references a set of textures by semantic:
- `material_diffuse`
- `material_specular`
- `material_normal`

At render time:
- Textures are bound to fixed texture units
- Uniform samplers are updated once per draw call
- The same material system works for both forward and deferred passes

This approach keeps shader code clean and avoids material-specific branching inside the rendering pipeline.

---

### Instancing-Friendly Design

Asset loading is tightly integrated with the engine’s **instancing system**.

When a model is marked as instanced:
- Mesh data is shared across all instances
- Per-instance transformation matrices are stored in GPU buffers
- Only instance data changes per frame

This makes it possible to render:
- Dense environments
- Repeated props
- Debug geometry

with minimal CPU overhead and excellent scalability.

---

### Bounding Volumes

While loading models, the engine computes bounding data:

- Axis-aligned bounding box (AABB)
- Bounding sphere (center + radius)

These values are later reused for:
- View frustum culling
- Instance visibility checks
- Debug visualization

Computing these bounds at load time avoids costly per-frame calculations and keeps the runtime pipeline efficient.

---

### Summary

The asset loading system provides:

- Robust support for industry-standard model formats
- Efficient GPU buffer creation
- Texture caching and reuse
- Seamless integration with deferred rendering and instancing

By front-loading all heavy work during asset initialization, the runtime rendering loop remains lightweight, predictable, and scalable — even with complex scenes and large object counts.

## 4. Rendering Pipeline – Deferred Rendering

The core of the engine is built around a **Deferred Rendering** pipeline.
This approach was chosen to efficiently handle scenes containing many dynamic light sources while keeping rendering costs predictable.

In contrast to forward rendering, deferred shading decouples **geometry complexity** from **lighting complexity**, making it particularly well-suited for modern real-time scenes.

---

### Why Deferred Rendering?

In a forward renderer, lighting is computed per object, per light, which quickly becomes expensive as the number of lights increases.

Deferred rendering reverses this logic:

- Geometry is rendered once
- Lighting is computed later in screen space
- The cost of lighting depends primarily on screen resolution, not scene complexity

This allows the engine to support:
- Dozens of dynamic lights
- Complex lighting models
- Screen-space effects (SSAO, shadows, post-processing)

with minimal impact on CPU performance.

---

### Overview of the Pipeline

The rendering pipeline is split into multiple well-defined passes:

1. **Geometry Pass** → Fill the G-Buffer  
2. **SSAO Pass** (optional)  
3. **Lighting Pass** → Compute lighting in screen space  
4. **Forward Pass** → Skybox and debug geometry  
5. **Post-Processing Pass** → Final image output  

Each pass operates on explicit framebuffer targets, making the pipeline easy to debug and extend.

---

### Geometry Pass (G-Buffer)

During the geometry pass, all visible geometry is rendered **without lighting** into a set of textures known as the **G-Buffer**.

The engine stores the following per-pixel attributes:

- **World-space position**
- **World-space normal**
- **Albedo (base color)**
- **Specular intensity**

These textures are written using multiple render targets (MRT) in a single pass.

Key characteristics:
- Depth testing enabled
- No blending
- No lighting calculations
- Heavy vertex processing, minimal fragment cost

The output of this pass fully describes the scene’s visible surfaces and is reused by all subsequent passes.

---

### Normal Encoding

To reduce memory bandwidth, normals are encoded before being written to the G-Buffer:

- Normals are remapped from `[-1, 1]` to `[0, 1]`
- Decoded later during the lighting pass

This optimization reduces storage cost while preserving sufficient precision for lighting calculations.

---

### Screen-Space Lighting Pass

Once the G-Buffer is filled, lighting is computed in a **single fullscreen pass**.

A screen-aligned quad samples the G-Buffer and evaluates lighting per pixel.

For each fragment:
1. World position is reconstructed
2. Normal is decoded and normalized
3. Material properties are retrieved
4. Lighting equations are evaluated

Supported light types:
- Directional lights
- Point lights (with attenuation)
- Spot lights

Each light contributes additively to the final color, making the system naturally scalable.

---

### Shadow Integration

Shadows are seamlessly integrated into the deferred lighting pass.

For each light that casts shadows:
- A dedicated shadow map is generated beforehand
- Shadow visibility is evaluated during lighting
- Diffuse and specular contributions are attenuated accordingly

Both **directional shadow maps** and **omnidirectional point light shadows** (cubemaps) are supported.

Because shadow sampling happens in screen space, shadow cost remains independent of scene complexity.

---

### SSAO Integration

Screen-Space Ambient Occlusion (SSAO) is applied as a separate pass between geometry and lighting.

The SSAO texture:
- Samples depth and normals from the G-Buffer
- Estimates local occlusion
- Is blurred to reduce noise

The resulting occlusion factor is then multiplied with ambient lighting during the lighting pass, improving depth perception and contact shadows.

---

### Forward Pass (Hybrid Rendering)

Certain elements are rendered **after** the deferred lighting pass using a traditional forward pipeline:

- Skybox
- Light gizmos
- Debug geometry
- Transparent objects (if needed)

This hybrid approach avoids common deferred rendering limitations, such as transparency handling, while keeping the main lighting system efficient.

---

### Advantages and Trade-offs

**Advantages:**
- Scales well with many lights
- Predictable performance
- Clean separation of geometry and lighting
- Ideal for screen-space effects

**Trade-offs:**
- Higher memory usage (G-Buffer)
- Limited support for transparency
- More complex pipeline compared to forward rendering

For this project, the benefits clearly outweighed the drawbacks.

---

### Summary

Deferred rendering is the backbone of the engine’s visual pipeline.

By separating geometry from lighting, the engine achieves:
- High performance with many dynamic lights
- A clean, extensible multi-pass architecture
- Seamless integration of shadows and SSAO

This foundation enables advanced visual effects while keeping the rendering loop efficient and maintainable.

## 5. Lighting & Shadow Mapping

Lighting is one of the most critical and performance-sensitive parts of a real-time renderer.
In this engine, lighting is fully integrated into the **deferred shading pipeline**, allowing multiple dynamic light sources and shadows to coexist efficiently.

The lighting system is designed to be:
- Scalable (many lights)
- Physically intuitive (classic lighting models)
- Debuggable (clear separation of passes)

---

### Lighting Model

The engine uses a **Blinn–Phong lighting model**, chosen for its balance between visual quality and performance.

For each pixel, the lighting equation is composed of:
- Ambient term
- Diffuse term
- Specular term

All lighting calculations are performed during the **lighting pass**, using data sampled from the G-Buffer:
- World-space position
- World-space normal
- Albedo (base color)
- Specular intensity

Because lighting is evaluated in screen space, the cost of lighting scales with screen resolution rather than scene complexity.

---

### Supported Light Types

The lighting system supports multiple light types simultaneously:

- **Directional Light**
  - Used to represent large-scale light sources such as the sun
  - Constant direction across the entire scene
- **Point Light**
  - Emits light in all directions from a single position
  - Distance-based attenuation
- **Spot Light**
  - Directional cone with inner and outer cutoff angles
  - Used for flashlights or focused light sources

All lights are evaluated additively during the lighting pass, making it easy to combine multiple light sources without additional geometry passes.

---

### Shadow Mapping Overview

Shadows are implemented using **shadow mapping**, a two-step process:

1. Render the scene from the point of view of the light into a depth texture
2. Compare fragment depth against the stored depth during lighting

Shadow calculations are fully integrated into the deferred lighting pass, allowing shadows to affect both diffuse and specular lighting contributions.

---

### Directional Light Shadows

Directional lights use a classic **2D depth shadow map**.

Process:
1. The scene is rendered from the light’s point of view using an orthographic projection
2. Depth values are stored in a shadow map texture
3. During lighting, each fragment is transformed into light space
4. Depth comparison determines whether the fragment is shadowed

To reduce common artifacts such as **shadow acne** and **peter panning**, the engine applies:
- A dynamic depth bias based on surface normal
- Percentage-Closer Filtering (PCF) for soft shadow edges

This produces stable, visually pleasing shadows suitable for large outdoor scenes.

---

### Point Light Shadows (Omnidirectional)

Point lights require shadows in all directions.
This is achieved using **cubemap shadow maps**.

Key characteristics:
- The scene is rendered six times per point light (one for each cubemap face)
- Depth values are stored in a cubemap texture
- During lighting, the distance from the fragment to the light is compared against the stored depth

While more expensive than directional shadows, point light shadows remain efficient thanks to:
- Limited number of shadow-casting lights
- Screen-space evaluation during the lighting pass

---

### Shadow Filtering & Quality

To improve shadow quality and reduce aliasing, the engine applies:

- **PCF (Percentage-Closer Filtering)**  
  Multiple depth samples are averaged to soften shadow edges.
- **Depth bias tuning**  
  Prevents self-shadowing artifacts without introducing floating shadows.

These techniques strike a balance between visual fidelity and performance, suitable for real-time applications.

---

### Lighting Management

All light data is centralized through a dedicated **Light Manager**.

Responsibilities include:
- Storing active lights
- Uploading light parameters to GPU uniforms
- Enforcing maximum light counts
- Enabling or disabling shadow casting per light

This centralized approach keeps the renderer stateless and avoids scattered lighting logic across the codebase.

---

### Debugging & Visualization

Lighting and shadows are notoriously difficult to debug.
To address this, the engine provides several debugging tools:

- Light gizmos rendered in world space
- G-Buffer visualization modes
- Shadow map inspection
- Real-time parameter tweaking via ImGui

These tools significantly reduced iteration time when tuning lighting, shadow bias, and attenuation values.

---

### Summary

The lighting and shadow system provides:

- Support for multiple dynamic light types
- Efficient screen-space lighting via deferred shading
- Directional and omnidirectional shadow mapping
- High-quality shadows with PCF filtering
- Robust debugging and visualization tools

This system forms the visual backbone of the engine and enables complex, well-lit scenes without sacrificing performance.

## 6. Screen-Space Ambient Occlusion (SSAO)

While direct lighting defines the overall illumination of a scene, it often lacks subtle contact shadows and depth cues.
To address this, the engine implements **Screen-Space Ambient Occlusion (SSAO)** as a post-process integrated into the deferred rendering pipeline.

SSAO enhances visual realism by approximating how ambient light is occluded in creases, corners, and areas where geometry is close together.

---

### Why SSAO?

Ambient occlusion is difficult to compute accurately in real time.
SSAO provides a practical compromise by operating entirely in screen space.

Benefits include:
- Improved depth perception
- Enhanced contact shadows
- No need for additional geometry passes
- Compatibility with deferred rendering

Because SSAO relies on data already present in the G-Buffer, it integrates naturally into the pipeline.

---

### SSAO Pipeline Overview

The SSAO implementation consists of three main steps:

1. **Sampling nearby geometry in view space**
2. **Estimating occlusion per pixel**
3. **Applying a blur to reduce noise**

The final SSAO texture is then used during the lighting pass to modulate ambient lighting.

---

### Sampling Kernel

For each pixel, the SSAO shader samples a set of random vectors distributed within a hemisphere oriented along the surface normal.

Key properties of the sampling kernel:
- Typically 64 samples
- Biased towards the origin for higher precision near the surface
- Precomputed on the CPU and uploaded to the GPU

These samples are rotated per-pixel using a small noise texture to reduce visible banding and pattern repetition.

---

### View-Space Reconstruction

SSAO calculations are performed in **view space** to ensure consistent depth comparisons.

Process:
1. World-space positions are reconstructed from the G-Buffer
2. Positions are transformed into view space
3. Normals are transformed accordingly
4. Sample positions are projected back into screen space for depth comparison

This approach avoids perspective distortion and produces stable occlusion results.

---

### Occlusion Estimation

For each sample:
- The sample position is projected into screen space
- The depth stored in the G-Buffer is compared against the sample depth
- If nearby geometry blocks ambient light, the occlusion factor increases

A smooth range check is applied to avoid harsh transitions and reduce halo artifacts.

The occlusion values are accumulated and normalized to produce a final ambient occlusion factor per pixel.

---

### Noise Reduction & Blur

Raw SSAO output is inherently noisy due to random sampling.
To mitigate this, the engine applies a **bilateral blur** pass.

This blur:
- Preserves edges by considering depth and normal similarity
- Smooths noise without bleeding across geometry boundaries

The result is a clean, stable SSAO texture suitable for real-time use.

---

### Integration with Lighting

During the lighting pass, the SSAO value is used to modulate the ambient lighting component:

- Fully occluded areas receive less ambient light
- Open areas remain mostly unaffected

This keeps SSAO subtle and physically plausible, avoiding overly dark or stylized results.

---

### Performance Considerations

SSAO is one of the more expensive screen-space effects.
To keep performance acceptable, several optimizations are applied:

- Reduced resolution SSAO buffer
- Limited kernel size
- Efficient blur pass
- Optional toggling via debug UI

These optimizations allow SSAO to be enabled without significantly impacting frame rate.

---

### Summary

The SSAO implementation adds depth and realism to the scene by:

- Enhancing contact shadows
- Improving spatial perception
- Integrating seamlessly with deferred lighting
- Maintaining real-time performance through targeted optimizations

Although approximate by nature, SSAO significantly improves visual quality and plays a key role in the overall rendering pipeline.

## 7. Challenges & Debugging

Building a modern rendering engine goes far beyond implementing algorithms.
A significant part of the work involved identifying subtle bugs, performance bottlenecks, and visual artifacts — and systematically fixing them.

This section highlights the most important challenges encountered during development and the strategies used to solve them.

---

### Debugging the Deferred Pipeline

Deferred rendering introduces multiple framebuffer passes, which can make debugging difficult.

Common issues included:
- Empty or incorrect G-Buffer outputs
- Mismatched texture formats
- Incorrect texture bindings between passes

To address this, each G-Buffer attachment was visualized independently:
- Albedo
- Normals
- Positions
- Specular

This made it possible to quickly identify whether an issue originated in the geometry pass or the lighting pass.

---

### RenderDoc & GPU Inspection

**RenderDoc** was used extensively to inspect GPU state and diagnose rendering issues.

Key use cases:
- Verifying framebuffer attachments
- Inspecting G-Buffer contents per draw call
- Validating uniform values and texture bindings
- Stepping through draw calls and render passes

By analyzing GPU captures frame by frame, many issues that were invisible at the CPU level became immediately obvious.

---

### Shadow Mapping Artifacts

Shadow mapping introduced several well-known artifacts:

- **Shadow acne** (self-shadowing)
- **Peter panning** (detached shadows)
- Hard shadow edges

Solutions applied:
- Dynamic depth bias based on surface normal
- Percentage-Closer Filtering (PCF)
- Careful tuning of near/far planes in shadow projections

Balancing shadow quality and stability required multiple iterations and extensive visual testing.

---

### SSAO Noise & Stability

Initial SSAO implementations produced significant noise and flickering.

Challenges included:
- Random sampling patterns
- Depth precision issues
- Temporal instability during camera movement

Fixes:
- Kernel sample distribution biased toward the origin
- Per-pixel noise rotation
- Bilateral blur pass

These improvements resulted in stable, visually pleasing ambient occlusion without excessive blur.

---

### Performance Bottlenecks

Performance profiling revealed several critical bottlenecks:

- Excessive draw calls
- Redundant state changes
- CPU overhead during scene submission

Optimizations included:
- GPU instancing for repeated geometry
- Centralized resource management
- Reduced per-frame allocations
- Limiting active lights per frame

These changes dramatically improved performance, especially in scenes with hundreds or thousands of objects.

---

### Debug Visualization Tools

To speed up iteration, several real-time debugging tools were implemented:

- Light gizmos rendered directly in the scene
- G-Buffer debug modes
- Shadow map previews
- ImGui-based parameter tweaking

Being able to visualize internal engine state in real time significantly reduced debugging time and made complex issues easier to reason about.

---

### Lessons Learned

Perhaps the most important lesson was that **most rendering bugs are not algorithmic, but architectural**.

Clear data ownership, explicit resource management, and well-defined rendering passes made debugging tractable — even as the engine grew in complexity.

---

### Summary

The main challenges encountered during development included:

- Debugging multi-pass rendering pipelines
- Managing shadow artifacts
- Stabilizing SSAO
- Optimizing performance at scale

Overcoming these challenges required a combination of strong tooling, careful design decisions, and iterative testing — all of which proved invaluable learning experiences.

## 8. Conclusion & Next Steps

This project was an in-depth exploration of **modern real-time rendering** and the practical challenges involved in building a graphics engine from the ground up.

By implementing a full **deferred rendering pipeline**, complete with dynamic lighting, shadow mapping, SSAO, and GPU instancing, I gained a much deeper understanding of how contemporary 3D engines operate at a low level.

---

### What I Learned

Beyond the individual techniques, this project reinforced several key principles:

- Rendering performance is often limited by **lighting and bandwidth**, not geometry
- Clear separation between systems makes complex pipelines easier to debug
- Explicit GPU resource management is essential for stability and scalability
- Debugging tools are as important as rendering algorithms themselves

Working directly with OpenGL and GLSL provided valuable insight into:
- GPU/CPU synchronization
- Multi-pass rendering workflows
- Common rendering artifacts and their solutions
- Performance-oriented engine design

---

### Technical Highlights

- Deferred shading with G-Buffer architecture
- Directional and omnidirectional shadow mapping
- Screen-Space Ambient Occlusion with bilateral blur
- Massive GPU instancing
- Hybrid rendering (deferred + forward)
- Real-time debugging tools (ImGui, RenderDoc)

---

### Next Steps

While the engine already supports a wide range of modern rendering features, several extensions are planned:

- Bloom and HDR tone mapping
- Physically Based Rendering (PBR)
- Cascaded Shadow Maps (CSM) for directional lights
- Frustum culling and GPU-driven visibility
- Temporal effects (TAA, temporal SSAO)
- Vulkan or DirectX 12 backend exploration

These additions would further push the engine toward production-level rendering techniques.

---

### Final Thoughts

This project was a significant milestone in my journey as a graphics programmer.
It combined theory, low-level API usage, performance optimization, and real-world debugging into a single cohesive experience.

More importantly, it confirmed my interest in **graphics programming and real-time rendering**, and provided a strong foundation for future engine and rendering work.

The full source code and builds are available on GitHub for anyone interested in exploring the implementation details.

