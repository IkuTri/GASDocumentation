---
tags:
  - GAS-System/Effect
  - GAS-System/Sample/Example
  - iku/draft
---
## Stun
Typically with stuns, we want to cancel all of a *Character's* active *GameplayAbilities*, prevent new *GameplayAbility* activations, and prevent movement throughout the duration of the stun. The Sample Project's Meteor *GameplayAbility* applies a stun on hit targets.

To cancel the target's active *GameplayAbilities*, we call *AbilitySystemComponent->CancelAbilities()* when the stun [*GameplayTag* is added](#concepts-gt-change).

To prevent new *GameplayAbilities* from activating while stunned, the *GameplayAbilities* are given the stun *GameplayTag* in their [*Activation Blocked Tags* *GameplayTagContainer*](#concepts-ga-tags).

To prevent movement while stunned, we override the *CharacterMovementComponent's* *GetMaxSpeed()* function to return 0 when the owner has the stun *GameplayTag*.
