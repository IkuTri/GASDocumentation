---
tags:
  - GAS-System
  - iku/draft
---
# Gameplay Cues
## Concepts
*Gameplay Cues* (GC) execute non-gameplay related things. They're typically predicted, replicated, unless the latter is explicitly *Executed*, *Added*, or *Removed* locally.
- sound effects
- particle effects
- camera shakes


## Triggering Gameplay Cues
We trigger *Gameplay Cues* by sending a *Gameplay Tag* parented with it, attached with an *Event* type.
- The docs call this the *Gameplay Cue Tag*.
- This information is sent to the *Gameplay Cue Manager* via the ASC.
- *Gameplay Cue Notify* variant classes and other *Actors* that implement the *Interface* (`IGameplayCueInterface`) can subscribe to these events.
>I assume the Manager here is a function of the ASC.

Just to reiterate,  *Gameplay Tags* for your *Gameplay Cues* need to start with the parent *Gameplay Tag* of that Cue itself.

### Example of *Gameplay Cue Tag*
For example, a valid "*Gameplay Cue* *Gameplay Tag*" might be (Editor thinks this is the *Gameplay Cue Tag*)

>Made A.B.C into NATO phonetics, might help with verbosity if translated / tokenization. Shows hierarchy

```GAS
GameplayCue.Alpha.Beta.Charlie
```

### Event Types
- *Execute* Cue
- *Add / Added* Cue
- *Remove* Cue
### Gameplay Cue Notify / Notifies
There are two classes of *Gameplay Cue Notifies*, Static and Actor.
- Note that these can be local? Check
>I think Gameplay Cue as a concept is the tag setup, and the *Gameplay cue Notify* type links to the event type.

