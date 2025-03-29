---
tags:
  - GAS-System/Advanced
  - Networking
  - Prediction
  - iku/draft
---
# Prediction
GAS comes out of the box with support for client-side prediction; however, it does not predict everything.
The definitive source for GAS-related prediction is `GameplayPrediction.h` in the plugin source code.

>There are other types of prediction. If I knew anything about them, I would put them here. the subtitling is for ease of using the find function.
## About Client Side Prediction
To expand on this: Client-side prediction in GAS means that the client does not have to wait for the server's permission to activate a *Gameplay Ability* and apply *Gameplay Effects*. 

- It can "predict" the server giving it permission to do this and predict the targets that it would apply *Gameplay Effects* to. 
- The *Game Server* then runs the *Gameplay Ability* network latency-time after the client activates and tells the client if he was correct or not in his predictions. 
- If the *Game Client* was wrong in any of his predictions, he will "roll back" his changes from his "mispredictions" to match the server.

- *Instant Gameplay Effects* (like *Cost GEs*) that change *Attributes* can be predicted on yourself seamlessly,
- predicting *Instant* *Attribute* changes to other characters will show a brief anomaly or "blip" in their *Attributes*. 
- Predicted *Instant* *Gameplay Effects* are actually treated like *Infinite* *Gameplay Effects* so that they can be rolled back if *mispredicted*. 
- When the server's *Gameplay Effect* is applied, there potentially exists two of the same *Gameplay Effect's* causing the *Modifier* to be applied twice or not at all for a brief moment.
	- It will eventually correct itself but sometimes the blip is noticeable to players.


### What to use Client Side Prediction on
Epic's mindset is to only predict what you "can get away with".

- For example, Paragon and Fortnite do not predict damage. 
- Most likely they use an [[Gameplay Effect Execution Calculation |Execute Calc]] for their damage which cannot be predicted anyway.
- This is not to say that you can't try to predict certain things like damage. 
	- By all means if you do it and it works well for you then that's great.

> ... we are also not all in on a "predict everything: seamlessly and automatically" solution. We still feel player prediction is best kept to a minimum (meaning: predict the minimum amount of stuff you can get away with).

Source: *Dave Ratti from Epic's comment from the new [[Questions and Answers from Dave Ratti - Part 1]]*
Or [[Questions and Answers from Dave Ratti - Part 2]]


---

## Prediction Concepts

