# Drawing Graphics on Apple Vision with the Metal Rendering API

## Table of Contents

1. [Introduction](#introduction)
   1. [Why Write This Article?](#why-write-this-article)
   1. [`Metal`](#metal)
   2. [`Compositor Services`](#compositor-services)
2. [Creating and configuring a `LayerRenderer`](#creating-and-configuring-a-layerrenderer)
   1. [Variable Rate Rasterization (Foveation)](#variable-rate-rasterization-foveation)
   2. [Organising the Metal Textures Used for Presenting the Rendered Content](#organising-the-metal-textures-used-for-presenting-the-rendered-content)
3. [Vertex Amplification](#vertex-amplification)
   1. [Preparing to Render with Support for Vertex Amplification](#preparing-to-render-with-support-for-vertex-amplification)
   2. [Enabling Vertex Amplification for a Render Pass](#enabling-vertex-amplification-for-a-render-pass)
   3. [Specifying the Viewport Mappings for Both Render Targets](#specifying-the-viewport-mappings-for-both-render-targets)
   4. [Computing the View and Projection Matrices for Each Eye](#computing-the-view-and-projection-matrices-for-each-eye)
   5. [Adding Vertex Amplification to our Shaders](#adding-vertex-amplification-to-our-shaders)
      1. [Vertex Shader](#vertex-shader)
      2. [Fragment Shader](#fragment-shader)
4. [Updating and Encoding a Frame of Content](#updating-and-encoding-a-frame-of-content)
   1. [Rendering on a Separate Thread](#rendering-on-a-separate-thread)
   2. [Fetching a Next Frame for Drawing](#fetching-a-next-frame-for-drawing)
   3. [Getting Predicted Render Deadlines](#getting-predicted-render-deadlines)
   4. [Updating Our App State Before Rendering](#updating-our-app-state-before-rendering)
   5. [Waiting Until Optimal Rendering Time](#waiting-until-optimal-rendering-time)
   6. [Frame Submission Phase](#frame-submission-phase)
5. [Supporting Both Stereoscopic and non-VR Display Rendering](#supporting-both-stereoscopic-and-non-vr-display-rendering)
   1. [Two Rendering Paths. `LayerRenderer.Frame.Drawable` vs `MTKView`](#two-rendering-paths-layerrendererframedrawable-vs-mtkview)
   3. [Adapting our Vertex Shader](#adapting-our-vertex-shader)
6. [Gotchas](#gotchas)
   1. [Can't Render to a Smaller Resolution Pixel Buffer when Foveation is Enabled](#cant-render-to-a-smaller-resolution-pixel-buffer-when-foveation-is-enabled)
   2. [Postprocessing](#postprocessing)
   3. [True Camera Position](#true-camera-position)
   4. [Apple Vision Simulator](#apple-vision-simulator)

## Introduction

At the time of writing, Apple Vision Pro has been available for seven months, with numerous games released and an increasing number of developers entering this niche. When it comes to rendering, most opt for established game engines like Unity or Apple's high-level APIs like RealityKit. However, there's another option that's been available since the beginning: building your own rendering engine using the Metal API. Though challenging, this approach offers full control over the rendering pipeline, down to each byte and command submitted to the GPU on each frame.

> **_NOTE_**: visionOS 2.0 enables rendering graphics with the Metal API and compositing them in **mixed** mode with the user’s surroundings, captured by the device's cameras. This article focuses on developing Metal apps for fully immersive mode, though passthrough rendering will be discussed at the end. At the time of Apple Vision Pro release, visionOS 1.0 allowed for rendering with the Metal API in **immersive** mode only.

### Why Write This Article?

Mainly as a recap of all I have learned for myself. I used all of this while building [RAYQUEST](https://rayquestgame.com/), my first game for Apple Vision. I am not going to present any groundbreaking techniques or anything that you can not find in Apple documentation and official examples, aside from some gotchas I discovered while developing my game. In fact, I'd treat this article as an additional reading to the Apple examples. Read them first or read this article first. I will link to Apple's relevant docs and examples as much as possible as I explain the upcomming concepts.

### Metal

To directly quote Apple [docs](https://developer.apple.com/metal/):

> Metal is a modern, tightly integrated graphics and compute API coupled with a powerful shading language that is designed and optimized for Apple platforms. Its low-overhead model gives you direct control over each task the GPU performs, enabling you to maximize the efficiency of your graphics and compute software. Metal also includes an unparalleled suite of GPU profiling and debugging tools to help you improve performance and graphics quality.

I will not focus too much on the intristics of Metal in this article, however will mention that the API is mature, well documented and with plenty of tutorials and examples. I personally find it **very nice** to work with. If you want to learn it I suggest you read [this book](https://www.kodeco.com/books/metal-by-tutorials/v4.0). It is more explicit than APIs such as OpenGL ES, there is more planning involved in setting up your rendering pipeline and rendering frames, but is still very approachable and more beginner friendly then, say, Vulkan or DirectX12. Furthermore, Xcode has high quality built-in Metal profiler and debugger that allows for inspecting your GPU workloads and your shader inputs, code and outputs.

### Compositor Services

Compositor Services is a visionOS-specific API that bridges your SwiftUI code with your Metal rendering engine. It enables you to encode and submit drawing commands directly to the Apple Vision displays, which include separate screens for the left and right eye.

At the app’s initialization, Compositor Services automatically creates and configures a [`LayerRenderer`](https://developer.apple.com/documentation/compositorservices/layerrenderer) object to manage rendering on Apple Vision during the app lifecycle. This configuration includes texture layouts, pixel formats, foveation settings, and other rendering options. If no custom configuration is provided, Compositor Services defaults to its standard settings. Additionally, the `LayerRenderer` supplies timing information to optimize the rendering loop and ensure efficient frame delivery.

## Creating and configuring a `LayerRenderer`

In our scene creation code, we need to pass a type that adopts the [`CompositorLayerConfiguration`](https://developer.apple.com/documentation/compositorservices/compositorlayerconfiguration) protocol as a parameter to our scene content. The system will then use that configuration to create a `LayerRenderer` that will hold information such as the pixel formats of the final color and depth buffers, how the textures used to present the rendered content to Apple Vision displays are organised, whether foveation is enabled and so on. More on all these fancy terms a bit later. Here is some boilerplate code:

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

### Variable Rate Rasterization (Foveation)

Next thing we need to set up is whether to enable support for **foveation** in `LayerRenderer`. Foveation allows us to render at a higher resolution the content our eyes gaze directly at and render at a lower resolution everything else. That is very beneficial in VR as it allows for improved performance.

Apple Vision does eye-tracking and foveation automatically for us (in fact, it is not possible for developers to access the user's gaze **at all** due to security concerns). We need to setup our `LayerRenderer` to support it and we will get it "for free" during rendering. When we render to the `LayerRenderer` textures, Apple Vision will automatically adjust the resolution to be higher at the regions of the textures we directly gaze at. Here is the previous code that configures the `LayerRenderer`, updated with support for foveation:

```swift
func makeConfiguration(capabilities: LayerRenderer.Capabilities, configuration: inout LayerRenderer.Configuration) {
   // ...

   // Enable foveation
   let foveationEnabled = capabilities.supportsFoveation
   configuration.isFoveationEnabled = foveationEnabled
}
```

### Organising the Metal Textures Used for Presenting the Rendered Content

We established we need to render our content as two views to both Apple Vision left and right displays. We have three options when it comes to the organization of the textures' layout we use for drawing:

1. [`LayerRenderer.Layout.dedicated`](https://developer.apple.com/documentation/compositorservices/layerrenderer/layout/dedicated) - A layout that assigns a separate texture to each rendered view. Two eyes - two textures.
2. [`LayerRenderer.Layout.shared`](https://developer.apple.com/documentation/compositorservices/layerrenderer/layout/shared) - A layout that uses a single texture to store the content for all rendered views. One texture big enough for both eyes.
3. [`LayerRenderer.Layout.layered`](https://developer.apple.com/documentation/compositorservices/layerrenderer/layout/layered) - A layout that specifies each view’s content as a slice of a single 3D texture with two slices.

Which one should you use? Apple official examples use `.layered`. Ideally `.shared` or `.layered`, as having one texture to manage results in fewer things to keep track of, less commands to submit and less GPU context switches. Some important to Apple Vision rendering techniques such as vertex amplification do not work with `.dedicated`, which expects a separate render pass to draw content for each eye texture, so it is best avoided.

Let's update the configuration code once more:

```swift
func makeConfiguration(capabilities: LayerRenderer.Capabilities, configuration: inout LayerRenderer.Configuration) {
   // ...

   // Set the LayerRenderer's texture layout configuration
   let options: LayerRenderer.Capabilities.SupportedLayoutsOptions = foveationEnabled ? [.foveationEnabled] : []
   let supportedLayouts = capabilities.supportedLayouts(options: options)

   configuration.layout = supportedLayouts.contains(.layered) ? .layered : .shared
}
```

That takes care of the basic configuration for `LayerRenderer` for rendering our content. We set up our textures' pixel formats, whether to enable foveation and the texture layout to use for rendering. Let's move on to rendering our content.

### Vertex Amplification

Imagine we have a triangle we want rendered on Apple Vision. A triangle consists of 3 vertices. If we were to render it to a "normal" non-VR display we would submit 3 vertices to the GPU and let it draw them for us. On Apple Vision we have two displays. How do we go about it? A naive way would be to submit two drawing commands:

1. Issue draw command **A** to render 3 vertices to the left eye display.
2. Issue draw command **B** to render the same 3 vertices again, this time for the right eye display.

> **_NOTE_** Rendering everything twice and issuing double the amount of commands is required if you have chosen `.dedicated` texture layout when setting up your `LayerRenderer`.

This is not optimal as it doubles the commands needed to be submited to the GPU for rendering. A 3 vertices triangle is fine, but for more complex scenes with even moderate amounts of geometry it becomes unwieldly very fast. Thankfully, Metal allows us to submit the 3 vertices once for both displays via a technique called **Vertex Amplification**.

Taken from this great [article](https://developer.apple.com/documentation/metal/render_passes/improving_rendering_performance_with_vertex_amplification) on vertex amplification from Apple:

> With vertex amplification, you can encode drawing commands that process the same vertex multiple times, one per render target.

Does this sound useful? Well it is, because one "render target" from the quote above translates directly to one display on Apple Vision. Two displays for the left and right eyes - two render targets to which we can submit the same 3 vertices once, letting the Metal API "amplify" them for us, for free, with hardware acceleration, and render them to both displays **at the same time**. Vertex Amplification is not used only for rendering to both displays on Apple Vision and has its benefits in general graphics techniques such as Cascaded Shadowmaps, where we submit one vertex and render it to multiple "cascades", represented as texture slices, for more adaptive and better looking realtime shadows.

#### Preparing to Render with Support for Vertex Amplification

But back to vertex amplification as means for efficient rendering to both Apple Vision displays. Say we want to render the aforementioned 3 vertices triangle on Apple Vision. In order to render anything, on any Apple device, be it with a non-VR display or two displays set-up, we need to create a [`MTLRenderPipelineDescriptor`](https://developer.apple.com/documentation/metal/mtlrenderpipelinedescriptor) that will hold all of the state needed to render an object in a single render pass. Stuff like the vertex and fragment shaders to use, the color and depth pixel formats to use when rendering, the sample count if we use MSAA and so on. In the case of Apple Vision, we need to explicitly set the `maxVertexAmplificationCount` property when creating our `MTLRenderPipelineDescriptor`:

```swift
let pipelineStateDescriptor = MTLRenderPipelineDescriptor()
pipelineStateDescriptor.vertexFunction = vertexFunction
pipelineStateDescriptor.fragmentFunction = fragmentFunction
pipelineStateDescriptor.maxVertexAmplificationCount = 2
// ...
```

#### Enabling Vertex Amplification for a Render Pass

We now have a `MTLRenderPipelineDescriptor` that represents a graphics pipeline configuration with vertex amplification enabled. We can use it to create a render pipeline, represented by [`MTLRenderPipelineState`](https://developer.apple.com/documentation/metal/mtlrenderpipelinestate). Once this render pipeline has been created, the call to render this pipeline needs to be encoded into list of per-frame commands to be submitted to the GPU. What are examples of such commands? Imagine we are building a game with two objects and on each frame we do the following operations:

1. Set the clear color before rendering.
2. Set the viewport size.
3. Set the render target we are rendering to.
4. Clear the contents of the render target with the clear color set in step 1.
5. Set `MTLRenderPipelineState` for object A as active
6. Render object A.
7. Set `MTLRenderPipelineState` for object B as active
8. Render object B.
9. Submit all of the above commands to the GPU
10. Write the resulting pixel values to some pixel attachment

All of these rendering commands represent a single **render pass** that happens on each frame while our game is running. This render pass is configured via [`MTLRenderPassDescriptor`](https://developer.apple.com/documentation/metal/mtlrenderpassdescriptor). We need to configure the render pass to use foveation and output to two render targets simultaneously.

1. Enable foveation by supplying a [`rasterizationRateMap`](https://developer.apple.com/documentation/metal/mtlrasterizationratemap) property to our `MTLRenderPassDescriptor`. This property, represented by [`MTLRasterizationRateMap`](https://developer.apple.com/documentation/metal/mtlrasterizationratemap) is created for us behind the scenes by the Compositor Services. We don't have a direct say in its creation. Instead, we need to query it. On each frame, `LayerRenderer` will supply us with a [`LayerRenderer.Frame`](https://developer.apple.com/documentation/compositorservices/layerrenderer/frame) object. Among other things, `LayerRenderer.Frame` holds a [`LayerRenderer.Drawable`](https://developer.apple.com/documentation/compositorservices/layerrenderer/drawable). More on these objects later. For now, we need to know that this `LayerRenderer.Drawable` object holds not only the textures for both eyes we will render our content into, but also an array of `MTLRasterizationRateMap`s that hold the foveation settings for each display.
2. Set the amount of render targets we will render to by setting the [`renderTargetArrayLength`](https://developer.apple.com/documentation/metal/mtlrenderpassdescriptor/1437975-rendertargetarraylength) property. Since we are dealing with two displays, we set it to 2.

```swift
// Get the current frame from Compositor Services
guard let frame = layerRenderer.queryNextFrame() else {
   return
}

// Get the current frame drawable
guard let drawable = frame.queryDrawable() else {
   return
}

let renderPassDescriptor = MTLRenderPassDescriptor()
// ...

// both eyes ultimately have the same foveation settings. Let's use the left eye MTLRasterizationRateMap for both eyes
renderPassDescriptor.rasterizationRateMap = drawable.rasterizationRateMaps.first
renderPassDescriptor.renderTargetArrayLength = 2
```

> **_NOTE_** Turning on foveation prevents rendering to a pixel buffer with smaller resolution than the device display. Certain graphics techniques allow for rendering to a lower resolution pixel buffer and upscaling it before presenting it or using it as an input to another effect. That is a performance optimisation. Apple for example has the [MetalFX](https://developer.apple.com/documentation/metalfx) upscaler that allows us to render to a smaller pixel buffer and upscale it back to native resolution. That is not possible when rendering on visionOS with foveation enabled due to the `rasterizationRateMaps` property. That property is set internally by Compositor Services when a new `LayerRenderer` is created based on whether we turned on the [`isFoveationEnabled`](https://developer.apple.com/documentation/compositorservices/layerrenderer/configuration-swift.struct/isfoveationenabled) property in our layer configuration. We don't have a say in the direct creation of the `rasterizationRateMaps` property. We can not use smaller viewport sizes sizes when rendering to our `LayerRenderer` textures that have predefined rasterization rate maps because the viewport dimensions will not match. We can not change the dimensions of the predefined rasterization rate maps.
>
> With foveation disabled you **can** render to a pixel buffer smaller in resolution than the device display. You can render at, say, 75% of the native resolution and use MetalFX to upscale it to 100%. This approach works on Apple Vision.

Once created with the definition above, the render pass represented by a [`MTLRenderCommandEncoder`](https://developer.apple.com/documentation/metal/mtlrendercommandencoder). We use this `MTLRenderCommandEncoder` to encode our **rendering** commands from the steps above into a [`MTLCommandBuffer`](https://developer.apple.com/documentation/metal/mtlcommandbuffer) which is submitted to the GPU for execution. For a given frame, after these commands have been issued and submitted by the CPU to the GPU for encoding, the GPU will execute each command in correct order, produce the final pixel values for the specific frame, and write them to the final texture to be presented to the user.

> **_NOTE_** A game can and often does have multiple render passes per frame. Imagine you are building a first person racing game. The main render pass would draw the interior of your car, your opponents' cars, the world, the trees and so on. A second render pass will draw all of the HUD and UI on top. A third render pass might be used for drawing shadows. A fourth render pass might render the objects in your rearview mirror and so on. All of these render passes need to be encoded and submitted to the GPU on each new frame for drawing.

It is important to note that the commands to be encoded in a `MTLCommandBuffer` and submitted to the GPU are not only limited to rendering. We can submit "compute" commands to the GPU for general-purpose non-rendering work such as fast number crunching via the [`MTLComputeCommandEncoder`](https://developer.apple.com/documentation/metal/mtlcomputecommandencoder) (modern techniques for ML, physics, simulations, etc are all done on the GPU nowadays). Apple Vision internal libraries for example use Metal for all the finger tracking, ARKit environment recognition and tracking and so on. However, let's focus only on the rendering commands for now.

#### Specifying the Viewport Mappings for Both Render Targets

We already created a render pass with vertex amplification enabled. We need to instruct Metal on the correct viewport offsets and sizes for each render target before we render. We need to:

2. Specify the view mappings that hold per-output offsets to a specific render target and viewport.
3. Specify the viewport sizes for each render target.

The viewport sizes and view mappings into each render target depend on our textures' layout we specified when creating the `LayerRenderer` configuration used in Compositor Services earlier in the article. We should **never** hardcode these values ourselves. Instead, we can query this info from the current frame `LayerRenderer.Drawable`. It provides the information and textures we need to draw into for a given frame of content. We will explore these objects in more detail later on, but the important piece of information is that the `LayerRenderer.Drawable` we just queried will give us the correct viewport sizes and view mappings for each render target we will draw to.

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

let viewports = drawable.views.map { $0.textureMap.viewport }
renderEncoder.setViewports(viewports)

// Encode our rendering commands into the MTLRenderCommandEncoder

// Submit the MTLRenderCommandEncoder to the GPU for execution
```

#### Computing the View and Projection Matrices for Each Eye

Okay, we created our `LayerRenderer` that holds the textures we will render to, enabled foveation, and have vertex amplification enabled. Next we need to compute the correct view and projection matrices **for each eye** to use during rendering. If you have done computer graphics work or used a game engine like Unity, you know that usually we create a virtual camera that sits somewhere in our 3D world, is oriented towards a specific direction, has a specific field of view, a certain aspect ratio, a near and a far plane and other attributes. We use the view and projection matrix of the camera to transform a vertex's 3D position in our game world to clip space, which in turn is is further transformed by the GPU to finally end up in device screen space coordinates.

When rendering to a non-VR screen, it is up to us, as programmers, to construct this virtual camera and decide what values all of these properties will have. Since our rendered objects' positions are ultimately presented on a 2D screen that we look at from some distance, these properties do not have to be "physically based" to match our eyes and field of view. We can go crazy with really small range of field of view, use a portrait aspect ratio, some weird projection ("fish eye") and so on for rendering. Point being is that, in non-VR rendering, we are given leeway on how to construct the camera we use depending on the effect and look we are trying to achieve.

When rendering on Apple Vision we can not set these camera properties or augment them manually in any way as it might cause sickness. Changing the default camera properties will result in things looking "weird", and not matching our eyes (remember the initial eye setup you had to do when you bought your Apple Vision?). I can't imagine Apple being okay with publishing apps that augment the default camera projections as they might break the immersion, feel "off" and make the product look crappy.

My point is that we have to use the projection and view matrices given to us by Apple Vision. We are trying to simulate a world in immersive mode or mix our content with the real environment in mixed mode. It should feel natural to the user, as if she is not even wearing a device. We should not downscale the field of view, change the aspect ratio or mess with any other settings.

So on each frame, we need to query 2 view matrices representing each eye's position and orientation in the physical world. Similarly, we need to query 2 perspective projection matrices that encode the **correct** aspect, field of view, near and far planes for each eye from the current frame `LayerRenderer.Drawable`. Each eye's "view" is represented by [`LayerRenderer.Drawable.View
`](https://developer.apple.com/documentation/compositorservices/layerrenderer/drawable/view). Compositor Services provides a distinct view for each eye, i.e. each display. It is up to us to obtain these 4 matrices on each frame from both the left and right eye `LayerRenderer.Drawable.View` and use them to render our content to both of the displays. These 4 matrices are:

1. Left eye view matrix
3. Left eye projection matrix
2. Right eye view matrix
4. Right eye projection matrix

##### View Matrices

These matrices represent each eye's position and orientation **with regards to the world coordinate space**. As you move around your room the view matrices will change. Shorter people will get different view matrices then tall people. You sitting on a couch and looking to the left will produce different view matrices than you standing up and looking to the right.

Obtaining the view matrices for both eyes is a 3 step process:

1. Obtain Apple Vision view transform **pose** matrix that indicates the device position and orientation in the world coordinate system.

This is global and not tied to a specific eye. It has nothing do to with Compositor Services or the current frame's `LayerRenderer.Drawable`. Instead, to obtain it, we need to use ARKit and more specifically the visionOS-specific [`WorldTrackingProvider`](https://developer.apple.com/documentation/arkit/worldtrackingprovider), which is a source of live data about the device pose and anchors in a person’s surroundings. Here is some code:

```swift
// During app initialization
let worldTracking = WorldTrackingProvider()
let arSession = ARKitSession()

// During app update loop
Task {
  do {
    let dataProviders: [DataProvider] = [worldTracking]
    try await arSession.run(dataProviders)
  } catch {
    fatalError("Failed to run ARSession")
  }
}

// During app render loop
let deviceAnchor = worldTracking.queryDeviceAnchor(atTimestamp: time)

// Query Apple Vision world position and orientation anchor. If not available for some reason, fallback to an identity matrix
let simdDeviceAnchor = deviceAnchor?.originFromAnchorTransform ?? float4x4.identity
```

`simdDeviceAnchor` now holds Apple Vision head transform pose matrix.
   
2. Obtain the eyes' local transformation matrix

These matrices specify the position and orientation of the left and right eyes **releative** to the device's pose. Just like any eye-specific information, we need to query it from the current frame's `LayerRenderer.Drawable`. Here is how we obtain the left and right eyes local view matrices:

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

> **_NOTE_** Pay special attention to the `.inverse` part in the end! That is because Apple Vision expects us to use a reverse-Z projection. This is especially important for passthrough rendering with Metal on visionOS 2.0.

Hopefully this image illustrates the concept:

![Apple Vision eye matrices illustrated](vision-pro-matrices.png)

To recap so far, let's refer to the 4 matrices needed to render our content on Apple Vision displays. We already computed the first two, the eyes world view transformation matrices, so let's cross them out from our to-do list:

1. ~Left eye view matrix~
2. ~Right eye view matrix~
3. Left eye projection matrix
4. Right eye projection matrix

Two more projection matrices to go.

##### Left and Right Eyes Projection Matrices

These two matrices encode the perspective projections for each eye. Just like any eye-specific information, they very much rely on Compositor Services and the current `LayerRenderer.Frame`. How do we go about computing them?

Each `LayerRenderer.Drawable.View` for both eyes gives us a property called [`.tangents`](https://developer.apple.com/documentation/compositorservices/layerrenderer/drawable/view/4082271-tangents). It represents the values for the angles you use to determine the planes of the viewing frustum. We can use these angles to construct the volume between the near and far clipping planes that contains the scene’s visible content. We will use these tangent values to build the perspective projection matrix for each eye.

> **_NOTE_** The `.tangents` property is in fact deprecated on visionOS 2.0 and should not be used in new code. To obtain correct projection matrices for a given eye, one should use the new Compositor Services' [`.computeProjection`](https://developer.apple.com/documentation/compositorservices/layerrenderer/drawable/computeprojection(convention:viewindex:)) method. I will still cover doing it via the `.tangents` property for historical reasons.

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

> **_NOTE_** Notice that the far plane is encoded in the `.x` property, while the near plane is in the `.y` range. That is, and I can not stress it enough, because Apple expects us to use reverse-Z projection matrices.

> **_NOTE_** At the time of writing this article, at least on visionOS 1.0, the far plane (`depthRange.x`) in the reverse Z projection is actually positioned at negative infinity. I am not sure if this is the case in visionOS 2.0. Not sure why Apple decided to do this. Leaving it as-is at infiity will break certain techniques (for example subdividing the viewing frustum volume into subparts for Cascaded Shadowmaps). In RAYQUEST I actually artifically overwrite and cap this value at something like -500 before constructing my projection matrices. Remember what I said about never overwriting the default projection matrix attributes Apple Vision gives you? Well, I did it only in this case. It works well for immersive space rendering. I can imagine however that overwriting any of these values is a big no-no for passthrough rendering on visionOS 2.0 (which has a different way of constructing projection matrices for each eye alltogether via the [`.computeProjection`](https://developer.apple.com/documentation/compositorservices/layerrenderer/drawable/computeprojection(convention:viewindex:))).

Now that we have both `tangents` for each eye, we will utilise Apple's [Spatial](https://developer.apple.com/documentation/spatial) API. It will allow us to create and manipulate 3D mathematical primitives. What we are interested in particular is the [`ProjectiveTransform3D`](https://developer.apple.com/documentation/spatial/projectivetransform3d) that will allow us to obtain a perspective matrix for each eye given the tangents we queried earlier. Here is how it looks in code:

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

#### Adding Vertex Amplification to our Shaders

Usually when rendering objects we need to supply a pair of shaders: the vertex and fragment shader. Let's focus on the vertex shader first.

##### Vertex Shader

If you have done traditional, non-VR non-stereoscoping rendering, you know that you construct a virtual camera, position and orient it in the world and supply it to the vertex shader which in turn multiplies each vertex with the camera view and projection matrices to turn it from local space to clip space. If you made it this far in this article, I assume you have seen this in your shader language of choice:

```metal
typedef struct {
   matrix_float4x4 projectionMatrix;
   matrix_float4x4 viewMatrix;
   // ...
} CameraEyeUniforms;

typedef struct {
   float4 vertexPosition [[attribute(0)]];
} VertexIn;

typedef struct {
  float4 position [[position]];
  float2 texCoord [[shared]];
  float3 normal [[shared]];
} VertexOut;

vertex VertexOut myVertexShader(
   Vertex in [[stage_in]],
   constant CameraEyeUniforms &camera [[buffer(0)]]
) {
   VertexOut out = {
      .position = camera.projectionMatrix * camera.viewMatrix * in.vertexPosition,
      .texCoord = /* compute UV */,
      .normal = /* compute normal */
   };
   return out;
}

fragment float4 myFragShader() {
   return float4(1, 0, 0, 1);
}
```

This vertex shader expects a single pair of matrices - the view matrix and projection matrix.

> **_NOTE_** Take a look at the `VertexOut` definition. `texCoord` and `normal` are marked as `shared`, while `position` is not. That's because the position values will change depending on the current vertex amplification index. Both eyes have a different pair of matrices to transform each vertex with. The output vertex for the left eye render target will have different final positions than the output vertex for the right eye.
> I hope this makes clear why `texCoord` and `normal` are `shared`. The values are **not** view or projection dependent. Their values will always be uniforms across different render targets, regardless of with which eye are we rendering them. For more info check out this [article](https://developer.apple.com/documentation/metal/render_passes/improving_rendering_performance_with_vertex_amplification).

Remember we have two displays and two eye views on Apple Vision. Each view holds it's own respective view and projection matrices. We need a vertex shader that will accept 4 matrices - a view and projection matrices for each eye. 

Let's introduce a new struct:

```metal
typedef struct {
   CameraEyeUniforms camUniforms[2];
   // ...
} CameraBothEyesUniforms;
```

See what we did there? We treat the original `CameraUniforms` as a single eye and combine both eyes in `camUniforms`. With that out of the way, we need to instruct the vertex shader which pair of matrices to use exactly. How do we do that? Well, we get a special `amplification_id` property as input to our shaders. It allows us to query the index of which vertex amplification are we currently executing. We have two amplifications for both eyes, so now we can easily query our `camUniforms` array! Here is the revised vertex shader:

```metal
vertex VertexOut myVertexShader(
   ushort ampId [[amplification_id]],
   Vertex in [[stage_in]],
   constant CameraBothEyesUniforms &bothEyesCameras [[buffer(0)]]
) {
   constant CameraEyeUniforms &camera = bothEyesCameras.camUniforms[ampId];
   VertexOut out = {
      .position = camera.projectionMatrix * camera.viewMatrix * in.vertexPosition;
   };
   return out;
}
```

And that's it! Our output textures and render commands have been setup correctly, we have obtained all required matrices and compiled our vertex shader with support for vertex amplification.

##### Fragment Shader

I will skip on the fragment shader code for brevity sake. However will mention a few techniques that require us to know the camera positions and / or matrices in our fragment shader code:

1. Lighting, as we need to shade pixels based on the viewing angle between the pixel and the camera
2. Planar reflections
3. Many post-processing effects
4. Non-rendering techniques such as Screen Space Particle Collisions

In all these cases, we need the two pair of view + projection matrices for each eye. Fragment shaders also get the `amplification_id` property as input, so we can query the correct matrices in exactly the same way as did in the vertex shader above.

> **_NOTE_** Compute shaders **do not** get an `[[amplification_id]]` property. That makes porting and running view-dependent compute shaders harder when using two texture views in stereoscoping rendering. When adopting well established algorithms you may need to rethink them to account for two eyes and two textures.

All that's left to do is...

## Updating and Encoding a Frame of Content

### Rendering on a Separate Thread

Rendering on a separate thread is recommended in general but especially important on Apple Vision. That is because, during rendering, we will pause the render thread to wait until the optimal rendering time provided to us by Compositor Services. We want the main thread to be able to continue to run, process user inputs, network and so on in the meantime.

How do we go about this? Here is some code:

```swift
@main
struct MyApp: App {
  var body: some Scene {
    WindowGroup {
      ContentView()
    }
    ImmersiveSpace(id: "ImmersiveSpace") {
      CompositorLayer(configuration: ContentStageConfiguration()) { layerRenderer in
         let engine = GameEngine(layerRenderer)
         engine.startRenderLoop()
      }
    }
  }
}

class GameEngine {
   private var layerRenderer: LayerRenderer

   public init(_ layerRenderer: LayerRenderer) {
      self.layerRenderer = layerRenderer

       layerRenderer.onSpatialEvent = { eventCollection in
         // process spatial events
       }
   }

   public func startRenderLoop() {
      let renderThread = Thread {
         self.renderLoop()
       }
       renderThread.name = "Render Thread"
       renderThread.start()
   }

   private func renderLoop() {
      while true {
        if layerRenderer.state == .invalidated {
          print("Layer is invalidated")
          return
        } else if layerRenderer.state == .paused {
          layerRenderer.waitUntilRunning()
          continue
        } else {
          autoreleasepool {
            // render next frame here
            onRender()
          }
        }
      }
   }

   private func onRender() {
      // ...
   }
}
```

We start a separate thread that on each frame checks the [`LayerRenderer.State`](https://developer.apple.com/documentation/compositorservices/layerrenderer/state-swift.enum) property. Depending on this property value, it may skip the current frame, quit the render loop entirely or draw to the current frame. The main thread is unaffected and continues running other code and waits for spatial events.

### Fetching a Next Frame for Drawing

Remember all the code we wrote earlier that used `LayerRenderer.Frame`? We obtained the current `LayerRenderer.Drawable` from it and queried it for the current frame view and projection matrices, view mappings and so on. This `LayerRenderer.Frame` is obviously different across frames and we have to constantly query it before using it and encoding draw commands to the GPU. Let's expand upon the `onRender` method from the previous code snippet and query the next frame for drawing:

```swift
class GameEngine {
   // ...
   private func onRender() {
      guard let frame = layerRenderer.queryNextFrame() else {
         print("Could not fetch current render loop frame")
         return
       }
   }
}
```

### Getting Predicted Render Deadlines

We need to block our render thread until the optimal rendering time to start the submission phase given to us by Compositor Services. First we will query this optimal rendering time and use it later. Let's expand our `onRender` method:

```swift
private func onRender() {
   guard let frame = layerRenderer.queryNextFrame() else {
      print("Could not fetch current render loop frame")
      return
   }
   guard let timing = frame.predictTiming() else {
      return
   }
}
```

### Updating Our App State Before Rendering

Before doing any rendering, we need to update our app state. What do I mean by this? We usually have to process actions such as:

1. User input
2. Animations
3. Frustum Culling
5. Physics
6. Enemy AI
7. Audio

These tasks can be done on the CPU or on the GPU via compute shaders, it does not matter. What does matter is that we need to process and run all of them **before rendering**, because they will dictate what and how exactly do we render. As an example, if you find out that two enemy tanks are colliding during this update phase, you may want to color them differently during rendering. If the user is pointing at a button, you may want to change the appearance of the scene. Apple calls this the **update phase** in their docs btw.

> **_NOTE_** All of the examples above refer to non-rendering work. However we **can** do rendering during the update phase!
> The important distinction that Apple makes is whether we rely on the device anchor information during rendering. It is important to do only rendering work that **does not** depend on the device anchor in the update phase. Stuff like shadow map generation would be a good candidate for this. We render our content from the point of view of the sun, so the device anchor is irrelevant to us during shadowmap rendering.
> Remember, it still may be the case that we have to wait until optimal rendering time and skip the current frame. We do not have reliable device anchor information **yet** during the update phase.

We mark the start of the update phase by calling Compositor Service's [`startUpdate`](https://developer.apple.com/documentation/compositorservices/layerrenderer/frame/startupdate()) method. Unsurprisingly, after we are done with the update phase we call [`endUpdate`](https://developer.apple.com/documentation/compositorservices/layerrenderer/frame/endupdate()) that notifies Compositor Services that you have finished updating the app-specific content you need to render the frame. Here is our updated render method:

```swift
private func onRender() {
   guard let frame = layerRenderer.queryNextFrame() else {
      print("Could not fetch current render loop frame")
      return
   }
   guard let timing = frame.predictTiming() else {
      return
   }

   frame.startUpdate()

   // do your game's physics, animation updates, user input, raycasting and non-device anchor related rendering work here

   frame.endUpdate()
}
```

### Waiting Until Optimal Rendering Time

We already queried and know the optimal rendering time given to us by Compositor Services. After wrapping up our update phase, we need to block our render thread until the optimal time to start the submission phase of our frame. To block the thread we can use Compositor Service's [`LayerRenderer.Clock`](https://developer.apple.com/documentation/compositorservices/layerrenderer/clock) and more specifically its [`.wait()`](https://developer.apple.com/documentation/compositorservices/layerrenderer/clock/wait(until:tolerance:)) method:

```swift
private func onRender() {
   guard let frame = layerRenderer.queryNextFrame() else {
      print("Could not fetch current render loop frame")
      return
   }
   guard let timing = frame.predictTiming() else {
      return
   }

   frame.startUpdate()

   // do your game's physics, animation updates, user input, raycasting and non-device anchor related rendering work here

   frame.endUpdate()

   // block the render thread until optimal input time
   LayerRenderer.Clock().wait(until: timing.optimalInputTime)
}
```

### Frame Submission Phase

We have ended our update phase and waited until the optimal rendering time. It is time to start the **submission phase**. **That** is the right time to query the device anchor information, compute the correct view and projection matrices and submit any view-related drawing commands with Metal (basically all the steps we did in the "Vertex Amplification" chapter of this article).

Once we have submitted all of our drawing and compute commands to the GPU, we end the frame submission. The GPU will take all of the submited commands and execute them for us.

```swift
private func onRender() {
   guard let frame = layerRenderer.queryNextFrame() else {
      print("Could not fetch current render loop frame")
      return
   }
   guard let timing = frame.predictTiming() else {
      return
   }

   frame.startUpdate()

   // do your game's physics, animation updates, user input, raycasting and non-device anchor related rendering work here

   frame.endUpdate()

   LayerRenderer.Clock().wait(until: timing.optimalInputTime)

   frame.startSubmission()

   // we already covered this code, query device anchor position and orientation in physical world
   let deviceAnchor = worldTracking.queryDeviceAnchor(atTimestamp: time)
   let simdDeviceAnchor = deviceAnchor?.originFromAnchorTransform ?? float4x4.identity

   // submit all of your rendering and compute related Metal commands here

   // mark the frame as submitted and hand it to the GPU
   frame.endSubmission()
}
```

And that's it! To recap: we have to use a dedicated render thread for drawing and Compositor Services' methods to control its execution. We are presented with two phases: update and submit. We update our app state in the update phase and issue draw commands with Metal in the submit phase.

## Supporting Both Stereoscopic and non-VR Display Rendering 

As you can see, Apple Vision requires us to **always** think in terms of two eyes and two render targets. Our rendering code, matrices and shaders were built around this concept. So the question is, can we write a renderer that supports "traditional" non-VR and stereoscoping rendering simultaneously? Of course we can! Doing so however requires some careful planning and inevitably some preprocessor directives in your codebase.

### Two Rendering Paths. `LayerRenderer.Frame.Drawable` vs `MTKView`

On Apple Vision, you configure a `LayerRenderer` at init time and the system gives you `LayerRenderer.Frame.Drawable` on each frame to draw to. On macOS / iOS / iPadOS and so on, you create a `MTKView` and a `MTKViewDelegate` that allows you to hook into the system resizing and drawing updates. In both cases you present your rendered content to the user by drawing to the texture provided by the system for you. How would this look in code? How about this:

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

By using preprocessor directives in Swift, we can build our project for different targets. This way we can have two render paths for stereoscoping (Apple Vision) and normal 2D rendering (all other Apple hardware).

It should be noted that the 2D render path will omit all of the vertex amplification commands we prepared earlier on the CPU to be submitted to the GPU for drawing. Stuff like `renderEncoder.setVertexAmplificationCount(2, viewMappings: &viewMappings)` and `renderEncoder.setViewports(viewports)` is no longer needed.

### Adapting our Vertex Shader

The vertex shader we wrote earlier needs some rewriting to support non-Vertex Amplified rendering. That can be done easily with [Metal function constants](https://developer.apple.com/documentation/metal/using_function_specialization_to_build_pipeline_variants). Function constants allow us to compile one shader binary and then conditionally enable / disable things in it when using it to build render or compute pipelines. Take a look:

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

> **_NOTE_** It is important to note that even when rendering on Apple Vision you may need to render to a flat 2D texture. One example would be drawing shadows, where you put a virtual camera where the sun should be, render to a depth buffer and then project these depth values when rendering to the main displays to determine if a pixel is in shadow or not. Rendering from the Sun point of view in this case does not require multiple render targets or vertex amplification. With our updated vertex shader, we can now support both.

## Gotchas

I have hinted at some of these throughout the article, but let's recap them and write them down together.

### Can't Render to a Smaller Resolution Pixel Buffer when Foveation is Enabled

Turning on foveation prevents rendering to a pixel buffer with smaller resolution than the device display. Certain graphics techniques allow for rendering to a lower resolution pixel buffer and upscaling it before presenting it or using it as an input to another effect. That is a performance optimisation. Apple for example has the [MetalFX](https://developer.apple.com/documentation/metalfx) upscaler that allows us to render to a smaller pixel buffer and upscale it back to native resolution. That is not possible when rendering on visionOS with foveation enabled due to the [`rasterizationRateMaps`](https://developer.apple.com/documentation/compositorservices/layerrenderer/drawable/rasterizationratemaps) property. That property is set internally by Compositor Services when a new `LayerRenderer` is created based on whether we turned on the [`isFoveationEnabled`](https://developer.apple.com/documentation/compositorservices/layerrenderer/configuration-swift.struct/isfoveationenabled) property in our layer configuration. We don't have a say in the direct creation of the `rasterizationRateMaps` property. We can not use smaller viewport sizes sizes when rendering to our `LayerRenderer` textures that have predefined rasterization rate maps because the viewport dimensions will not match. We can not change the dimensions of the predefined rasterization rate maps.

### Postprocessing

Many Apple official examples use compute shaders to postprocess the final scene texture. Implementing sepia, vignette and other graphics techniques happen at the postprocessing stage.

Using compute shaders to write to the textures provided by Compositor Services' `LayerRenderer` is not allowed. That is because these textures do not have the [`MTLTextureUsage.shaderWrite`](https://developer.apple.com/documentation/metal/mtltextureusage/1515854-shaderwrite) flag enabled. We can not enable this flag post factum ourselves, because they were internally created by Compositor Services. So for postprocessing we are left with spawning fullscreen quads for each display and using fragment shader to implement our postprocessing effects. That is allowed because the textures provided by Compositor Services **do** have the [`MTLTextureUsage.renderTarget`](https://developer.apple.com/documentation/metal/mtltextureusage/1515701-rendertarget) flag enabled. That is the case with the textures provided by `MTKView` on other Apple hardware btw.

### True Camera Position

Remember when we computed the view matrices for both eyes earlier? 

```swift
let leftViewWorldMatrix = (deviceAnchorMatrix * leftEyeLocalMatrix.transform).inverse
let rightViewWorldMatrix = (deviceAnchorMatrix * rightEyeLocalMatrix.transform).inverse
```

These are 4x4 matrices and we can easily extract the translation out of them to obtain each eye's world position. Something like this:

```swift
extension SIMD4 {
  public var xyz: SIMD3<Scalar> {
    get {
      self[SIMD3(0, 1, 2)]
    }
    set {
      self.x = newValue.x
      self.y = newValue.y
      self.z = newValue.z
    }
  }
}

// SIMD3<Float> vectors representing the XYZ position of each eye
let leftEyeWorldPosition = leftViewWorldMatrix.columns.xyz
let rightEyeWorldPosition = rightViewWorldMatrix.columns.xyz
```

The question now is, what is the true camera single position? We might need it in our shaders, to implement certain effects, etc.

Since the difference between them is small, we can just pick a random eye and use its position as the unified camera world position. Or we can take their average:

```swift
let cameraWorldPosition = (leftEyeWorldPosition + rightEyeWorldPosition) * 0.5
```

I use this approach in my code.

> **_NOTE_** Using this approach breaks in Xcode's Apple Vision simulator! It renders the scene for just the left eye. You will need to use the `#if targetEnvironment(simulator)` preprocessor directive to use only the `leftEyeWorldPosition` when running your code in the simulator.

### Apple Vision Simulator

First of all, the Simulator renders your scene only for the left eye. It simply ignores the right eye. All of your vertex amplification code will work just fine, but the second vertex amplification will be ignored.

Secondly, it also lacks some features (which is the case when simulating other Apple hardware as well). MSAA for example is not allowed so you will need to use the `#if targetEnvironment(simulator)` directive and implement two code paths for with MSAA and without.