| GameplayCue Class                                                                                                                  | Event             | GameplayEffect Type  | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Notes                                                                                              |
| ---------------------------------------------------------------------------------------------------------------------------------- | ----------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| [GameplayCueNotify_Static](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UGameplayCueNotify_Static/index.html) | *Execute*         | Instant or Periodic  | Static *GameplayCueNotifies* operate on the `ClassDefaultObject` (meaning no instances) and are perfect for one-off effects like hit impacts.                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                    |
| [GameplayCueNotify_Actor](https://docs.unrealengine.com/en-US/BlueprintAPI/GameplayCueNotify/index.html)                           | *Add* or *Remove* | Duration or Infinite | An Actor *Gameplay Cue Notify* spawns a new instance when *Added*. Because these are instanced, they can do actions over time until they are Removed. These are good for looping sounds and particle effects that will be removed when the backing Duration or Infinite GameplayEffect is removed or by manually calling remove. These also come with options to manage how many are allowed to be Added at the same time so that multiple applications of the same effect only start the sounds or particles once. | check Auto Destroy on Remove otherwise subsequent calls to Add that GameplayCueTag won't work.<br> |

 They respond to different events and different types of *Gameplay Effects* can trigger them. 
 - Override the corresponding event with your logic.
- *Gameplay Cue Notifies* technically can respond to any of the events but this is typically how we use them.

**Note:** When using `GameplayCueNotify_Actor`, 
When using an ASC [Replication Mode](#concepts-asc-rm) other than Full, Add and Remove GC events will fire twice on Server players (listen server) - once for applying the GE and again from the "Minimal" NetMultiCast to the clients. However, WhileActive events will still only fire once. All events will only fire once on clients.





# In Practice
## *Sample Project* Details
The Sample Project includes two Gameplay Cue Notifies
- A `GameplayCueNotify_Actor` for stun and sprint effects.
- A `GameplayCueNotify_Static` for the FireGun's projectile impact. 

These GCs can be optimized further by [triggering them locally](#concepts-gc-local) instead of replicating them through a GE. I opted for showing the beginner way of using them in the Sample Project.


###### Within Details / Blueprint

From inside of a *Gameplay Effect* when it is successfully applied (not blocked by tags or immunity), fill in the *Gameplay Tags* of all the *Gameplay Cues* that should be triggered.

![GameplayCue Triggered from a GameplayEffect](https://github.com/tranek/GASDocumentation/raw/master/Images/gcfromge.png)

`UGameplayAbility` offers Blueprint nodes to Execute, Add, or Remove GameplayCues.

![GameplayCue Triggered from a GameplayAbility](https://github.com/tranek/GASDocumentation/raw/master/Images/gcfromga.png)


###### Within C++ / Code
In C++, you can call functions directly on the ASC (or expose them to Blueprint in your ASC subclass):


```c++
/** GameplayCues can also come on their own. These take an optional effect context to pass through hit result, etc */
void ExecuteGameplayCue(const FGameplayTag GameplayCueTag, FGameplayEffectContextHandle EffectContext = FGameplayEffectContextHandle());
void ExecuteGameplayCue(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);

/** Add a persistent gameplay cue */
void AddGameplayCue(const FGameplayTag GameplayCueTag, FGameplayEffectContextHandle EffectContext = FGameplayEffectContextHandle());
void AddGameplayCue(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);

/** Remove a persistent gameplay cue */
void RemoveGameplayCue(const FGameplayTag GameplayCueTag);
	
/** Removes any GameplayCue added on its own, i.e. not as part of a GameplayEffect. */
void RemoveAllGameplayCues();
```


---


----

## Idk

### Idk A

There are also the following sub-concepts
- Gameplay Cue Parameters
- 
### Idk B



---
### Gameplay Cue Manager
By default, the *Gameplay Cue Manager* will scan the entire game directory for *Gameplay Cue Notifies* and load them into *memory* on play. 

#### Declaring where the *Gameplay Cue Manager* Scans
We can change the path where the *Gameplay Cue Manager* scans by setting it in the `DefaultGame.ini`

```ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
GameplayCueNotifyPaths="/Game/GASDocumentation/Characters"
```

We do want the *Gameplay Cue Manager* to scan and find all of the *Gameplay Cue Notifies*.
- We don't want it to async load every single one on play. 
- This will put every *Gameplay Cue Notify* and all of their referenced sounds and particles into memory
- ...regardless if they're even used in a level.


In a large game like Paragon, this can be hundreds of megabytes of unneeded assets in memory and cause hitching and game freezes on startup.

An alternative to async loading every GameplayCue on startup is to only async load GameplayCues as they're triggered in-game. This mitigates the unnecessary memory usage and potential game hard freezes while async loading every GameplayCue in exchange for potentially delayed effects for the first time that a specific GameplayCue is triggered during play. This potential delay is nonexistent for SSDs. I have not tested on a HDD. If using this option in the UE Editor, there may be slight hitches or freezes during the first load of GameplayCues if the Editor needs to compile particle systems. This is not an issue in builds as the particle systems will already be compiled.

First we must subclass *UGameplayCueManager* and tell the *Ability System Globals* class to use our `UGameplayCueManager` subclass.
- This is in our *Default Game Properties* (`DefaultGame.ini.`)

```ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
GlobalGameplayCueManagerClass="/Script/ParagonAssets.PBGameplayCueManager"
```


In our `UGameplayCueManager` subclass, override `ShouldAsyncLoadRuntimeObjectLibraries()`.

```c++
virtual bool ShouldAsyncLoadRuntimeObjectLibraries() const override
{
	return false;
}
```




### Gameplay Cue Events
*Gameplay Cues* respond to specific `EGameplayCueEvents`:

| `EGameplayCueEvent` | Description                                                                                                                                                                                                                                                                                                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OnActive`          | Called when a *Gameplay Cue* is activated (added).                                                                                                                                                                                                                                                                             |
| `WhileActive`       | Called when *Gameplay Cue* is active, even if it wasn't actually just applied (Join in progress, etc). This is not Tick! It's called once just like `OnActive` when a `GameplayCueNotify_Actor` is added or becomes relevant. If you need Tick(), just use the `GameplayCueNotify_Actor`'s Tick(). It's an *AActor* after all. |
| `Removed`           | Called when a *Gameplay Cue* is removed. The Blueprint *Gameplay Cue* function that responds to this event is OnRemove.                                                                                                                                                                                                        |
| `Executed`          | Called when a *Gameplay Cue* is executed: instant effects or periodic Tick(). The Blueprint *Gameplay Cue* function that responds to this event is OnExecute.                                                                                                                                                                  |

- Use `OnActive` for anything in your *Gameplay Cue* that happen at the start of the *Gameplay Cue* but is okay if late joiners miss. 
- Use `WhileActive` for ongoing effects in the *Gameplay Cue* that you would want late joiners to see. 

#### Example
For example, if you have a *Gameplay Cue* for a tower structure in a MOBA exploding:
- You would put the initial explosion particle system and explosion sound in `OnActive` 
- and you would put any residual ongoing fire particles or sounds in the `WhileActive`. 

In this scenario, it wouldn't make sense for late joiners to replay the initial explosion from `OnActive`. You would want them to see the persistent, looping fire effects on the ground after the explosion happened from `WhileActive`. 

- `OnRemove` should clean up anything added in `OnActive` and `WhileActive`. 
- `WhileActive` will be called every time an *Actor* enters the relevancy range of a `GameplayCueNotify_Actor`. 
- `OnRemove` will be called every time an *Actor* leaves relevancy range of a `GameplayCueNotify_Actor`.


## More Concepts

### Local Gameplay Cues
The exposed functions for firing *Gameplay Cues* from *Gameplay Abilities* and the ASC are replicated by default. 
- Each *Gameplay Cue* event is a multicast *RPC*. 
	- This can cause a lot of RPCs.
	- GAS also enforces a maximum of two of the same *Gameplay Cue* RPCs per net update. 
- We avoid this by using *Local Gameplay Cues* where we can. 
- *Local Gameplay Cues* only Execute, Add, or Remove on the individual client.

Scenarios where we can use local GameplayCues:
* Projectile impacts
* Melee collision impacts
* GameplayCues fired from animation montages

#### Local *Gameplay Cue* Functions
You should add these to your ASC subclass:
>Note the filter and the AutoCreateRefTerm, maybe I can find notes on this later

If a *Gameplay Cue* was Added locally, it should be Removed locally. If it was Added via replication, it should be Removed via replication.
##### Header
>right?
```c++
UFUNCTION(BlueprintCallable, Category = "GameplayCue", Meta = (AutoCreateRefTerm = "GameplayCueParameters", GameplayTagFilter = "GameplayCue"))
void ExecuteGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);

UFUNCTION(BlueprintCallable, Category = "GameplayCue", Meta = (AutoCreateRefTerm = "GameplayCueParameters", GameplayTagFilter = "GameplayCue"))
void AddGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);

UFUNCTION(BlueprintCallable, Category = "GameplayCue", Meta = (AutoCreateRefTerm = "GameplayCueParameters", GameplayTagFilter = "GameplayCue"))
void RemoveGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);
```

##### Implementation
```c++
void UPAAbilitySystemComponent::ExecuteGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters & GameplayCueParameters)
{
	UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(GetOwner(), GameplayCueTag, EGameplayCueEvent::Type::Executed, GameplayCueParameters);
}

void UPAAbilitySystemComponent::AddGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters & GameplayCueParameters)
{
	UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(GetOwner(), GameplayCueTag, EGameplayCueEvent::Type::OnActive, GameplayCueParameters);
	UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(GetOwner(), GameplayCueTag, EGameplayCueEvent::Type::WhileActive, GameplayCueParameters);
}

