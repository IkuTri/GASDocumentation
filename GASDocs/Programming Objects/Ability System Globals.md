---
tags:
  - iku/fine
---

# *Ability System Globals*
Known as [`UAbilitySystemGlobals`](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UAbilitySystemGlobals/index.html) this class holds global information about GAS.  Most of the variables can be set from the `DefaultGame.ini`.
- Generally you won't have to interact with this class, but you should be aware of its existence. 
## Subclassing *Ability System Globals*
If you need to subclass things like the *Gameplay Cue Manager* or the *Gameplay Effect Context*, do that through the `AbilitySystemGlobals`.

###  Set the class name in the `DefaultGame.ini`:
```
[/Script/GameplayAbilities.AbilitySystemGlobals]
AbilitySystemGlobalsClassName="/Script/ParagonAssets.PAAbilitySystemGlobals"
```

### Automatic Calling in 5.3+
>There's no info between setting up your DefaultGame.ini, and the next header. Perhaps since it mentions this below process is automatic in 5.3+, the docs might presume it's fine and good to go after this sole first step.
#### Using `InitGlobalData()` between Unreal Engine 4.24 and 5.2
Between UE 4.24 and 5.2, it is necessary to call `UAbilitySystemGlobals::Get().InitGlobalData()` to use [`TargetData`](#concepts-targeting-data).
- Otherwise you will get errors related to `ScriptStructCache` and clients will be disconnected from the server. 
- This function only needs to be called once in a project. 

- Fortnite calls it from `UAssetManager::StartInitialLoading()` 
- Paragon called it from `UEngine::Init()`.

- I find that putting it in `UAssetManager::StartInitialLoading()` is a good place as shown in the Sample Project. 
- I would consider this boilerplate code that you should copy into your project to avoid issues with `TargetData`. 
- Starting in 5.3 it is called automatically.

If you run into a crash while using the `AbilitySystemGlobals` `GlobalAttributeSetDefaultsTableNames`, you may need to call `UAbilitySystemGlobals::Get().InitGlobalData()` later like Fortnite in the `AssetManager` or in the `GameInstance`.



