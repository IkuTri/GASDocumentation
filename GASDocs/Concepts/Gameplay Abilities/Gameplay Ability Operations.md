---
tags:
  - GAS-System/Ability
  - How-To
  - iku/check
  - iku/draft
---
# Gameplay Ability Operations

## Granting Abilities
Granting a *Gameplay Ability* to an *ASC* adds it to the *ASC's* list of *Activatable Abilities*

This allows it to activate the *Gameplay Ability* at will if it meets the *Gameplay Tag* requirements.
- We grant *Gameplay Abilities* on the server which then automatically replicates the [[Gameplay Ability Spec]] to the owning client. 
- Other clients / simulated proxies do not receive the *Gameplay Ability Spec*.

The Sample Project stores a `TArray<TSubclassOf<UGDGameplayAbility>>` on the *Character* class that it reads from and grants when the game starts:
```c++
void AGDCharacterBase::AddCharacterAbilities()
{
	// Grant abilities, but only on the server	
	if (Role != ROLE_Authority || !AbilitySystemComponent.IsValid() || AbilitySystemComponent->bCharacterAbilitiesGiven)
	{
		return;
	}

	for (TSubclassOf<UGDGameplayAbility>& StartupAbility : CharacterAbilities)
	{
		AbilitySystemComponent->GiveAbility(
			FGameplayAbilitySpec(StartupAbility, GetAbilityLevel(StartupAbility.GetDefaultObject()->AbilityID), static_cast<int32>(StartupAbility.GetDefaultObject()->AbilityInputID), this));
	}

	AbilitySystemComponent->bCharacterAbilitiesGiven = true;
}
```

When granting these *Gameplay Abilities*:
- we're creating *Gameplay Ability Specs* with the *UGameplayAbility* class,
- the ability level, 
- the input that it is bound to,
- and the *Source Object* or who gave this *Gameplay Ability* to this *ASC*.



## Activating Abilities
### Pre-Activation
Before a *Gameplay Ability* calls `UGameplayAbility::Activate()`, it calls `UGameplayAbility::CanActivateAbility()`. 
- This function checks if the owning *ASC* can afford the cost (`UGameplayAbility::CheckCost()`) 
- It also ensures that the *Gameplay Ability* is not on cooldown (`UGameplayAbility::CheckCooldown()`).


The default might not always be the desirable way to activate a *Gameplay Ability*. 

The *ASC* provides four other methods of activating *Gameplay Abilities*:   
- Via *Gameplay Tag*,
- Via *Gameplay Ability* class
- Via a *Gameplay Ability Spec* handle
- Or by a standard (Unreal Engine) *Event*. 