void UPAAbilitySystemComponent::RemoveGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters & GameplayCueParameters)
{
	UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(GetOwner(), GameplayCueTag, EGameplayCueEvent::Type::Removed, GameplayCueParameters);
}
```


---

### Gameplay Cue Parameters
*Gameplay Cues* receive a *Gameplay Cue Parameters* Struct. (`FGameplayCueParameters`) 
It contains extra information as a parameter. (I think this means you store your info into what ends up being a parameter for other classes to read?)

- Manual triggers of Cues through a *Gameplay Ability* function means that you need to (manually) fill this Struct yourself. 

If you manually trigger the *Gameplay Cue* from a function on the *Gameplay Ability*, or the ASC, then you must manually fill in the GameplayCueParameters structure that is passed to the GameplayCue. 

If the *Gameplay Cue* is triggered by a *Gameplay Effect*, then the following variables are automatically filled in *Gameplay Cue Parameters*:

* AggregatedSourceTags
* AggregatedTargetTags
* GameplayEffectLevel
* AbilityLevel
* Effect Context
* Magnitude (if the *Gameplay Effect* has an *Attribute* for magnitude selected in the dropdown, above the GameplayCue tag container, and a corresponding Modifier that affects that Attribute)

The SourceObject variable in the GameplayCueParameters structure is potentially a good place to pass arbitrary data to the GameplayCue when triggering the GameplayCue manually.

**Note:** Some of the variables in the parameters structure like Instigator might already exist in the EffectContext. The EffectContext can also contain a FHitResult for location of where to spawn the GameplayCue in the world. Subclassing EffectContext is potentially a good way to pass more data to GameplayCues, especially those triggered by a GameplayEffect.

See the 3 functions in [UAbilitySystemGlobals](#concepts-asg) that populate the GameplayCueParameters structure for more information. They are virtual so you can override them to autopopulate more information.



```c++
/** Initialize GameplayCue Parameters */
virtual void InitGameplayCueParameters(FGameplayCueParameters& CueParameters, const FGameplayEffectSpecForRPC &Spec);
virtual void InitGameplayCueParameters_GESpec(FGameplayCueParameters& CueParameters, const FGameplayEffectSpec &Spec);
virtual void InitGameplayCueParameters(FGameplayCueParameters& CueParameters, const FGameplayEffectContextHandle& EffectContext);

