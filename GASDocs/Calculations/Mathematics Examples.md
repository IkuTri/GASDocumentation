---
tags:
  - iku/insane
---

# Mathematics and Utility
This page has some examples for niche things. Might be useful.
## Generating a Random Number on Client and Server
Sometimes you need to generate a "random" number inside of a *Gameplay Ability*.

- The client and the server will both want to generate the same random numbers. 
- To do this, we must set the *random seed* to be the same at the time of *Gameplay Ability* activation. 

You will want to set the *random seed* each time you activate the *Gameplay Ability*. This in case the *Client* mispredicts activation, and its random number sequence becomes out of sync with the *Server*'s.

| Seed Setting Method                                                          | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Use the activation prediction key                                            | The *Gameplay Ability* activation prediction key is an int16 guaranteed to be synchronized and available in both the client and server in the `Activation()`. You can set this as the *random seed* on both the client and the server. The downside to this method is that the prediction key always starts at zero each time the game starts and consistently increments the value to use between generating keys. This means each match will have the exact same random number sequence. This may or may not be random enough for your needs. |
| Send a seed through an event payload when you activate the *GameplayAbility* | Activate your *Gameplay Ability* by event and send the randomly generated seed from the client to the server via the replicated event payload. This allows for more randomness but the client could easily hack their game to only send the same seed value every time. Also activating *Gameplay Abilities* by event will prevent them from activating from the input bind.                                                                                                                                                                    |


If your random deviation is small, most players won't notice that the sequence is the same every game and using the activation prediction key as the *random seed* should work for you. If you're doing something more complex that needs to be hacker proof, perhaps having the *Server* initiate the *Gameplay Ability* would work better where the server can create the prediction key or generate the *random seed* to send via an event payload.
<a name="cae-crit"></a>
## Critical Hits
Tranek handles critical hits inside of the damage *Execution Calculation*.

- The *Gameplay Effect* will have a *Gameplay Tag* on it like `Effect.CanCrit`. 
- The *Execution Calculation* checks if the *Gameplay Effect Spec* has that `Effect.CanCrit` *Gameplay Tag*. 
- If the *Gameplay Tag* exists, the *Execution Calculation* generates a random number corresponding to the critical hit chance.
	- (*Attribute* captured from the *Source*) 
- Then adds the critical hit damage if it succeeded. 
	- (also an *Attribute* captured from the *Source*)


Since I don't predict damage, I don't have to worry about synchronizing the random number generators on the *Client* and *Server*.
This is due to the fact that the *Execution Calculation* will only run on the *Server*. 

If you tried to do this predictively using an *MMC* to do your damage calculation, you would have to get a reference to the *random seed* from:

*Gameplay Effect Spec*->*Gameplay Effect Context*->*Gameplay Ability Instance*.
- See how [GASShooter](https://github.com/tranek/GASShooter) does headshots.
- It's the same concept except that it does not rely on a random number for chance and instead checks the *FHitResult* bone name.


## Allowing only the best 
>IIRC
## Non-Stacking Gameplay Effects but Only the Greatest Magnitude Actually Affects the Target
Slow effects in *Paragon* did not stack.

- Each slow instance applied and kept track of their lifetimes as normal.
- However, only the greatest magnitude slow effect actually affected the *Character*.
- GAS provides for this scenario out of the box with *AggregatorEvaluateMetaData*. 
- See `AggregatorEvaluateMetaData()` for details and implementation.


## Generate Target Data While Game is Paused
If you need to pause the game while waiting to generate [*TargetData*](#concepts-targeting-data) from a *WaitTargetData* *Ability Task* from your player, I suggest instead of pausing to use *slomo 0*.