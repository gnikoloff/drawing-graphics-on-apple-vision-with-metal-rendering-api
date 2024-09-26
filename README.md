# Rendering 3D graphics on Apple Vision with the Metal API

1. Introduction
   1. Metal API
   2. Compositor Services
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
5. Base / Forward MSAA Pass
   1. Opaque Objects
   2. Skybox
   3. Transparent Objects
   4. Resolving MSAA Texture
6. Bloom Pass
7. Composite Pass
