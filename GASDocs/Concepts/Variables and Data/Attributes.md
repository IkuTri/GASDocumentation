---
tags:
  - GAS-System
  - iku/fine
  - iku/check
---

# Attributes
>Kinda wondering if people say "Attributes" over "Gameplay Attributes" due to the overlap with "Gameplay Abilities"
## Core Concepts
- *Attributes* have a *Base Value* and a *Current Value*.
	- In code these are `BaseValue` / `CurrentValue`.
-  *Attributes* are defined within an *Attribute Set*, which handles *replication* for values marked as such.
	- The [[Attribute Sets]] page has more information.
- There are *Meta Attributes* which are "placeholders for temporary values" and are used for inter Attribute calculations.
	- "Damage" (Raw Damage with No Type / Element / Etc) is a key example in the original docs
- In code, `Attributes` are float values defined by the struct [`FGameplayAttributeData`](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/FGameplayAttributeData/index.html). 
>I think this "F"-named thing and Attribute Set are one and the same.

- [ ] Check code representation of Attribute to confirm

> [!NOTE] BaseValue vs CurrentValue Example
>The `BaseValue` is the permanent value of the `Attribute`.
>- `CurrentValue` is the `BaseValue` plus temporary modifications from `GameplayEffects`.
>- For example, your `Character` may have a movespeed `Attribute` with a `BaseValue` of 600 units/second. 
>- Since there are no `GameplayEffects` modifying the movespeed yet, the `CurrentValue` is also 600 u/s. 
>- If she gets a temporary 50 u/s movespeed buff, the `BaseValue` stays the same at 600 u/s
>- The `CurrentValue` is now 600 + 50 for a total of 650 u/s.
>- When the movespeed buff expires, the `CurrentValue` reverts back to the `BaseValue` of 600 u/s.
>
>


>Tips Move pls (replicated now)
- Permanent changes to the `BaseValue` come from `Instant` `GameplayEffects`.
- `Duration` and `Infinite` `GameplayEffects` change the `CurrentValue`.
- Periodic `GameplayEffects` are treated like instant `GameplayEffects` and change the `BaseValue`.


---



