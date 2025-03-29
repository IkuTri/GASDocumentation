---
tags:
  - GAS-System
  - iku/draft
---

# Attribute Sets
You likely want to reference [[Attributes]] first, as the *Attribute Set* is comprised of them.

## Attribute Set Definition (C++)
The *Attribute Sets* defines, holds, and manages changes to *Gameplay Attributes*.

### Coding Guidelines
- Developers should subclass from [`UAttributeSet`](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UAttributeSet/index.html). 
- *Attributes* are internally referred to as `AttributeSetClassName.AttributeName`.
- Creating an *Attribute Sets* in an `OwnerActor's` constructor automatically registers it with its ASC. 

### Attribute Set Design 
*Attribute Sets* have negligible memory overhead. Your *Ability System Component* may have one or many. 
- How many to use is an organizational decision left up to the developer.

>Consider the following when designing your game and how it works.

#### Method: Monolithic Attribute Set
It is acceptable to have one large monolithic *Attribute Sets* shared by every *Actor Object* in your game.
- The Owning Actor's ASC would only be using attributes if needed while ignoring unused attributes.

#### Method: Group Attribute Sets by Usage

Alternatively, you may choose to have more than one *Attribute Sets* representing groupings of *Gameplay Attributes* that you selectively add to your `Actors` as needed. 

In a MOBA game, heroes might need mana but minions might not. Therefore the heroes would get the mana *Attribute Sets* and minions would not.

>What I was thinking sorta. See if I end up with my own method here.

For example, you could have:
- an *Attribute Sets* for health related *Gameplay Attributes*, 
- an *Attribute Sets* for mana related *Gameplay Attributes*, and so on.

#### Method: Subclassing Attribute Sets
Additionally, *Attribute Sets* can be subclassed as another means of selectively choosing which *Gameplay Attributes* an `Actor` has.

> [!NOTE] Reminder
>  *Gameplay Attributes* are internally referred to as `AttributeSetClassName.AttributeName`.
> 




When you subclass an *Attribute Sets*, all of the *Gameplay Attributes* from the parent class will still have the parent class's name as the prefix.

While you can have more than one *Attribute Sets*, you should not have more than one *Attribute Sets* of the same class on an `ASC`. If you have more than one *Attribute Sets* from the same class, it won't know which *Attribute Sets* to use and will just pick one.


------------

### Advanced Methods and Scenarios
<a name="concepts-as-design-subcomponents"></a>
#### Subcomponents with Individual Attributes
In the scenario where you have multiple damageable components on a `Pawn` like individually damageable armor pieces:

I recommend that if you know the 
maximum number of damageable components that a `Pawn` could have
make that many health *Gameplay Attributes* on one *Attribute Sets* - DamageableCompHealth0, DamageableCompHealth1, etc. 
to represent logical 'slots' for those damageable components.


