---
title: "Micro Optimizations Episode 2"
date: 2016-10-17T00:38:15+02:00
draft: false
---

# How NOT to loop
## Simple Problem

Another optimization I tackled lately was a dynamic system to enable / disable other players’ weapon animation updates, based on distance to local player. To give you a bit of a context, there could be between 100 – 400 active weapons at the same time.

I first noticed a problem as our weapon tick function was showing up high on the offenders list, which was interesting because I did not remember anything significant should be going on there.

Digging deeper into the issue, it turns out the weapon tick function is implemented JUST for the following code:
```	
void ABWeapon::Tick(float deltaTime)
{
  ABPlayerController* pc =
    Cast(GEngine->GetFirstLocalPlayerController(GetWorld()));
 
  if (!pc->IsDead())
  {
    float distSq =
    (pc->GetPawn()->GetActorLocation() - GetActorLocation).SizeSquared();
 
    EnableAnimation(distSq < m_maxEnableDistance); 
  }
}
 
void ABWeapon::EnableAnimation(bool bEnable)
{
  m_skelMesh->SetComponentTickEnabled(bEnable);
}
```

Imaging the following, hundreds of weapons are ticking every frame to do the exact same thing: check for distance to local player and then enable / disable animation update.

This is supposed to be an optimization technique in the first place, IT SHOULD BE FAST AS HELL!

The problem with this code is that it also ignores important basic assumptions:

- You do not need to tick to perform this check every single frame.
- Ticking in UE4 is expensive. Different objects from different classes are all in the same “ticking objects” array, resulting in both instruction-cache and data-cache misses.
- The local player does not change during the weapon lifetime, and DEFINITELY not during a frame.

## Simple Solution

Let’s leave the first point out for now and follow the original design requirements by checking every frame, how much faster can we make it? First step, I disabled the weapon tick and removed the function completely. Second step, I reversed the loop logic, so instead of all weapons checking for the local player, the local player checks for all weapons. See below:
```	
// This function was already implemented
void ABPlayerController::PlayerTick(float deltaTime)
{
  // some code
 
  FVector loc = GetPawn()->GetActorLocation();
  for (auto& w : ABWeapon::s_weapons) // static list of all active weapons
  {
    w->EnableAnimation((loc - w->GetActorLocation()).SizeSquared());
  }
}
 
// Weapon's EnableAnimation function changed to the following
void ABWeapon::EnableAnimation(float distSq)
{
  m_skelMesh->SetComponentTickEnabled(distSq < m_maxEnableDistance);
}
```

This way we have a list of all weapons where we loop just once, known variables are calculated in advance, and we just need to go over them in a tight loop and repeat the same code. Note that the player’s tick was already implemented.

The cost of checking + toggling animations for ~300 weapons __went down from 1.2ms to 0.2ms__ on PS4. 1ms per-frame saving and a 6x performance increase.