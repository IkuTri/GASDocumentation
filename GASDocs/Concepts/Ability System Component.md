---
tags:
  - Unreal/Programming-Types/Component
  - iku/fine
---

# (Gameplay) *Ability System Component*
>The core of it all

## How the ASC Works
Source:  [*UAbilitySystemComponent*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UAbilitySystemComponent/index.html)

The *Ability System Component*, or "ASC" is the heart of GAS.

-  It's a `UActorComponent`that handles all interactions with the system.
- The *Actor* with the ASC attached to it is referred to as the *Owner Actor*.
- The physical representation of ASC is called the *Avatar Actor*.
- Any Unreal Engine *Actor* needs to "own" one to have:
	- *Gameplay Abilities*
	- *Gameplay Effects*
	- *Attributes*

The ASC manages and replicates what it possesses, with the exception of *Attributes* needing an *Attribute Set*. Developers are expected but not required to subclass this.
>(I guess Tranek means it manages the attributes via the sets, since you can have one huge set and not use all of it at once)

 The *Owner Actor* and the *Avatar Actor* can be the same *Actor*, or different ones. Keep in mind that they would still have one ASC, together or separate.
 
 - If your *Actor* will respawn, and need persistence of *Attributes* or *Gameplay Effects* between spawns, then the ideal location is on the *Player State*.
	 - For this, the *Owner Actor* is the *Player State* and the *Avatar Actor* is the hero's *Character* class. 
- Both together might be good in the case of a simple AI minion in a MOBA game. 

### Note about Player State based ASCs
If your ASC is on your *Player State*, then you will need to increase the *Net Update Frequency* of your *Player State*. 

