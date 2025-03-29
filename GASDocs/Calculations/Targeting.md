---
tags:
  - iku/draft
  - GAS-System/Advanced
---

# Targeting

### Target Data
[*FGameplayAbilityTargetData*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/FGameplayAbilityTargetData/index.html) is a generic structure for targeting data meant to be passed across the network. *TargetData* will typically hold *AActor*/*UObject* references, *FHitResults*, and other generic location/direction/origin information. However, you can subclass it to put essentially anything that you want inside of them as a simple means to [pass data between the client and server in *GameplayAbilities*](#concepts-ga-data). The base struct *FGameplayAbilityTargetData* is not meant to be used directly but instead subclassed. *GAS* comes with a few subclassed *FGameplayAbilityTargetData* structs out of the box located in *GameplayAbilityTargetTypes.h*.

*TargetData* is typically produced by [*Target Actors*](#concepts-targeting-actors) or **created manually** and consumed by [*AbilityTasks*](#concepts-at) and [*GameplayEffects*](#concepts-ge) via the [*EffectContext*](#concepts-ge-context). As a result of being in the *EffectContext*, [*Executions*](#concepts-ge-ec), [*MMCs*](#concepts-ge-mmc), [*GameplayCues*](#concepts-gc), and the functions on the backend of the [*AttributeSet*](#concepts-as) can access the *TargetData*.

We don't typically pass around the *FGameplayAbilityTargetData* directly, instead we use a [*FGameplayAbilityTargetDataHandle*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/FGameplayAbilityTargetDataHandle/index.html) which has an internal TArray of pointers to *FGameplayAbilityTargetData*. This intermediate struct provides support for polymorphism of the *TargetData*.

An example of inheritting from *FGameplayAbilityTargetData*:
```c++
USTRUCT(BlueprintType)
struct MYGAME_API FGameplayAbilityTargetData_CustomData : public FGameplayAbilityTargetData
{
    GENERATED_BODY()
public:

    FGameplayAbilityTargetData_CustomData()
    { }

    UPROPERTY()
    FName CoolName = NAME_None;

    UPROPERTY()
    FPredictionKey MyCoolPredictionKey;

    // This is required for all child structs of FGameplayAbilityTargetData
    virtual UScriptStruct* GetScriptStruct() const override
    {
    	return FGameplayAbilityTargetData_CustomData::StaticStruct();
    }

	// This is required for all child structs of FGameplayAbilityTargetData
    bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
    {
	    // The engine already defined NetSerialize for FName & FPredictionKey, thanks Epic!
        CoolName.NetSerialize(Ar, Map, bOutSuccess);
        MyCoolPredictionKey.NetSerialize(Ar, Map, bOutSuccess);
        bOutSuccess = true;
        return true;
    }
}

template<>
struct TStructOpsTypeTraits<FGameplayAbilityTargetData_CustomData> : public TStructOpsTypeTraitsBase2<FGameplayAbilityTargetData_CustomData>
{
	enum
	{
        WithNetSerializer = true // This is REQUIRED for FGameplayAbilityTargetDataHandle net serialization to work
	};
};
```
For adding the target data to a handle:
```c++
UFUNCTION(BlueprintPure)
FGameplayAbilityTargetDataHandle MakeTargetDataFromCustomName(const FName CustomName)
{
	// Create our target data type, 
	// Handle's automatically cleanup and delete this data when the handle is destructed, 
	// if you don't add this to a handle then be careful because this deals with memory management and memory leaks so its safe to just always add it to a handle at some point in the frame!
	FGameplayAbilityTargetData_CustomData* MyCustomData = new FGameplayAbilityTargetData_CustomData();
	// Setup the struct's information to use the inputted name and any other changes we may want to do
	MyCustomData->CoolName = CustomName;
	
	// Make our handle wrapper for Blueprint usage
	FGameplayAbilityTargetDataHandle Handle;
	// Add the target data to our handle
	Handle.Add(MyCustomData);
	// Output our handle to Blueprint
	return Handle
}
```

For getting values it requires doing type safety checking, because the only way to get values from the handle's target data is by using generic C/C++ casting for it which is *NOT* type safe which can cause object slicing and crashes. For type checking there are multiple ways of doing this(however you want honestly) two common ways are:
- Gameplay Tag(s): You can use a subclass hierarchy where you know that anytime a certain code architecture's functionality occurs, you can cast for the base parent type and get its gameplay tag(s) and then compare against those for casting for inherited classes.
- Script Struct & Static Structs: You can instead do direct class comparison(which can involve a lot of IF statements or making some template functions), below is an example of doing this but basically you can get the script struct from any *FGameplayAbilityTargetData*(this is a nice advantage of it being a *USTRUCT* and requiring any inherited classes to specify the struct type in *GetScriptStruct*) and compare if its the type you're looking for. Below is an example of using these functions for type checking:
```c++
UFUNCTION(BlueprintPure)
FName GetCoolNameFromTargetData(const FGameplayAbilityTargetDataHandle& Handle, const int Index)
{   
    // NOTE, there is two versions of this '::Get(int32 Index)' function; 
    // 1) const version that returns 'const FGameplayAbilityTargetData*', good for reading target data values 
    // 2) non-const version that returns 'FGameplayAbilityTargetData*', good for modifying target data values
    FGameplayAbilityTargetData* Data = Handle.Get(Index); // This will valid check the index for you 
    
    // Valid check we have something to use, null data means nothing to cast for
    if(Data == nullptr)
    {
       	return NAME_None;
    }
    // This is basically the type checking pass, static_cast does not have type safety, this is why we do this check.
    // If we don't do this then it will object slice the struct and thus we have no way of making sure its that type.
    if(Data->GetScriptStruct() == FGameplayAbilityTargetData_CustomData::StaticStruct())
    {
        // Here is when you would do the cast because we know its the correct type already
        FGameplayAbilityTargetData_CustomData* CustomData = static_cast<FGameplayAbilityTargetData_CustomData*>(Data);    
        return CustomData->CoolName;
    }
    return NAME_None;
}
```



<a name="concepts-targeting-actors"></a>
### Target Actors
*GameplayAbilities* spawn [*TargetActors*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/AGameplayAbilityTargetActor/index.html) with the *WaitTargetData* *AbilityTask* to visualize and capture targeting information from the world. *TargetActors* may optionally use [*GameplayAbilityWorldReticles*](#concepts-targeting-reticles) to display current targets. Upon confirmation, the targeting information is returned as [*TargetData*](#concepts-targeting-data) which can then be passed into *GameplayEffects*.
 
*TargetActors* are based on *AActor* so they can have any kind of visible component to represent **where** and **how** they are targeting such as static meshes or decals. Static meshes may be used to visualize placement of an object that your character will build. Decals may be used to show an area of effect on the ground. The Sample Project uses [*AGameplayAbilityTargetActor_GroundTrace*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/AGameplayAbilityTargetActor_Grou-/index.html) with a decal on the ground to represent the damage area of effect for the Meteor ability. They also don't need to display anything either. For example it wouldn't make sense to display anything for a hitscan gun that instantly traces a line to its target as used in [GASShooter](https://github.com/tranek/GASShooter).

They capture targeting information using basic traces or collision overlaps and convert the results as *FHitResults* or *AActor* arrays to *TargetData* depending on the *TargetActor* implementation. The *WaitTargetData* *AbilityTask* determines when the targets are confirmed through its *TEnumAsByte<EGameplayTargetingConfirmation::Type> ConfirmationType* parameter. When **not** using *TEnumAsByte<EGameplayTargetingConfirmation::Type::Instant*, the *TargetActor* typically performs the trace/overlap on *Tick()* and updates its location to the *FHitResult* depending on its implementation. While this performs a trace/overlap on *Tick()*, it's generally not terrible since it's not replicated and you typically don't have more than one (although you could have more) *TargetActor* running at a time. Just be aware that it uses *Tick()* and some complex *TargetActors* might do a lot on it like the rocket launcher's secondary ability in GASShooter. While tracing on *Tick()* is very responsive to the client, you may consider lowering the tick rate on the *TargetActor* if the performance hit is too much. In the case of *TEnumAsByte<EGameplayTargetingConfirmation::Type::Instant*, the *TargetActor* immediately spawns, produces *TargetData*, and destroys. *Tick()* is never called. 

| *EGameplayTargetingConfirmation::Type* | When targets are confirmed                                                                                                                                                                                                                                                                                                                                     |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| *Instant*                              | The targeting happens instantly without special logic or user input deciding when to 'fire'.                                                                                                                                                                                                                                                                   |
| *UserConfirmed*                        | The targeting happens when the user confirms the targeting when the [ability is bound to a *Confirm* input](#concepts-ga-input) or by calling *UAbilitySystemComponent::TargetConfirm()*. The *TargetActor* will also respond to a bound *Cancel* input or call to *UAbilitySystemComponent::TargetCancel()* to cancel targeting.                              |
| *Custom*                               | The GameplayTargeting Ability is responsible for deciding when the targeting data is ready by calling *UGameplayAbility::ConfirmTaskByInstanceName()*. The *TargetActor* will also respond to *UGameplayAbility::CancelTaskByInstanceName()* to cancel targeting.                                                                                              |
| *CustomMulti*                          | The GameplayTargeting Ability is responsible for deciding when the targeting data is ready by calling *UGameplayAbility::ConfirmTaskByInstanceName()*. The *TargetActor* will also respond to *UGameplayAbility::CancelTaskByInstanceName()* to cancel targeting. Should not end the *AbilityTask* upon data production.                                       |

Not every EGameplayTargetingConfirmation::Type is supported by every *TargetActor*. For example, *AGameplayAbilityTargetActor_GroundTrace* does not support *Instant* confirmation.

The *WaitTargetData* *AbilityTask* takes in a *AGameplayAbilityTargetActor* class as a parameter and will spawn an instance on each activation of the *AbilityTask* and will destroy the *TargetActor* when the *AbilityTask* ends. The *WaitTargetDataUsingActor* *AbilityTask* takes in an already spawned *TargetActor*, but still destroys it when the *AbilityTask* ends. Both of these *AbilityTasks* are inefficient in that they either spawn or require a newly spawned *TargetActor* for each use. They're great for prototyping, but in production you might explore optimizing it if you have cases where you are constantly producing *TargetData* like in the case of an automatic rifle. GASShooter has a custom subclass of [*AGameplayAbilityTargetActor*](https://github.com/tranek/GASShooter/blob/master/Source/GASShooter/Public/Characters/Abilities/GSGATA_Trace.h) and a new [*WaitTargetDataWithReusableActor*](https://github.com/tranek/GASShooter/blob/master/Source/GASShooter/Public/Characters/Abilities/AbilityTasks/GSAT_WaitTargetDataUsingActor.h) *AbilityTask* written from scratch that allows you to reuse a *TargetActor* without destroying it.

*TargetActors* are not replicated by default; however, they can be made to replicate if that makes sense in your game to show other players where the local player is targeting. They do include default functionality to communicate with the server via RPCs on the *WaitTargetData* *AbilityTask*. If the *TargetActor*'s *ShouldProduceTargetDataOnServer* property is set to *false*, then the client will RPC its *TargetData* to the server on confirmation via *CallServerSetReplicatedTargetData()* in *UAbilityTask_WaitTargetData::OnTargetDataReadyCallback()*. If *ShouldProduceTargetDataOnServer* is *true*, the client will send a generic confirm event, *EAbilityGenericReplicatedEvent::GenericConfirm*, RPC to the server in *UAbilityTask_WaitTargetData::OnTargetDataReadyCallback()* and the server will do the trace or overlap check upon receiving the RPC to produce data on the server. If the client cancels the targeting, it will send a generic cancel event, *EAbilityGenericReplicatedEvent::GenericCancel*, RPC to the server in *UAbilityTask_WaitTargetData::OnTargetDataCancelledCallback*. As you can see, there are a lot of delegates on both the *TargetActor* and the *WaitTargetData* *AbilityTask*. The *TargetActor* responds to inputs to produce and broadcast *TargetData* ready, confirm, or cancel delegates. *WaitTargetData* listens to the *TargetActor*'s *TargetData* ready, confirm, and cancel delegates and relays that information back to the *GameplayAbility* and to the server. If you send *TargetData* to the server, you may want to do validation on the server to make sure the *TargetData* looks reasonable to prevent cheating. Producing the *TargetData* directly on the server avoids this issue entirely, but will potentially lead to mispredictions for the owning client.

Depending on the particular subclass of *AGameplayAbilityTargetActor* that you use, different *ExposeOnSpawn* parameters will be exposed on the *WaitTargetData* *AbilityTask* node. Some common parameters include:

| Common *TargetActor* Parameters | Definition                                                                                                                                                                                                                                                                                                               |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Debug                           | If *true*, it will draw debug tracing/overlapping information whenever the *TargetActor* performs a trace in non-shipping builds. Remember, non-*Instant* *TargetActors* will perform a trace on *Tick()* so these debug draw calls will also happen on *Tick()*.                                                        |
| Filter                          | [Optional] A special struct for filtering out (removing) *Actors* from the targets when the trace/overlap happens. Typical use cases are to filter out the player's *Pawn*, require targets be of a specific class. See [Target Data Filters](#concepts-target-data-filters) for more advanced use cases. |
| Reticle Class                   | [Optional] Subclass of *AGameplayAbilityWorldReticle* that the *TargetActor* will spawn.                                                                                                                                                                                                                                 |
| Reticle Parameters              | [Optional] Configure your Reticles. See [Reticles](#concepts-targeting-reticles).                                                                                                                                                                                                                                        |
| Start Location                  | A special struct for where tracing should start from. Typically this will be the player's viewpoint, a weapon muzzle, or the *Pawn*'s location.                                                                                                                                                                          |

With the default *TargetActor* classes, *Actors* are only valid targets when they are directly in the trace/overlap. If they leave the trace/overlap (they move or you look away), they are no longer valid. If you want the *TargetActor* to remember the last valid target(s), you will need to add this functionality to a custom *TargetActor* class. I refer to these as persistent targets as they will persist until the *TargetActor* receives confirmation or cancellation, the *TargetActor* finds a new valid target in its trace/overlap, or the target is no longer valid (destroyed). GASShooter uses persistent targets for its rocket launcher's secondary ability's homing rockets targeting.



<a name="concepts-target-data-filters"></a>
### Target Data Filters
Using both the *Make GameplayTargetDataFilter* and *Make Filter Handle* nodes, you can filter out the player's *Pawn* or select only a specific class. If you need more advanced filtering, you can subclass *FGameplayTargetDataFilter* and override the *FilterPassesForActor* function. 
```c++
USTRUCT(BlueprintType)
struct GASDOCUMENTATION_API FGDNameTargetDataFilter : public FGameplayTargetDataFilter
{
	GENERATED_BODY()

	/** Returns true if the actor passes the filter and will be targeted */
	virtual bool FilterPassesForActor(const AActor* ActorToBeFiltered) const override;
};
```

However, this will not work directly into the *Wait Target Data* node as it requires a *FGameplayTargetDataFilterHandle*. A new custom *Make Filter Handle* must be made to accept the subclass:
```c++
FGameplayTargetDataFilterHandle UGDTargetDataFilterBlueprintLibrary::MakeGDNameFilterHandle(FGDNameTargetDataFilter Filter, AActor* FilterActor)
{
	FGameplayTargetDataFilter* NewFilter = new FGDNameTargetDataFilter(Filter);
	NewFilter->InitializeFilterContext(FilterActor);

	FGameplayTargetDataFilterHandle FilterHandle;
	FilterHandle.Filter = TSharedPtr<FGameplayTargetDataFilter>(NewFilter);
	return FilterHandle;
}
```


