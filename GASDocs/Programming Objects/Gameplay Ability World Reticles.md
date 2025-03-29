---
tags:
  - GAS-System
  - iku/draft
---


# Gameplay Ability World Reticles
[*AGameplayAbilityWorldReticles*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/AGameplayAbilityWorldReticle/index.html) (*Reticles*) visualize **who** you are targeting when targeting with non-*Instant* confirmed [*TargetActors*](#concepts-targeting-actors). 


- *TargetActors* are responsible for the spawn and destroy lifetimes for all *Reticles*.
- *Reticles* are *AActors* so they can use any kind of visual component for representation. 
- A common implementation as seen in [GASShooter](https://github.com/tranek/GASShooter) is to use a *WidgetComponent* to display a UMG Widget in screen space (always facing the player's camera). 
- *Reticles* do not know which *AActor* that they're on, but you could subclass in that functionality on a custom *TargetActor*.
- *TargetActors* will typically update the *Reticle*'s location to the target's location on every *Tick()*.

GASShooter uses *Reticles* to show locked-on targets for the rocket launcher's secondary ability's homing rockets. 
- The red indicator on the enemy is the *Reticle*. 
- The similar white image is the rocket launcher's crosshair.
![Reticles in GASShooter](https://github.com/tranek/GASDocumentation/raw/master/Images/gameplayabilityworldreticle.png)

*Reticles* come with a handful of *BlueprintImplementableEvents* for designers (they're intended to be developed in Blueprints):

```C++
/** Called whenever bIsTargetValid changes value. */
UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void OnValidTargetChanged(bool bNewValue);

/** Called whenever bIsTargetAnActor changes value. */
UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void OnTargetingAnActor(bool bNewValue);

UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void OnParametersInitialized();

UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void SetReticleMaterialParamFloat(FName ParamName, float value);

UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void SetReticleMaterialParamVector(FName ParamName, FVector value);
```

*Reticles* can optionally use [*FWorldReticleParameters*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/FWorldReticleParameters/index.html) provided by the *TargetActor* for configuration. 


The default struct only provides one variable *FVector AOEScale*. While you can technically subclass this struct, the *TargetActor* will only accept the base struct.

It seems a little short-sighted to not allow this to be subclassed with default *TargetActors*. However, if you make your own custom *TargetActor*, you can provide your own custom reticle parameters struct and manually pass it to your subclass of *AGameplayAbilityWorldReticles* when you spawn them.


### Replication
*Reticles* are not replicated by default, but can be made replicated if it makes sense for your game to show other players who the local player is targeting.

*Reticles* will only display on the current valid target with the default *TargetActors*.

For example, if you're using a *AGameplayAbilityTargetActor_SingleLineTrace* to trace for a target:
- the *Reticle* will only appear when the enemy is directly in the trace path.
- If you look away, the enemy is no longer a valid target and the *Reticle* will disappear.
- If you want the *Reticle* to stay on the last valid target, you will want to customize your *TargetActor* to remember the last valid target and keep the *Reticle* on them.

I refer to these as *"Persistent Targets"* as they will persist until the *Target Actor* receives confirmation or cancellation, the *Target Actor* finds a new valid target in its trace/overlap, or the target is no longer valid (destroyed).  GASShooter uses persistent targets for its rocket launcher's secondary ability's homing rockets targeting.

