# Rendering 3D graphics on Apple Vision with the Metal API

## Dissecting a Frame of RAYQUEST

1. Introduction
   1. `Metal` API
   2. `Compositor Services` API
2. Stereoscoping Rendering
   1. Creating and configuring a `LayerRenderer`
      1. Variable Rate Rasterization (Foveation)
      2. Organising the Metal textures used for presenting the rendered content on Apple Vision
   3. Vertex Amplification
      1. Preparing to render with Support for Vertex Amplification
      2. Encoding and submitting a render pass to the GPU
      3. Enabling Vertex Amplification for a Render Pass
      4. Computing the View and Projection Matrices for Each Eye
      5. Adding Vertex Amplification to our shaders
      6. Rendering
   3. Supporting both stereoscopic and flat 2D display rendering
      1. Two Rendering Paths. `LayerRenderer.Frame.Drawable` vs `MTKView`.
      3. Adapting our Vertex Shader
3. Updating and Encoding a Frame of Content
   1. Rendering on a Separate Thread
   2. Fetching a Next Frame for Drawing
   3. Handling User Input
   4. Waiting Until Optimal Rendering Time
   5. Issuing Draw Calls
   6. Frame Submission
  

4. Dissecting a Frame of RAYQUEST
5. Pre-Rendering Tasks
   1. Capturing user input via ARKit
   2. Compute
   3. Animation / Tweening
   4. Frame Prediction
      1. Querying the Next Frame
      2. Waiting Until Optimal Rendering Time
      3. Frame Submission
6. Base / Forward MSAA Pass
   1. Opaque Objects
   2. Skybox
   3. Transparent Objects
   4. Resolving MSAA Texture
7. Bloom Pass
8. Composite Pass
9. Passthrough Rendering

## Introduction

At the time of writing, Apple Vision Pro has been available for seven months, with numerous games released and an increasing number of developers entering this niche. When it comes to rendering, most opt for established game engines like Unity or Apple's high-level APIs like RealityKit. However, there's another option that's been available since visionOS 1.0: building your own rendering engine using the Metal API. Though challenging, this approach offers full control over the rendering pipeline, down to each byte and command submitted to the GPU on each frame.

> **Note**: visionOS 2.0 enables rendering graphics with the Metal API and compositing them in mixed mode with the user’s surroundings, captured by the device's cameras. This article focuses on developing Metal apps for fully immersive mode, though passthrough rendering will be discussed at the end. At the time of Apple Vision Pro release, visionOS 1.0 allowed for rendering with the Metal API in **immersive** mode only.

### Metal

To directly quote Apple:

> Metal is a modern, tightly integrated graphics and compute API coupled with a powerful shading language that is designed and optimized for Apple platforms. Its low-overhead model gives you direct control over each task the GPU performs, enabling you to maximize the efficiency of your graphics and compute software. Metal also includes an unparalleled suite of GPU profiling and debugging tools to help you improve performance and graphics quality.

I will not focus too much on the intristics of Metal in this article, however will mention that the API is mature, well documented and with plenty of tutorials and examples. I personally find it **very nice** to work with. It is more explicit than an API such as OpenGL ES, there is more planning involved in setting up your rendering pipeline and rendering frames, but is still very approachable and more beginner friendly then, say, Vulkan or DirectX12. Furthermore, Xcode has high quality built-in Metal profiler and debugger that allows for inspecting your GPU workloads and your shader inputs, code and outputs.

### Compositor Services

Compositor Services is a visionOS-specific API that bridges your SwiftUI code with your Metal rendering engine. In other words, it allows you to render directly via your custom Metal renderer to the Apple Vision displays, which include separate displays for the left and right eye.

