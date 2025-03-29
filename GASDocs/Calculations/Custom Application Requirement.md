---
tags:
  - iku/draft
---

# Overview: *Custom Application Requirement*
[*CustomApplicationRequirement*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UGameplayEffectCustomApplication-/index.html) (CAR) classes give the designers advanced control over whether a 
*Gameplay Effect* can be applied versus the simple *Gameplay Tag* checks on the *Gameplay Effect*. 

These can be implemented in Blueprint by overriding `CanApplyGameplayEffect()` and in C++ by overriding `CanApplyGameplayEffect_Implementation()`.

## Examples of when to use *CARs*:
* *Target* needs to have a certain amount of an *Attribute*
* *Target* needs to have a certain number of stacks of a *Gameplay Effect*

CARs can also do more advanced things like checking if an instance of this *Gameplay Effect* is already on the *Target* and [changing the duration](#concepts-ge-duration) of the existing instance instead of applying a new instance (return false for `CanApplyGameplayEffect()`).

