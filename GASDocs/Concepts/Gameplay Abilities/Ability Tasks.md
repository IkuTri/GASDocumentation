---
tags:
  - GAS-System
  - iku/draft
---
# Ability Tasks

## An Ability Task Conceptually
>Going "conceptually" over "definition" due to that latter term being a C++ concept in itself


*Gameplay Abilities* only execute in one frame. This does not allow for much flexibility on its own. Alternatively, latent actions called *Ability Tasks* happen over time.

They're comprised of:
* A static function that creates new instances of the *Ability Task*
* Delegates that are broadcasted on when the *Ability Task* completes its purpose
* An `Activate()` function to start its main job, bind to external delegates, etc.
* An `OnDestroy()` function for cleanup, including external delegates that it bound to
* Callback functions for any external delegates that it bound to
* Member variables and any internal helper functions

They can also require responding to *delegates* fired at some point later in time.
 
### Built in *Ability Tasks*
GAS comes with many *Ability Tasks* out of the box:
* Tasks for moving Characters with *Root Motion Source*
* A task for playing animation montages
* Tasks for responding to *Attribute* changes
* Tasks for responding to *Gameplay Effect* changes
* Tasks for responding to player input
* and more

#### Hard Limit on *Ability Task* time.
The `UAbilityTask` constructor enforces a hardcoded game-wide maximum of 1000 concurrent *Ability Tasks* running at the same time. 

Keep this in mind when designing *Gameplay Abilities* for games that can have hundreds of characters in the world at the same time - like RTS games.


## Other Ability Task Notes

### *Ability Tasks* and Output Delegates
*Ability Tasks* can only declare one type of output delegate. 
- All of your output delegates must be of this type, regardless if they use the parameters or not. 
- Pass default values for unused delegate parameters.


### Setting *Ability Tasks* to run on Simulated Clients
>Seemingly mandatory for movement GAs. See below.
- *Ability Tasks* only run on the Client or Server that is running the owning *Gameplay Ability*.
- However, *AbilityTasks* can be set to run on simulated clients by setting *bSimulatedTask = true;* in the *Ability Task* constructor.
- This (enables) overriding *virtual void InitSimulatedTask(UGameplayTasksComponent& InGameplayTasksComponent);*, 
- Allows setting any member variables to be replicated. 

This is only useful in rare situations like movement *Ability Tasks* where you don't want to replicate every movement change but instead simulate the entire movement *Ability Task*. 
>Um, so, basically, every motion based Ability Task does this. A little much making this niche, huh?
- All of the *RootMotionSource* *AbilityTasks* do this. 
- See *AbilityTask_MoveToLocation.h/.cpp* as an example.

#### Root Motion Source Ability Tasks
GAS comes with *Ability Tasks* for moving *Characters* over time for things like knockbacks, complex jumps, pulls, and dashes using *Root Motion Sources* hooked into the *CharacterMovementComponent*.

`Note:` Predicting *RootMotionSource* *AbilityTasks* works up to engine version 4.19 and 4.25+. Prediction is bugged for engine versions 4.20-4.24; however, the *AbilityTasks* still perform their function in multiplayer with minor net corrections and work perfectly in single player. It is possible to cherry pick the [prediction fix](https://github.com/EpicGames/UnrealEngine/commit/94107438dd9f490e7b743f8e13da46927051bf33#diff-65f6196f9f28f560f95bd578e07e290c) from 4.25 into a custom 4.20-4.24 engine.

---
### Making *Ability Tasks* Tick
*Ability Tasks* can *Tick* if you set *bTickingTask = true;* in the *Ability Task* constructor and override *virtual void TickTask(float DeltaTime);*. 

This is useful when you need to lerp values smoothly across frames. 
- See *AbilityTask_MoveToLocation.h/.cpp* as an example.

---
# Sample Project Examples

The Sample comes with two custom *AbilityTasks*:
1. *PlayMontageAndWaitForEvent* is a combination of the default *PlayMontageAndWait* and *WaitGameplayEvent* *AbilityTasks*. This allows animation montages to send gameplay events from *AnimNotifies* back to the *GameplayAbility* that started them. Use this to trigger actions at specific times during animation montages.
2. *WaitReceiveDamage* listens for the *OwnerActor* to receive damage. The passive armor stacks *GameplayAbility* removes a stack of armor when the hero receives an instance of damage.


---
# Using Ability Tasks

## Example Ability Task Use
To create and activate an *Ability Task* in C++ (From *GDGA_FireGun.cpp*):
```c++
UGDAT_PlayMontageAndWaitForEvent* Task = UGDAT_PlayMontageAndWaitForEvent::PlayMontageAndWaitForEvent(this, NAME_None, MontageToPlay, FGameplayTagContainer(), 1.0f, NAME_None, false, 1.0f);
Task->OnBlendOut.AddDynamic(this, &UGDGA_FireGun::OnCompleted);
Task->OnCompleted.AddDynamic(this, &UGDGA_FireGun::OnCompleted);
Task->OnInterrupted.AddDynamic(this, &UGDGA_FireGun::OnCancelled);
Task->OnCancelled.AddDynamic(this, &UGDGA_FireGun::OnCancelled);
Task->EventReceived.AddDynamic(this, &UGDGA_FireGun::EventReceived);
Task->ReadyForActivation();
```

In Blueprint, we just use the Blueprint node that we create for the *Ability Task*.
- We don't have to call `ReadyForActivation()`. 
- That is automatically called by :
	- `Engine/Source/Editor/GameplayTasksEditor/Private/K2Node_LatentGameplayTaskCall.cpp`. 
- *K2Node_LatentGameplayTaskCall* also automatically calls `BeginSpawningActor()` and `FinishSpawningActor()` if they exist in your *Ability Task* class
	- (see `AbilityTask_WaitTargetData`). 
To reiterate,
- `K2Node_LatentGameplayTaskCall` only does automagic sorcery for Blueprint. 
- In C++, we have to manually call `ReadyForActivation()`, `BeginSpawningActor()`, and `FinishSpawningActor()`.
- To manually cancel an *Ability Task*, just call `EndTask()` on the *Ability Task* object in Blueprint (called *Async Task Proxy*) or in C++.

![Blueprint WaitTargetData AbilityTask](https://github.com/tranek/GASDocumentation/raw/master/Images/abilitytask.png)





<a name="concepts-at-rms"></a>



<a name="concepts-gc"></a>