At the app’s initialization, Compositor Services allows us to configure a [`LayerRenderer`](https://developer.apple.com/documentation/compositorservices/layerrenderer) object, created behind the scenes, to handle rendering on Apple Vision throughout the app’s lifecycle. This configuration includes texture layouts, pixel formats, foveation settings, and other rendering options. If no custom configuration is provided, Compositor Services defaults to standard settings. The LayerRenderer also supplies timing information to help manage your app’s rendering loop and deliver frames efficiently.

## Stereoscoping Rendering

In our scene creation code, we need to pass a type that adopts `CompositorLayerConfiguration` as a parameter to our scene content. The system will then use that configuration to create a `LayerRenderer` that will hold information such as the pixel formats of the final color and depth buffers, how the textures used to present the rendered content to Apple Vision's displays are organised, whether foveation is enabled and so on. Here is some boilerplate code:

```swift
struct ContentStageConfiguration: CompositorLayerConfiguration {
  func makeConfiguration(capabilities: LayerRenderer.Capabilities, configuration: inout LayerRenderer.Configuration) {
      // Specify the formats for both the color and depth output textures that Apple Vision will create for us.
      configuration.depthFormat = .depth32Float
      configuration.colorFormat = .bgra8Unorm_srgb

      // TODO: we will adjust the rest of the configuration further down in the article.
  }
}

@main
struct MyApp: App {
  var body: some Scene {
    WindowGroup {
      ContentView()
    }
    ImmersiveSpace(id: "ImmersiveSpace") {
      CompositorLayer(configuration: ContentStageConfiguration()) { layerRenderer in
         // layerRenderer is what we will use for rendering, frame timing and other presentation info in our engine
      }
    }
  }
}
```

#### Variable Rate Rasterization (Foveation)

Next thing we need to set up is whether to enable support for **foveation** in `LayerRenderer`. Foveation allows us to render at a higher resolution the content our eyes gaze directly at and render at a lower resolution everything else. That is very beneficial in VR as it allowes for improved performance.

Apple Vision does eye-tracking and foveation automatically for us (in fact, it is not possible for developers to access the user's gaze **at all** due to security concerns). So we need to setup our `LayerRenderer` to support it and we will get it "for free" during rendering. When we render to the `LayerRenderer` textures, Apple Vision will automatically adjust the resolution to be higher at the regions of the textures we directly gaze at. Here is the previous code that configures the `LayerRenderer`, updated with support for foveation:

```swift
func makeConfiguration(capabilities: LayerRenderer.Capabilities, configuration: inout LayerRenderer.Configuration) {
   // ...

   // Enable foveation
   let foveationEnabled = capabilities.supportsFoveation
   configuration.isFoveationEnabled = foveationEnabled
}
```

#### Organising the Metal textures used for presenting the rendered content

We established we need to render our content as two views to both Apple Vision displays. We have three options when it comes to the organization of the textures' layout we use for drawing:

1. `LayerRenderer.Layout.dedicated` - A layout that assigns a separate texture to each rendered view. So two eyes - two textures.
2. `LayerRenderer.Layout.shared` - A layout that uses a single texture to store the content for all rendered views. One texture big enough for both eyes.
3. `LayerRenderer.Layout.layered` - A layout that specifies each view’s content as a slice of a single 3D texture with two slices total.

Which one should you use? Apple official examples use `.layered`. Ideally `.shared` or `.layered` as having one texture to manage results in fewer things to keep track of, less commands to submit and less GPU context switches. I have seen people out there using two separate textures (`.dedicated`) but I don't see why.

Let's update the configuration code once more:

```swift
func makeConfiguration(capabilities: LayerRenderer.Capabilities, configuration: inout LayerRenderer.Configuration) {
   // ...

   // Set the LayerRenderer's texture layout configuration
   let options: LayerRenderer.Capabilities.SupportedLayoutsOptions = foveationEnabled ? [.foveationEnabled] : []
   let supportedLayouts = capabilities.supportedLayouts(options: options)
   configuration.layout = supportedLayouts.contains(.layered) ? .layered : .dedicated
}
```

That takes care of configuring the `LayerRenderer` for rendering our content at initialization time. Let's move on to rendering our content.

### Vertex Amplification

Imagine we have a triangle we want rendered on Apple Vision. A triangle consists of 3 vertices. If we were to render to a normal display we would submit 3 vertices to the GPU and let it draw them for us. On Apple Vision we have two displays. How do we go about it? A naive way would be to submit two drawing commands:

1. Issue a draw command to render 3 vertices to the left eye display.
2. Issue another draw command to render the same 3 vertices again, this time for the right eye display.

This is not optimal as it doubles the commands needed to be submited to the GPU for rendering. A 3 vertices triangle is fine, but for more complex scenes with even moderate amounts of geometry it becomes unwieldly very fast. Thankfully, Metal allows us to submit the 3 vertices once for both displays via a process called **Vertex Amplification**.

Taken from this great [article](https://developer.apple.com/documentation/metal/render_passes/improving_rendering_performance_with_vertex_amplification) on Vertex Amplification from Apple:

> With vertex amplification, you can encode drawing commands that process the same vertex multiple times, one per render target.

Does this sound useful? Well it is, because one "render target" from the quote above translates directly to one display on Apple Vision. Two displays for the left and right eyes - two render targets to which we can submit the same 3 vertices once, letting the Metal API "amplify" them for us, for free, with hardware acceleration, and render them to both displays **at the same time**. Vertex Amplification is not used only for rendering to both displays on Apple Vision and has it's benefits in general graphics techniques such as Cascaded Shadowmaps, where we submit one vertex and render it to multiple "cascades", represented as texture slices, for more adaptive and better looking realtime shadows.

#### Preparing to render with Support for Vertex Amplification

But back to Vertex Amplification as means for efficient rendering to both Apple Vision displays. Say we want to render the aformentioned 3 vertices triangle on Apple Vision. In order to render anything, on any Apple device, be it with a traditional display or two displays set-up, we need to create a `MTLRenderPipelineDescriptor` that will hold all of the state needed to render an object in a single render pass. Stuff like the vertex and fragment shaders to use, the color and depth pixel formats to use when rendering, the sample count if we use MSAA and so on. In the case of Apple Vision, we need to explicitly set the `maxVertexAmplificationCount` property when creating our `MTLRenderPipelineDescriptor`:

```swift
let pipelineStateDescriptor = MTLRenderPipelineDescriptor()
pipelineStateDescriptor.vertexFunction = vertexFunction
pipelineStateDescriptor.fragmentFunction = fragmentFunction
pipelineStateDescriptor.maxVertexAmplificationCount = 2
// ...
```

#### Encoding and submitting a render pass to the GPU

We now have a `MTLRenderPipelineDescriptor` that represents a graphics pipeline configuration with Vertex Amplification enabled. We can use it to create a render pipeline, represented by `MTLRenderPipelineState`. Once this render pipeline has been created created, the call to render this pipeline needs to be encoded into list of per-frame commands to be submitted to the GPU. What are examples of such commands? Imagine we are building a game with two objects and on each frame we do the following operations:

1. Set the clear color before rendering.
2. Set the viewport size.
3. Set the render target we are rendering to.
4. Clear the contents of the render target with the clear color set in step 1.
5. Set `MTLRenderPipelineState` for object A
6. Render object A.
7. Set `MTLRenderPipelineState` for object B
8. Render object B.
9. Finally submit all of the above commands to the GPU

All of these rendering commands represent a **render pass** that happens on each frame while our game is running. This render pass is represented by a [`MTLRenderCommandEncoder`](https://developer.apple.com/documentation/metal/mtlrendercommandencoder). We use this `MTLRenderCommandEncoder` to encode our **rendering** commands from steps 1 to 7 into a [`MTLCommandBuffer`](https://developer.apple.com/documentation/metal/mtlcommandbuffer) which is submitted to the GPU for execution. For a given frame, after these commands have been submitted by the CPU to the GPU for encoding, the GPU will execute each command in correct order, produce the final pixel values for the specific frame, and write them to the final texture to be presented to the user.

It is important to note that the commands to be encoded in a `MTLCommandBuffer` and submitted to the GPU are not only limited to rendering. We can submit commands to the GPU for general-purpose non-rendering work such as fast number crunching via the [`MTLComputeCommandEncoder`](https://developer.apple.com/documentation/metal/mtlcomputecommandencoder) (modern techniques for ML, physics, simulations, etc are all done on the GPU nowadays). Apple Vision's internal libraries for example use Metal for all the finger tracking, ARKit environment recognition and tracking and so on. However, let's focus only on the rendering commands for now.

#### Enabling Vertex Amplification for a Render Pass

When creating a render pass and submitting render commands for a frame via a `MTLRenderCommandEncoder`, we need to enable Vertex Amplification. This consists of three steps:

1. Specifying the number of amplifications to create.
2. Specifying view mappings that hold per-output offsets to a specific render target and viewport.
3. Specifying the viewport sizes for each render target.

Remember, we are dealing with two render targets on Apple Vision. So the number of amplifications is, of course, 2. The viewport sizes and view mappings into each render target depend on our textures' layout we specified when creating the `LayerRenderer` configuration used in Compositor Services above. We should **never** hardcode these values ourselves. Instead, we can query this info from the current frame's [`LayerRenderer.Frame`](https://developer.apple.com/documentation/compositorservices/layerrenderer/frame) from the `LayerRenderer` object visionOS created for us during the app initialization. Among other things, this `LayerRenderer.Frame` holds a [`LayerRenderer.Drawable`](https://developer.apple.com/documentation/compositorservices/layerrenderer/drawable) that provides the textures and information we need to draw a frame of content. We will explore these objects in more detail later on, but the important piece of information is that the `LayerRenderer.Drawable` we just queried will give us the correct viewport sizes and view mappings for each render target we will draw to.

```swift
// Get the current frame from Compositor Services
guard let frame = layerRenderer.queryNextFrame() else {
   return
}

// Get the current frame drawable
guard let drawable = frame.queryDrawable() else {
   return
}

// Creates a MTLRenderCommandEncoder
guard let renderEncoder = commandBuffer.makeRenderCommandEncoder(descriptor: renderPassDescriptor) else {
  return
}

// Query the current frame drawable view offset mappings for each render target
var viewMappings = (0 ..< 2).map {
   MTLVertexAmplificationViewMapping(
     viewportArrayIndexOffset: UInt32($0),
     renderTargetArrayIndexOffset: UInt32($0)
   )
}
// Set the number of amplifications and the correct view offset mappings for each render target
renderEncoder.setVertexAmplificationCount(2, viewMappings: &viewMappings)

// Query the current frame drawable viewport sizes for each render target
let viewports = drawable.views.map { $0.textureMap.viewport }
// Set the viewport sizes for each render target
renderEncoder.setViewports(viewports)

// Encode our rendering commands into the MTLRenderCommandEncoder

// Submit the MTLRenderCommandEncoder to the GPU for execution
```

#### Computing the View and Projection Matrices for Each Eye

Okay, we enabled foveation, created our `LayerRenderer` that holds the textures we will render to, and have vertex amplification enabled. Next we need to compute the correct view and projection matrices **for each eye** to use for rendering. If you have done computer graphics work or used a game engine like Unity, you know that we create a virtual camera that sits somewhere in our 3D world, is oriented to point at a specific direction, has a specific field of view, a certain aspect ratio, a near and a far plane and so on. We use the view and projection matrix of the camera to transform a vertex's 3D position in our game world to clip space, which in turn is is further transformed by the GPU to finally end up in device screen space coordinates.

When rendering to a 2D screen, it is up to us, as programmers, to construct this virtual camera and decide what values all of these properties will have. Since our rendered objects' positions are ultimately presented on a 2D screen that we look at from some distance, these properties do not have to be "physically based" to match our eyes and field of view. We can go crazy with really small range of field of view, use a portrait aspect ratio, some weird projection ("fish eye") and so on for rendering.

When rendering on Apple Vision we can not set these camera properties or augment them manually in any way. Doing otherwise will result in things looking "weird", and not matching our eyes (remember the initial eye setup you had to do when you bought your Apple Vision?). I can't imagine Apple being okay with publishing apps that augment the default camera projections as they might break the immersion and make the product look crappy.

So on each frame, we need to query 2 view matrices representing each eye's position in the physical world. Similarly, we need to query 2 perspective projection matrices that encode the **correct** aspect, field of view, near and far planes and so on for each eye from the current frame `LayerRenderer.Drawable`. Each eye's "view" is represented by [`LayerRenderer.Drawable.View
`](https://developer.apple.com/documentation/compositorservices/layerrenderer/drawable/view). Compositor Services provides a view for each distinct render viewpoint. It is up to us to obtain these 4 matrices on each frame from both left and right eye views and use them to render our content to both the screens. These 4 matrices are:

1. Left eye view matrix
2. Right eye view matrix
3. Left eye projection matrix
4. Right eye projection matrix

##### View Matrices

These matrices represent each eye's position and orientation **with regards to the world coordinate space**. As you move around your room the view matrices will change. Shorter people will get different view matrices then tall people. You sitting on a couch and looking to the left will produce different view matrices than you standing up and looking to the right.

Obtaining any of these matrices is a 3 step process:

1. Obtain Apple Vision's view transform **pose** matrix that indicates Apple Vision's position and orientation in the world coordinate system.

This is global and not tied to a specific eye. It has nothing do to with Compositor Services or the current frame's `Drawable`. Instead, to obtain it, we need to use ARKit and more specifically the visionOS-specific [`WorldTrackingProvider`](https://developer.apple.com/documentation/arkit/worldtrackingprovider), which is a source of live data about the device pose and anchors in a person’s surroundings. Here is some code:

```swift
// During app initialization
let worldTracking = WorldTrackingProvider()
let arSession = ARKitSession()

// During app update loop
Task {
  do {
    let dataProviders: [DataProvider] = [Self.worldTracking]
    try await arSession.run(dataProviders)
  } catch {
    fatalError("Failed to run ARSession")
  }
}

// During app render loop
let deviceAnchor = worldTracking.queryDeviceAnchor(atTimestamp: time)

// Query Apple Vision's world position and orientation anchor. If not available for some reason, fallback to an identity matrix
let simdDeviceAnchor = deviceAnchor?.originFromAnchorTransform ?? float4x4.identity
```

`simdDeviceAnchor` now holds Apple Vision's transform pose matrix.
   
2. Obtain the eyes' local transformation matrix

This matrices specify the position and orientation of the left and right eyes **releative** to the device's pose. Just like any eye-specific information, we need to query it from the current frame's `Drawable`. Here is how we obtain the left and right eyes local view matrices:

```swift
let leftViewLocalMatrix = drawable.views[0].transform
let rightViewLocalMatrix = drawable.views[1].transform
```

3. Multiply the device pose matrix by each eye local transformation matrix to obtain each eye view transform matrix in the world coordinate space.

To get the final world transformation matrix for each eye we multiply the matrix from step 1. by both eyes' matrices from step 2:

```swift
let leftViewWorldMatrix = (deviceAnchorMatrix * leftEyeLocalMatrix.transform).inverse
let rightViewWorldMatrix = (deviceAnchorMatrix * rightEyeLocalMatrix.transform).inverse
```

> **_NOTE:_** Pay special attention to the `.inverse` part in the end! That is because Apple Vision expects us to use a reverse-Z projection. This is especially important for passthrough rendering with Metal on visionOS 2.0.

Hopefully this image illustrates the concept:

![Apple Vision eye matrices illustrated](vision-pro-matrices.png)

`simdDeviceAnchor` would be 

To recap so far, let's refer to the 4 matrices needed to render our content on Apple Vision's displays. We already computed the first two, the eyes world view transformation matrices, so let's cross them out from our to-do list:

1. ~Left eye view matrix~
2. ~Right eye view matrix~
3. Left eye projection matrix
4. Right eye projection matrix

Two more projection matrices to go.

##### Left and Right Eyes Projection Matrices

These two matrices encode the perspective projections for each eye. Just like any eye-specific information, they very much rely on Compositor Services and the current `LayerRenderer.Frame`. How do we go about computing them?

Each `LayerRenderer.Drawable.View` for both eyes gives us a property called [`.tangents`](https://developer.apple.com/documentation/compositorservices/layerrenderer/drawable/view/4082271-tangents). It represents the values for the angles you use to determine the planes of the viewing frustum. We can use these angles to construct the volume between the near and far clipping planes that contains the scene’s visible content. We will use these tangent values to build the perspective projection matrix for each eye.

Let's obtain the tangent property for both eyes:

```swift
let leftViewTangents = drawable.views[0].tangents
let rightViewTangents = drawable.views[0].tangents
```

We will also need to get the near and far planes to use in our projections. They are the same for both eyes. We can query them from the current frame's `LayerRenderer.Drawable` like so:

```swift
let farPlane = drawable.depthRange.x
let nearPlane = drawable.depthRange.y
```

> **_NOTE:_** Notice that the far plane is encoded in the `.x` property, while the near plane is in the `.y` range. That is, and I can not stress it enough, because Apple expects us to use reverse-Z projection matrices.

> **_NOTE:_** At the time of writing this article, the far plane (`depthRange.x`) is actually positioned at infinity. Not sure why Apple decided to do this. Leaving it as-is will break certain techniques (for example subdividing the viewing frustum volume into subparts for Cascaded Shadowmaps). In RAYQUEST I actually artifically overwrite and cap this value at something like -500 before constructing my projection matrices. Remember what I said about never overwriting the default projection matrix attributes Apple Vision gives you? Well I did it only in this case. It works well for immersive space rendering. I can imagine however that overwriting any of these values is a big no-no for passthrough rendering on visionOS 2.0 (which has a different way of constructing projection matrices for each eye alltogether btw).

Now that we have them, we will utilise Apple's new [Spatial](https://developer.apple.com/documentation/spatial) API. It will allow us to create and manipulate 3D mathematical primitives. What we are interested in particular is the [`ProjectiveTransform3D`](https://developer.apple.com/documentation/spatial/projectivetransform3d) that will allow us to obtain a perspective matrix for each eye given the tangents we queried earlier. Here is how it looks in code:

```swift
let leftViewProjectionMatrix = ProjectiveTransform3D(
  leftTangent: Double(leftViewTangents[0]),
  rightTangent: Double(leftViewTangents[1]),
  topTangent: Double(leftViewTangents[2]),
  bottomTangent: Double(leftViewTangents[3]),
  nearZ: depthRange.x,
  farZ: depthRange.y,
  reverseZ: true
)

let rightViewProjectionMatrix = ProjectiveTransform3D(
  leftTangent: Double(rightViewTangents[0]),
  rightTangent: Double(rightViewTangents[1]),
  topTangent: Double(rightViewTangents[2]),
  bottomTangent: Double(rightViewTangents[3]),
  nearZ: depthRange.x,
  farZ: depthRange.y,
  reverseZ: true
)
```

And that's it! We have obtained all 4 matrices needed to render our content. The global view and projection matrix for the left and the right eyes.

Armed with these 4 matrices we can now move on to writing our shaders for stereoscoping rendering.

#### Adding Vertex Amplification to our shaders

Usually when rendering objects to a texture we need to supply a pair of shaders: the vertex and fragment shaders. Let's not focus on the fragment shader, as it normally will be the same for both eyes (unless you want to give the user headache).

If you have done traditional, non-VR non-stereoscoping rendering, you know that you construct a virtual camera, position and orient it in the world and supply it to the vertex shader which in turn multiplies each vertex with the camera view and projection matrices to turn it from local space to clip space. If you made it this far in this article, I assume you have seen this in your shader language of choice:

```metal
typedef struct {
   matrix_float4x4 projectionMatrix;
   matrix_float4x4 viewMatrix;
   // ...
} CameraEyeUniforms;

typedef struct {
  float4 position [[position]];
  float2 texCoord [[shared]];
  float3 normal [[shared]];
} VertexOut;

vertex VertexOut myVertexShader(constant CameraEyeUniforms &camera [[buffer(0)]]) {
   VertexOut out = {
      .position = camera.projectionMatrix * camera.viewMatrix * vertexPosition,
      .texCoord = /* compute UV */,
      .normal = /* compute normal */
   };
   return out;
}

fragment float4 myFragShader() {
   return float4(1, 0, 0, 1);
}
```

This vertex shader expects 2 camera matrices - the view and projection matrices.

> **_NOTE:_** Take a look at the `VertexOut` definition. `texCoord` and `normal` are marked as `shared`, while `position` is not. That's because the position values will change depending on the current Vertex Amplification index. Both eyes have a different pair of matrices to transform each vertex with. The output vertex for the left eye render target will have different final positions than the output vertex for the right eye.
> I hope this makes clear why `texCoord` and `normal` are `shared`. The values are **not** view or projection dependent. Their values will always be uniforms across different render targets, regardless of with which eye are we rendering them. For more info check out this [article](https://developer.apple.com/documentation/metal/render_passes/improving_rendering_performance_with_vertex_amplification).

Remember we have two displays and two eye views on Apple Vision. Each view holds it's own respective view and projection matrices. We need a vertex shader that will accept 4 matrices - a view and projection matrices for each eye. 

Let's first rewrite our `CameraUniforms` struct:

```metal
typedef struct {
   CameraEyeUniforms camUniforms[2];
   // ...
} CameraBothEyesUniforms;
```

See what we did there? We treat the original `CameraUniforms` as a single eye and combine both eyes in `camUniforms`. With that out of the way, we need to instruct the vertex shader which matrices to use exactly. How do we do that? Well, we get a special `amplification_id` property as input to our shaders. It allows us to query the index of which Vertex Amplification are we currently executing. We have two amplifications for both eyes, so now we can easily query our `camUniforms` array! Here is the revised vertex shader:

```metal
vertex VertexOut myVertexShader(
   ushort ampId [[amplification_id]],
   constant CameraBothEyesUniforms &bothEyesCameras [[buffer(0)]]
) {
   constant CameraEyeUniforms &camera = bothEyesCameras.uniforms[ampId];
   VertexOut out = {
      .position = camera.projectionMatrix * camera.viewMatrix * vertexPosition;
   };
   return out;
}
```

And that's it! Our output textures and render commands have been setup correctly, we have obtained all required matrices and compiled our shaders with support for Vertex Amplification. All that's left to do is...

#### Rendering

When it comes to actually issuing draw commands, drawing on Apple Vision does not differ from any other platforms. All [drawing commands](https://developer.apple.com/documentation/metal/mtlrendercommandencoder/render_pass_drawing_commands) such as `drawPrimitices` and `drawIndexedPrimitives` work exactly the same. We issue a draw call, encode it and submit it to the GPU.

### Supporting both stereoscopic and flat 2D display rendering 

As you can see, Apple Vision requires us to **always** think in terms of two eyes and two render targets. Our rendering code, matrices and shaders were built around this concept. So the question is, can we write a renderer that supports "traditional" and stereoscoping rendering simultaneously? Of course we can! Doing so however requires some carefull planning and inevitably some preprocessor directives in your codebase.

#### Two Rendering Paths. `LayerRenderer.Frame.Drawable` vs `MTKView`

On Apple Vision, you configure a `LayerRenderer` at init time and the system gives you `LayerRenderer.Frame.Drawable` on each frame to draw to. On macOS / iOS / iPadOS and so on, you create a `MTKView` and a `MTKViewDelegate` that allows you to hook into the system resizing and drawing updates. In both cases you present your rendered content to the user by drawing to the texture provided by the system to you. How would this look in code? How about this:

```swift
open class Renderer {
   #if os(visionOS)
      public var currentDrawable: LayerRenderer.Drawable
   #else
      public var currentDrawable: MTKView
   #endif

   private func renderFrame() {
      // prepare frame, run animations, collect user input, etc

      #if os(visionOS)
         // prepare a two sets of view and projection matrices for both eyes
         // render to both render targets simultaneously 
      #else
         // prepare a view and projection matrix for a single virtual camera
         // render to single render target
      #endif

      // submit your rendering commands to the GPU for rendering
   }
}
```

By using preprocessor directives in Swift, we can build our project for different targets. This way we can have two render paths for stereoscoping and normal 2D rendering.

It should be noted that the 2D render path will omit all of the vertex amplification commands we prepared earlier on the CPU to be submitted to the GPU for drawing. Stuff like `renderEncoder.setVertexAmplificationCount(2, viewMappings: &viewMappings)` and `renderEncoder.setViewports(viewports)` is no longer needed.

#### Adapting our Vertex Shader

The vertex shader we wrote earlier needs some rewriting to support non-Vertex Amplified rendering. That can be done easily with [Metal function constants](https://developer.apple.com/documentation/metal/using_function_specialization_to_build_pipeline_variants). If you don't know what they are or how to use them, please refer to the linked article first. They basically allow us to compile one shader binary and then conditionally enable / disable things in it when using it to build render or compute pipelines. Take a look:

```metal
typedef struct {
   matrix_float4x4 projectionMatrix;
   matrix_float4x4 viewMatrix;
} CameraUniforms;

typedef struct {
   CameraEyeUniforms camUniforms[2];
} CameraBothEyesUniforms;

typedef struct {
  float4 position [[position]];
} VertexOut;

constant bool isAmplifiedRendering [[function_constant(0)]];
constant bool isNonAmplifiedRendering = !isAmplifiedRendering;

vertex VertexOut myVertexShader(
   ushort ampId                                    [[amplification_id]],
   constant CameraUniforms &camera                 [[buffer(0), function_constant(isNonAmplifiedRendering]],
   constant CameraBothEyesUniforms &cameraBothEyes [[buffer(1), function_constant(isAmplifiedRendering)]]
) {
   if (isAmplifiedRendering) {
      constant CameraEyeUniforms &camera = bothEyesCameras.uniforms[ampId];
      out.position = camera.projectionMatrix * camera.viewMatrix * vertexPosition;
   } else {
      out.position = camera.projectionMatrix * camera.viewMatrix * vertexPosition;
   }
   return out;
}

fragment float4 myFragShader() {
   return float4(1, 0, 0, 1);
}
```

Our updated shader supports both flat 2D and stereoscoping rendering. All we need to set the `isAmplifiedRendering` function constant when creating a `MTLRenderPipelineState` and supply the correct matrices to it.

> **_NOTE:_** It is important to note that even when rendering on Apple Vision you may need to render to a flat 2D texture. One example would be drawing shadows, where you put a virtual camera where the sun should be, render to a depth buffer and then project these depth values when rendering to the main displays to determine if a pixel is in shadow or not. Rendering from the Sun point of view in this case does not require Vertex Amplification.
> With our updated vertex shader, we can now support both.










## Dissecting a Frame of RAYQUEST

[RAYQUEST](https://rayquestgame.com/) is my first game published on Apple Vision. It utilises Compositor Services and Metal to deliver advanced graphics at 4K resolution at 90 frames per second. This article will take one random frame of it captured during gameplay and show you how is it rendered by taking you on a trip through the complete rendering pipeline.

So here is the frame we will examine:

![Random frame captured during gameplay of RAYQUEST](rayquest-random-frame.png)

> **_NOTE:_** Before continuing, I would like to mention that aside from the rendering engine capabilities and techniques covered in this article, a lot of the boilerplate, setup code and theory has been covered in this great [official example](https://developer.apple.com/documentation/compositorservices/drawing_fully_immersive_content_using_metal) by Apple. In fact, the origins of the RAYQUEST code lead straight to this example. I strongly suggest it as an accompanying reading to this article.
