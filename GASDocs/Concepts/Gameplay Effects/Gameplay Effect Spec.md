---
tags:
  - iku/check
  - GAS-System/Effect/Spec
  - iku/draft
---
#iku/tag-target
# Gameplay Effect Spec
The *Gameplay Effect Spec* is the instance of it's parent *Gameplay Effect*.
It's used for applying the effect to the target instead of using the GE itself.
>uh I think tag target is something I want to triangulate together
## Overview
- *Gameplay Effect Specs* can be freely created and modified at runtime.
- *Gameplay Effect* can not, and should be created by designers prior to runtime. 
- When applied successfully, they return a new struct called `FActiveGameplayEffect`.
#iku/tag-target
### Contents of the *Gameplay Effect Spec*
The following default to the equivalent parent value of the *Gameplay Effect*, unless modified. 

>*periodic* here for the period effect variable

- The *level*, *duration*, and *periodic* of the *Spec*, if applicable.
- The *Spec*'s creator, via `GameplayEffectContextHandle`
	- [ ] Check this if 
* Current *Stack Count*, based of the *Stack Limit* imposed by the *Gameplay Effect*.
* *Attributes* that were captured at the time of creation, due to snapshotting.
* Relevant *Input Bindings*
* The "Runtime State"

The Target Gets:
* `DynamicGrantedTags`, in addition to the *Gameplay Tags* that the *Gameplay Effect* grants.
* `DynamicAssetTags` that the *Gameplay Effect Spec* has itself
* `AssetTags` that the *Gameplay Effect* has "in addition to"
* "`SetByCaller` `TMaps`".

*The Negative Space admires Your Presence*
### Creating a Gameplay Effect Spec
- This is done with `UAbilitySystemComponent::MakeOutgoingSpec()` which is `BlueprintCallable`.
	- They do not have to be immediately applied.
- The documentation Epic provides actually might be useful as it lists what functions this has by default.
	-  [*Gameplay Effect Spec*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/FGameplayEffectSpec/index.html) (`GESpec`) 
- Code: from the *GameplayEffect's* "ClassDefaultObject".
- [ ] Look into last bullet point

>Guessing this is to avoid the ASC not knowing how to handle it otherwise if you send the whole GE over. You just want the GE's results, right? 

## Common Use Cases
### Use Case Examples
- It is common to pass a *Gameplay Effect Spec* to a projectile created from an ability.

### Moving Float Values with SetByCallers (via *Gameplay Effect Spec*)
> I think this is how you send the actual variables around? The docs mention how to define everything but that...?


`SetByCallers` allow the `GameplayEffectSpec` to carry float values associated with a *Gameplay Tag* or `FName` around. 
 
 It is common to pass numerical data generated inside of an ability to either of the following:
 - [`GameplayEffectExecutionCalculations`](#concepts-ge-ec) 
 - [`ModifierMagnitudeCalculations`](#concepts-ge-mmc) .

They are stored in their respective `TMaps` on the `GameplayEffectSpec`.
- `TMap<FGameplayTag, float>`
- `TMap<FName, float>` . 

These can be used as:
- `Modifiers` on the *Gameplay Effect* 
- As generic means of ferrying floats around.

| `SetByCaller` Use | Notes                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Modifiers`       | Must be defined ahead of time in the *Gameplay Effect* class. Can only use the *Gameplay Tag*version. If one is defined on the *Gameplay Effect* class but the `GameplayEffectSpec` does not have the corresponding tag and float value pair, the game will have a runtime error on application of the `GameplayEffectSpec` and return 0. This is a potential problem for a `Divide` operation. See [`Modifiers`](#concepts-ge-mods). |
| Elsewhere         | Does not need to be defined ahead of time anywhere. Reading a `SetByCaller` that does not exist on a `GameplayEffectSpec` can return a developer defined default value with optional warnings.                                                                                                                                                                                                                                      |

To assign `SetByCaller` values in Blueprint, use the Blueprint node for the version that you need (*Gameplay Tag* or `FName`):

![Assigning SetByCaller](https://github.com/tranek/GASDocumentation/raw/master/Images/setbycaller.png)

To read a `SetByCaller` value in Blueprint, you will need to make custom nodes in your Blueprint Library.

To assign `SetByCaller` values in C++, use the version of the function that you need (*Gameplay Tag*or `FName`):

```c++
void FGameplayEffectSpec::SetSetByCallerMagnitude(FName DataName, float Magnitude);
```
```c++
void FGameplayEffectSpec::SetSetByCallerMagnitude(FGameplayTag DataTag, float Magnitude);
```

To read a `SetByCaller` value in C++, use the version of the function that you need (*Gameplay Tag* or `FName`):

```c++
float GetSetByCallerMagnitude(FName DataName, bool WarnIfNotFound = true, float DefaultIfNotFound = 0.f) const;
```
```c++
float GetSetByCallerMagnitude(FGameplayTag DataTag, bool WarnIfNotFound = true, float DefaultIfNotFound = 0.f) const;
```

I recommend using the *Gameplay Tag* version over the `FName` version. This can prevent spelling errors in Blueprint.





## Replication
When a *Gameplay Ability* is granted on the server:

- The server replicates the *Spec* to the owning client for activation.
- Activating a *Spec* will create an instance depending on its *Instancing Policy*.
- This will not happen for *Non-Instanced* *Gameplay Abilities* of the *Gameplay Ability* .
