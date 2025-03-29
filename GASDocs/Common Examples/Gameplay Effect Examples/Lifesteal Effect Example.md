---
tags:
  - GAS-System/Effect
  - GAS-System/Sample/Example
  - iku/draft
---

## Lifesteal Effect
I handle lifesteal inside of the damage [*Execution Calculation*](#concepts-ge-ec). The *Gameplay Effect* will have a *Gameplay Tag* on it like Effect.CanLifesteal. The *Execution Calculation* checks if the *GameplayEffectSpec* has that *Effect.CanLifesteal* *GameplayTag*. If the *GameplayTag* exists, the *ExecutionCalculation* [creates a dynamic *Instant* *GameplayEffect*](#concepts-ge-dynamic) with the amount of health to give as the modifier and applies it back to the *Source's* *ASC*.

```c++
if (SpecAssetTags.HasTag(FGameplayTag::RequestGameplayTag(FName("Effect.Damage.CanLifesteal"))))
{
	float Lifesteal = Damage * LifestealPercent;

	UGameplayEffect* GELifesteal = NewObject<UGameplayEffect>(GetTransientPackage(), FName(TEXT("Lifesteal")));
	GELifesteal->DurationPolicy = EGameplayEffectDurationType::Instant;

	int32 Idx = GELifesteal->Modifiers.Num();
	GELifesteal->Modifiers.SetNum(Idx + 1);
	FGameplayModifierInfo& Info = GELifesteal->Modifiers[Idx];
	Info.ModifierMagnitude = FScalableFloat(Lifesteal);
	Info.ModifierOp = EGameplayModOp::Additive;
	Info.Attribute = UPAAttributeSetBase::GetHealthAttribute();

	SourceAbilitySystemComponent->ApplyGameplayEffectToSelf(GELifesteal, 1.0f, SourceAbilitySystemComponent->MakeEffectContext());
}
```

