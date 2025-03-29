---
tags:
  - GAS-System
  - GAS-System/Advanced
  - Networking
  - Replication
  - iku/fine
---
# *Gameplay Ability* Replication

### Replication Policy
Don't use this option. The name is misleading and you don't need it. 

[[Gameplay Ability Spec]]s are replicated from the server to the owning client by default.

As mentioned above, Gameplay Abilities* don't run on simulated proxies**. 
They use *Ability Tasks* and *Gameplay Cues* to replicate or RPC visual changes to the simulated proxies. 

Dave Ratti from Epic has stated his desire to [remove this option in the future](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89).


### Net Execution Policy
A *Gameplay Ability* has it's own *Net Execution Policy*.

- It determines who runs the *Gameplay Ability*, and in what order.
- It also determines if a *Gameplay Ability* will be locally [predicted](#concepts-p). 
- *Gameplay Abilities* use [*Ability Tasks*](#concepts-at) for actions that happen over time.
	- like waiting for an event, 
	- waiting for an attribute change
	- waiting for players to choose a target, 
	- or moving a *Character* with *Root Motion Source*. 

 Simulated clients will not run *Gameplay Abilities*. Instead, when the server runs the ability:
- anything that visually needs to play on the simulated proxies (like animation montages) will be replicated or RPC'd.
- This would be through *Ability Tasks* or [*GameplayCues*](#concepts-gc) for cosmetic things like sounds and particles.

- All *Gameplay Abilities* will have their `ActivateAbility()` function overridden with your gameplay logic. 
- Additional logic can be added to `EndAbility()` that runs when the *Gameplay Ability* completes or is canceled.


(Editor guessing) *Gameplay Abilities* (They) include default behavior for optional cost and cooldown *Gameplay Effects*.


| *Net Execution Policy* | Description                                                                                                                                                                                                         |     |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- |
| *Local Only*           | The *Gameplay Ability* is only run on the owning client. This could be useful for abilities that only make local cosmetic changes. Single player games should use *Server Only*.                                    |     |
| *Local Predicted*      | *Local Predicted* *Gameplay Abilities* activate first on the owning client and then on the server. The server's version will correct anything that the client predicted incorrectly. See [Prediction](#concepts-p). |     |
| *Server Only*          | The *Gameplay Ability* is only run on the server. Passive *Gameplay Abilities* will typically be *Server Only*. Single player games should use this.                                                                |     |
| *Server Initiated*     | *Server Initiated* *Gameplay Abilities* activate first on the server and then on the owning client. I personally haven't used these much if any.                                                                    |     |

### Net Security Policy
A *Gameplay Ability*'s *Net Security Policy* determines where should an ability execute on the network. It provides protection from clients attempting to execute restricted abilities.
>Switched back to tics.

- [ ] Check this and verify


| *Net Security Policy*   | Description                                                                                                                                        |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ClientOrServer`        | No security requirements. Client or server can trigger execution and termination of this ability freely.                                           |
| `ServerOnlyExecution`   | A client requesting execution of this ability will be ignored by the server. Clients can still request that the server cancel or end this ability. |
| `ServerOnlyTermination` | A client requesting cancellation or ending of this ability will be ignored by the server. Clients can still request execution of the ability.      |
| `ServerOnly`            | Server controls both execution and termination of this ability. A client making any requests will be ignored.                                      |
|                         |                                                                                                                                                    |

----




## Passing Data to Abilities
The general paradigm for *Gameplay Abilities* is 

1. Activate the Ability
2. Generate the Data (Editor assumes this is *Spec* )
3. Apply the Data (Editor is guessing to the *Target* of the GA or the GE in question.)
4. (Deactivate? Garbage Collect?????) End.

Tranek: Activate->Generate Data->Apply->End. 

### Passing External Data with *Gameplay Abilities*
Sometimes you need to act on existing / external data. *Gameplay Abilities* provide a few options:
- [ ] See overlap with *Gameplay Effects*

| Method                                            | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Activate *Gameplay Ability* by Event              | Activate a *Gameplay Ability* with an event containing a payload of data. <br>The event's payload is replicated from client to server for local predicted *Gameplay Abilities*.<br><br>Use the two *Optional Object* or the [*TargetData*](#concepts-targeting-data) variables for arbitrary data that does not fit any of the existing variables.<br><br>The downside to this is that it prevents you from activating the ability with an input bind.<br><br>To activate a *Gameplay Ability* by event, the *Gameplay Ability* must have its *Triggers* set up in the *Gameplay Ability*. <br><br>Assign a *GameplayTag* and pick an option for *GameplayEvent*. <br><br>To send the event, use the function<br>`UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(AActor* Actor, FGameplayTag EventTag, FGameplayEventData Payload)`. |
| Use `WaitGameplayEvent` *Ability Task*            | Use the *WaitGameplayEvent* *AbilityTask* to tell the *Gameplay Ability* to listen for an event with payload data after it activates. The event payload and the process to send it is the same as activating *Gameplay Abilities* by event. The downside to this is that events are not replicated by the *Ability Task* and should only be used for *Local Only* and *Server Only* *Gameplay Abilities*. You potentially could write your own *Ability Task* that will replicate the event payload.                                                                                                                                                                                                                                                                                                                                           |
| Use *TargetData*                                  | A custom *TargetData* struct is a good way to pass arbitrary data between the client and server.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Store Data on the *Owner Actor* or *Avatar Actor* | Use replicated variables stored on the *OwnerActor*, *AvatarActor*, or any other object that you can get a reference to. <br><br>This method is the most flexible and will work with *Gameplay Abilities* activated by input binds. <br>However, it does not guarantee the data will be synchronized from replication at the time of use. <br><br>You must ensure that ahead of time - meaning:<br>- if you set a replicated variable and then immediately activate a *Gameplay Ability* there is no guarantee the order that will happen on the receiver due to potential packet loss.                                                                                                                                                                                                                                                        |




## Ability Batching
Traditional *Gameplay Ability* lifecycle involves a minimum of two or three RPCs from the client to the server.

1. `CallServerTryActivateAbility()`
2. `ServerSetReplicatedTargetData()` (Optional)
3. `ServerEndAbility()`

If a *Gameplay Ability* performs all of these actions in one atomic grouping in a frame, we can optimize this workflow.

- This would involve batching (to combine) all two or three RPCs into one RPC. 
- *GAS* refers to this RPC optimization as *Ability Batching*.
- The common example of when to use *Ability Batching* is for hitscan guns. 
- Hitscan guns activate, do a line trace, send the [*TargetData*](#concepts-targeting-data) to the server, and end the ability all in one atomic group in one frame. 
- The [GASShooter](https://github.com/tranek/GASShooter) sample project demonstrates this technique for its hitscan guns.

### Example Semi-Automatic Weapon Replication
Semi-Automatic guns are the best case scenario and batch:
- The `CallServerTryActivateAbility()`, 
- A `ServerSetReplicatedTargetData()` (the bullet hit result), 
- And `ServerEndAbility()` into one RPC instead of three RPCs.
### Example: Automatic Weapon Replication
Full-Automatic/Burst guns batch:
- `CallServerTryActivateAbility()` 
- `ServerSetReplicatedTargetData()` for the first bullet into one RPC, instead of two RPCs. 
- Each subsequent bullet is its own `ServerSetReplicatedTargetData()` RPC. 
- Finally, `ServerEndAbility()` is sent as a separate RPC when the gun stops firing.

This is a worst case scenario where we only save one RPC on the first bullet instead of two.
##### Alternative Method for Automatic Fire
This scenario could have also been implemented with activating the ability via a [*Gameplay Event*](#concepts-ga-data).
- This would send the bullet's *TargetData* in with the *EventPayload* to the server from the client. 
- *TargetData* would have to be generated externally vs batching generating inside the ability itself.

### Enabling Ability Batching
*Ability Batching* is disabled by default on the [*ASC*](#concepts-asc). 

To enable *Ability Batching*, override ShouldDoServerAbilityRPCBatch() to return true:

```c++
virtual bool ShouldDoServerAbilityRPCBatch() const override { return true; }
```

#### Create Struct
 *Ability Batching* needs a `FScopedServerAbilityRPCBatcher` struct beforehand. It has special code in each of the functions that can be batched.
 
When `FScopedServerAbilityRPCBatcher` has an GA falls into scope, it will try to batch it.
 - It automatically RPCs this batch struct to the server in `UAbilitySystemComponent::EndServerAbilityRPCBatch()`. 
- It intercepts the call from the sending RPC to pack the message into a batch struct.


When `FScopedServerAbilityRPCBatcher` falls out of scope, any abilities activated will not try to batch. 
- The server receives the batch RPC in `UAbilitySystemComponent::ServerAbilityRPCBatch_Internal(FServerAbilityRPCBatch& BatchInfo).` 
- The `BatchInfo` parameter will contain flags for if the ability should end 
- and if input was pressed at the time of activation
- and the *TargetData* if that was included.

This is a good function to put a breakpoint on to confirm that your batching is working properly. 

#### Enable Logging with Cvar
Alternatively, use the cvar to enable special ability batching logging.
```Unreal_Console
AbilitySystem.ServerRPCBatching.Log 1
```

#### Code Snippet Example (In Use?)
This mechanism can only be done in C++ and can only activate abilities by their `FGameplayAbilitySpecHandle`.

GASShooter reuses the same batched *Gameplay Ability* for semi-automatic and full-automatic guns.
Both never directly call `EndAbility()`.
- (it is handled outside of the ability by a local-only ability 
	- that manages player input 
	- and the call to the batched ability based on the current firemode). 
- Since all of the RPCs must happen within the scope of the `FScopedServerAbilityRPCBatcher`, 
- I provide the `EndAbilityImmediately` parameter so that the controlling/managing local-only can specify:
	- whether this ability should batch the `EndAbility()` call (semi-automatic), 
	- or not batch the `EndAbility()` call (full-automatic) and the `EndAbility()` call will happen sometime later in its own RPC.

```c++
bool UGSAbilitySystemComponent::BatchRPCTryActivateAbility(FGameplayAbilitySpecHandle InAbilityHandle, bool EndAbilityImmediately)
{
	bool AbilityActivated = false;
	if (InAbilityHandle.IsValid())
	{
		FScopedServerAbilityRPCBatcher GSAbilityRPCBatcher(this, InAbilityHandle);
		AbilityActivated = TryActivateAbility(InAbilityHandle, true);

		if (EndAbilityImmediately)
		{
			FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(InAbilityHandle);
			if (AbilitySpec)
			{
				UGSGameplayAbility* GSAbility = Cast<UGSGameplayAbility>(AbilitySpec->GetPrimaryInstance());
				GSAbility->ExternalEndAbility();
			}
		}

		return AbilityActivated;
	}

	return AbilityActivated;
}
```


#### Blueprint Node Example from *Sample Project*
GASShooter exposes a Blueprint node to allow batching abilities which the aforementioned local-only ability uses to trigger the batched ability.

![Activate Batched Ability](https://github.com/tranek/GASDocumentation/raw/master/Images/batchabilityactivate.png)




## Additional Options


<a name="concepts-ga-definition-remotecancel"></a>
### Server Respects Remote Ability Cancellation
This option causes trouble more often than not. It means if the client's *Gameplay Ability* ends either due to cancellation or natural completion, it will force the server's version to end whether it completed or not. The latter issue is the important one, especially for locally predicted *Gameplay Abilities* used by players with high latencies. Generally you will want to disable this option.

<a name="concepts-ga-definition-repinputdirectly"></a>
### Replicate Input Directly
Setting this option will always replicate input press and release events to the server. Epic recommends not using this and instead relying on the *Generic Replicated Events* that are built into the existing input related [*Ability Tasks*](#concepts-at) if you have your [input bound to your *ASC*](#concepts-ga-input).

Epic's comment:
```c++
/** Direct Input state replication. These will be called if bReplicateInputDirectly is true on the ability and is generally not a good thing to use. (Instead, prefer to use Generic Replicated Events). */
UAbilitySystemComponent::ServerSetInputPressed()
```



## Example for Local Prediction:
### Activation sequence for **locally predicted** *Gameplay Abilities*:
1. **Owning client** calls *TryActivateAbility()*
2. Calls *InternalTryActivateAbility()*
3. Calls *CanActivateAbility()* and returns whether *GameplayTag* requirements are met, if the *ASC* can afford the cost, if the *Gameplay Ability* is not on cooldown, and if no other instances are currently active
4. Calls *CallServerTryActivateAbility()* and passes it the *Prediction Key* that it generates
5. Calls *CallActivateAbility()*
6. Calls *PreActivate()* Epic refers to this as "boilerplate init stuff"
7. Calls *ActivateAbility()* finally activating the ability

**Server** receives *CallServerTryActivateAbility()*
1. Calls *ServerTryActivateAbility()*
2. Calls *InternalServerTryActivateAbility()* 
3. Calls *InternalTryActivateAbility()*
4. Calls *CanActivateAbility()* and returns whether *GameplayTag* requirements are met, if the *ASC* can afford the cost, if the *Gameplay Ability* is not on cooldown, and if no other instances are currently active
5. Calls *ClientActivateAbilitySucceed()* if successful telling it to update its *ActivationInfo* that its activation was confirmed by the server and broadcasting the *OnConfirmDelegate* delegate. This is not the same as input confirmation.
6. Calls *CallActivateAbility()*
7. Calls *PreActivate()* Epic refers to this as "boilerplate init stuff"
8. Calls *ActivateAbility()* finally activating the ability

If at any time the server fails to activate, it will call *ClientActivateAbilityFailed()*, immediately terminating the client's *Gameplay Ability* and undoing any predicted changes.

---

