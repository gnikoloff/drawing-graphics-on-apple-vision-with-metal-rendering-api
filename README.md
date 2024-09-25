# Rendering fast 3D graphics on Apple Vision with the Metal API

1. Introduction
2. Metal API
3. Compositor Services
4. Stereoscoping Rendering
  4.1. Computing the View Matrices for Each Eye
  4.2. Vertex Amplification
  4.3. Variable Rate Rasterization (Foveation)
6. Pre-Rendering Tasks
 6.1. Compute
 6.2. Animation / Tweening
 6.2. Frame Prediction
   6.2.1. Querying the Next Frame
   6.2.2. Waiting Until Optimal Rendering Time
   6.2.3. Frame Submission
8. Base / Forward MSAA Pass
 7.1. Opaque Objects
 7.2. Skybox
 7.3. Transparent Objects
 7.4. Resolving MSAA Texture
9. Bloom Pass
10. Composite Pass
