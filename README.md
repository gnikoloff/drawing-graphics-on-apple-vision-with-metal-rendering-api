# Rendering 3D graphics on Apple Vision with the Metal API

1. Introduction
   1. `Metal` API
   2. `Compositor Services` API
3. Dissecting a Frame of RAYQUEST
4. Stereoscoping Rendering
   1. Configuring a `CompositorLayer` for rendering at initialization time
      1. Variable Rate Rasterization (Foveation)
      2. Organising the Metal textures used for presenting the rendered content
   3. Vertex Amplification
      1. Configuring a `MTLRenderPipelineDescriptor` with Support for Vertex Amplification
      2. Encoding and submitting a render pass to the GPU
      3. Enabling Vertex Amplification for a Render Pass
      4. Computing the View and Projection Matrices for Each Eye
      5. Adding Vertex Amplification to our Vertex Shader
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

Apple Vision Pro has been out for 7 months at the time of writing this article. Many games have been released for it since then and more and more game developers enter this niche. When it comes to rendering, most developers jump to an existing game engine such as Unity or use Apple's high level rendering APIs such as RealityKit. However, a third way exists, one that has been there since visionOS 1.0 - rolling your own rendering engine using the Metal API. While challenging, this method is incredibly rewarding and gives you complete control over the entire rendering pipeline down to each individual byte and command submitted to the GPU.

> **_NOTE:_** visionOS 2.0 allows us to render graphics via the Metal API and overlay (composite) them in **mixed** mode with the user's surroundings caputed by the device cameras' feed. This article focuses on writing Metal apps for fully **immersive** mode, although passthrough rendering will be mentioned at the end.

### Metal API

Taken directly from Apple's official website:

> Metal is a modern, tightly integrated graphics and compute API coupled with a powerful shading language that is designed and optimized for Apple platforms. Its low-overhead model gives you direct control over each task the GPU performs, enabling you to maximize the efficiency of your graphics and compute software. Metal also includes an unparalleled suite of GPU profiling and debugging tools to help you improve performance and graphics quality.

I will not focus too much on the intristics of Metal in this article, however will mention that the API is mature, well documented and **incredibly nice** to work with. It is more explicit than an API such as OpenGL ES, there is more planning involved in setting up your rendering pipeline, but is still very approachable and more beginner friendly then, say, Vulkan or DirectX12. Furthermore, Xcode's built-in Metal profiler and debugger are hands down the **best** tools to debug and inspect computer graphics I have seen so far.

### Compositor Services

Compositor Services is visionOS-specific API that lets you draw directly to the Apple Vision Pro displays (there is one for each eye). It provides a bridge between your SwiftUI code and your Metal rendering engine code by giving you a layer, which contains the Metal types, textures, and other information you need. This layer also provides timing information to help you manage your app’s rendering loop and deliver frames of content in a timely manner.

## Dissecting a Frame of RAYQUEST