In your damageable component class instance, assign the slot number `Attribute` that can be read by `GameplayAbilities` or [`Executions`](#concepts-ge-ec) to know which `Attribute` to apply damage to. `Pawns` that have less than the maximum number or zero of damageable components are fine. Just because a *Attribute Sets* has an `Attribute`, doesn't mean that you have to use it. Unused *Gameplay Attributes* take up trivial amount of memory.

If your subcomponents need many *Gameplay Attributes* each, there's potentially an unbounded number of subcomponents, the subcomponents can detach and be used by other players (e.g. weapons), or for any other reason this approach doesn't work for you, I'd recommend switching away from *Gameplay Attributes* and instead store plain old floats on the components. See [Item Attributes](#concepts-as-design-itemattributes).


## Attribute Set Operations
>Note that this should refer to the Set and not individual variables.

### Adding and Removing *Attribute Sets* at Runtime
*Attribute Sets* can be added and removed from an `ASC` at runtime. however, removing `AttributeSets` can be dangerous. 

For example:
- if an *Attribute Sets* is removed on a client before the server...
- and an *Gameplay Attribute* value change is replicated to client...
- ...the *Gameplay Attribute* won't find its *Attribute Set* and crash the game.

#### Example: On weapon add to inventory:
```c++
AbilitySystemComponent->GetSpawnedAttributes_Mutable().AddUnique(WeaponAttributeSetPointer);
AbilitySystemComponent->ForceReplication();
```

#### Example: On weapon remove from inventory:
```c++
AbilitySystemComponent->GetSpawnedAttributes_Mutable().Remove(WeaponAttributeSetPointer);
AbilitySystemComponent->ForceReplication();
```



------

## Working with Item Attributes (Example: Weapon Ammo)
There's a few ways to implement equippable items with *Gameplay Attributes* (weapon ammo, armor durability, etc). All of these approaches store values directly on the item. This is necessary for items that can be equipped by more than one player over its lifetime.

1. Use plain floats on the item (**Recommended**)
2. Separate *Attribute Sets*  on the item
3. Separate ASC on the item


----

### Method: Plain Floats on the Item (Fortnite / GASShooter)
Instead of *Gameplay Attributes*, store plain float values on the item class instance. 
- As mentioned, Fortnite!
- The Project Example the original docs created alongside it also uses this method.

#### Firearm / Gun Example
For a gun, directly as replicated floats (`COND_OwnerOnly`) on the gun instance:
- store the max clip size
- current ammo in clip
- reserve ammo
- etc 

- If weapons share reserves, you would move that onto the character as an `Attribute` in a shared ammo *Attribute Sets* 
	- (reload abilities can use a `Cost GE` to pull from reserve ammo into the gun's float clip ammo). 
- Since you're not using *Gameplay Attributes* for current clip ammo, you will need to:
	- override some functions in `UGameplayAbility` 
	- This checks and applies the cost against the floats on the gun. 
 

Making the gun the `SourceObject` in the [`GameplayAbilitySpec`](https://github.com/tranek/GASDocumentation#concepts-ga-spec) when granting the ability means you'll have access to the gun that granted the ability inside the ability.

To prevent the gun from replicating back the ammo amount and clobbering the local ammo amount during automatic fire, disable replication while the player has a `IsFiring` `GameplayTag` in `PreReplication()`. You're essentially doing your own local prediction here.

```c++
void AGSWeapon::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
	Super::PreReplication(ChangedPropertyTracker);

	DOREPLIFETIME_ACTIVE_OVERRIDE(AGSWeapon, PrimaryClipAmmo, (IsValid(AbilitySystemComponent) && !AbilitySystemComponent->HasMatchingGameplayTag(WeaponIsFiringTag)));
	DOREPLIFETIME_ACTIVE_OVERRIDE(AGSWeapon, SecondaryClipAmmo, (IsValid(AbilitySystemComponent) && !AbilitySystemComponent->HasMatchingGameplayTag(WeaponIsFiringTag)));
}
```

#### Benefits:
4. Avoids limitations of using `AttributeSets` (see below)

#### Limitations:
5. Can not use existing `GameplayEffect` workflow (`Cost GEs` for ammo use, etc)
6. Requires work to override key functions on `UGameplayAbility` to check and apply ammo costs against the gun's floats




------


### Method: Attribute Set on the Item
>Honestly, this seems like what Arc Inventory is doing
- The old version of the *Project Example* uses this
	- The link provided: [Older version of GASShooter](https://github.com/tranek/GASShooter/tree/df5949d0dd992bd3d76d4a728f370f2e2c827735).
- Separate *Attribute Set* on the item
	- gets added to the player's *Ability System Component*
	- "on adding it to the player's inventory can work"
- has some major limitations.


> [!NOTE] Tranek's Comment
> I had this working in early versions of [GASShooter](https://github.com/tranek/GASShooter) for the weapon ammo. The weapon stores its *Gameplay Attributes* such as max clip size, current ammo in clip, reserve ammo, etc in an *Attribute Sets* that lives on the weapon class. If weapons share reserve ammo, you would move the reserve ammo onto the character in a shared ammo *Attribute Sets*. When a weapon is added to the player's inventory on the server, the weapon would add its *Attribute Sets* to the player's `ASC::SpawnedAttributes`. The server would then replicate this down to the client. If the weapon is removed from the inventory, it would remove its *Attribute Sets* from the `ASC::SpawnedAttributes`.



When the *Attribute Sets* lives on something other than the `OwnerActor` (say a weapon), you'll initially get some compilation errors in the *Attribute Sets*. 

The fix is to construct the *Attribute Sets* in `BeginPlay()` instead of in the constructor and to implement `IAbilitySystemInterface` (set the pointer to the `ASC` when you add the weapon to the player inventory) on the weapon.

```c++
void AGSWeapon::BeginPlay()
{
	if (!AttributeSet)
	{
		AttributeSet = NewObject<UGSWeaponAttributeSet>(this);
	}
	//...
}
```



#### Benefits:
7. Can use existing `GameplayAbility` and `GameplayEffect` workflow (`Cost GEs` for ammo use, etc)
8. Simple to setup for a very small set of items

#### Limitations:
1. You have to make a new *Attribute Sets* class for every weapon type.
2. `ASCs` can only functionally have one *Attribute Sets* instance of a class since changes to an `Attribute` look for the first instance of their *Attribute Sets* class in the `ASCs` `SpawnedAttributes` array. Additional instances of the same *Attribute Sets* class are ignored.
3. You can only have one of each type of weapon in the player's inventory due to previous reason of one *Attribute Sets* instance per *Attribute Sets* class.
4. Removing an *Attribute Sets* is dangerous. 
	1. In GASShooter if the player killed himself from a rocket, the player would immediately remove the rocket launcher from his inventory (including its *Attribute Sets* from the `ASC`). 
	2. When the server replicated that the rocket launcher's ammo `Attribute` changed, the *Attribute Sets* no longer existed on the client's `ASC` and the game crashed.



----

### Method: Individual Ability System Components on the Item
Putting a whole ASC on each item is an extreme approach. 

- I have not personally done this nor have I seen it in the wild. 
- It would take a lot of engineering to make it work.

#### Epic Games Commentary on Individual ASC Usage
*Dave Ratti from Epic's answer to [community questions #6](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89)*
>My formatting applied here as well - this I kept verbatim since that's what the docs are intending to do.
>I'll put my translation down / commentary
##### Question 
 Is it viable to have several AbilitySystemComponents which have the same owner but different avatars (e.g. on pawn and weapon/items/projectiles with Owner set to PlayerState)?


##### Answer
"The first problem I see there would be implementing the `IGameplayTagAssetInterface` and `IAbilitySystemInterface` on the *Owning Actor*."

The former may be possible: just aggregate the tags from all all ASCs (but watch out -`HasAllMatchingGameplayTags` may be met only via cross ASC aggregation.

It wouldn't be enough to just forward that calls to each ASC and OR the results together). 

But the later is even trickier: which ASC is the authoritative one? 

If someone wants to apply a GE -which one should receive it? 

Maybe you can work these out but this side of the problem will be the hardest: owners will have multiple ASCs beneath them."

"Separate ASCs on the pawn and the weapon can make sense on its own though. E.g, distinguishing between tags that describe the weapon vs those that describe the owning pawn. 

Maybe it does make sense that tags granted to the weapon also “apply” to the owner and nothing else (e.g, attributes and GEs are independent but the owner will aggregate the owned tags like I describe above). 

This could work out, I am sure. 

But having multiple ASCs with the same owner may get dicey."


##### Commentary
>I think the problem / what they're getting at here is by doing this, you try to build a custom GAS style system with ASC's. Most of the stuff provided by Epic aims to solve

> That comment is based on thinking that - this is why Tom Looman's C++ tutorial aims to teach you how to roll your own instead of um, adherence to GAS. You'd avoid doing this by building out GAS or modifying each of it's provided "parts" (deriving / subclassing the preprogrammed objects)

>That said, I still need to finish watching it, but ironically since it's a stream of consciousness I'd love to edit that transcript.


#### Benefits:
5. Can use existing `GameplayAbility` and `GameplayEffect` workflow (`Cost GEs` for ammo use, etc)
6. Can reuse *Attribute Sets* classes (one on each weapon's ASC)

#### Limitations:
7. Unknown engineering cost
8. Is it even possible?


-----

### Functions Provided
#### 4.4.5 PreAttributeChange()
`PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)` is one of the main functions in the *Attribute Sets* to respond to changes to an `Attribute's` `CurrentValue` before the change happens. It is the ideal place to clamp incoming changes to `CurrentValue` via the reference parameter `NewValue`.

For example to clamp movespeed modifiers the Sample Project does it like so:
```c++
if (Attribute == GetMoveSpeedAttribute())
{
	// Cannot slow less than 150 units/s and cannot boost more than 1000 units/s
	NewValue = FMath::Clamp<float>(NewValue, 150, 1000);
}
```
The `GetMoveSpeedAttribute()` function is created by the macro block that we added to the `AttributeSet.h` ([Defining Attributes](#concepts-as-attributes)).

This is triggered from any changes to *Gameplay Attributes*, whether using `Attribute` setters (defined by the macro block in `AttributeSet.h` ([Defining Attributes](#concepts-as-attributes))) or using [`GameplayEffects`](#concepts-ge).

**Note:** Any clamping that happens here does not permanently change the modifier on the `ASC`. It only changes the value returned from querying the modifier. This means anything that recalculates the `CurrentValue` from all of the modifiers like [`GameplayEffectExecutionCalculations`](#concepts-ge-ec) and [`ModifierMagnitudeCalculations`](#concepts-ge-mmc) need to implement clamping again.

**Note:** Epic's comments for `PreAttributeChange()` say not to use it for gameplay events and instead use it mainly for clamping. The recommended place for gameplay events on `Attribute` change is `UAbilitySystemComponent::GetGameplayAttributeValueChangeDelegate(FGameplayAttribute Attribute)` ([Responding to Attribute Changes](#concepts-a-changes)).



<a name="concepts-as-postgameplayeffectexecute"></a>
#### 4.4.6 PostGameplayEffectExecute()
`PostGameplayEffectExecute(const FGameplayEffectModCallbackData & Data)` only triggers after changes to the `BaseValue` of an `Attribute` from an instant [`GameplayEffect`](#concepts-ge). This is a valid place to do more `Attribute` manipulation when they change from a `GameplayEffect`.

For example, in the Sample Project we subtract the final damage `Meta Attribute` from the health `Attribute` here. If there was a shield `Attribute`, we would subtract the damage from it first before subtracting the remainder from health. The Sample Project also uses this location to apply hit react animations, show floating Damage Numbers, and assign experience and gold bounties to the killer. By design, the damage `Meta Attribute` will always come through an instant `GameplayEffect` and never the `Attribute` setter.

Other *Gameplay Attributes* that will only have their `BaseValue` changed from instant `GameplayEffects` like mana and stamina can also be clamped to their maximum value counterpart *Gameplay Attributes* here.

**Note:** When `PostGameplayEffectExecute()` is called, changes to the `Attribute` have already happened, but they have not replicated back to clients yet so clamping values here will not cause two network updates to clients. Clients will only receive the update after clamping.



<a name="concepts-as-onattributeaggregatorcreated"></a>
#### 4.4.7 OnAttributeAggregatorCreated()
`OnAttributeAggregatorCreated(const FGameplayAttribute& Attribute, FAggregator* NewAggregator)` triggers when an `Aggregator` is created for an `Attribute` in this set. It allows custom setup of [`FAggregatorEvaluateMetaData`](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/FAggregatorEvaluateMetaData/index.html). `AggregatorEvaluateMetaData` is used by the `Aggregator` in evaluating the `CurrentValue` of an `Attribute` based on all the [`Modifiers`](#concepts-ge-mods) applied to it. By default, `AggregatorEvaluateMetaData` is only used by the `Aggregator` to determine which `Modifiers` qualify with the example of `MostNegativeMod_AllPositiveMods` which allows all positive `Modifiers` but restricts negative `Modifiers` to only the most negative one. This was used by Paragon to only allow the most negative move speed slow effect to apply to a player regardless of how many slow effects where on them at any one time while applying all positive move speed buffs. `Modifiers` that don't qualify still exist on the `ASC`, they just aren't aggregated into the final `CurrentValue`. They can potentially qualify later once conditions change, like in the case if the most negative `Modifier` expires, the next most negative `Modifier` (if one exists) then qualifies.

To use AggregatorEvaluateMetaData in the example of only allowing the most negative `Modifier` and all positive `Modifiers`:

```c++
virtual void OnAttributeAggregatorCreated(const FGameplayAttribute& Attribute, FAggregator* NewAggregator) const override;
```

```c++
void UGSAttributeSetBase::OnAttributeAggregatorCreated(const FGameplayAttribute& Attribute, FAggregator* NewAggregator) const
{
	Super::OnAttributeAggregatorCreated(Attribute, NewAggregator);

	if (!NewAggregator)
	{
		return;
	}

	if (Attribute == GetMoveSpeedAttribute())
	{
		NewAggregator->EvaluationMetaData = &FAggregatorEvaluateMetaDataLibrary::MostNegativeMod_AllPositiveMods;
	}
}
```

Your custom `AggregatorEvaluateMetaData` for qualifiers should be added to `FAggregatorEvaluateMetaDataLibrary` as static variables.


