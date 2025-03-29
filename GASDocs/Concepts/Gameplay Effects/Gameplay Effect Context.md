---
tags:
  - iku/fine
---

# *Gameplay Effect Context*
The [`GameplayEffectContext`](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/FGameplayEffectContext/index.html) structure holds information about a `GameplayEffectSpec's` instigator and [`TargetData`](#concepts-targeting-data).

This is also a good structure to subclass to pass arbitrary data around between places like:
- *Modifier Magnitude Calculation*
- *Gameplay Effect Execution Calculations*
- *Attribute Sets*
- *Gameplay Cues*

## Usage
1. Subclass `FGameplayEffectContext`
2. Override `FGameplayEffectContext::GetScriptStruct()`
3. Override `FGameplayEffectContext::Duplicate()`
4. (if your new data needs to be replicated) Override `FGameplayEffectContext::NetSerialize()`
5. Implement `TStructOpsTypeTraits` for your subclass, like the parent struct `FGameplayEffectContext` has
6. Override `AllocGameplayEffectContext()` in your [`AbilitySystemGlobals`](#concepts-asg) class to return a new object of your subclass

[GASShooter](https://github.com/tranek/GASShooter) uses a subclassed `GameplayEffectContext` to add `TargetData` which can be accessed in `GameplayCues`, specifically for the shotgun since it can hit more than one enemy.