Unless mentioned specifically - Activating a *Gameplay Ability* ~~by event~~ allows you to [pass in a payload of data with the event](#concepts-ga-data).)

- [ ] Confirm this 
### Activating a Gameplay Ability with the default method:
If a *Gameplay Ability* is assigned an input action, it will be automatically activated if:
- the input is pressed
- it meets its *Gameplay Tag* requirements. 

>This might be each method in code, but I should confirm. If so, remember that you need the implementation, and not just this header code.
```c++
UFUNCTION(BlueprintCallable, Category = "Abilities")
bool TryActivateAbilitiesByTag(const FGameplayTagContainer& GameplayTagContainer, bool bAllowRemoteActivation = true);

UFUNCTION(BlueprintCallable, Category = "Abilities")
bool TryActivateAbilityByClass(TSubclassOf<UGameplayAbility> InAbilityToActivate, bool bAllowRemoteActivation = true);

bool TryActivateAbility(FGameplayAbilitySpecHandle AbilityToActivate, bool bAllowRemoteActivation = true);

bool TriggerAbilityFromGameplayEvent(FGameplayAbilitySpecHandle AbilityToTrigger, FGameplayAbilityActorInfo* ActorInfo, FGameplayTag Tag, const FGameplayEventData* Payload, UAbilitySystemComponent& Component);

FGameplayAbilitySpecHandle GiveAbilityAndActivateOnce(const FGameplayAbilitySpec& AbilitySpec, const FGameplayEventData* GameplayEventData);
```


### Via Unreal Engine Event
To activate a *Gameplay Ability* by event:

1. the *Gameplay Ability* must have its *Triggers* set up in the *Gameplay Ability*. 
2. Assign a *Gameplay Tag* 
3. Pick an option for *Gameplay Event*. 

To send the event, use the function:

```
UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(AActor* Actor, FGameplayTag EventTag, FGameplayEventData Payload)
```

**Notes:**
- Activating a *Gameplay Ability* by event allows you to pass in a payload with data.
 - When activating a *Gameplay Ability* from event in Blueprint, you must use the "Activate Ability From Event" node.
### Via a Gameplay Ability Trigger (Tag)
*Gameplay Ability* *Triggers* also allow you to activate the *Gameplay Ability* when a *GameplayTag* is added or removed.

**Note:** Don't forget to call `EndAbility()` when the *Gameplay Ability* should terminate unless you have a *Gameplay Ability* that will always run like a passive ability.


### Reporting Failed Activations with *Gameplay Tags*
Abilities have default logic to tell you why an ability activation failed. 

To enable this, you must set up the *Gameplay Tags* that correspond to the default failure cases.

>Wait, so add to default game ini but also where? in a project???
 - Add these tags (or your own naming convention) to your project:
 
```
+GameplayTagList=(Tag="Activation.Fail.BlockedByTags",DevComment="")
+GameplayTagList=(Tag="Activation.Fail.CantAffordCost",DevComment="")
+GameplayTagList=(Tag="Activation.Fail.IsDead",DevComment="")
+GameplayTagList=(Tag="Activation.Fail.MissingTags",DevComment="")
+GameplayTagList=(Tag="Activation.Fail.Networking",DevComment="")
+GameplayTagList=(Tag="Activation.Fail.OnCooldown",DevComment="")
```

Then add them to the [*GASDocumentation\Config\DefaultGame.ini*](https://github.com/tranek/GASDocumentation/blob/master/Config/DefaultGame.ini#L8-L13):
>These are in a example files
```
[/Script/GameplayAbilities.AbilitySystemGlobals]
ActivateFailIsDeadName=Activation.Fail.IsDead
ActivateFailCooldownName=Activation.Fail.OnCooldown
ActivateFailCostName=Activation.Fail.CantAffordCost
ActivateFailTagsBlockedName=Activation.Fail.BlockedByTags
ActivateFailTagsMissingName=Activation.Fail.MissingTags
ActivateFailNetworkingName=Activation.Fail.Networking
```

Now whenever an ability activation fails, this corresponding *Gameplay Tag* will be included in output log messages.
- You can also make them visible on the "showdebug AbilitySystem" hud.


>Is this a specific file in the Sample Project?
```
LogAbilitySystem: Display: InternalServerTryActivateAbility. Rejecting ClientActivation of Default__GA_FireGun_C. InternalTryActivateAbility failed: Activation.Fail.BlockedByTags
LogAbilitySystem: Display: ClientActivateAbilityFailed_Implementation. PredictionKey :109 Ability: Default__GA_FireGun_C
```

#### In Editor Example
![Activation Failed Tags Displayed in showdebug AbilitySystem](https://github.com/tranek/GASDocumentation/raw/master/Images/activationfailedtags.png)



## Getting Active Abilities
Beginners often ask: "How can I get the active ability?"

Tranek speculates that this is perhaps to set variables on it, or to cancel it. 

Regardless, the thinking clashes with how GAS is designed:
- More than one *Gameplay Ability* can be active at a time -- so there is no one "active ability". 
- Instead, you must search *Activatable Abilities*, a list of granted *Gameplay Abilities* that the *ASC* owns.
- You'd also be looking for the one matching the [*Asset* or *Granted* *GameplayTag*](#concepts-ga-tags) that you are looking for.

>I think if you wanted to emulate having a priority, you'd have to consider using a variable (Attribute) from an Attribute Set for it, right? And with this, does "Active" involve a default (or implied) tag to represent "active" ?

- [ ] Might be somewhere lol

*Activatable Abilities* works with `UAbilitySystemComponent::GetActivatableAbilities()`, and returns a `TArray<FGameplayAbilitySpec>` for you to iterate over.

## Canceling Abilities
To cancel a *Gameplay Ability* from within:
1. you call *CancelAbility()*. 
2. This will call *EndAbility()* 
3. and set its *WasCancelled* parameter to true.



To cancel a *Gameplay Ability* externally, the *ASC* provides a few functions:

```c++
/** Cancels the specified ability CDO. */
void CancelAbility(UGameplayAbility* Ability);	

/** Cancels the ability indicated by passed in spec handle. If handle is not found among reactivated abilities nothing happens. */
void CancelAbilityHandle(const FGameplayAbilitySpecHandle& AbilityHandle);

/** Cancel all abilities with the specified tags. Will not cancel the Ignore instance */
void CancelAbilities(const FGameplayTagContainer* WithTags=nullptr, const FGameplayTagContainer* WithoutTags=nullptr, UGameplayAbility* Ignore=nullptr);

/** Cancels all abilities regardless of tags. Will not cancel the ignore instance */
void CancelAllAbilities(UGameplayAbility* Ignore=nullptr);

/** Cancels all abilities and kills any remaining instanced abilities */
virtual void DestroyActiveState();
```

**Note:** I have found that *CancelAllAbilities* doesn't seem to work right if you have a *Non-Instanced* *Gameplay Abilities*. It seems to hit the *Non-Instanced* *Gameplay Ability* and give up. *CancelAbilities* can handle *Non-Instanced* *Gameplay Abilities* better and that is what the Sample Project uses (Jump is a non-instanced *Gameplay Ability*). Your mileage may vary.





----





## Leveling Up Abilities
There are two common methods for leveling up an ability:

| Level Up Method                            | Description                                                                                                                                                                                                      |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ungrant and Regrant at the New Level       | Ungrant (remove) the *Gameplay Ability* from the *ASC* and regrant it back at the next level on the server. This terminates the *Gameplay Ability* if it was active at the time.                                   |
| Increase the *GameplayAbilitySpec's* Level | On the server, find the *GameplayAbilitySpec*, increase its level, and mark it dirty so that replicates to the owning client. This method does not terminate the *Gameplay Ability* if it was active at the time. |

The main difference between the two methods is if you want active *Gameplay Abilities* to be canceled at the time of level up. You will most likely use both methods depending on your *Gameplay Abilities*. I recommend adding a *bool* to your `UGameplayAbility` subclass specifying which method to use.

----


## *Gameplay Tags* for *Gameplay Abilities*
- *Gameplay Abilities* come with *Gameplay Tag Containers* with built-in logic. 
- None of these *Gameplay Tags* are replicated.

| *GameplayTag Container*     | Description                                                                                                                                                                                     |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| *Ability Tags*              | *GameplayTags* that the *Gameplay Ability* owns. These are just *GameplayTags* to describe the *Gameplay Ability*.                                                                              |
| *Cancel Abilities with Tag* | Other *Gameplay Abilities* that have these *GameplayTags* in their *Ability Tags* will be canceled when this *Gameplay Ability* is activated.                                                   |
| *Block Abilities with Tag*  | Other *Gameplay Abilities* that have these *GameplayTags* in their *Ability Tags* are blocked from activating while this *Gameplay Ability* is active.                                          |
| *Activation Owned Tags*     | These *GameplayTags* are given to the *GameplayAbility's* owner while this *Gameplay Ability* is active. Remember these are not replicated.                                                     |
| *Activation Required Tags*  | This *Gameplay Ability* can only be activated if the owner has **all** of these *GameplayTags*.                                                                                                 |
| *Activation Blocked Tags*   | This *Gameplay Ability* cannot be activated if the owner has **any** of these *GameplayTags*.                                                                                                   |
| *Source Required Tags*      | This *Gameplay Ability* can only be activated if the *Source* has **all** of these *GameplayTags*. The *Source* *GameplayTags* are only set if the *Gameplay Ability* is triggered by an event. |
| *Source Blocked Tags*       | This *Gameplay Ability* cannot be activated if the *Source* has **any** of these *GameplayTags*. The *Source* *GameplayTags* are only set if the *Gameplay Ability* is triggered by an event.   |
| *Target Required Tags*      | This *Gameplay Ability* can only be activated if the *Target* has **all** of these *GameplayTags*. The *Target* *GameplayTags* are only set if the *Gameplay Ability* is triggered by an event. |
| *Target Blocked Tags*       | This *Gameplay Ability* cannot be activated if the *Target* has **any** of these *GameplayTags*. The *Target* *GameplayTags* are only set if the *Gameplay Ability* is triggered by an event.   |



## Instancing Gameplay Abilities
### Gameplay Ability Instancing
A *Gameplay Ability* has what's called an *Instancing Policy*.
It determines if, and how, the *Gameplay Ability* is instanced when it does get activated.

| *Instancing Policy*     | Description                                                                                        | Example of when to use                                                                                                                                                                                                                                                                                                                                                                                               |
| ----------------------- | -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Instanced Per Actor     | Each *ASC* only has one instance of the *Gameplay Ability* that is reused between activations.     | This will probably be the *Instancing Policy* that you use the most. You can use it for any ability and provides persistence between activations. The designer is responsible for manually resetting any variables between activations that need it.                                                                                                                                                                 |
| Instanced Per Execution | Every time a *Gameplay Ability* is activated, a new instance of the *Gameplay Ability* is created. | The benefit of these *Gameplay Abilities* is that the variables are reset everytime you activate. These provide worse performance than *Instanced Per Actor* since they will spawn new *Gameplay Abilities* every time they activate. The Sample Project does not use any of these.                                                                                                                                  |
| Non-Instanced           | The *Gameplay Ability* operates on its *ClassDefaultObject*. No instances are created.             | This has the best performance of the three but is the most restrictive in what can be done with it. *Non-Instanced* *Gameplay Abilities* cannot store state, meaning no dynamic variables and no binding to *AbilityTask* delegates. The best place to use them is for frequently used simple abilities like minion basic attacks in a MOBA or RTS. The Sample Project's Jump *Gameplay Ability* is *Non-Instanced*. |

### Using *Gameplay Ability Spec*
Editor here, it seems the other heavily implied option for instancing an ability is to use the [[Gameplay Ability Spec]] feature.
- Tranek seems to have more written down for the *Gameplay Effect Spec*
- This might imply you want to figure out 
	- What should be a *Gameplay Ability* (or it's *Spec*)
	- What should be a *Gameplay Effect* (or that's *Spec*)
	- what to instance in general and when
