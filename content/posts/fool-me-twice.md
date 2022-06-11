---
title: "Fool Me Twice"
date: 2022-04-15T22:20:19+02:00
draft: false
cover:
    image: "/posts/images/cpu-profiler-window.png"
    alt: "Unity CPU Profiler Window"
---
 
Nothing's better than gaining 160ms per frame just by toggling a few checkboxes. Well, maybe gaining 200ms is better, but that's not the point.
 
In December 2021, I reviewed a first-person shooter mobile game from a well-known franchise. The game is developed for iOS and Android and uses Unity's new [Universal Rendering Pipeline (URP)](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@12.1/manual/index.html).
 
The game suffered from long frame times on low- and mid-end Android devices; on average, the frame time was 200ms, which translates to 5 frames per second. I have seen PowerPoint presentations that ran faster. Clearly, lots of optimization work was needed, or was it?
 
# Duplicate rendering
 
I used the [Unity CPU Profiler](https://docs.unity.cn/2021.3/Documentation/Manual/ProfilerCPU.html) and noticed that the game waits on the GPU to finish rendering for about 180ms per frame, so I started looking into the GPU workload, and how I can reduce it. The game seemed simple, as it only rendered a basic terrain and a few characters from a top-down view. Nothing looked so complex that it needed 180ms to render.
 
To understand what is happening on the device, I took a frame capture of the game running on Android using [RenderDoc](https://renderdoc.org/) and noticed a section labeled (Default) after the main opaque rendering pass. All draw calls under (Default) except for one draw call were duplicates of draw calls in the previous opaque rendering pass. The unique draw call was a black rectangle to cover a part of the screen.
 
The game rendered everything twice, the scene geometry, props, trees, the player, his weapon, the NPCs, etc. The dev team had no idea why this behavior was happening. I checked the URP asset settings and finally found the reason for the duplicate rendering.
 
It was a wrong setting in the forward renderer data asset *UniversalRenderPipelineAsset_Renderer*.
 
Unity uses this data asset to control various parts of the URP forward renderer. Under the *Renderer Features* section, I found that someone on the original dev team added a renderer feature with the name (Default) a while ago. They meant to render a black bar on the bottom of the screen to use for game controls and ads. However, they forgot to set up the *Opaque* and *Transparent Layer Masks*. The (Default) pass was set to render AfterRenderingOpaques, but its *Opaque Layer Mask* property included the same layers rendered by default under the forward renderer’s Opaque Layer Mask property. As a result, the game rendered all objects in both layers twice, wasting GPU time while having no noticeable effect on the screen.
 
Turning off the duplicate layers in the Default renderer pass resulted in rendering only the required black rectangle and reduced GPU frame time from 180ms to 90ms. Great work, but still not at the target frame rate.
 
# GPU workload
 
I checked the project settings and noticed that *Compute Skinning* was enabled, which moves skinned mesh computations to the GPU. While having Compute Skinning enabled could reduce CPU time, it almost always increases GPU time. It is usually better to disable Compute Skinning on low-end platforms, which is what I did.
 
Additionally, I noticed that *Graphics Jobs* were disabled in the player settings. Enabling Graphics Jobs uses worker threads to use the worker thread and submit draw calls faster to the GPU, so the latter doesn't wait for commands.
 
Toggling both options reduced the frame time from 90ms to 50ms. We were getting closer to our goal, and all we had to do was to change a few checkboxes.
 
# Final touches
 
The game used the device's native resolution, which could be an overkill for lower-end devices with a weak GPU and a large screen with a high pixel count. I used the URP *Render Scale* feature to render the game at 0.75 of the native resolution while keeping the UI at the native resolution. This change reduced the GPU frame time from 50ms to 35ms.
 
Finally, the team left a few expensive operations enabled on all devices, for example, Soft Shadows and Anisotropic Filtering (forced on for all textures). Disabling those options for lower-end mobile devices reduced the GPU time from 35ms to 20ms.
 
As a result of previous optimizations, the game ran at a solid 30FPS on the target low-end device and could reach 60FPS on some frames.
 
It was a fun optimizing experience for me. Also, it was my first time using RenderDoc on Android, and I was surprised by how easy it was to get it to work.
