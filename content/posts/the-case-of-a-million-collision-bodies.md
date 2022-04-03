---
title: "The Case of a Million Collision Bodies"
date: 2018-04-17T11:34:13+02:00
draft: false
cover:
    image: "/posts/images/million_boxes.jpg"
    alt: "The Case of a Million Collision Bodies"
    caption: "The Case of a Million Collision Bodies"
---

Recently I was looking into the performance of a game’s dedicated server, one problematic area was PhysX performance. To get an idea of how it was like, the average PhysX frame time for a 32-players game was around 80ms with spikes of over 400ms. A quick round of removing unneeded meshes helped, but we were still far from calling it a day.

# Overview

To understand the context, the project is an open-world survival game with a large map, containing a lot of meshes with collisions. The game is [Memories of Mars](https://www.memoriesofmars.com/), developed by Limbic Entertainment and published by 505 Games. Memories of Mars is a pretty cool game where you can join 64-players games with player-made buildings, the game is more than that so go check out the website 😉

YAGER provided assistance with profiling and optimizing performance of the game’s dedicated server. To start out I needed to understand why is PhysX performing slow, although Unreal Engine’s stat system is strong, it did not give me enough information to determine the root cause(s) of the problem. I needed to get more detailed information, more in-depth profiles.

My favorite profilers are the ones built for latest-gen consoles, PIX for Xbox One and Razor for PlayStation 4. Unfortunately those were not available as the dedicated server runs on PC. By this time Microsoft has released PIX on Windows and I was quite eager to try it out, so I thought I’d give it a shot (PIX for Windows is available for free [here](https://blogs.msdn.microsoft.com/pix/download/).

# Profiling UE4 using PIX

I went ahead and downloaded and set up PIX, the next step was instrumenting PhysX SDK code to get more insight on what was going on, luckily the SDK source code is freely available with UE4 at __"\Engine\Source\ThirdParty\PhysX\"__"

First step was taking some [Function Summary Captures](https://blogs.msdn.microsoft.com/pix/function-summary-captures/) through PIX, digging deeper into the PhysX task quickly revealed that the high cost is due to a sheer number of collision bodies.

![total_phys](/posts/images/total_phys.png)

The profile snippet below shows around 40k allocations + 9.5k reallocations happening during a physics update frame, again due to the large number of collision bodies.

![count_phys](/posts/images/count_phys.png)

The physics frame cost is all spent in the broad phase part of the collision detection algorithm, as shown below.

![tasks_phys](/posts/images/tasks_phys.png)

## PIX Markers and Events

It was time to understand more, using [PIX events and markers](https://blogs.msdn.microsoft.com/pix/winpixeventruntime/) along with [Timing Captures](https://blogs.msdn.microsoft.com/pix/timing-captures/) is a great way to get detailed information of where is your performance going. I went ahead and modified __"\Engine\Source\ThirdParty\PhysX\PhysX_3.4\Source\LowLevelAABB\src\BpBroadPhaseSap"__ with the following code after the includes:

```	
#define WINDOWS_LEAN_AND_MEAN
#include "windows.h"
#define USE_PIX 1
#include "WinPixEventRuntime/pix3.h"
#pragma comment(lib, "WinPixEventRuntime.lib")
```


Added the following line to the beginning of __BroadPhaseSap::setUpdateData__

```
PIXScopedEvent(PIX_COLOR(120, 14, 27), "Sap::setUpdateData");
```

And the next 2 lines to the beginning of __BroadPhaseSap::postUpdate__

```	
PIXScopedEvent(PIX_COLOR(0, 0, 128), "SapPostUpdate");
PIXSetMarker(PIX_COLOR(0, 128, 0), "Context: %l", mContextID);
```

Inside __BroadPhaseSap::update__ I added the following __PIXScopedEvent__ calls:
```	
void BroadPhaseSap::update(PxBaseTask* continuation)
{
    PX_UNUSED(continuation);
 
    PX_PROFILE_ZONE("BroadPhase.SapUpdate", mContextID);
 
    PIXScopedEvent(PIX_COLOR(255, 0, 0), "SapUpdate");
    {
        PIXScopedEvent(PIX_COLOR(128, 0, 0), "batchRemove");
        batchRemove();
    }
 
    //Check that the overlap pairs per axis have been reset.
    PX_ASSERT(0==mBatchUpdateTasks[0].getPairsSize());
    PX_ASSERT(0==mBatchUpdateTasks[1].getPairsSize());
    PX_ASSERT(0==mBatchUpdateTasks[2].getPairsSize());
 
    {
        PIXScopedEvent(PIX_COLOR(64, 64, 0), "Batch[0]");
        mBatchUpdateTasks[0].runInternal();
    }
 
    {
        PIXScopedEvent(PIX_COLOR(128, 128, 0), "Batch[1]");
        mBatchUpdateTasks[1].runInternal();
    }
 
    {
        PIXScopedEvent(PIX_COLOR(192, 192, 0), "Batch[2]");
        mBatchUpdateTasks[2].runInternal();
    }
}
```

Added the following code to the beginning of __BroadPhaseSap::batchCreate__
```	
PIXScopedEvent(PIX_COLOR(0, 0, 128), "batchCreate");
PIXSetMarker(PIX_COLOR(0, 128, 0), "Size: %d", mCreatedSize);
```

Finally, I heavily modified __BroadPhaseSap::batchRemove__ to include the following PIX markers & events (huge function, apologies in advance):
```	
void BroadPhaseSap::batchRemove()
{
    PIXSetMarker(PIX_COLOR(0, 255, 0), "Boxes: %d", mBoxesSize);
    if(!mRemovedSize)   return; // Early-exit if no object has been removed
    PIXSetMarker(PIX_COLOR(255, 255, 0), "Removed: %d", mRemovedSize);
 
    //The box count is incremented when boxes are added to the create list but these boxes
    //haven't yet been added to the pair manager or the sorted axis lists.  We need to 
    //pretend that the box count is the value it was when the bp was last updated.
    //Then, at the end, we need to set the box count to the number that includes the boxes
    //in the create list and subtract off the boxes that have been removed.
    PxU32 currBoxesSize=mBoxesSize;
    mBoxesSize=mBoxesSizePrev;
 
    {
        PIXScopedEvent(PIX_COLOR(0, 255, 255), "bigLoop");
        for (PxU32 Axis = 0; Axis < 3; Axis++)
        {
            ValType* const BaseEPValue = mEndPointValues[Axis];
            BpHandle* const BaseEPData = mEndPointDatas[Axis];
            PxU32 MinMinIndex = PX_MAX_U32;
            {
                PIXScopedEvent(PIX_COLOR(0, 255, 0), "removedLoop");
                for (PxU32 i = 0; i < mRemovedSize; i++)
                {
                    PX_ASSERT(mRemoved[i] < mBoxesCapacity);
 
                    const PxU32 MinIndex = mBoxEndPts[Axis][mRemoved[i]].mMinMax[0];
                    PX_ASSERT(MinIndex < mBoxesCapacity * 2 + 2);
                    PX_ASSERT(getOwner(BaseEPData[MinIndex]) == mRemoved[i]);
 
                    const PxU32 MaxIndex = mBoxEndPts[Axis][mRemoved[i]].mMinMax[1];
                    PX_ASSERT(MaxIndex < mBoxesCapacity * 2 + 2);
                    PX_ASSERT(getOwner(BaseEPData[MaxIndex]) == mRemoved[i]);
 
                    PX_ASSERT(MinIndex < MaxIndex);
 
                    BaseEPData[MinIndex] = PX_REMOVED_BP_HANDLE;
                    BaseEPData[MaxIndex] = PX_REMOVED_BP_HANDLE;
 
                    if (MinIndex < MinMinIndex)
                        MinMinIndex = MinIndex;
                }
            }
 
            PxU32 ReadIndex = MinMinIndex;
            PxU32 DestIndex = MinMinIndex;
            const PxU32 Limit = mBoxesSize * 2 + NUM_SENTINELS;
 
            {
                PIXScopedEvent(PIX_COLOR((BYTE) Axis * 127, 92, 68), "whileLoop");
                while (ReadIndex != Limit)
                {
                    Ps::prefetchLine(&BaseEPData[ReadIndex], 128);
                    while (ReadIndex != Limit && BaseEPData[ReadIndex] == PX_REMOVED_BP_HANDLE)
                    {
                        Ps::prefetchLine(&BaseEPData[ReadIndex], 128);
                        ReadIndex++;
                    }
                    if (ReadIndex != Limit)
                    {
                        if (ReadIndex != DestIndex)
                        {
                            BaseEPValue[DestIndex] = BaseEPValue[ReadIndex];
                            BaseEPData[DestIndex] = BaseEPData[ReadIndex];
                            PX_ASSERT(BaseEPData[DestIndex] != PX_REMOVED_BP_HANDLE);
                            if (!isSentinel(BaseEPData[DestIndex]))
                            {
                                BpHandle BoxOwner = getOwner(BaseEPData[DestIndex]);
                                PX_ASSERT(BoxOwner < mBoxesCapacity);
                                mBoxEndPts[Axis][BoxOwner].mMinMax[isMax(BaseEPData[DestIndex])] = BpHandle(DestIndex);
                            }
                        }
                        DestIndex++;
                        ReadIndex++;
                    }
                }
            }
        }
    }
 
    {
        PIXScopedEvent(PIX_COLOR(0, 255, 128), "smallLoop");
        for (PxU32 i = 0; i < mRemovedSize; i++)
        {
            const PxU32 handle = mRemoved[i];
            mBoxEndPts[0][handle].mMinMax[0] = PX_REMOVED_BP_HANDLE;
            mBoxEndPts[0][handle].mMinMax[1] = PX_REMOVED_BP_HANDLE;
            mBoxEndPts[1][handle].mMinMax[0] = PX_REMOVED_BP_HANDLE;
            mBoxEndPts[1][handle].mMinMax[1] = PX_REMOVED_BP_HANDLE;
            mBoxEndPts[2][handle].mMinMax[0] = PX_REMOVED_BP_HANDLE;
            mBoxEndPts[2][handle].mMinMax[1] = PX_REMOVED_BP_HANDLE;
        }
    }
 
    {
        PIXScopedEvent(PIX_COLOR(0, 128, 255), "bitmapSet");
        const PxU32 bitmapWordCount = 1 + (mBoxesCapacity >> 5);
        Cm::TmpMem<PxU32, 128> bitmapWords(bitmapWordCount);
        PxMemZero(bitmapWords.getBase(), sizeof(PxU32)*bitmapWordCount);
        Cm::BitMap bitmap;
        bitmap.setWords(bitmapWords.getBase(), bitmapWordCount);
        for (PxU32 i = 0; i < mRemovedSize; i++)
        {
            PxU32 Index = mRemoved[i];
            PX_ASSERT(Index < mBoxesCapacity);
            PX_ASSERT(0 == bitmap.test(Index));
            bitmap.set(Index);
        }
 
        PIXScopedEvent(PIX_COLOR(0, 64, 128), "RemovePairs");
        mPairs.RemovePairs(bitmap);
    }
 
    mBoxesSize=currBoxesSize;
    mBoxesSize-=mRemovedSize;
    mBoxesSizePrev=mBoxesSize-mCreatedSize;
}
```

## Profiling Results

In this build the normal average time for broad phase SAP update is 5ms (smaller marker to the left), with regular spikes of up to 33 ms (huge marker on the right) as shown in the screenshot below.

![sap_timeline](/posts/images/sap_timeline.png)

To figure out the cause of the spikes, note the results of profiling __BroadPhaseSap::update__

![sap_detail](/posts/images/sap_detail.png)

For a 33 ms spike, most of the cost came from __BroadPhaseSap::batchRemove__, this function removes colliding boxes which are not colliding anymore. It does so by looping through the list of endPoints to remove the collision, the example above shows how the large number of boxes (930,479) is resulting in a very high cost for just iterating over the list, as 3 while loops (one per Axis) go over the list of all boxes * 2, this means 3 loops of up to 2 million iterations each in worst case.

Additionally, there are cases where the spikes are caused by broad phase SAP postUpdate, highest spike recorded was 39 ms.

![postsap_timeline](/posts/images/postsap_timeline.png)

Further analysis of postUpdate shows source of the slowdown:

![postsap_detail](/posts/images/postsap_detail.png)

For a 39 ms spike, most of the cost came from __BroadPhaseSap::batchCreate__, this function add colliding boxes which were not colliding before. It inserts the new endPoints for the collisions and performs pruning over the collisions, the example above shows how the large number of boxes (930,479) is again resulting in a very high cost for both the insertion (25 ms) and the pruning (14 ms) stages.

Sometimes both spikes happen in the same frame (which is expected to happen even more when more interesting things are happening in the game), and you get the cost of both combined. In this example a spike of 28 ms for __SapUpdate__ was immediately followed by a 39 ms spike for __SapPostUpdate__.

![doublesap_timeline](/posts/images/doublesap_timeline.png)

Since it was clear the high physics cost was due to a large number of collisions, the next step was to list and optimize the worst offenders. In some cases it was pretty straightforward and in other cases it was tricky. Eventually physics performance got much better but it was still not there yet, for a 64-players game with a dedicated server that should run at 30fps, physics performance was simply not there yet. I had to figure out another way to get more performance without having to retouch all the content, or having to reduce how much content is in the game.

There was yet another option to try out, a new collision algorithm was introduced in PhysX 3.3 release, Multi-box Prune (MBP for short) was introduced as an alternative to SAP.


# SAP vs. MBP

- SAP (Sweep-and-Prune): used by default, good performance with many sleeping objects, performance drops significantly when many objects are moving or when many objects are added / deleted from broad-phase
- MBP (Multi-box-Prune): new algorithm added in PhysX SDK 3.3, has higher base cost but does not suffer from the same issues as with SAP when many objects are moving or when many objects are inserted

[Source](http://docs.nvidia.com/gameworks/content/gameworkslibrary/physx/guide/Manual/RigidBodyCollision.html#broad-phase-algorithms)

Let’s repeat the same profiling scenario with 1, 10, 16, and 32 players. We will do two runs per scenario, once using SAP, and the other time using MBP.


## Unreal Profiles

__1 Player__

SAP has a lower base cost (Min: 2.0 ms), but has much higher (Max: 17.4 ms) and more frequent (Avg: 3.9 ms) spikes. Note how most PhysX spikes coincide with game thread spikes.

![SAP](/posts/images/sap.webp)

MBP has a higher base cost (Min: 2.8 ms), however it has much smaller spikes (Max: 5.2 ms), and is more stable (Avg: 3.0 ms) than SAP. The remaining game thread spikes are not related to PhysX in this profile.

![MBP](/posts/images/mbp.webp)

__10 Players__

The same profiling scenario was repeated with 10 clients connected to see how PhysX performance scales with more and more players connected and moving around. SAP does not scale very well with a high base cost (Min: 40 ms), huge spikes (Max: 215 ms) and more of them (Avg: 138 ms). PhysX cost directly relates to all game thread spikes, and is responsible for more than 50% of the total cost on average.

![SAP_STAT](/posts/images/sap_stat.webp)

MBP scaled pretty well in this test with an overall lower base cost (Min: 2.29 ms), much smaller spikes (Max: 9 ms) and much more stable cost overall (Avg: 3.4 ms).

![MBP_STAT](/posts/images/mbp_stat.webp)

__16 Players__

Repeating the test with the same scenario we use for periodic playtest (server running alone on one machine) gives us the most precise results for performance. We again tested SAP vs. MBP for 16 players, the results were as below (uses optimized static meshes):

SAP has a base cost of 2 ms, highest spike recorded at 10 ms, overall average of 3 ms.

![16_SAP](/posts/images/16_sap.png)

MBP repeatedly shows better scaling, with a base cost of 2 ms as well, highest spike recorded at 3.4 ms, overall average of 2.1 ms.

![16_MBP](/posts/images/16_mbp.png)

__32 Players__

Finally, we repeated the playtest for 32 players. SAP had a base cost of 3.7 ms, highest recorded spike at 25 ms, and overall average of 6.8 ms.

![32_SAP](/posts/images/32_sap.png)

MBP showed great scaling for 32 players, with a base cost of 3 ms, highest recorded spike at 4.5 ms, and a great overall average of 3.2 ms.

![32_MBP](/posts/images/32_mbp.png)


## PIX Profiles

__1 Player__

PIX profiles confirm SAP has the same behavior as before, the base time (small orange-yellow events in the middle) is around 2 ms with occasional spikes, either in Sap::Update (large orange spike to the right) or Sap::PostUpdate (large blue spike to the left).

![SAP.PIX.Timeline](/posts/images/sap-pix-timeline.png)

On the other hand, MBP cost was much smaller in scale and had much smaller spikes (only the spikes are visible in the profiler, base cost is too small to view).

![MBP.PIX.Timeline](/posts/images/mbp-pix-timeline.png)

Highest recorded SAP spikes were as before in __SapUpdate__ (5 ms) or __SapPostUpdate__ (6.7 ms). In worst case, spikes could happen in both functions in the same frame.

![SAP.PIX.Details.01](/posts/images/sap-pix-details-01.png)

![SAP.PIX.Details.02](/posts/images/sap-pix-details-02.png)

On the other hand, spikes in MBP are smaller in magnitude.

![MBP.PIX.Details](/posts/images/mbp-pix-details.png)


__10 Players__

SAP scaled up pretty badly and the profile was dominated by PhysX, in some cases a SapUpdate spike was immediately followed by a SapPostUpdate spike resulting ina total spike of over 200 ms.

![SAP_250](/posts/images/sap_250.png)

SapUpdate spike up to 124 ms.

![SAP_Update](/posts/images/sap_update.png)

SapPostUpdate spike up to 80 ms.

![SAP_PostUpdate](/posts/images/sap_postupdate.png)

For MBP, highest recorded spike in PIX was 2.4 ms.

![MBP_Update](/posts/images/mbp_update.png)

# Conclusion

MBP is a clear winner for our case, with lower base cost and a much more stable average cost. No content was removed and the game can still live up to its full potential. If you are doing a game with a lot of colliding objects and if your PhysX performance suffers, consider switching to MBP. Note: optimizing meshes is STILL important and will result in great performance gains anyway, even when using MBP. Switching to a different collision algorithm is not an excuse for badly optimized content 😉

# Bonus: Get MBP Now!

You can enable MBP in your game easily by doing the following code change in __"\Engine\Source\Runtime\Engine\Private\PhysicsEngine\PhysScene.cpp"__

```	
void FPhysScene::InitPhysScene(uint32 SceneType)
{
...
    if (UPhysicsSettings::Get()->bEnableEnhancedDeterminism) {
        PSceneDesc.flags |= PxSceneFlag::eENABLE_ENHANCED_DETERMINISM;
    }
 
    // Begin Code Change
    if (UPhysicsSettings::Get()->bEnableMBPForBroadphase)
    {
        PSceneDesc.broadPhaseType = PxBroadPhaseType::eMBP;
    }
 
    bool bIsValid = PSceneDesc.isValid();
    if (!bIsValid)
    {
        UE_LOG(LogPhysics, Log, TEXT("Invalid PSceneDesc"));
    }
 
    // Create scene, and add to map
    PxScene* PScene = GPhysXSDK->createScene(PSceneDesc);
 
    if (UPhysicsSettings::Get()->bEnableMBPForBroadphase && PScene->getBroadPhaseType())
    {
        TArray<PxBounds3> Regions;
        const uint32 SubDivs = UPhysicsSettings::Get()->MBPRegionSubdiv;
        Regions.SetNumUninitialized(SubDivs * SubDivs);
 
        const PxBounds3 LevelBounds = PxBounds3::centerExtents(PxVec3(PxZero), U2PVector(UPhysicsSettings::Get()->MBPWorldBounds));
 
        PxU32 Num = PxBroadPhaseExt::createRegionsFromWorldBounds(Regions.GetData(), LevelBounds, SubDivs, 2);
        check(Num == SubDivs * SubDivs);
 
        for (const PxBounds3& Bounds : Regions)
        {
            PxBroadPhaseRegion Region;
            Region.bounds = Bounds;
            Region.userData = nullptr;
            PScene->addBroadPhaseRegion(Region);
        }
    }
 
    // End Code Change
 
#if WITH_APEX
    // Build the APEX scene descriptor for the PhysX scene
    apex::SceneDesc ApexSceneDesc;
...
```


Add the following lines to __"\Engine\Source\Runtime\Engine\Private\PhysicsEngine\PhysicsSettings.cpp"__
```	
    , PhysXTreeRebuildRate(10)
    // Begin Code Change
    , bEnableMBPForBroadphase(true)
    , MBPWorldBounds(100000)
    , MBPRegionSubdiv(16)
    // End Code Change
{
    SectionName = TEXT("Physics");
}
```

Add the following to the class UPhysicsSettings in __"\Engine\Classes\PhysicsEngine\PhysicsSettings.h"__
```	
public:
    UPROPERTY(config, EditAnywhere, Category = Simulation)
    bool bEnableMBPForBroadphase;
 
    /** Specifies the maximum world bounds. Needed for MBP to work correctly! */
    UPROPERTY(config, EditAnywhere, Category = Simulation, meta = (EditCondition = "bEnableMBPForBroadphase"))
    FVector MBPWorldBounds;
 
    /** Defines in how many horizontal and vertical subdivisions the world should be divided */
    UPROPERTY(config, EditAnywhere, Category = Simulation, meta = (EditCondition = "bEnableMBPForBroadphase"))
    uint32 MBPRegionSubdiv;
```

## Note

MBPWorldBounds should be set up correctly based on your world bounds, it should cover all of your world. Physics objects outside those bounds will not be correctly simulated.