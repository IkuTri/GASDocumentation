---
tags:
  - iku/draft
---
# Gameplay Effect Operations
Applying, Removing, and Stacking *Gameplay Effects* can be applied in many ways:
- from functions on [`GameplayAbilities`](#concepts-ga) 
- and functions on the `ASC` 

They usually take the form of `ApplyGameplayEffectTo`. 
>I guess this means using or overriding it somehow.
## Using Gameplay Effects
The different functions (that the base GE provides) are essentially convenience functions.
- Each function will eventually call `UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf()` on the *Object Target*.

### Applying a *Gameplay Effect* Independently 
Example concept: A projectile applying a GE to it's *Object Target*, which we presume has it's own ASC.
To apply *Gameplay Effects* outside of a *Gameplay Ability*:

- Get the *Object Target*'s ASC
- Use one of its functions to `ApplyGameplayEffectToSelf`.
>um, use that function, right? Or is the doc implying there might be a custom solution you could choose? (if it existed)
#### Binding
You can listen for when any `Duration` or `Infinite` *Gameplay Effects* are applied to an `ASC` by binding to its delegate:
```c++
AbilitySystemComponent->OnActiveGameplayEffectAddedDelegateToSelf.AddUObject(this, &APACharacterBase::OnActiveGameplayEffectAddedCallback);
```


####  Callback Function
```c++
virtual void OnActiveGameplayEffectAddedCallback(UAbilitySystemComponent* Target, const FGameplayEffectSpec& SpecApplied, FActiveGameplayEffectHandle ActiveHandle);
```

- The server will always call this function regardless of replication mode. 
- The autonomous proxy will only call this for replicated *Gameplay Effects* in `Full` and `Mixed` replication modes. 
- Simulated proxies will only call this in `Full` [replication mode](#concepts-asc-rm).


## Removing Gameplay Effects
*Gameplay Effects* can be removed in many ways. They usually take the form of `RemoveActiveGameplayEffect`. 
- Using functions on (from?) the Gameplay Ability (that removes it?)
- Using functions on the ASC itself.

The different functions are essentially convenience functions that will eventually call `FActiveGameplayEffectsContainer::RemoveActiveEffects()` on the `Target`.


To remove *Gameplay Effects* outside of a *Gameplay Ability*:
- Get the *Object Target*'s ASC
- Use one of its functions to `RemoveActiveGameplayEffect`.

### Binding
You can listen for when any `Duration` or `Infinite` *Gameplay Effects* are removed from an `ASC` by binding to its delegate:
```c++
AbilitySystemComponent->OnAnyGameplayEffectRemovedDelegate().AddUObject(this, &APACharacterBase::OnRemoveGameplayEffectCallback);
```
The callback function:
```c++
virtual void OnRemoveGameplayEffectCallback(const FActiveGameplayEffect& EffectRemoved);
```

The server will always call this function regardless of replication mode. The autonomous proxy will only call this for replicated *Gameplay Effects* in `Full` and `Mixed` replication modes. Simulated proxies will only call this in `Full` [replication mode](#concepts-asc-rm).

## Stacking Gameplay Effects
*Gameplay Effects* by default will apply new instances of the *Gameplay Effect Spec* that don't know or care about previously existing instances of the *Gameplay Effect Spec* on application. *Gameplay Effects* can be set to stack where instead of a new instance of the *Gameplay Effect Spec* is added, the currently existing `GameplayEffectSpec's` stack count is changed. Stacking only works for `Duration` and `Infinite` *Gameplay Effects*.

There are two types of stacking: Aggregate by Source and Aggregate by Target.

| Stacking Type       | Description                                                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Aggregate by Source | There is a separate instance of stacks per Source `ASC` on the Target. Each Source can apply X amount of stacks.                     |
| Aggregate by Target | There is only one instance of stacks on the Target regardless of Source. Each Source can apply a stack up to the shared stack limit. |

Stacks also have policies for expiration, duration refresh, and period reset. They have helpful hover tooltips in the *Gameplay Effect* Blueprint.

The Sample Project includes a custom Blueprint node that listens for *Gameplay Effect* stack changes. The HUD UMG Widget uses it to update the amount of passive armor stacks that the player has. This `AsyncTask` will live forever until manually called `EndTask()`, which we do in the UMG Widget's `Destruct` event. See `AsyncTaskEffectStackChanged`'s header and implementation file..

![Listen for GameplayEffect Stack Change BP Node](https://github.com/tranek/GASDocumentation/raw/master/Images/gestackchange.png)






<a name="concepts-ge-stacking"></a>
