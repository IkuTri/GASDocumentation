---
tags:
  - iku/check
  - iku/draft
  - GAS-System/Effect
---


## Common Things to do with Gameplay Effects
---
### How to Toggle the Activation of a *Gameplay Effect*
Duration and Infinite type  *Gameplay Effects* can be temporarily turned off and on after application if their *Ongoing Tag Requirements* are not met/met ([Gameplay Effect Tags](#concepts-ge-tags)). 
- Turning off a *Gameplay Effect* removes the effects of its *Modifiers* and applied *Gameplay Tags* 
- ...but it does not remove the *Gameplay Effect*. 
- Turning the *Gameplay Effect* back on reapplies its *Modifiers* and *Gameplay Tags*.

### How to Change the Duration of an Active *Gameplay Effect* 
To change the time remaining for a *Cooldown Gameplay Effect* or any *Duration* *Gameplay Effect*, we need to:

- Change the *GameplayEffectSpec's* *Duration*, 
- Update its *StartServerWorldTime*, 
- Update its *CachedStartServerWorldTime*, 
- Update its *StartWorldTime*, 
- Rerun the check on the duration with *CheckDuration()*.

Doing this on the server and marking the *FActiveGameplayEffect* dirty will replicate the changes to clients.


**Note:** This does involve a "const_cast" and may not be Epic's intended way of changing durations, but it seems to work well so far.
- [ ] figure out if this is literal code or just a ... tranek thing


#### Example
```c++
bool UPAAbilitySystemComponent::SetGameplayEffectDurationHandle(FActiveGameplayEffectHandle Handle, float NewDuration)
{
	if (!Handle.IsValid())
	{
		return false;
	}

	const FActiveGameplayEffect* ActiveGameplayEffect = GetActiveGameplayEffect(Handle);
	if (!ActiveGameplayEffect)
	{
		return false;
	}

	FActiveGameplayEffect* AGE = const_cast<FActiveGameplayEffect*>(ActiveGameplayEffect);
	if (NewDuration > 0)
	{
		AGE->Spec.Duration = NewDuration;
	}
	else
	{
		AGE->Spec.Duration = 0.01f;
	}

	AGE->StartServerWorldTime = ActiveGameplayEffects.GetServerWorldTime();
	AGE->CachedStartServerWorldTime = AGE->StartServerWorldTime;
	AGE->StartWorldTime = ActiveGameplayEffects.GetWorldTime();
	ActiveGameplayEffects.MarkItemDirty(*AGE);
	ActiveGameplayEffects.CheckDuration(Handle);

	AGE->EventSet.OnTimeChanged.Broadcast(AGE->Handle, AGE->StartWorldTime, AGE->GetDuration());
	OnGameplayEffectDurationChange(*AGE);

	return true;
}
```


### How to Recalculate *Modifiers* Manually for a *Gameplay Effect*
For recalculating *Duration* or *Infinite* *Gameplay Effects*:

Say you have an *MMC* that uses data that doesn't come from *Attributes*:
- You can call `UAbilitySystemComponent::ActiveGameplayEffects.SetActiveGameplayEffectLevel(FActiveGameplayEffectHandle ActiveHandle, int32 NewLevel)`
	- with the same level that it already has 
	- using `UAbilitySystemComponent::ActiveGameplayEffects.GetActiveGameplayEffect(ActiveHandle).Spec.GetLevel()`.
- *Modifiers* that are based on backing *Attributes* automatically update when those backing *Attributes* update. 

In code, the key (available ?) functions of `SetActiveGameplayEffectLevel()` used to update the *Modifiers* are:
```C++
MarkItemDirty(Effect);
Effect.Spec.CalculateModifierMagnitudes();
// Private function otherwise we'd call these three functions without needing to set the level to what it already is
UpdateAllAggregatorModMagnitudes(Effect);
```

### How to Deal with Instancing by Using *Gameplay Effect Spec*

*Gameplay Effects* are not typically instantiated as they have a *Spec* for doing so instead.
>Calling it "Spec" is my shorthand, by the way. Not sure you'll see it elsewhere.


This is called




- Successfully applied *Gameplay Effect Specs* are then added to a new struct called `FActiveGameplayEffect`
- The *ASC* keeps track of these in a special container struct, which the documentation refers to as  *Active Gameplay Effects*.