## Derived Attributes
To make an `Attribute` that has some or all of its value derived from one or more other `Attributes`, use an `Infinite` `GameplayEffect` with one or more `Attribute Based` or [`MMC`](#concepts-ge-mmc) [`Modifiers`](#concepts-ge-mods). The `Derived Attribute` will update automatically when an `Attribute` that it depends on is updated.

The final formula for all the `Modifiers` on a `Derived Attribute` is the same formula for `Modifier Aggregators`. If you need calculations to happen in a certain order, do it all inside of an `MMC`.

```
((CurrentValue + Additive) * Multiplicitive) / Division
```

**Note:** If playing with multiple clients in PIE, you need to disable `Run Under One Process` in the Editor Preferences otherwise the `Derived Attributes` will not update when their independent `Attributes` update on clients other than the first.

In this example, we have an `Infinite` `GameplayEffect` that derives the value of `TestAttrA` from the `Attributes`, `TestAttrB` and `TestAttrC`, in the formula `TestAttrA = (TestAttrA + TestAttrB) * ( 2 * TestAttrC)`. `TestAttrA` recalculates its value automatically whenever any of the `Attributes` update their values.

![Derived Attribute Example](https://github.com/tranek/GASDocumentation/raw/master/Images/derivedattribute.png)

## Meta Attributes

*Meta Attributes* are treated as placeholders for temporary values that are intended to interact with `Attributes`. 
 - For example, we commonly define damage as a *Meta Attribute*.
 - Despite being a good design pattern, they are not mandatory.


### How Meta Attributes work
- `Meta Attributes` provide a good logical separation when handling gameplay calculations.
- Instead of a `GameplayEffect` directly changing our health `Attribute`, we use a `Meta Attribute` called damage as a placeholder.
- The damage value can be modified with buffs and debuffs in an [`GameplayEffectExecutionCalculation`](#concepts-ge-ec).
- This can be further manipulated in the `AttributeSet`. 
- `Meta Attributes` are not typically replicated.


> [!NOTE] For example:
>- Subtracting the damage from a current shield `Attribute`, before finally subtracting the remainder from the health `Attribute`. 
>- The damage `Meta Attribute` has no persistence between `GameplayEffects` and is overriden by every one. 

#### Working Example
When considering how to calculate damage, ask yourself:
-  "How much damage did we do?" 
-  "What do we do with this damage?". 

This logical separation means our `Gameplay Effects` and `Execution Calculations` don't need to know how the Target handles the damage. 

- `Gameplay Effect` determines how much damage
- `AttributeSet` decides what to do with that damage.

#### Key Points for Usage

- Not all characters may have the same `Attributes`, especially if you use subclassed `AttributeSets`.
- The base `AttributeSet` class may only have a health `Attribute`, but a subclassed `AttributeSet` may add a shield `Attribute`.
- The subclassed `AttributeSet` with the shield `Attribute` would distribute the damage received differently than the base `AttributeSet` class.

 - If you only ever have the following:
	- One `Execution Calculation` used for all instances of damage
	- One `Attribute Set` class shared by all characters,

- Then you may be fine doing the damage distribution inside of the `Execution Calculation`, and directly modifying those `Attributes`. 
	- You'll only be sacrificing flexibility, but that may be okay for you.

---









# Designing with Attributes and Attribute Sets in mind
*Gameplay Attributes* can represent anything, from the amount of health, a character's level - even the number of charges that a potion has.
- Any gameplay-related numbers that you want an *Actor* based object to own is perfect for this.
- They should generally only be modified by [`GameplayEffects`](#concepts-ge) so that the ASC can [predict](#concepts-p) the changes.


## Exposing to Details / Blueprint

- If you don't want an `Attribute` to show up in the Editor's list of `Attributes`, you can use the `Meta = (HideInDetailsView)` `property specifier`.




## Common User Errors when Developing

#### Dealing with / Declaring Maximum Values
- Often beginners to GAS will confuse `BaseValue` with a maximum value for an `Attribute` and try to treat it as such.
	- Maximum values that can change or are referenced in abilities or UI should be treated as separate.

>So right now, my game has a Base, Current, and Max.

##### Experimental Solution for Min/Max Values
- For hardcoded maximum and minimum values, there is a way to define a `DataTable` with `FAttributeMetaData` that can set maximum and minimum values.
	- **Epic's comment above the struct calls it a "work in progress".** 
	- See `AttributeSet.h` for more information. 


##### Recommended Solution for Min/Max Values

- To prevent confusion, I recommend that maximum values that can be referenced in abilities or UI be made as separate `Attributes` 
	- Hardcoded maximum and minimum values only used for clamping `Attributes` be defined as hardcoded floats in the `AttributeSet`.
- Clamping for changes to `CurrentValue` is discussed in `PreAttributeChange()`
	- See [[#]]
- Clamping for changes to the `BaseValue` is discussed [PostGameplayEffectExecute()](#concepts-as-postgameplayeffectexecute)  from `GameplayEffects`.



----



# Attribute Operations
>Tips Move pls
- Permanent changes to the `BaseValue` come from `Instant` `GameplayEffects`.
- `Duration` and `Infinite` `GameplayEffects` change the `CurrentValue`.
- Periodic `GameplayEffects` are treated like instant `GameplayEffects` and change the `BaseValue`.

## To-Do List
- [ ] Define *Gameplay Attributes* with *Attribute Sets*
- [ ] Choose a method to Initialize them with

The methods below are summarized here:
- Epic recommends an *Instant Gameplay Effect*
- Reading the header and doing that instead
>bruh

## Steps to Setup an Attribute Set

### Defining Attributes (C++)
It is recommended to add this block of macros to the top of every `AttributeSet` header file to automatically generate getter and setter functions.
```c++
// Uses macros from AttributeSet.h
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)
```


#### Example for replicated "Health"

###### Header
A replicated health attribute would be defined like this:

```c++
UPROPERTY(BlueprintReadOnly, Category = "Health", ReplicatedUsing = OnRep_Health)
FGameplayAttributeData Health;
ATTRIBUTE_ACCESSORS(UGDAttributeSetBase, Health)
```

Also define the `OnRep` function in the header:
```c++
UFUNCTION()
virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);
```

###### Implementation
The .cpp file for the `AttributeSet` should fill in the `OnRep` function with the `GAMEPLAYATTRIBUTE_REPNOTIFY` macro used by the prediction system:
```c++
void UGDAttributeSetBase::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(UGDAttributeSetBase, Health, OldHealth);
}
```

Finally, the `Attribute` needs to be added to `GetLifetimeReplicatedProps`:
```c++
void UGDAttributeSetBase::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME_CONDITION_NOTIFY(UGDAttributeSetBase, Health, COND_None, REPNOTIFY_Always);
}
```

`REPNOTIFY_Always` tells the `OnRep` function to trigger if the local value is already equal to the value being repped down from the Server (due to prediction). 

By default it won't trigger the `OnRep` function if the local value is the same as the value being repped down from the Server.

If the `Attribute` is not replicated like a `Meta Attribute`, then the `OnRep` and `GetLifetimeReplicatedProps` steps can be skipped.

### Initializing Attributes
There are multiple ways to initialize (set their `BaseValue` and consequently their `CurrentValue` to some initial value).

- Epic recommends using an instant `GameplayEffect`. 
	- This is the method used in the Sample Project too.
		- See `GE_HeroAttributes` Blueprint for an example of initializing
		- The application of this `GameplayEffect` happens in C++.

If you used the `ATTRIBUTE_ACCESSORS` macro when you defined your `Attributes`, an initialization function will automatically be generated on the `AttributeSet` for each `Attribute` that you can call at your leisure in C++.

```c++
// InitHealth(float InitialValue) is an automatically generated function for an Attribute 'Health' defined with the `ATTRIBUTE_ACCESSORS` macro
AttributeSet->InitHealth(100.0f);
```

See `AttributeSet.h` for more ways to initialize `Attributes`.

**Note:** Prior to 4.24, `FAttributeSetInitterDiscreteLevels` did not work with `FGameplayAttributeData`. It was created when `Attributes` were raw floats and will complain about `FGameplayAttributeData` not being `Plain Old Data` (`POD`). This is fixed in 4.24 https://issues.unrealengine.com/issue/UE-76557.





## Responding to Attribute Changes

### Functions Provided?
#### GetGameplayAttributeValueChangeDelegate(...)
To listen for when an `Attribute` changes to update the UI or other gameplay, use `UAbilitySystemComponent::GetGameplayAttributeValueChangeDelegate(FGameplayAttribute Attribute)`. This function returns a delegate that you can bind to that will be automatically called whenever an `Attribute` changes. The delegate provides a `FOnAttributeChangeData` parameter with the `NewValue`, `OldValue`, and `FGameplayEffectModCallbackData`. **Note:** The `FGameplayEffectModCallbackData` will only be set on the server.

```c++
AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(AttributeSetBase->GetHealthAttribute()).AddUObject(this, &AGDPlayerState::HealthChanged);
```

```c++
virtual void HealthChanged(const FOnAttributeChangeData& Data);
```

The Sample Project binds to the `Attribute` value changed delegates on the `GDPlayerState` to update the HUD and to respond to player death when health reaches zero.

A custom Blueprint node that wraps this into an `ASyncTask` is included in the Sample Project. It is used in the `UI_HUD` UMG Widget to update the health, mana, and stamina values. This `AsyncTask` will live forever until manually called `EndTask()`, which we do in the UMG Widget's `Destruct` event. See `AsyncTaskAttributeChanged.h/cpp`.

![Listen for Attribute Change BP Node](https://github.com/tranek/GASDocumentation/raw/master/Images/attributechange.png)


