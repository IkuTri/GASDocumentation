---
tags:
  - GAS-System
  - iku/draft
---


# Q&A With Epic Game's Dave Ratti - Part 1
>While the repo I forked is MIT licensed, I rather keep Dave's comments intact. I did format *concepts* and `code` so you can see a bit better.
## Community Questions #1
>I used "part 1" vs "#1" so the file name is valid in obsidian / your computer
1

Source: [Dave Ratti responses to the Unreal Slackers Discord Server community questions about GAS](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89):

## Scoped Prediction Windows outside of a Designed Gameplay Ability
### Question
How can we create scoped prediction windows on demand outside or irrespective of `GameplayAbilities`? (#1)

### Example
For example, how can a fire and forget projectile locally predict a damage `GameplayEffect` when it hits an enemy?

### Answer
 The PredictionKey system is not really meant to do this. 
 
 Fundamentally this systems works by a client initiating a predictive action, telling the server about it with a key, and then both client and server running the same thing and associating predictive side effects with the given prediction key. 
 
 For example, “I am predictively activating an ability” or “I have produced target data and am going to predictively run the part of the ability graph after the `WaitTargetData` task”.

 With this pattern, the PredictionKey “bounces” off the server and comes back to the client via `UAbilitySystemComponent::ReplicatedPredictionKeyMap` (replicated property). 
 
 Once the key is replicated back from the server, the *client* is able to undo all of the locally predictive side effects (*Gameplay Cues*, *Gameplay Effects*): the replicated versions *will be there* and if they aren’t then it was a misprediction. Knowing exactly when to undo the predictive side effects is crucial here: if you are too early you will see gaps, if you are too late you will have “double”. (Note this is referring to stateful prediction, like a looping *Gameplay Cue* of a duration based *Gameplay Effect*. “Burst” *Gameplay Cues* and instant *Gameplay Effects* are never “undone” or rolled back. They are just skipped on the client if there is a prediction key associated with them).

 To further hit home the point: it’s crucial that predictive action is something the server does not do on their own, but only does so when the client tells them to. So having a generic “Create a key on demand and tell the server so I can run something” does not work unless that “something” is something the server will only do once told to by the client.

### About Example
 Backing up to the original question: something like a fire and forget projectile. 
 
 Both *Paragon* and *Fornite* have projectile actor classes that use *Gameplay Cues*. 
 
 However we do not use the Prediction Key system to do these. Instead we have a concept on Non-Replicated *Gameplay Cues*. *Gameplay Cues* that just fire off locally and are skipped by the server completely. Essentially all these are direct calls to `UGameplayCueManager::HandleGameplayCue`. They do not route through the `UAbilitySystemComponent` so no prediction key checks / early returns are made.

 The downside with non replicated *Gameplay Cues* is that, well, they are not replicated. 
 
 So its up to the "projectile class"/*blueprint* to make sure the code paths that call these functions are running on everyone. We have for cues startup (called in `BeginPlay`), explosion, hit wall/character, etc.

 These type of events are already generated client side, so calling into a non replicated gameplay cue was no big deal. Complicated blueprints can be tricky, and are up to the author to make sure they understand what is running where.

## Countering Cheaters regarding locally predicted Scoped Prediction Windows (#2)

#### Example
When using a `WaitNetSync` `AbilityTask` with `OnlyServerWait` to create a scoped prediction window in a locally predicted `GameplayAbility`, could players potentially cheat by delaying their packets to the Server to control `GameplayAbility` timing since the Server is waiting for their RPC with their prediction key? Was this ever an issue in Paragon or Fortnite, and if so, what did Epic do to remedy it?

 Yes, this is a valid concern. Any ability blueprint running on the server that is waiting for a client “signal” is potentially vulnerable to lag switch type exploits.

 *Paragon* had a custom targeting task similar to `UAbilityTask_WaitTargetData`.
 
  In this task we had timeouts, or a “max delay” that we would wait on the client for instantaneous targeting modes. If the targeting mode was waiting for user confirmation (button press) then it would be ignored since the user is allowed to take his time. But for abilities that instantly confirmed targeting we would only wait a certain amount of time before either A) generating the target data server side or B) canceling the ability.

 We never had such mechanisms for `WaitNetSync`, which we used pretty sparingly.

 I don’t believe Fortnite makes use of anything like this though. The weapon abilities in Fortnite are special cased batched to a single fortnite-specific RPC: one RPC to activate the ability, provide target data, and end the ability. So weapon abilities are intrinsically not vulnerable to this in Battle Royale.

 My take is that this is something that could probably be solved system wide but I don’t see us making the change ourselves anytime soon. Spot fixing `WaitNetSync` to include a max delay for the case you mention is probably a reasonable task, but again - unlikely we will do this on our end in the immediate future.

## About Gameplay Effect Replication Choices when Designing Gameplay (#3)
3. Which `EGameplayEffectReplicationMode` did *Paragon* and *Fortnite* use and what are Epic’s recommendations for when to use each?

 Both games essentially use Mixed mode for their player controlled characters and Minimal for AI controlled (AI minions, jungle creeps, AI Husks, etc). This is what I would recommend most people using the system in a multiplayer game. The sooner into your project you set these, the better.

 Fortnite goes a few steps further with its optimizations. It actually does not replicate the `UAbilitySystemComponent` at all for simulated proxies. 
 
 The component and attribute subobjects are skipped inside `::ReplicateSubobjects()` on the owning *Fortnite* player state class. 
 
 We do push the bare minimum replicated data from the ability system component to a structure on the pawn itself (basically, a subset of attribute values and a white list subset of tags that we replicate down in a bitmask). 
 We call this a “proxy”. 
 
 On the receiving side we take the proxy data, replicated on the pawn, and push it back into ability system component on the player state. So you do have an ASC for each player in FNBR, it just doesn’t directly replicate: instead it replicates data via a minimal proxy struct on the pawn and then routes back to the ASC on receiving side. This is advantage since its A) a more minimal set of data B) takes advantage of pawn relevancy.

 I’m not sure if it is still necessary with other server side optimizations that have been done since then (Replication Graph, etc) and it is not the most maintainable pattern.


## About mitigating latency when removing *Gameplay Effects* (#4)
### Question
 Since we cannot predict the removal of *Gameplay Effects* as per `GameplayPrediction.h`, are there any strategies for mitigating the effects of latency on removing *Gameplay Effects*? For example, when removing a movement speed slow, we currently have to wait for the Server to replicate the *Gameplay Effect* removal resulting in a snap of the player’s character position.

### Answer
 This is a tough one and I don’t have a good answer. We generally skirted around these problems with tolerances and smoothing. I totally agree that ability system and precise synchronization with the character movement system is not in a good place and something we do want to fix.

 I had a shelf of allowing predictive removal of GEs but could never work out all edge cases before having to move on. This doesn’t solve everything though since character movement still has an internal saved move buffer that does not know anything about the ability system and possible movement speed modifiers, etc. It is still possible to get into correction feedback loops even outside of not being able to predict the removal of GEs.

 If you think you have a case that is truly desperate, you are able to predictively add a GE that would inhibit your movement speed GEs. I’ve never done this myself but have theorized about it before. It may be able to help with a certain class of problem.

## About *Ability System Component* Placement (#5)
### Question
What are Epic’s internal rules, guidelines, or recommendations for where the *Ability System Component* should live, and what should its *Owner* be?

### Example
We know that it lives on the *Player State* in *Paragon* and *Fortnite* and on the *Character* in the *Action RPG Sample*. 
### Answer
 In general I would say anything that does not need to respawn should have the *Owner* and *Avatar Actor* be the same thing. Anything like AI enemies, buildings, world props, etc.

 Anything that does respawn should have the *Owner Actor* and *Avatar Actor* be different so that the *Ability System Component* does not need to be saved off / recreated / restored after a respawn. *Player State* is the logical choice it is replicated to all clients (where as *Player Controller* is not). The downside is *Player States* are always relevant so you can run into problems in 100 player games (See notes on what FN did in question #3).


## About Multiple *Ability System Components* (#6)
### Question
Is it viable to have several `AbilitySystemComponents` which have the same owner but different avatars?

### Example
(e.g. on pawn and weapon/items/projectiles with `Owner` set to `PlayerState`)?
### Answer
 The first problem I see there would be implementing the `IGameplayTagAssetInterface` and `IAbilitySystemInterface` on the owning actor. The former may be possible: just aggregate the tags from all ASCs (but watch out - `HasAllMatchingGameplayTags` may be met only via cross ASC aggregation. It wouldn't be enough to just forward that calls to each ASC and OR the results together). But the later is even trickier: which ASC is the authoritative one? If someone wants to apply a GE - which one should receive it? Maybe you can work these out but this side of the problem will be the hardest: owners will multiple ASCs beneath them.

 Separate ASCs on the pawn and the weapon can make sense on its own though. E.g, distinguishing between tags the describe the weapon vs those that describe the owning pawn. Maybe it does make sense that tags granted to the weapon also “apply” to the owner and nothing else (E.g, attributes and GEs are independent but the owner will aggregate the owned tags like I describe above). This could work out, I am sure. But having multiple ASCs with the same owner may get dicey.


## Stop Server Overwrites of Cooldowns for Locally Predicted Abilities (#7)
### Question
Is there a way to stop the *Server* from overwriting the cooldown duration of locally predicted abilities on the *Owning Client*?
### Example
In scenarios of high latency, this would let the Owning Client "try" to activate the ability again when its local cooldown expires but it is still on cooldown on the Server. 
By the time the *Owning Client*'s activation request reaches the *Server* over the network, the *Server* may be off cooldown or the *Server* might be able to queue the activation request for the remaining milliseconds that it has left. 

Otherwise as is, clients with higher latency have a longer delay before when they can reactivate an ability versus those with less latency.

This is most apparent with very low cooldown abilities like a basic attack that can be less than one second of cooldown.

If there isn't a way to stop the *Server* from overwriting the cooldown duration of locally predicted abilities, what is Epic's strategy for mitigating the effects of high latency on reactivating abilities? 


#### TL;DR
To word it another example-based way, how did *Epic* design *Paragon*'s basic attacks and other abilities so that high latency players could attack or activate at the same speed as low latency players with local prediction?


### Answer
 The short answer there is not a way to prevent this and Paragon definitely had the problem. Higher latency connections would have a lower ROF with basic attacks.

 I attempted to fix this by adding “GE reconciliation” where latency was taken into account when calculating GE duration. Essentially allowing the server to eat some of the total GE time so that the effective time of the GE client side would be 100% consistent with any amount of latency (though fluctuations could still cause issues). However I never got this working in a state that could ship and the project moved fast and we just never fully addressed it.

 Fortnite does its own bookkeeping for weapon firing rates: it does not use GEs for cooldowns on weapons. I would recommend this if this is a critical problem for your game.


### About the Roadmap for Gameplay Ability System
What is Epic’s roadmap for the *Gameplay Ability System* plugin? Which features does Epic plan to add in 2019 and beyond?

 We feel that overall the system is pretty stable at this point and we don’t have anyone working on major new features. Bug fixes and small improvements occasionally are made for Fortnite or from UDN/pull requests, but that is it right now.

 Longer term, I think we will eventually do a “V2” or some big changes. We learned a lot from writing this system and feel we got a lot right and a lot wrong. I would love a chance to correct those mistakes and improve some of the fatal flaws that were pointed out above.

 If a V2 was to ever come, providing an upgrade path would be of utmost importance. We would never make a V2 and leave Fortnite on V1 forever: there would be some path or procedures that would automatically migrate as much as possible, though there would still almost certainly be some manual remaking required.

 The high priority fixes would be:
 * Better interoperability with the character movement system. Unifying client prediction.
 * GE removal prediction (question #4)
 * GE latency reconciliation (question #7)
 * Generalized network optimizations such as batching RPCs and proxy structures. Mostly the stuff that we’ve done for Fortnite but find ways to break it down into more generalized form, at least so that games can write their own game specific optimizations more easily.

 The more general refactor type of changes I would consider making:
 * I would like to look at fundamentally moving away from having GEs reference spreadsheet values directly, instead they would be able to emit parameters and those parameters could be filled by some higher level object that is bound to spreadsheet values. The problem with the current model is that GEs become unsharable due to their tight coupling with the curve table rows. I think a generalized system for parameterization could be written and be the underpinning of a V2 system.
 * Reduce number of “policies” on `UGameplayAbility`. I would remove ReplicationPolicy and InstancingPolicy. Replication is, imo, almost never actually needed and causes confusion. InstancingPolicy should be replaced instead by making `FGameplayAbilitySpec` a `UObject` that can be subclassed. This should have been the “non instantiated ability object” that has events and is blueprintable. The UGameplayAbility should be the “instanced per execution” object. It could be optional if you need to actually instantiate: instead “non instanced” abilities would be implemented via the new UGameplayAbilitySpec object. 
 * The system should provide more “middle level” constructs such as “filtered GE application container” (data drive what GEs to apply to which actors with higher level gameplay logic), “Overlapping volume support” (apply the “Filtered GE application container” based on collision primitive overlap events), etc. These are building blocks that every project ends up implementing in their own way. Getting them right is non trivial so I think we should do a better job providing some basic implementations. 
 * In general, reducing boilerplate needed to get your project up and running. Possibly a separate module “Ex library” or whatever that could provide things like passive abilities or basic hitscan weapons out of the box. This module would be optional but would get you up and running quickly.
 * I would like to move *Gameplay Cues* to a separate module that is not coupled with the ability system. I think there are a lot of improvements that could be made here.


 This is only my personal opinion and not a commitment from anyone. I think the most realistic course of action will be as new engine tech initiatives come through, the ability system will need to be updated and that will be a time to do this sort of thing. These initiatives could be related to scripting, networking, or physics/character movement. This is all very far looking ahead though so I cannot give commitments or estimates on timelines.



<a name="resources-daveratti-community2"</a



<a name="changelog"</a
