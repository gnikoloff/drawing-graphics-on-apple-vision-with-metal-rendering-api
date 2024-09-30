# Rendering 3D graphics on Apple Vision with the Metal API

1. Introduction
2. Dissecting a Frame of RAYQUEST
3. `Metal` API
4. `Compositor Services` API
5. Stereoscoping Rendering
   1. Configuring a `CompositorLayer` for rendering at initialization time
      1. Organising the Metal textures used for presenting the rendered content
      2. Variable Rate Rasterization (Foveation)
   3. Vertex Amplification
   4. Computing the View Matrices for Each Eye
6. Pre-Rendering Tasks
   1. Capturing user input via ARKit
   2. Compute
   3. Animation / Tweening
   4. Frame Prediction
      1. Querying the Next Frame
      2. Waiting Until Optimal Rendering Time
      3. Frame Submission
7. Base / Forward MSAA Pass
   1. Opaque Objects
   2. Skybox
   3. Transparent Objects
   4. Resolving MSAA Texture
8. Bloom Pass
9. Composite Pass
10. Passthrough Rendering

## Introduction

Apple Vision Pro has been out for 7 months at the time of writing this article. Many games have been released for it since then and more and more game developers enter this niche. When it comes to rendering, most developers jump to an existing game engine such as Unity or use Apple's high level rendering APIs such as RealityKit. However, a third way exists, one that has been there since visionOS 1.0 - rolling your own rendering engine using the Metal API. While challenging, this method is incredibly rewarding and gives you complete control over the entire rendering pipeline down to each individual byte and command submitted to the GPU.

> **_NOTE:_** visionOS 2.0 allows us to render graphics via the Metal API and overlay (composite) them in **mixed** mode with the user's surroundings caputed by the device cameras' feed. This article focuses on writing Metal apps for fully **immersive** mode, although passthrough rendering will be mentioned at the end.

## Dissecting a Frame of RAYQUEST

[RAYQUEST](https://rayquestgame.com/) is my first game published on Apple Vision. It utilises Compositor Services and Metal to deliver advanced graphics at 4K resolution at 90 frames per second. This article will take one random frame of it captured during gameplay and show you how is it rendered by taking you on a trip through the complete rendering pipeline.

So here is the frame we will examine:

![Random frame captured during gameplay of RAYQUEST](rayquest-random-frame.png)

> **_NOTE:_** Before continuing, I would like to mention that aside from the rendering engine capabilities and techniques covered in this article, a lot of the boilerplate, setup code and theory has been covered in this great [official example](https://developer.apple.com/documentation/compositorservices/drawing_fully_immersive_content_using_metal) by Apple. I strongly suggest it as an accompanying reading to this article.

## Metal API

Taken directly from Apple's official website:

> Metal is a modern, tightly integrated graphics and compute API coupled with a powerful shading language that is designed and optimized for Apple platforms. Its low-overhead model gives you direct control over each task the GPU performs, enabling you to maximize the efficiency of your graphics and compute software. Metal also includes an unparalleled suite of GPU profiling and debugging tools to help you improve performance and graphics quality.

I will not focus too much on the intristics of Metal in this article, however will mention that the API is mature, well documented and **incredibly nice** to work with. It is more explicit than an API such as OpenGL ES, there is more planning involved in setting up your rendering pipeline, but is still very approachable and more beginner friendly then, say, Vulkan or DirectX12. Furthermore, Xcode's built-in Metal profiler and debugger are hands down the **best** tools to debug and inspect computer graphics I have seen so far.

## Compositor Services

Compositor Services is visionOS-specific API that lets you draw directly to the Apple Vision Pro displays (there is one for each eye). It provides a bridge between your SwiftUI code and your Metal rendering engine code by giving you a layer, which contains the Metal types, textures, and other information you need. This layer also provides timing information to help you manage your app’s rendering loop and deliver frames of content in a timely manner.

## Stereoscoping Rendering

At the app's initialization time, Compositor Services will allow us to configure a `LayerRenderer` object that it will create for us to be used for rendering on Apple Vision. It is up to us to configure the texture layouts, pixel formats used, and other rendering options. If we don’t provide a custom configuration, Compositor Services uses a set of default configuration values.

First thing we need to realize is that unlike traditional apps where we render our graphics to a single device screen, be it on a MacBook or iPhone, on Apple Vision we need to render to two displays - one for the left eye and one for the right eye. That means we need a set of textures for each eye's view or a single texture big enough to accommodate the views of both eyes. Let's take a look at organising the textures' layout.

### Configuring a `CompositorLayer` for rendering at initialization time

In our scene creation code, we need to pass a type that adopts `CompositorLayerConfiguration` as a parameter to the scene content. The system will then use that information to configure the Metal textures we will use for presenting the rendered content to both of the displays on Apple Vision. Here is how this looks in code:

```swift
struct ContentStageConfiguration: CompositorLayerConfiguration {
  func makeConfiguration(capabilities: LayerRenderer.Capabilities, configuration: inout LayerRenderer.Configuration) {
      // specify the formats for both the color and depth output textures that Apple Vision will create for us
      configuration.depthFormat = .depth32Float
      configuration.colorFormat = .bgra8Unorm_srgb

      // TODO: configure the rest of 
  }
}
```

#### Organising the Metal textures used for presenting the rendered content

We have three options when it comes to the organization of the textures' layout we use for drawing:

1. `LayerRenderer.Layout.dedicated` - A layout that assigns a separate texture to each rendered view.
2. `LayerRenderer.Layout.shared` - A layout that uses a single texture to store the content for all rendered views.
3. `LayerRenderer.Layout.layered` - A layout that specifies each view’s content as a slice of a single 3D texture with two slices total.

Which one should you use? Ideally `.shared` or `.layered` as having one texture to manage results in fewer things to keep track of, less commands to submit and less GPU context switches. I have seen people out there using two separate textures (`.dedicated`) but I don't see why. Apple official examples use `.layered`.
