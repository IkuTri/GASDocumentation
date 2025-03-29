---
tags:
  - GAS-System/Built-In/Effect
  - iku/fine
  - GAS-System/Effect
---
# Cooldown Gameplay Effect
*Gameplay Abilities* have an optional *Cooldown Gameplay Effect*, specifically allowing a time duration before another activation.
- The *Gameplay Ability* checks for the *Cooldown Gameplay Tag* instead of the presence of the *Cooldown Gameplay Effect*.

- [ ]  To look into: Stacking *Cooldown Charges* of an ability.
## Requirements
- This should be a `Duration` *Gameplay Effect*.
- No `Modifiers` are allowed.
- It should have an unique *Gameplay Tag* per *Gameplay Ability*, or per *Ability Slot* .
- This tag would be in the `GameplayEffect's` `GrantedTags` ("*Cooldown Gameplay Tag*").

## Prediction
- They are meant to be predicted by default.
- Do not use *Execution Calculations*.
- *Modifier Magnitude Calculations* perfectly acceptable and encouraged for complex cooldown calculations.

## Designing Gameplay Abilities to Use Cooldowns
>I think original text got nuked

It's possible to design a single GE that is the concept of an *Ability Cooldown* for your game, and then use it for every GA you design.

>Your GA's would have their own set cooldown numbers (variables in some Attribute Set) that get sent as a Spec, then the one Cooldown Effect (code) works with that number to set that GA's cooldown into action.

- The *Spec* is designed for in the moment calculations. 
- The code for the "*Cost Gameplay Effect*" is only written once, in one place.
-  **This only works for `Instanced` abilities.**
- Two are techniques for reusing it. See the Methods below.

---
## Methods