```



## Advanced Tech
### Preventing Gameplay Cues from Firing
>Not part of the Gameplay Cue Manager?

Sometimes we don't want GameplayCues to fire. For example if we block an attack, we may not want to play the hit impact attached to the damage GameplayEffect or play a custom one instead. We can do this inside of [GameplayEffectExecutionCalculations](#concepts-ge-ec) by calling OutExecutionOutput.MarkGameplayCuesHandledManually() and then manually sending our GameplayCue event to the Target or Source's ASC.

If you never want any GameplayCues to fire on a specific ASC, you can set AbilitySystemComponent->bSuppressGameplayCues = true;.

### Gameplay Cue Batching
Each GameplayCue triggered is an unreliable NetMulticast RPC. In situations where we fire multiple GCs at the same time, there are a few optimization methods to condense them down into one RPC or save bandwidth by sending less data.
#### Method: Manual RPC
Say you have a shotgun that shoots eight pellets. That's eight trace and impact GameplayCues. [GASShooter](https://github.com/tranek/GASShooter) takes the lazy approach of combining them into one RPC by stashing all of the trace information into the [EffectContext](#concepts-ge-ec) as [TargetData](#concepts-targeting-data). While this reduces the RPCs from eight to one, it still sends a lot of data over the network in that one RPC (~500 bytes). A more optimized approach is to send an RPC with a custom struct where you efficiently encode the hit locations or maybe you give it a random seed number to recreate/approximate the impact locations on the receiving side. The clients would then unpack this custom struct and turn back into [locally executed GameplayCues](#concepts-gc-local).

How this works:
1. Declare a `FScopedGameplayCueSendContext`. 
	- This suppresses UGameplayCueManager::FlushPendingCues() until it falls out of scope
	- meaning all GameplayCues will be queued up until the FScopedGameplayCueSendContext falls out of scope.

2. Override UGameplayCueManager::FlushPendingCues()
- Have it merge GameplayCues that can be batched together
- Have it based on some custom GameplayTag into your custom struct
- RPC it to clients.

3. Clients receive the custom struct and unpack it into locally executed GameplayCues.

This method can also be used when you need specific parameters for your GameplayCues that don't fit with what GameplayCueParameters offer and you don't want to add them to the EffectContext like damage numbers, crit indicator, broken shield indicator, was fatal hit indicator, etc.

https://forums.unrealengine.com/development-discussion/c-gameplay-programming/1711546-fscopedgameplaycuesendcontext-gameplaycuemanager

<a name="concepts-gc-batching-gcsonge"></a>
#### Method: Multiple GCs on one GE
All of the *Gameplay Cues* on a *Gameplay Effect* are sent in one RPC already.
By default:

`UGameplayCueManager::InvokeGameplayCueAddedAndWhileActive_FromSpec() `
- This will send the whole `GameplayEffectSpec` (but converted to `FGameplayEffectSpecForRPC`) in the unreliable `NetMulticast`
	- This is regardless of the ASC's Replication Mode.
	- This could potentially be a lot of bandwidth depending on what is in the GameplayEffectSpec.
	- We can potentially optimize this by setting the *cvar* `AbilitySystem.AlwaysConvertGESpecToGCParams` to 1.
	- This will convert `GameplayEffectSpecs` to `FGameplayCueParameter` structures and RPC those instead of the whole `FGameplayEffectSpecForRPC`. 
	- This potentially saves bandwidth but also has less information, depending on how the GESpec is converted to `GameplayCueParameters` and what your GCs need to know.



<a name="concepts-gc-events"></a>
### Gameplay Cue Reliability
GameplayCues in general should be considered unreliable and thus unsuited for anything that directly affects gameplay.

#### Executed *Gameplay Cues*
These *Gameplay Cues* are applied via unreliable multicasts and are always unreliable.

#### *Gameplay Cues* applied from *Gameplay Effects*
Autonomous proxy reliably receives `OnActive`, `WhileActive`, and `OnRemove`.

- `FActiveGameplayEffectsContainer::NetDeltaSerialize()` calls `UAbilitySystemComponent::HandleDeferredGameplayCues()` 
	- This to call `OnActive` and `WhileActive`. 
- `FActiveGameplayEffectsContainer::RemoveActiveGameplayEffectGrantedTagsAndModifiers()` makes the call to `OnRemoved`.
	* Simulated proxies reliably receive `WhileActive` and `OnRemove`  
- `UAbilitySystemComponent::MinimalReplicationGameplayCues`'s replication calls `WhileActive` and `OnRemove`. 
	- The `OnActive` event is called by an unreliable multicast.
#### *Gameplay Cues* applied without a *Gameplay Effect*:
* Autonomous proxy reliably receives `OnRemove`  
- The `OnActive` and `WhileActive` events are called by an unreliable multicast.
* Simulated proxies reliably receive `WhileActive` and `OnRemove`  
- `UAbilitySystemComponent::MinimalReplicationGameplayCues`'s replication calls `WhileActive` and `OnRemove`. 
	- The `OnActive` event is called by an unreliable multicast.

If you need something in a *Gameplay Cue* to be 'reliable', then apply it from a *Gameplay Effect* and use `WhileActive` to add the FX and `OnRemove` to remove the FX.




