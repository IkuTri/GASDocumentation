---
tags:
  - GAS-System
  - iku/draft
  - GAS-System/Calculation
---

# Modifier Magnitude Calculation
Unreal Reference Page: [*ModifierMagnitudeCalculations*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UGameplayModMagnitudeCalculation/index.html) 
(*ModMagCalc* or MMC) are powerful classes used as *Modifiers* in *Gameplay Effects*. 

## Overview
*MMCs'* strength lies in their capability to capture the value of any number of *Attributes*.

- This can be on the *Source* or the *Target* of *Gameplay Effect* 
- Allows full access to the *Gameplay Effect Spec* to read *Gameplay Tags* and `SetByCallers`.
- They function similarly to [[Gameplay Effect Execution Calculation]] but are less powerful.
	- Most importantly they can be predicted. See [[Prediction]].
- Their sole purpose is to return a float value from `CalculateBaseMagnitude_Implementation()`. 
- You can subclass and override this function in Blueprint and C++.


### Using with *Gameplay Effects*
MMCs can be used in any duration of *Gameplay Effects* - *Instant*, *Duration*, *Infinite*, or *Periodic*.

| Snapshot | Source or Target | Captured on *GameplayEffectSpec* | Automatically updates when *Attribute* changes for *Infinite* or *Duration* *GE* |
| -------- | ---------------- | -------------------------------- | -------------------------------------------------------------------------------- |
| Yes      | Source           | Creation                         | No                                                                               |
| Yes      | Target           | Application                      | No                                                                               |
| No       | Source           | Application                      | Yes                                                                              |
| No       | Target           | Application                      | Yes                                                                              |

The resultant float from an MMC can further be modified in the *Gameplay Effect's* *Modifier* by:
- a coefficient
- a pre and post coefficient addition.



-----

### Capturing Attributes with *Modifier Magnitude Calculation*
*Snapshotting* deals with when you want to grab the value from the *Attribute*.

- *Snapshotting* captures the *Attribute* when the *Gameplay Effect Spec* is created. (Other Page)
- Not *Snapshotting* captures the *Attribute* when the *Gameplay Effect Spec* is applied. (Other Page)

(Either way?) (the MMC?) automatically updates when the *Attribute* changes for *Infinite* and *Duration* *Gameplay Effects*. 
- [ ]  Check

Capturing *Attributes* recalculates their `CurrentValue` from existing mods on the ASC. 

This recalculation will not run `PreAttributeChange` in the *Ability Set* so any clamping must be done here again.
### Example: Mana Drain Effect
>Personally I'd consider this a "Mana Drain" over a Mana "poison", but you do you.


An example *MMC* that captures the *Target's* mana *Attribute* reduces it from a "poison effect".
- This is where the amount reduced changes depending on how much mana the *Target* has.
- It also depends on a tag that the *Target* might have:

If you don't add the `FGameplayEffectAttributeCaptureDefinition` to `RelevantAttributesToCapture` in the MMC's constructor and try to capture *Attributes*:
- You will get an error about a missing Spec while capturing. 
- If you don't need to capture *Attributes*, then you don't have to add anything to `RelevantAttributesToCapture`.

#### Code Sample:
```c++
UPAMMC_PoisonMana::UPAMMC_PoisonMana()
{

	//ManaDef defined in header FGameplayEffectAttributeCaptureDefinition ManaDef;
	ManaDef.AttributeToCapture = UPAAttributeSetBase::GetManaAttribute();
	ManaDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target;
	ManaDef.bSnapshot = false;

	//MaxManaDef defined in header FGameplayEffectAttributeCaptureDefinition MaxManaDef;
	MaxManaDef.AttributeToCapture = UPAAttributeSetBase::GetMaxManaAttribute();
	MaxManaDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target;
	MaxManaDef.bSnapshot = false;

	RelevantAttributesToCapture.Add(ManaDef);
	RelevantAttributesToCapture.Add(MaxManaDef);
}

float UPAMMC_PoisonMana::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec & Spec) const
{
	// Gather the tags from the source and target as that can affect which buffs should be used
	const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
	const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

	FAggregatorEvaluateParameters EvaluationParameters;
	EvaluationParameters.SourceTags = SourceTags;
	EvaluationParameters.TargetTags = TargetTags;

	float Mana = 0.f;
	GetCapturedAttributeMagnitude(ManaDef, Spec, EvaluationParameters, Mana);
	Mana = FMath::Max<float>(Mana, 0.0f);

	float MaxMana = 0.f;
	GetCapturedAttributeMagnitude(MaxManaDef, Spec, EvaluationParameters, MaxMana);
	MaxMana = FMath::Max<float>(MaxMana, 1.0f); // Avoid divide by zero

	float Reduction = -20.0f;
	if (Mana / MaxMana > 0.5f)
	{
		// Double the effect if the target has more than half their mana
		Reduction *= 2;
	}
	
	if (TargetTags->HasTagExact(FGameplayTag::RequestGameplayTag(FName("Status.WeakToPoisonMana"))))
	{
		// Double the effect if the target is weak to PoisonMana
		Reduction *= 2;
	}
	
	return Reduction;
}
```




