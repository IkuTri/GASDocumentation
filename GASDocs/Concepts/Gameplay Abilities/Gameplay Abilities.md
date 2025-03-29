---
tags:
  - GAS-System
  - iku/draft
---
# Gameplay Abilities
[*Gameplay Abilities*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/UGameplayAbility/index.html) (*GA*) are any actions or skills that an *Actor* can do in the game.
- More than one *Gameplay Ability* can be active at one time.
- These can be made in Blueprint or C++
- Some are included by default
- 
## Gameplay Ability Operations (Lifecycle)
The Editor decided to place this as it's own page. It covers the lifecycle.

[[Gameplay Ability Operations]]
- Granting
- Activating (and Pre-Activation Requirements)
- Getting
- Canceling
- Instancing (with *Spec*)

Tranek made two flowcharts:
### Flowchart: *Gameplay Ability*
Flowchart of a simple *Gameplay Ability*:
>Why simple vs advanced?
![Simple GameplayAbility Flowchart](https://github.com/tranek/GASDocumentation/raw/master/Images/abilityflowchartsimple.png)

### Flowchart: *Gameplay Ability* (Advanced)

Complex abilities can be implemented using multiple *Gameplay Abilities* that interact (activate, cancel, etc) with each other.
![Complex GameplayAbility Flowchart](https://github.com/tranek/GASDocumentation/raw/master/Images/abilityflowchartcomplex.png)




## Determining Use Cases for *Gameplay Abilities*
### Examples of *Gameplay Abilities*:
* Jumping
* Sprinting
* Shooting a gun
* Passively blocking an attack every X number of seconds
* Using a potion
* Opening a door
* Collecting a resource
* Constructing a building
### Anti-Examples of *Gameplay Abilities*:
Things that should not be implemented. These are not rules, just my recommendations. Your design and implementations may vary.
* Basic movement input
* Some interactions with UIs
* Don't use a *Gameplay Ability* to purchase an item from a store.

## Included Base Gameplay Ability Types
>I guess sorta a misnomer that's helpful.
### Passive Abilities
To implement passive *Gameplay Abilities* that automatically activate and run continuously:
1. override `UGameplayAbility::OnAvatarSet()` 
2. This is automatically called when a *Gameplay Ability* is granted, and the *Avatar Actor* is set.
3. call `TryActivateAbility()`.


Keep in mind that:
- Tranek recommends adding a *bool* to your custom *UGameplayAbility* class.
	- This would help specifying if the *Gameplay Ability* should be activated when granted.
- Passive *Gameplay Abilities* will typically have a [*Net Execution Policy*](#concepts-ga-net) of *Server Only*.


#### Sample Project Example:
The Sample Project does this for its passive armor stacking ability.

```c++
void UGDGameplayAbility::OnAvatarSet(const FGameplayAbilityActorInfo * ActorInfo, const FGameplayAbilitySpec & Spec)
{
	Super::OnAvatarSet(ActorInfo, Spec);

	if (bActivateAbilityOnGranted)
	{
		ActorInfo->AbilitySystemComponent->TryActivateAbility(Spec.Handle, false);
	}
}
```

Epic describes this function as the correct place to initiate passive abilities and to do *BeginPlay* type things.

### Leveling Abilities
*Gameplay Abilities* come with default functionality to:
- have a level to modify the amount of change to attributes

### Changing Functionality
- or to change the *GameplayAbility's* functionality.

## Optional Included Gameplay Effects
*Gameplay Abilities* come with functionality for optional costs and cooldowns. 
- They come in the form of a *Gameplay Effect*.

### Abilities that need a designed Cost
Costs are predefined amounts of *Attributes* for the usage of a *Gameplay Ability*.
- [[Cost Gameplay Effect]]
- Implemented with an *Instant* *Gameplay Effect*
### Abilities that have a designed Cooldown
Cooldowns are timers that prevent the reactivation of a *Gameplay Ability* until it expires.
- [[Cooldown Gameplay Effect]]
- Implemented with a *Duration* *Gameplay Effect*




# About Binding Input to the ASC
The ASC allows input actions to be bound to *Gameplay Abilities* when the GA is granted to it.
- To confirm, the ASC uses *Gameplay Tags* to check the GA's requirements.
- Each IA required to use the built-in *Ability Tasks*.
- Additionally, it accepts generic *Confirm* and *Cancel* inputs.  
	- Used by *Ability Tasks* for confirming things like *Target Actors* or canceling them.

## Steps Involved to Bind ASC

- First create an enum that translates the input action name to a byte.
- The enum name must match exactly to the name used for the input action in the project settings.
- The *DisplayName* does not matter.

## Binding to Input without Activating Abilities
If you don't want your *Gameplay Abilities* to automatically activate when an input is pressed but still bind them to input to use with *Ability Tasks*, you can add a new bool variable to your *UGameplayAbility* subclass, *bActivateOnInput*, that defaults to *true* and override *UAbilitySystemComponent::AbilityLocalInputPressed()*.

```c++
void UGSAbilitySystemComponent::AbilityLocalInputPressed(int32 InputID)
{
	// Consume the input if this InputID is overloaded with GenericConfirm/Cancel and the GenericConfim/Cancel callback is bound
	if (IsGenericConfirmInputBound(InputID))
	{
		LocalInputConfirm();
		return;
	}

	if (IsGenericCancelInputBound(InputID))
	{
		LocalInputCancel();
		return;
	}

	// ---------------------------------------------------------

	ABILITYLIST_SCOPE_LOCK();
	for (FGameplayAbilitySpec& Spec : ActivatableAbilities.Items)
	{
		if (Spec.InputID == InputID)
		{
			if (Spec.Ability)
			{
				Spec.InputPressed = true;
				if (Spec.IsActive())
				{
					if (Spec.Ability->bReplicateInputDirectly && IsOwnerActorAuthoritative() == false)
					{
						ServerSetInputPressed(Spec.Handle);
					}

					AbilitySpecInputPressed(Spec);

					// Invoke the InputPressed event. This is not replicated here. If someone is listening, they may replicate the InputPressed event to the server.
					InvokeReplicatedEvent(EAbilityGenericReplicatedEvent::InputPressed, Spec.Handle, Spec.ActivationInfo.GetActivationPredictionKey());
				}
				else
				{
					UGSGameplayAbility* GA = Cast<UGSGameplayAbility>(Spec.Ability);
					if (GA && GA->bActivateOnInput)
					{
						// Ability is not active, so try to activate it
						TryActivateAbility(Spec.Handle);
					}
				}
			}
		}
	}
}
```


### Example: *Sample Project* Header
```c++
UENUM(BlueprintType)
enum class EGDAbilityInputID : uint8
{
	// 0 None
	None			UMETA(DisplayName = "None"),
	// 1 Confirm
	Confirm			UMETA(DisplayName = "Confirm"),
	// 2 Cancel
	Cancel			UMETA(DisplayName = "Cancel"),
	// 3 LMB
	Ability1		UMETA(DisplayName = "Ability1"),
	// 4 RMB
	Ability2		UMETA(DisplayName = "Ability2"),
	// 5 Q
	Ability3		UMETA(DisplayName = "Ability3"),
	// 6 E
	Ability4		UMETA(DisplayName = "Ability4"),
	// 7 R
	Ability5		UMETA(DisplayName = "Ability5"),
	// 8 Sprint
	Sprint			UMETA(DisplayName = "Sprint"),
	// 9 Jump
	Jump			UMETA(DisplayName = "Jump")
};
```

If your *ASC* lives on the *Character*, then in *SetupPlayerInputComponent()* include the function for binding to the *ASC*:

### Example: *Sample Project* Implementation
```c++
// Bind to AbilitySystemComponent
FTopLevelAssetPath AbilityEnumAssetPath = FTopLevelAssetPath(FName("/Script/GASDocumentation"), FName("EGDAbilityInputID"));
AbilitySystemComponent->BindAbilityActivationToInputComponent(PlayerInputComponent, FGameplayAbilityInputBinds(FString("ConfirmTarget"),
	FString("CancelTarget"), AbilityEnumAssetPath, static_cast<int32>(EGDAbilityInputID::Confirm), static_cast<int32>(EGDAbilityInputID::Cancel)));
```


## Coding Hazards for *Gameplay Abilities* 
### Race Conditions
If your ASC  lives on the *Player State*, there is a potential race condition inside of `SetupPlayerInputComponent()` where the *Player State* may not have replicated to the client yet. 

Therefore, I recommend attempting to bind to input in `SetupPlayerInputComponent()` and `OnRep_PlayerState()`. 

`OnRep_PlayerState()` is not sufficient by itself because there could be a case where the *Actor* could have their *Input Component* be null when *Player State* replicates before the *Player Controller* tells the client to call `ClientRestart()` which creates the *Input Component*.


The Sample Project demonstrates attempting to bind in both locations with a *boolean* gating the process so it only actually binds the input once.

**Note:** In the Sample Project *Confirm* and *Cancel* in the *enum* don't match the input action names in the project settings (`ConfirmTarget` and `CancelTarget`), but we supply the mapping between them in `BindAbilityActivationToInputComponent()`. 


These are special since we supply the mapping and they don't have to match, but they can match. All other inputs in the *enum* must match the input action names in the project settings.

For *Gameplay Abilities* that will only ever be activated by one input (they will always exist in the same "slot" like a MOBA), I prefer to add a variable to my `UGameplayAbility` subclass where I can define their input. I can then read this from the `ClassDefaultObject` when granting the ability.



# Assorted Classes
## Ability Sets
*GameplayAbilitySets* are convenience `UDataAsset` classes for holding input bindings and lists of startup *Gameplay Abilities* for Characters with logic to grant the *Gameplay Abilities*. 
Subclasses can also include extra logic or properties. 


- Paragon had a *Gameplay Ability Set* per hero that included all of their given *Gameplay Abilities*.
### Tranek Commentary
I find this class to be unnecessary at least given what I've seen of it so far. 
The Sample Project handles all of the functionality of *GameplayAbilitySets* inside of the *GDCharacterBase* and its subclasses.

> Um okay but I think this is for huge projects. Like, I might actually need this lmao

### My Commentary
- Kinda wonder if this is a good fit for a perk pool, but unsure since would you really want to load all perks per weapon for a gamemode?



