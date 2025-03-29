---
tags:
  - GAS-System/Built-In/Effect
  - iku/fine
  - GAS-System/Effect
---
- (if your game has interchangeable abilities assigned to slots that share a cooldown)
	- I think an example would be different class ability options in Destiny 2, they're driven by the slot's base cooldown stat.


---
# Cost Gameplay Effect
Costs are how much of an *Attribute* needs to have to be able to activate the *Gameplay Ability*. 

## Requirements
- It should be an `Instant` *Gameplay Effect* 
- It should have one or more *Modifiers* that subtract from *Attributes* the Owner's ASC has.

## Prediction
- They are meant to be predicted. It is recommended to maintain that capability.
- Do not use *Execution Calculations*.
- *Modifier Magnitude Calculations* are perfectly acceptable and encouraged for complex cost calculations.

#iku/tag-target
## Designing Gameplay Abilities to Use Cost
When starting out, you will most likely have one unique *Cost Gameplay Effect* per *Gameplay Ability* that is designed to have one. A more advanced technique is to reuse it for multiple *Gameplay Abilities* and just modify the *Gameplay Effect Spec*.

- The *Spec* is designed for in the moment calculations. 
- The code for the "*Cost Gameplay Effect*" is only written once, in one place.
-  **This only works for `Instanced` abilities.**
- Two are techniques for reusing it. See the Methods below.
#iku/tag-target


## Methods

### Method: Use an MMC
This is the easiest method. Create an [`MMC`](#concepts-ge-mmc) that reads the cost value from the *Gameplay Ability* instance which you can get from the 
 *Gameplay Effect Spec* (in code as `FGameplayEffectSpec`).

```c++
float UPGMMC_HeroAbilityCost::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec & Spec) const
{
	const UPGGameplayAbility* Ability = Cast<UPGGameplayAbility>(Spec.GetContext().GetAbilityInstance_NotReplicated());

	if (!Ability)
	{
		return 0.0f;
	}

	return Ability->Cost.GetValueAtLevel(Ability->GetAbilityLevel());
}
```

In this example the cost value is an `FScalableFloat` on the *Gameplay Ability* child class that I added to it.
```c++
UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cost")
FScalableFloat Cost;
```

![Cost GE With MMC](https://github.com/tranek/GASDocumentation/raw/master/Images/costmmc.png)
##### Method: Override the Function
1. **Override `UGameplayAbility::GetCostGameplayEffect()`.** 
2. Override this function
3. [create a *Gameplay Effect* at runtime](#concepts-ge-dynamic) that reads the cost value on the *Gameplay Ability*.

- [ ]  Connect information about Dynamic Gameplay Effects








## Replication Info? Merge

After a *Gameplay Ability* calls *Activate()*, it can optionally commit the cost and cooldown at any time using *UGameplayAbility::CommitAbility()* which calls *UGameplayAbility::CommitCost()* and *UGameplayAbility::CommitCooldown()*. The designer may choose to call *CommitCost()* or *CommitCooldown()* separately if they shouldn't be committed at the same time. Committing cost and cooldown calls *CheckCost()* and *CheckCooldown()* one more time and is the last chance for the *Gameplay Ability* to fail related to them. The owning *ASC's* *Attributes* could potentially change after a *Gameplay Ability* is activated, failing to meet the cost at time of commit. Committing the cost and cooldown can be [locally predicted](#concepts-p) if the [prediction key](#concepts-p-key) is valid at the time of commit.