### Method: Set Caller
**Use a [`SetByCaller`](#concepts-ge-spec-setbycaller).** This is the easiest method. Set the duration of your shared *Cooldown Gameplay Effect* to `SetByCaller` with a `GameplayTag`. 
#### Step: Define Gameplay Ability in Preparation
On your *Gameplay Ability* subclass, define the following:
- A float / `FScalableFloat` for the duration
- A `FGameplayTagContainer` for the unique *Cooldown Gameplay Tag*, 
- A temporary `FGameplayTagContainer` that we will use as the return pointer of the union of our *Cooldown Gameplay Tag* and the `Cooldown GE's` tags.
```c++
UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cooldown")
FScalableFloat CooldownDuration;

UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cooldown")
FGameplayTagContainer CooldownTags;

// Temp container that we will return the pointer to in GetCooldownTags().
// This will be a union of our CooldownTags and the Cooldown GE's cooldown tags.
UPROPERTY(Transient)
FGameplayTagContainer TempCooldownTags;
```
#### Step: Overriding to get Cooldown Tags
Then override `UGameplayAbility::GetCooldownTags()` to return the union of our `Cooldown Tags` and any existing `Cooldown GE's` tags.
```c++
const FGameplayTagContainer * UPGGameplayAbility::GetCooldownTags() const
{
	FGameplayTagContainer* MutableTags = const_cast<FGameplayTagContainer*>(&TempCooldownTags);
	MutableTags->Reset(); // MutableTags writes to the TempCooldownTags on the CDO so clear it in case the ability cooldown tags change (moved to a different slot)
	const FGameplayTagContainer* ParentTags = Super::GetCooldownTags();
	if (ParentTags)
	{
		MutableTags->AppendTags(*ParentTags);
	}
	MutableTags->AppendTags(CooldownTags);
	return MutableTags;
}
```
#### Step: Overriding to Apply the *Cooldown Effect Tags* into the *Gameplay Effect Spec*
Finally, override `UGameplayAbility::ApplyCooldown()` to inject our `Cooldown Tags` and to add the `SetByCaller` to the cooldown `GameplayEffectSpec`.
```c++
void UPGGameplayAbility::ApplyCooldown(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo * ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo) const
{
	UGameplayEffect* CooldownGE = GetCooldownGameplayEffect();
	if (CooldownGE)
	{
		FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(CooldownGE->GetClass(), GetAbilityLevel());
		SpecHandle.Data.Get()->DynamicGrantedTags.AppendTags(CooldownTags);
		SpecHandle.Data.Get()->SetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag(FName(  OurSetByCallerTag  )), CooldownDuration.GetValueAtLevel(GetAbilityLevel()));
		ApplyGameplayEffectSpecToOwner(Handle, ActorInfo, ActivationInfo, SpecHandle);
	}
}
```


##### Sub-Step: In Editor:
In this picture, the cooldown's duration `Modifier` is set to `SetByCaller` with a `Data Tag` of `Data.Cooldown`. `Data.Cooldown` would be `OurSetByCallerTag` in the code above.

![Cooldown GE with SetByCaller](https://github.com/tranek/GASDocumentation/raw/master/Images/cooldownsbc.png)


### Method: Use an MMC
This has the same setup as above, except for:
- Setting the `SetByCaller`as the duration on the *Cooldown Gameplay Effect*
- Setting the SetByCaller in `ApplyCooldown`. 
#### Step: Setting the Duration Up and pointing to the MMC
Set the duration to be a `Custom Calculation Class` and point to the new `MMC` that we will make.

```c++
UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cooldown")
FScalableFloat CooldownDuration;

UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cooldown")
FGameplayTagContainer CooldownTags;

// Temp container that we will return the pointer to in GetCooldownTags().
// This will be a union of our CooldownTags and the Cooldown GE's cooldown tags.
UPROPERTY(Transient)
FGameplayTagContainer TempCooldownTags;
```

#### Step: Overriding to get Cooldown Tags
Then override `UGameplayAbility::GetCooldownTags()` to return the union of our `Cooldown Tags` and any existing `Cooldown GE's` tags.
```c++
const FGameplayTagContainer * UPGGameplayAbility::GetCooldownTags() const
{
	FGameplayTagContainer* MutableTags = const_cast<FGameplayTagContainer*>(&TempCooldownTags);
	MutableTags->Reset(); // MutableTags writes to the TempCooldownTags on the CDO so clear it in case the ability cooldown tags change (moved to a different slot)
	const FGameplayTagContainer* ParentTags = Super::GetCooldownTags();
	if (ParentTags)
	{
		MutableTags->AppendTags(*ParentTags);
	}
	MutableTags->AppendTags(CooldownTags);
	return MutableTags;
}
```

#### Step: Overriding to Apply the *Cooldown Effect Tags* into the *Gameplay Effect Spec*
Override `UGameplayAbility::ApplyCooldown()` to inject our *Cooldown Effect Tags* into the cooldown *Gameplay Effect Spec*.
```c++
void UPGGameplayAbility::ApplyCooldown(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo * ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo) const
{
	UGameplayEffect* CooldownGE = GetCooldownGameplayEffect();
	if (CooldownGE)
	{
		FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(CooldownGE->GetClass(), GetAbilityLevel());
		SpecHandle.Data.Get()->DynamicGrantedTags.AppendTags(CooldownTags);
		ApplyGameplayEffectSpecToOwner(Handle, ActorInfo, ActivationInfo, SpecHandle);
	}
}
```

```c++
float UPGMMC_HeroAbilityCooldown::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec & Spec) const
{
	const UPGGameplayAbility* Ability = Cast<UPGGameplayAbility>(Spec.GetContext().GetAbilityInstance_NotReplicated());

	if (!Ability)
	{
		return 0.0f;
	}

	return Ability->CooldownDuration.GetValueAtLevel(Ability->GetAbilityLevel());
}
```


##### Sub-Step: What it looks like in Editor
![Cooldown GE with MMC](https://github.com/tranek/GASDocumentation/raw/master/Images/cooldownmmc.png)



#### Step: Getting the Remaining Time of the *Cooldown Gameplay Effect*
Get the Cooldown Gameplay Effect's Remaining Time:
```c++
bool APGPlayerState::GetCooldownRemainingForTag(FGameplayTagContainer CooldownTags, float & TimeRemaining, float & CooldownDuration)
{
	if (AbilitySystemComponent && CooldownTags.Num() > 0)
	{
		TimeRemaining = 0.f;
		CooldownDuration = 0.f;

		FGameplayEffectQuery const Query = FGameplayEffectQuery::MakeQuery_MatchAnyOwningTags(CooldownTags);
		TArray< TPair<float, float> > DurationAndTimeRemaining = AbilitySystemComponent->GetActiveEffectsTimeRemainingAndDuration(Query);
		if (DurationAndTimeRemaining.Num() > 0)
		{
			int32 BestIdx = 0;
			float LongestTime = DurationAndTimeRemaining[0].Key;
			for (int32 Idx = 1; Idx < DurationAndTimeRemaining.Num(); ++Idx)
			{
				if (DurationAndTimeRemaining[Idx].Key > LongestTime)
				{
					LongestTime = DurationAndTimeRemaining[Idx].Key;
					BestIdx = Idx;
				}
			}

			TimeRemaining = DurationAndTimeRemaining[BestIdx].Key;
			CooldownDuration = DurationAndTimeRemaining[BestIdx].Value;

			return true;
		}
	}

	return false;
}
```

**Note:** Querying the cooldown's time remaining on clients requires that they can receive replicated *Gameplay Effects*. This will depend on their `ASC's` [replication mode](#concepts-asc-rm).


---
## Listening for Cooldown Begin and End
- To listen for when a cooldown begins, you can either respond to when the *Cooldown Gameplay Effect* is applied by binding to:
	- `AbilitySystemComponent->OnActiveGameplayEffectAddedDelegateToSelf` 
- or when the *Cooldown Gameplay Tag* is added by binding to 
	- `AbilitySystemComponent->RegisterGameplayTagEvent(CooldownTag, EGameplayTagEventType::NewOrRemoved)`. 
- I recommend listening for when the *Cooldown Gameplay Effect* is added because you also have access to the *Gameplay Effect Spec* that applied it. 
- From this you can determine if the *Cooldown Gameplay Effect* is the locally predicted one or the Server's correcting one.
- To listen for when a cooldown ends, you can either respond to when the *Cooldown Gameplay Effect* is removed by binding to:
	- `AbilitySystemComponent->OnAnyGameplayEffectRemovedDelegate()` 
- or when the *Cooldown Gameplay Tag* is removed by binding to:
	- `AbilitySystemComponent->RegisterGameplayTagEvent(CooldownTag, EGameplayTagEventType::NewOrRemoved)`.
- I recommend listening for when the *Cooldown Gameplay Tag* is removed because when the Server's corrected *Cooldown Gameplay Effect* comes in, it will remove our locally predicted one.
	- This would result in the `OnAnyGameplayEffectRemovedDelegate()` to fire even though we're still on cooldown. 
- The *Cooldown Gameplay Tag* will not change during the removal of the predicted *Cooldown Gameplay Effect* and the application of the Server's corrected *Cooldown Gameplay Effect*

**Note:** Listening for a *Gameplay Effect* to be added or removed on clients requires that they can receive replicated *Gameplay Effects*. This will depend on their `ASC's` [replication mode](#concepts-asc-rm).

The Sample Project includes a custom Blueprint node that listens for cooldowns beginning and ending.

The HUD UMG Widget uses it to update the amount of time remaining on the Meteor's cooldown. This `AsyncTask` will live forever until manually called `EndTask()`, which we do in the UMG Widget's `Destruct` event. See `AsyncTaskCooldownChanged`'s header and implementation
![Listen for Cooldown Change BP Node](https://github.com/tranek/GASDocumentation/raw/master/Images/cooldownchange.png)

<a name="concepts-ge-cooldown-prediction"></a>
## Predicting Cooldowns
Cooldowns cannot really be predicted currently. 
- We can start UI cooldown timer's when the locally predicted *Cooldown Gameplay Effect* is applied...
- ...but the `GameplayAbility's` actual cooldown is tied to the server's cooldown's time remaining.
- Depending on the player's latency, the locally predicted cooldown could expire but the *Gameplay Ability* would still be on cooldown on the server and this would prevent the `GameplayAbility's` immediate re-activation until the server's cooldown expires.

The Sample Project handles this by graying out the Meteor ability's UI icon when the locally predicted cooldown begins and then starting the cooldown timer once the server's corrected *Cooldown Gameplay Effect* comes in.

A gameplay consequence of this is that players with high latencies have a lower rate of fire on short cooldown abilities than players with lower latencies putting them at a disadvantage. Fortnite avoids this by their weapons having custom bookkeeping that do not use cooldown *Gameplay Effects*.

Allowing for true predicted cooldowns (player could activate a *Gameplay Ability* when the local cooldown expires but the server is still on cooldown) is something that Epic would like to implement someday in a [future iteration of GAS](#concepts-p-future).




---


