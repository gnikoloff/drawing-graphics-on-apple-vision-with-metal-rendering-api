# Rendering 3D graphics on Apple Vision with the Metal API

1. Introduction
   1. Metal API
   2. Compositor Services
2. Dissecting a Frame of RAYQUEST
3. Stereoscoping Rendering
   1. Computing the View Matrices for Each Eye
   2. Vertex Amplification
   3. Variable Rate Rasterization (Foveation)
4. Pre-Rendering Tasks
   1. Compute
   2. Animation / Tweening
   3. Frame Prediction
      1. Querying the Next Frame
      2. Waiting Until Optimal Rendering Time
      3. Frame Submission
5. Bloom Pass
6. Base / Forward MSAA Pass
   1. Opaque Objects
   2. Skybox
   3. Transparent Objects
   4. Resolving MSAA Texture
7. Composite Pass
8. Passthrough Rendering

## Introduction

Apple Vision Pro has been out for 7 months at the time of writing this article. Many games have been released for it since then and more and more game developers enter this niche. When it comes to rendering, most developers jump to an existing game engine such as Unity or use Apple's high level rendering APIs such as RealityKit. However a third way exists, one that has been there since visionOS 1.0 - rolling your own rendering engine using the Metal API. While challenging, this method is incredibly rewarding and gives you complete control over the rendering down to each individual byte and command submitted to the GPU.

> **_NOTE:_**  visionOS 2.0 allows us to render graphics via the Metal API and overlay (composite) them in **mixed** mode with the user's surroundings caputed by the device cameras' feed. This article focuses on writing Metal apps for fully **immersive** mode, although passthrough rendering will be mentioned at the end.