It defaults to a very low value, and can cause delays or perceived lag before changes to things like *Attributes* and *Gameplay Tags* happen on the *clients*. 
- Be sure to enable [*Adaptive Network Update Frequency*](https://docs.unrealengine.com/en-US/Gameplay/Networking/Actors/Properties/index.html#adaptivenetworkupdatefrequency), *Fortnite* uses it.


## The ASC's Interface
Both, the *Owner Actor* and the *Avatar Actor* if different *Actors*, should implement the `IAbilitySystemInterface`. 
- *ASCs* interact with each other internally to the system by looking for this interface function.
- This interface has one function that must be overridden, which returns a pointer to its *ASC*. 

```C++
 UAbilitySystemComponent* GetAbilitySystemComponent() const
```


## What the ASC Holds

### Active Gameplay Effects
- The *ASC* holds its current active *Gameplay Effects* in 
```C++
FActiveGameplayEffectsContainer ActiveGameplayEffects
```

### Active Gameplay Abilities
The *ASC* holds its granted *Gameplay Abilities* in
```C++
FGameplayAbilitySpecContainer ActivatableAbilities
```

### Items held ? Is tis about the thing?

Any time that you plan to iterate over `ActivatableAbilities.Items`, be sure to add `ABILITYLIST_SCOPE_LOCK();` 
- Place this above your loop to lock the list from changing (due to removing an ability). 
- Every `ABILITYLIST_SCOPE_LOCK();` in scope increments *AbilityScopeLockCount* and then decrements when it falls out of scope. 
- Do not try to remove an ability inside the scope of *ABILITYLIST_SCOPE_LOCK();* 
	- (the clear ability functions check *AbilityScopeLockCount* internally to prevent removing abilities if the list is locked).

---


# Replication

## Available Replication Modes
The ASC defines three different replication modes for replicating *Gameplay Effects*, *Gameplay Tags*, and *Gameplay Cues*.
- Remember that *Attributes* are replicated by their *Attribute Set*.


| Replication Mode | When to Use                             | Description                                                                                                                    |
| ---------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| *Full*           | Single Player                           | Every *GameplayEffect* is replicated to every client.                                                                          |
| *Mixed*          | Multiplayer, player controlled *Actors* | *GameplayEffects* are only replicated to the owning client. Only *GameplayTags* and *GameplayCues* are replicated to everyone. |
| *Minimal*        | Multiplayer, AI controlled *Actors*     | *GameplayEffects* are never replicated to anyone. Only *GameplayTags* and *GameplayCues* are replicated to everyone.           |

*Mixed* replication mode expects the *Owner Actor's* (owner of the ASC being used) encompassing owner to be the *Controller*.
- The owner of the *Player State* is the *Controller* by default.
- The owner of the *Character* is not the *Controller* by default.
- If using *Mixed* replication mode with the *Owner Actor* not the *Player State*, then you need to call `SetOwner()` on the *Owner Actor* with a valid *Controller*.

Starting with 4.24, `PossessedBy()` now sets the owner of the *Pawn* to the new *Controller*.


# Setup and Initialization
## Owning Actor Constructs the ASC
ASCs are typically in the *OwnerActor's* constructor and explicitly marked replicated. **This must be done in C++**.

```c++
AGDPlayerState::AGDPlayerState()
{
	// Create ability system component, and set it to be explicitly replicated
	AbilitySystemComponent = CreateDefaultSubobject<UGDAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
	AbilitySystemComponent->SetIsReplicated(true);
	//...
}
```

The *ASC* needs to be initialized with its *Owner Actor* and *Avatar Actor* on both the server and the client. You want to initialize after the *Pawn's* *Controller* has been set (after possession). Single player games only need to worry about the server path.

For player controlled characters where the *ASC* lives on the *Pawn*, I typically initialize on the server in the *Pawn's* `PossessedBy()` function and initialize on the client in the *Player Controller's* `AcknowledgePossession()` function.

```c++
void APACharacterBase::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	if (AbilitySystemComponent)
	{
		AbilitySystemComponent->InitAbilityActorInfo(this, this);
	}

	// ASC MixedMode replication requires that the ASC Owner's Owner be the Controller.
	SetOwner(NewController);
}
```

```c++
void APAPlayerControllerBase::AcknowledgePossession(APawn* P)
{
	Super::AcknowledgePossession(P);

	APACharacterBase* CharacterBase = Cast<APACharacterBase>(P);
	if (CharacterBase)
	{
		CharacterBase->GetAbilitySystemComponent()->InitAbilityActorInfo(CharacterBase, CharacterBase);
	}

	//...
}
```



## Setup: Player State as Owner
For player controlled characters where the ASC lives on the *Player State*, I typically initialize the server in the *Pawn's* `PossessedBy()` function and initialize on the client in the *Pawn's* `OnRep_PlayerState()` function. This ensures that the *Player State* exists on the client.

```c++
// Server only
void AGDHeroCharacter::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		// Set the ASC on the Server. Clients do this in OnRep_PlayerState()
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// AI won't have PlayerControllers so we can init again here just to be sure. No harm in initing twice for heroes that have PlayerControllers.
		PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
	}
	
	//...
}
```

```c++
// Client only
void AGDHeroCharacter::OnRep_PlayerState()
{
	Super::OnRep_PlayerState();

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		// Set the ASC for clients. Server does this in PossessedBy.
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// Init ASC Actor Info for clients. Server will init its ASC when it possesses a new Actor.
		AbilitySystemComponent->InitAbilityActorInfo(PS, this);
	}

	// ...
}
```

If you get the error message below, then you did not initialize your ASC on the client.

```UnrealLog
LogAbilitySystem: Warning: Can't activate LocalOnly or LocalPredicted ability %s when not local!
```



---




# Common Situations for the ASC to handle
## Gameplay Tag Operations

### Accepting *Gameplay Tag Containers*
The *ASC* also has another helper function that takes in a *Gameplay Tag Container* as a parameter to assist in searching instead of manually iterating over the list of *Gameplay Ability Specs*. 


The `bOnlyAbilitiesThatSatisfyTagRequirements` parameter will only return *Gameplay Ability Specs* that:
- satisfy their *GameplayTag* requirements 
- can be activated right now. 

 For example, you could have two basic attack *Gameplay Abilities*,  one with a weapon and one with bare fists: 
 
- the correct one activates depending on if a weapon is equipped setting the *Gameplay Tag* requirement. 
	- See Epic's comment on the function for more information.

> I presume the code below is the location of Epic's comment.


```c++
UAbilitySystemComponent::GetActivatableGameplayAbilitySpecsByAllMatchingTags(const FGameplayTagContainer& GameplayTagContainer, TArray < struct FGameplayAbilitySpec* >& MatchingGameplayAbilities, bool bOnlyAbilitiesThatSatisfyTagRequirements = true)
```




Once you get the *FGameplayAbilitySpec* that you are looking for, you can call *IsActive()* on it.

### Responding to Changes in Gameplay Tags
The *ASC* provides a delegate for when *GameplayTags* are added or removed. It takes in a *EGameplayTagEventType* that can specify only to fire when the *GameplayTag* is added/removed or for any change in the *GameplayTag's* *TagMapCount*.

```c++
AbilitySystemComponent->RegisterGameplayTagEvent(FGameplayTag::RequestGameplayTag(FName("State.Debuff.Stun")), EGameplayTagEventType::NewOrRemoved).AddUObject(this, &AGDPlayerState::StunTagChanged);
```

The callback function has a parameter for the *GameplayTag* and the new *TagCount*.
```c++
virtual void StunTagChanged(const FGameplayTag CallbackTag, int32 NewCount);
```

### Loading Gameplay Tags from Plugin .ini Files
If you create a plugin with its own .ini files with *GameplayTags*, you can load that plugin's *GameplayTag* .ini directory in your plugin's *StartupModule()* function.

For example, this is how the CommonConversation plugin that comes with Unreal Engine does it:

```c++
void FCommonConversationRuntimeModule::StartupModule()
{
	TSharedPtr<IPlugin> ThisPlugin = IPluginManager::Get().FindPlugin(TEXT("CommonConversation"));
	check(ThisPlugin.IsValid());
	
	UGameplayTagsManager::Get().AddTagIniSearchPath(ThisPlugin->GetBaseDir() / TEXT("Config") / TEXT("Tags"));

	//...
}
```

This would look for the directory `Plugins\CommonConversation\Config\Tags` and load any .ini files with *GameplayTags* in them into your project when the Engine starts up if the plugin is enabled.


## Others
> If there were more, I'd put them here. WIP?
------------



