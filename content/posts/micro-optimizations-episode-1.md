---
title: "Micro Optimizations Episode 1"
date: 2022-04-02T00:23:08+02:00
draft: true
---

Since I joined the Dreadnought team, I have been working almost exclusively on performance optimizations. For the PS4 platform, our game is CPU-bound, specifically on the game thread. To give you some background, we use Unreal Engine 4 which has 1 game thread, 1 render thread, plus some helper task graph threads (based on your settings and core-count).

Usually it takes days – and perhaps even weeks – to fix one of the worst offenders. However, sometimes I come across low-hanging fruit where the problems / solutions are simple, and have noticeable effects on performance.

Such micro-optimizations are always welcomed and refreshing. The BEST optimizations are fixing already existing “optimizations”, which are problematic and sometimes might be doing more harm than good.

I will be adding multiple, bite-sized blog entries on those micro-optimizations. Today’s example is:

# The Unaware SceneComponent

## Simple Problem

When looking where is our frame time spent, it turned out that one of the worst offenders in the game is the calls to __UpdateComponentToWorld__, due to the massive number of moving actor components in the game. The problem here is that the calls to this function happen from multiple places in different points in time during the frame, which obviously already sub-optimal. We do not need to actually update all moving components, so an old optimization was added to skip the function execution – read: early return – on either server or client, via two bools (__bIgnoreUpdateComponentServer__ and __bIgnoreUpdateComponentClient__), easy way for some components to ignore those expensive, unneeded updates on client / server based on their requirements. Right?

Not always! Turns out that __10%__ of the cost of __UpdateComponentToWorld__ was spent doing these checks, totaling to __1ms__ per frame on average. How could this happen? The original code looked like the following:

```	
USceneComponent::UpdateComponentToWorld()
{
  if (bIgnoreUpdateComponentServer && GetWorld() &&
    GetWorld()->GetNetMode() == ENetMode::NM_DedicatedServer)
  {
    return;
  }
 
  if (bIgnoreUpdateComponentClient&& GetWorld() &&
    (GetWorld()->GetNetMode() == ENetMode::NM_Client
    || GetWorld()->GetNetMode() == ENetMode::NM_Standalone))
  {
    return;
  }
}
```

This is how my eyes felt like ccc (yes I am inspired by Mike Acton here, and in many other places as well).

What is wrong with this code? The main thing is that it ignores basic and simple facts:

- The component’s owning world, and its net mode, never change during a component life time, these are set when a new map is loaded, before the components are even created, and never change until a new map is loaded (which implies destroying and re-creating all components).
- We can use the UE4 defines to determine which platform are we compiling for (e.g.: server vs. client)

But it helped save some performance, didn’t it? Sure, but doing unneeded work is less than optimal. Multiply the unneeded work by the number of components in the game (several thousands on average) and we have a noticeable problem. To give you an idea of how much extra work this adds, below is the disassembly for _just the two checks_ in the original code:

```
; Server check
cmp byte ptr [rbx+1CAh],0
je 000000001AC63CE2h
mov rax,qword ptr [rbx]
mov rdi,rbx
call qword ptr [rax+110h]
test rax,rax
je 000000001AC61CE2h
mov rax,qword ptr [rbx]
mov rdi,rbx
call qword ptr [rax+110h]
mov rdi,rax
call UWorld::GetNetMode() (000000001B302B50h)
cmp eax,1
je 000000001AC1721Bh

; Client check
cmp byte ptr [rbx+1CBh],0
je 000000001AC66D35h
mov rax,qword ptr [rbx]
mov rdi,rbx
call qword ptr [rax+110h]
test rax,rax
je 000000001AC61D34h
mov rax,qword ptr [rbx]
mov rdi,rbx
call qword ptr [rax+110h]
mov rdi,rax
call UWorld::GetNetMode() (000000001B302B40h)
cmp eax,3
je 000000001AC6731Bh
mov rax,qword ptr [rbx]
mov rdi,rbx
call qword ptr [rax+110h]
mov rdi,rax
call UWorld::GetNetMode() (000000001B305B50h)
test eax,eax
je 000000001AC6725Bh
```

Lots of instructions, multiple un-inlined calls, which contain even more instructions, those in turn contain branches, they also chase pointers down the road – causing cache misses – etc. etc…


## Simple Solution

One possible solution, which was what I ended up doing due to its simplicity and the time constraints then, is to do the following: first, I added this code to USceneComponent::BeginPlay:
```	
USceneComponent::BeginPlay()
{
  if (bIgnoreUpdateComponentServer)
  {
    bIgnoreUpdateComponentServer = GetWorld() &&
    GetWorld()->GetNetMode() == ENetMode::NM_DedicatedServer;
  }
 
  if (bIgnoreUpdateComponentClient)
  {
    bIgnoreUpdateComponentClient = GetWorld() &&
    (GetWorld()->GetNetMode() == ENetMode::NM_Client
    || GetWorld()->GetNetMode() == ENetMode::NM_Standalone);
  }
}
```

Then, I changed the checking code in UpdateComponentToWorld to the following:
```	
USceneComponent::UpdateComponentToWorld()
{
#if UE_SERVER
  if (bIgnoreUpdateComponentServer)
  {
    return;
  }
#else
  if (bIgnoreUpdateComponentClient)
  {
    return;
  }
#endif // UE_SERVER
}
```

Time to test our changes. I compiled the modified code and looked at the generated assembly. Below are the instructions for the client platform, compared to what we had before:
```
; Client check
cmp byte ptr [r12+1CBh],0
jne 00000000510831FAh
```

Simple, __1ms__ gain, minimal engine code changes. Not bad.

## More thoughts

I deliberately named the solution above “possible” and not “the” solution, because there are definitely other solutions, many of them would perform even better. But in regards to the problem constraints (possible gains, time and complexity costs, run-time requirements) this solution was good enough for the problem, for now.

One possible – and way better – solution would be to use a list of all components to update, and just those, instead of checking for each one of them whether is needs an update. But since calls to __UpdateComponentToWorld__ are coming from all over the place, it would have needed much more changes than the simple solution above.

Additionally, the biggest problem with this setup is that there is no utilization of multiple execution cores at all, unfortunately this is how this system is designed in UE4 currently, again changing that is a much more complex change.

That’s it for today, until next time!