### Prediction Key
GAS's prediction works on the concept of a *Prediction Key*. This is an integer identifier that the client generates activating a *Gameplay Ability*.
* This is the *Activation Prediction Key*.
* Client sends this prediction key to the server with `CallServerTryActivateAbility()`.
* Client adds this prediction key to all *Gameplay Effects* that it applies while the prediction key is valid.
* Client's prediction key falls out of scope. 
* Further predicted effects in the same *Gameplay Ability* need a new [Scoped Prediction Window](#concepts-p-windows).


* Server receives the prediction key from the client.
* Server adds this prediction key to all *Gameplay Effects* that it applies.
* Server replicates the prediction key back to the client.


* Client receives replicated *Gameplay Effects* from the server with the prediction key used to apply them. If any of the replicated *GameplayEffects* match the *GameplayEffects* that the client applied with the same prediction key, they were predicted correctly. There will temporarily be two copies of the *GameplayEffect* on the target until the client removes its predicted one.
* Client receives the prediction key back from the server. This is the *Replicated Prediction Key*.
	- This prediction key is now marked stale.
	* Client removes **all** *Gameplay Effects* that it created with the now stale replicated prediction key. 
* *Gameplay Effects* replicated by the server will persist.
* Any *Gameplay Effects* that the client added and didn't receive a matching replicated version from the server were mispredicted.

Prediction keys are guaranteed to be valid during an atomic grouping of instructions "window" in *GameplayAbilities* starting with *Activation* from the activation prediction key.

You can think of this as being only valid during one frame. 
- Any callbacks from latent action *AbilityTasks* will no longer have a valid prediction key unless the *AbilityTask* has a built-in Synch Point which generates a new [Scoped Prediction Window](#concepts-p-windows).
[[]]
### Creating New Prediction Windows in Abilities
To predict more actions in callbacks from *AbilityTasks*:
- we need to create a new *Scoped Prediction Window* 
- with a new *Scoped Prediction Key*. 

This is sometimes referred to as a *Synch Point* between the client and server.

Some *AbilityTasks* like all of the input related ones come with built-in functionality to create a new scoped prediction window, meaning atomic code in the *Ability Tasks* callbacks have a valid scoped prediction key to use. 

Other tasks like the *WaitDelay* task do not have built-in code to create a new scoped prediction window for its callback.

If you need to predict actions after an *Ability Task* that does not have built-in code to create a scoped prediction window like `WaitDelay`, we must manually do that using the `WaitNetSync` *Ability Task* with the option `OnlyServerWait`. 

When the client hits a `WaitNetSync` with `OnlyServerWait`, it generates a new scoped prediction key based on the *GameplayAbility's* activation prediction key, RPCs it to the server, and adds it to any new *Gameplay Effects* that it applies. 

When the server hits a `WaitNetSync` with `OnlyServerWait`, it waits until it receives the new scoped prediction key from the client before continuing.

This *Scoped Prediction Key* does the same dance as activation prediction keys - applied to *Gameplay Effects* and replicated back to clients to be marked stale. The scoped prediction key is valid until it falls out of scope, meaning the scoped prediction window has closed. So again, only atomic operations, nothing latent, can use the new scoped prediction key.

You can create as many scoped prediction windows as you need.

If you would like to add the synch point functionality to your own custom *AbilityTasks*, look at how the input ones essentially inject the *WaitNetSync* *Ability Task* code into them.

**Note:** When using *WaitNetSync*, this does block the server's *Gameplay Ability* from continuing execution until it hears from the client. This could potentially be abused by malicious users who hack the game and intentionally delay sending their new scoped prediction key. While Epic uses the *WaitNetSync* sparingly, it recommends potentially building a new version of the *AbilityTask* with a delay that automatically continues without the client if this is a concern for you.

The Sample Project uses *WaitNetSync* in the Sprint *Gameplay Ability* to create a new scoped prediction window every time we apply the stamina cost so that we can predict it. Ideally we want a valid prediction key when applying costs and cooldowns.

If you have a predicted *GameplayEffect* that is playing twice on the owning client, your prediction key is stale and you're experiencing the "redo" problem. You can usually solve this by putting a *WaitNetSync* *AbilityTask* with *OnlyServerWait* right before you apply the *Gameplay Effect* to create a new scoped prediction key.


### Predictively Spawning Actors
Spawning *Actors* predictively on clients is an advanced topic. GAS does not provide functionality to handle this out of the box (the *SpawnActor* *AbilityTask* only spawns the *Actor* on the server). The key concept is to spawn a replicated *Actor* on both the client and the server.

If the *Actor* is just cosmetic or doesn't serve any gameplay purpose, the simple solution is to override the *Actor's* *IsNetRelevantFor()* function to restrict the server from replicating to the owning client. The owning client would have his locally spawned version and the server and other clients would have the server's replicated version.
```c++
bool APAReplicatedActorExceptOwner::IsNetRelevantFor(const AActor * RealViewer, const AActor * ViewTarget, const FVector & SrcLocation) const
{
	return !IsOwnedBy(ViewTarget);
}
```

If the spawned *Actor* affects gameplay like a projectile that needs to predict damage, then you need advanced logic that is outside of the scope of this documentation. Look at how UnrealTournament predictively spawns projectiles on Epic Games' GitHub. They have a dummy projectile spawned only on the owning client that synchs up with the server's replicated projectile.

## Designing with Prediction in Mind:
### Problems that GAS's prediction implementation is trying to solve
1. "Can I do this?" Basic protocol for prediction.
2. "Undo" How to undo side effects when a prediction fails.
3. "Redo" How to avoid replaying side effects that we predicted locally but that also get replicated from the server.
4. "Completeness" How to be sure we /really/ predicted all side effects.
5. "Dependencies" How to manage dependent prediction and chains of predicted events.
6. "Override" How to override state predictively that is otherwise replicated/owned by the server.

Source: From `GameplayPrediction.h`
### What is predicted:
* *Gameplay Ability* activation
* Triggered Events
* *Gameplay Effect* application:
	* *Attribute* modification (EXCEPTIONS: Executions do not currently predict, only attribute modifiers)
	* *Gameplay Tag* modification
	* *Gameplay Cue* events (both from within predictive gameplay effect and on their own)
* Montages
* Movement (built into UE's *UCharacterMovement*)
### What is not predicted:
* *Gameplay Effect* removal
	* Options on dealing with this and it's knock-on effects are below
* *Gameplay Effect* periodic effects (dots ticking)
Source: From `GameplayPrediction.h`

### Prediction and Damage
Regarding predicting damage, I personally do not recommend it despite it being one of the first things that most people try when starting with GAS. 
While you can, doing so is tricky. If you mispredict applying damage, the player will see the enemy's health jump back up. 

### Prediction and Death
I especially do not recommend trying to predict death.
- Say you mispredict a *Character's* death and it starts ragdolling only to stop ragdolling and continue shooting at you when the server corrects it.



## Dealing with Not Predictable Code
### Option: Dealing with Gameplay Effect Removal by Predicting it's Inversion
While we can predict *Gameplay Effect* application, we cannot predict it's removal. 
One way that we can work around this limitation is to predict the inverse effect when we want to remove a *Gameplay Effect*. 

- Say we predict a movement speed slow of 40%. 
- We can predictively remove it by applying a movement speed buff of 40%. 
- Then remove both *Gameplay Effects* at the same time. 
- This is not appropriate for every scenario and support for predicting *Gameplay Effect* removal is still needed. 

Dave Ratti from Epic has expressed desire to add it to a [future iteration of GAS](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89).
### Option: Avoiding Gameplay Effect Inversion
Because we cannot predict the removal of *Gameplay Effects*, there might be another scenario where inversion won't work.
- For example, we cannot fully predict *Gameplay Ability* cooldowns.  (There is no inverse *Gameplay Effect* workaround for them.)

- The server's replicated *Cooldown GE* will exist on the client and any attempts to bypass this (with *Minimal* replication mode for example) will be rejected by the server. 
- This means clients with higher latencies take longer to tell the server to go on cooldown and to receive the removal of the server's *Cooldown GE*. 
- This means players with higher latencies will have a lower rate of fire than players with lower latencies, giving them a disadvantage against lower latency players. 

Fortnite avoids this issue by using custom bookkeeping instead of *Cooldown GEs*.
>Honestly, I think this is covered somewhere? Or is that just the ammo factoid?
- [ ] Update this if you can.

# Future of Prediction in GAS
- `GameplayPrediction.h` states in the future they could potentially add functionality for predicting *Gameplay Effect* removal and periodic *Gameplay Effects*.
- Dave Ratti from Epic has [expressed interest](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89) in fixing the *latency reconciliation* problem for predicting cooldowns, disadvantaging players with higher latencies versus players with lower latencies.
- The new [*Network Prediction* plugin](#concepts-p-npp) by Epic is expected to be fully interoperable with the GAS like the *Character Movement Component* was before it.
	- This is now the *Mover Plugin*

## Network Prediction Plugin
>Needs to actually be updated with new info, hold on

- [ ]  Added new info
### New Info
This is now the *Mover Component*, as part of the *Mover Plugin*

### Old Info
Epic recently started an initiative to replace the *Character Movement Component* with a new *Network Prediction* plugin. 

This plugin is still in its very early stages but is available to very early access on the Unreal Engine GitHub. It's too soon to tell which future version of the Engine that it will make its experimental beta debut in.