[RAYQUEST](https://rayquestgame.com/) is my first game published on Apple Vision. It utilises Compositor Services and Metal to deliver advanced graphics at 4K resolution at 90 frames per second. This article will take one random frame of it captured during gameplay and show you how is it rendered by taking you on a trip through the complete rendering pipeline.

So here is the frame we will examine:

![Random frame captured during gameplay of RAYQUEST](rayquest-random-frame.png)

> **_NOTE:_** Before continuing, I would like to mention that aside from the rendering engine capabilities and techniques covered in this article, a lot of the boilerplate, setup code and theory has been covered in this great [official example](https://developer.apple.com/documentation/compositorservices/drawing_fully_immersive_content_using_metal) by Apple. I strongly suggest it as an accompanying reading to this article.

## Stereoscoping Rendering

At the app's initialization time, Compositor Services will allow us to configure a `LayerRenderer` object that will be created for us behind the scenes to be used for rendering on Apple Vision. It is up to us to configure the texture layouts, pixel formats used, and other rendering options. If we don’t provide a custom configuration, Compositor Services uses a set of default configuration values.

First thing we need to realize is that unlike traditional apps where we render our graphics to a single device screen, be it on a MacBook or iPhone, on Apple Vision we need to render to two displays - one for the left eye and one for the right eye. That means we need a set of textures for each eye's view or a single texture big enough to accommodate the views of both eyes. Let's take a look at organising the textures' layout.

### Configuring a `CompositorLayer` for rendering at initialization time

In our scene creation code, we need to pass a type that adopts `CompositorLayerConfiguration` as a parameter to our scene content. The system will then use that configuration to create a `LayerRenderer` that will hold information such as the pixel formats of the final color and depth buffers, how the textures used to present the rendered content to Apple Vision's displays are organised, whether foveation is enabled and so on. Here is boilerplate code:

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
struct TestingApp: App {
  var body: some Scene {
    WindowGroup {
      ContentView()
    }
    ImmersiveSpace(id: "ImmersiveSpace") {
      // Supply the CompositorLayerConfiguration we created to the CompositorLayer.
      CompositorLayer(configuration: ContentStageConfiguration()) { layerRenderer in
         // The system uses the CompositorLayerConfiguration we supplied to create a LayerRenderer we can use for rendering.
      }
    }
  }
}
```

#### Variable Rate Rasterization (Foveation)

Next thing we need to set up is whether to enable support for **foveation** in LayerRenderer. What is foveation you ask? Taken straight from the [Wikipedia article]([https://en.wikipedia.org/wiki/Foveated_imaging](https://en.wikipedia.org/wiki/Foveated_rendering)):

> Foveated rendering is a rendering technique which uses an eye tracker integrated with a virtual reality headset to reduce the rendering workload by greatly reducing the image quality in the peripheral vision (outside of the zone gazed by the fovea)

In other words, foveation allows us to render at a higher resolution the content our eyes gaze directly at and render everything else at a lower resolution. This works exactly like in real life, where you see things you gaze at clearly while everything else in the periphery is blurier and less focused. Apple Vision does eye-tracking and foveation automatically for us (in fact, it is not possible for developers to access the user's gaze **at all** due to security concerns). So we need to setup our LayerRenderer to use it and we will get it for free during rendering. When we render to the LayerRenderer textures, Apple Vision will automatically adjust the resolution to be higher at the regions of the textures we directly gaze at. Here is the previous code that configures the `LayerRenderer`, updated with support for foveation:

```swift
struct ContentStageConfiguration: CompositorLayerConfiguration {
  func makeConfiguration(capabilities: LayerRenderer.Capabilities, configuration: inout LayerRenderer.Configuration) {
      configuration.depthFormat = .depth32Float
      configuration.colorFormat = .bgra8Unorm_srgb

      // Enable foveation
      let foveationEnabled = capabilities.supportsFoveation
      configuration.isFoveationEnabled = foveationEnabled
  }
}
```

#### Organising the Metal textures used for presenting the rendered content

We established we need to render our content as two views to both Apple Vision displays. We have three options when it comes to the organization of the textures' layout we use for drawing:

1. `LayerRenderer.Layout.dedicated` - A layout that assigns a separate texture to each rendered view.
2. `LayerRenderer.Layout.shared` - A layout that uses a single texture to store the content for all rendered views.
3. `LayerRenderer.Layout.layered` - A layout that specifies each view’s content as a slice of a single 3D texture with two slices total.

Which one should you use? Ideally `.shared` or `.layered` as having one texture to manage results in fewer things to keep track of, less commands to submit and less GPU context switches. I have seen people out there using two separate textures (`.dedicated`) but I don't see why. Apple official examples use `.layered`.

Let's update the configuration code once more:

```
struct ContentStageConfiguration: CompositorLayerConfiguration {
  func makeConfiguration(
    capabilities: LayerRenderer.Capabilities,
    configuration: inout LayerRenderer.Configuration) {
      configuration.depthFormat = .depth32Float
      configuration.colorFormat = .bgra8Unorm_srgb

    let foveationEnabled = capabilities.supportsFoveation
    configuration.isFoveationEnabled = foveationEnabled

    let options: LayerRenderer.Capabilities.SupportedLayoutsOptions = foveationEnabled ? [.foveationEnabled] : []
    let supportedLayouts = capabilities.supportedLayouts(options: options)

    configuration.layout = supportedLayouts.contains(.layered) ? .layered : .dedicated
  }
}
```

That takes care of configuring a `LayerRenderer` for rendering our content at initialization time. Let's move on to rendering.

### Vertex Amplification

Imagine we have a triangle we want rendered on Apple Vision. A triangle consists of 3 vertices. If we were to render to a normal display we would submit 3 vertices to the GPU and let it draw them for us. On Apple Vision we have two displays. How do we go about it? A naive way would be to submit two drawing commands:

1. Issue a draw command to render 3 vertices to the left eye display.
2. Issue another draw command to render the same 3 vertices again, this time for the right eye display.

This is not optimal as it doubles the commands needed to be submited to the GPU for rendering. A 3 vertices triangle is fine, but for complex scenes with even moderate amounts of geometry it becomes unwieldly very fast. Thankfully, Metal allows us to submit the 3 vertices once for both displays via a process called **vertex amplification**.

Taken from this great [article](https://developer.apple.com/documentation/metal/render_passes/improving_rendering_performance_with_vertex_amplification) on vertex amplification from Apple:

> With vertex amplification, you can encode drawing commands that process the same vertex multiple times, one per render target.

Does this sound useful? Well it is, because one "render target" from the quote above translates directly to one display on Apple Vision. Two displays for the left and right eyes - two render targets to which we can submit the same 3 vertices once, letting the Metal API "amplify" them for us, for free, with hardware acceleration, and render them to both displays **at the same time**. Vertex amplification is not used only for rendering to both displays on Apple Vision and has it's benefits in graphics techniques such as Cascaded Shadowmaps, where we submit one vertex and render it to multiple "cascades", represented as texture slices, for more adaptive and better looking realtime shadows.

#### Configuring a `MTLRenderPipelineDescriptor` with Support for Vertex Amplification

But back to vertex amplification as means for efficient rendering to both Apple Vision displays. Say we want to render the aformentioned 3 vertices triangle on Apple Vision. In order to render anything, on any Apple device, be it with a traditional display or two displays set-up, we need to create a `MTLRenderPipelineDescriptor` that will hold all of the state needed to render an object in a single render pass. Stuff like the vertex and fragment shaders to use, the color and depth pixel formats to use when rendering, the sample count if we use MSAA and so on. In the case of Apple Vision, we need to explicitly set the `maxVertexAmplificationCount` property when creating our `MTLRenderPipelineDescriptor`:

```swift
let pipelineStateDescriptor = MTLRenderPipelineDescriptor()
pipelineStateDescriptor.vertexFunction = vertexFunction
pipelineStateDescriptor.fragmentFunction = fragmentFunction
pipelineStateDescriptor.maxVertexAmplificationCount = 2
...
```

#### Encoding and submitting a render pass to the GPU

We now have a `MTLRenderPipelineDescriptor` that represents a graphics pipeline configuration with Vertex Amplification enabled. Once a graphics pipeline, represented by `MTLRenderPipelineState`, is created, the function call for rendering this pipeline needs to be encoded into list of per-frame commands to be submitted to the GPU. What are examples of such commands? Imagine we are building a game with two objects and on each frame we do the following operations:

1. Set the clear color before rendering.
2. Set the viewport size.
3. Set the render target we are rendering to.
4. Set `MTLRenderPipelineState` for object A
5. Render object A.
6. Set `MTLRenderPipelineState` for object B
7. Render object B.
9. Finally submit all of the above commands to the GPU

All of these rendering commands represent a **render pass** that happens on each frame while our game is running. This render pass is represented by a `MTLRenderCommandEncoder`. We use this `MTLRenderCommandEncoder` to encode our **rendering** commands from steps 1 to 7 into a `MTLCommandBuffer` which is submitted to the GPU for execution. The GPU will execute each command in correct order, writing the final frame values to the final texture to be presented to the user.

It is important to note that the commands to be encoded in a `MTLCommandBuffer` and submitted to the GPU are not only limited to rendering. We can submit commands to the GPU for general-purpose non-rendering work such as fast number crunching (modern techniques for ML, physics, simulations, etc are all done on the GPU nowadays). However, let's focus only on the rendering commands for now.

#### Enabling Vertex Amplification for a Render Pass

When creating a `MTLRenderCommandEncoder` to submit commands for drawing we need to enable Vertex Amplification for this render pass. This compromises of two steps:

1. Specifying the number of amplifications to create.
2. Specifying view mappings that hold per-output offsets to a specific render target and viewport.
3. Specifying the viewport sizes for each render target

Remember, we are dealing with two render targets on Apple Vision. So the number of amplifications is, of course, 2. The view mappings into each render target depend on our textures' layout we specified when creating the `LayerRenderer` configuration used in Compositor Services above. The render target viewport sizes are also given to us by the `LayerRenderer`. We should never hardcode these values ourselves. Instead, we can query the current frame `Drawable` given to us by Compositor Services and obtain them from it. Here is how this looks in code:

```
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
