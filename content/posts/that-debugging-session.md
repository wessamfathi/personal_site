---
title: "That Debugging Session"
date: 2016-11-06T10:58:15+02:00
draft: false
cover:
    image: "/posts/images/audio_device2.png"
    alt: "That Debugging Session"
    caption: "That Debugging Session"
---

## Disclaimer

This post is not about performance optimizations, but I find it… interesting to say the least, and perhaps you’ll agree!

## What happened?

My colleague and I were investigating a bug which started manifesting lately in Dreadnought, where the game would crash as it goes out of GPU memory on start-up. The problem is, this only happened on one specific machine configuration. To make it even more interesting, this only happened when the game used the DX12 renderer, my colleague – who is a graphics programmer – has also figured out that renaming / removing the start-up movies folder circumvents the issue.

## How did it happen?

Naturally we started looking at how much memory was allocated for the movies, but everything seemed normal. I went ahead and set up remote debugging on two different machine configurations (the crashing one and a normal one). Looking at the crash, the allocation request seemed innocent, it was a bit big but since the allocation was for an allocator’s buffer it should be fine. Some time later I discovered that there were also several hundred existing allocators, each having a buffer of the same size. Debugging continues, with added logs as well, later revealing that > 860 new allocator buffers are created before the crash, IN THE SAME FRAME.

Let me explain some background on the logic behind the allocations, UE4 uses what is called a multi allocator in DX12 mode to upload resources etc. to the GPU, this multi allocator creates a new allocator if none of the already existing allocators were able to serve the request. It seams innocent as the code works as intended, creating new buffers when needed. Finally, the allocator buffers get freed either manually or when the frame ends. However, in comparison with the other “normal” machine configuration, this one creates a maximum of 8 allocators per frame during the game’s lifetime. So something apparently went very wrong.

Turns out all those allocations are coming from the same callstack, from an infinite loop where the engine waits for the movie player to finish streaming all movies, and while doing so it forcibly ticks / renders Slate. In turn, Slate’s tick / render loop creates resources which lead to creating the buffers. Apparently this streaming never finishes for some  reason. On the other non-crashing machine, this code path is never taken as the engine never waits for movie streaming to finish.

## Why did it happen?

The engine initialization paths were in fact different on the two machines – whose configurations barely differ, and this should never be the case. Looking at when did the initialization paths diverge during the audio device initialization, if the audio device initialization fails then the code path to prepare and stream the movies is not followed as a side effect – read: bug. And the audio device initialization failed because of a hardware problem on this very specific machine! That’s right, the underlying issue was actually a hardware fault, but in a totally unrelated one to the crash happening, no audio device causing an out of GPU memory crash? Come on…

*__Game development is fun!__*