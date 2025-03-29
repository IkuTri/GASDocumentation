---
tags:
  - GAS-System
  - GAS-System/Effect
  - GAS-System/Ability
  - iku/draft
---

## Using Gameplay Abilities and Gameplay Effects Together

### How *Gameplay Effects*  Grant Abilities  
*Gameplay Effects* can grant new [*Gameplay Abilities*](#concepts-ga) to `ASCs`. Only `Duration` and `Infinite` *Gameplay Effects* can grant abilities.

A common usecase for this is when you want to force another player to do something like moving them from a knockback or pull. 
- You would apply a *Gameplay Effect* to them that grants them an automatically activating ability that does the desired action to them.
	- see [Passive Abilities](#concepts-ga-activating-passive) for how to automatically activate an ability when it is granted

Designers can choose which abilities a *Gameplay Effect* grants, what level to grant them at, what [input to bind](#concepts-ga-input) them at and the removal policy for the granted ability.

| Removal Policy             | Description                                                                                                                                                                     |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cancel Ability Immediately | The granted ability is canceled and removed immediately when the *Gameplay Effect* that granted it is removed from the Target.                                                   |
| Remove Ability on End      | The granted ability is allowed to finish and then is removed from the Target.                                                                                                   |
| Do Nothing                 | The granted ability is not affected by the removal of the granting *Gameplay Effect* from the Target. The Target has the ability permanently until it is manually removed later. |

### How *Gameplay Effects* hold *Gameplay Tags*
*Gameplay Effects* carry multiple [`GameplayTagContainers`](#concepts-gt). 

Designers will edit the `Added` and `Removed` `GameplayTagContainers` for each category and the result will show up in the `Combined` `GameplayTagContainer` on compilation.
- `Added` tags are new tags that this *Gameplay Effect* adds that its parents did not previously have. 
- `Removed` tags are tags that parent classes have but this subclass does not have.

| Category                          | Description                                                                                                                                                                                                                                                                                                                                                                        |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Gameplay Effect Asset Tags        | Tags that the *Gameplay Effect* has. They do not do any function on their own and serve only the purpose of describing the *Gameplay Effect*.                                                                                                                                                                                                                                        |
| Granted Tags                      | Tags that live on the *Gameplay Effect* but are also given to the `ASC` that the *Gameplay Effect* is applied to. They are removed from the `ASC` when the *Gameplay Effect* is removed. This only works for `Duration` and `Infinite` *Gameplay Effects*.                                                                                                                             |
| Ongoing Tag Requirements          | Once applied, these tags determine whether the *Gameplay Effect* is on or off. A *Gameplay Effect* can be off and still be applied. If a *Gameplay Effect* is off due to failing the Ongoing Tag Requirements, but the requirements are then met, the *Gameplay Effect* will turn on again and reapply its modifiers. This only works for `Duration` and `Infinite` *Gameplay Effects*. |
| Application Tag Requirements      | Tags on the Target that determine if a *Gameplay Effect* can be applied to the Target. If these requirements are not met, the *Gameplay Effect* is not applied.                                                                                                                                                                                                                      |
| Remove Gameplay Effects with Tags | *Gameplay Effects* on the Target that have any of these tags in their `Asset Tags` or `Granted Tags` will be removed from the Target when this *Gameplay Effect* is successfully applied.                                                                                                                                                                                            |

<a name="concepts-ge-immunity"></a>
### How *Gameplay Effects* apply Immunity 
*Gameplay Effects* can grant immunity, effectively blocking the application of other *Gameplay Effects*, based on [`GameplayTags`](#concepts-gt). 

While immunity can be effectively achieved through other means like `Application Tag Requirements`, using this system provides a delegate for when *Gameplay Effects* are blocked due to immunity `UAbilitySystemComponent::OnImmunityBlockGameplayEffectDelegate`.

`GrantedApplicationImmunityTags` checks if the Source `ASC` (including tags from the Source ability's `AbilityTags` if there was one) has any of the specified tags. This is a way to provide immunity from all *Gameplay Effects* from certain characters or sources based on their tags.

`Granted Application Immunity Query` checks the incoming `GameplayEffectSpec` if it matches any of the queries to block or allow its application.

The queries have helpful hover tooltips in the *Gameplay Effect* Blueprint.

###  How *Gameplay Effects* can be Dynamic at Runtime
Creating Dynamic *Gameplay Effects* at runtime is an advanced topic. You shouldn't have to do this too often.

Only `Instant` *Gameplay Effects* can be created at runtime from scratch in C++.
- `Duration` and `Infinite` *Gameplay Effects* cannot be created dynamically at runtime 
	- This is because when they replicate they look for the *Gameplay Effect* class definition that does not exist. 
- To achieve this functionality, you should instead make an archetype *Gameplay Effect* class like you would normally do in the Editor. 
- Then customize the `GameplayEffectSpec` instance with what you need at runtime.

> [!NOTE] Alternative Experimental
> 
`Instant` *Gameplay Effects* created at runtime can also be called from within a [local predicted](#concepts-p) *Gameplay Ability*. However, it is unknown yet if the dynamic creation can have side effects.

