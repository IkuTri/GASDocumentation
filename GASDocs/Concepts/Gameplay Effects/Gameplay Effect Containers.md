---
tags:
  - GAS-System/Advanced
  - GAS-System/Epic-ARPG
  - iku/fine
---

# *Gameplay Effect Containers*  (Not Vanilla GAS)
Epic's [Action RPG Sample Project](https://www.unrealengine.com/marketplace/en-US/product/action-rpg) implements a structure called *FGameplayEffectContainer*. 

These are not in vanilla GAS but are extremely handy for containing *Gameplay Effects* andTargetData*](#concepts-targeting-data). It automates some of the effort like creating *Gameplay Effect Specs* from *Gameplay Effects* and setting default values in its *Gameplay Effect Context*. 


Making a *Gameplay Effect Container* in a *Gameplay Ability* and passing it to spawned projectiles is very easy and straightforward. 

I opted not to implement the *Gameplay Effect Containers* in the included Sample Project to show how you would work without them in vanilla GAS,but I highly recommend looking into them and considering adding them to your project.

To access the *GESpecs* inside of the *GameplayEffectContainers* to do things like adding *SetByCallers*, break the *FGameplayEffectContainer* and access the *GESpec* reference by its index in the array of *GESpecs*. This requires that you know the index ahead of time of the *GESpec* that you want to access.

![SetByCaller with a GameplayEffectContainer](https://github.com/tranek/GASDocumentation/raw/master/Images/gecontainersetbycaller.png)



## Targeting *Gameplay Effect Containers*
*Gameplay Effect Containers* come with an optional, efficient means of producing *Target Data*.  This targeting takes place instantly when applied on the client and the server.
- More efficient than *TargetActors* because it runs on the CDO of the targeting object (no spawning and destroying of *Actors*),
- Lacks player input
- Happens instantly without needing confirmation,
- Cannot be canceled
- Cannot send data from the client to the server as it produces data on both

It works well for instant traces and collision overlaps. Epic's [Action RPG Sample Project](https://www.unrealengine.com/marketplace/en-US/product/action-rpg) includes two example types of targeting with its containers
- target the ability owner 
- pull *TargetData* from an event.

 It also implements one in Blueprint to do instant sphere traces at some offset (set by child Blueprint classes) from the player.
 - You can subclass *URPGTargetType* in C++ or Blueprint to make your own targeting types.


f