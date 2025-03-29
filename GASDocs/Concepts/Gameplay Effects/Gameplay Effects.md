---
tags:
  - GAS-System
  - iku/draft
---




# Gameplay Effects
*Gameplay Effects* are a *Class* designed to be used to change *Attributes* and *Gameplay Tags*.
- This can be done on the *Gameplay Ability* that applies the effect itself or other ASC Owners.
## Features
- They can cause immediate *Attribute* changes
- They can cause immediate *Gameplay Tags* changes
- They can apply status buff/debuffs
- They can add/execute *Gameplay Cues*.
- They instance with *Gameplay Effect Spec* instead of themselves.
	- This doc uses *Spec* for brevity, but I also think GES could be used.
	- This doc *italics* one and not the other (acronyms) intentionally.

Calling it "Spec" is my shorthand, not sure you'll see it elsewhere / on Tranek's repo.


### Examples
- movespeed
- stunning
- Character Stats (RPG Stats)
- etc


### Subtypes
- They have duration types (*Instant Effect*, *Duration Effect*, and *Infinite Effect*)

### Common Methods for Attribute Changes
*Gameplay Effects* change *Attributes* through:

| Acronyms            | Conceptual Name                         | Documentation                             | Class Name (in Code)    | Description | Relationship |
| ------------------- | --------------------------------------- | ----------------------------------------- | ----------------------- | ----------- | ------------ |
| GES? / Spec         | *Gameplay Effect Spec*                  | [[Gameplay Effect Spec]]                  | *UGameplayEffectSpec* ? |             |              |
| GEO?                | *Gameplay Effect Operations*            | [[Gameplay Effect Operations]]            |                         |             |              |
| GEM?                | *Gameplay Effect Modifiers*             | [[Gameplay Effect Modifiers]]             |                         |             |              |
| ExecCalc, Execution | *Gameplay Effect Execution Calculation* | [[Gameplay Effect Execution Calculation]] |                         |             |              |
| GEC?                | *Gameplay Effect Context*               | [[Gameplay Effect Context]]               |                         |             |              |



## Limitations
As code, the `UGameplayEffect` class is a meant to be **data-only**; defining a single gameplay effect. 
- No added logic is allowed.
- Typically, designers will create many Blueprint child classes of `UGameplayEffect`.


#iku/tag-target
>I think the idea here is that you might pair them. 
>The debuff would have a value or individual attributes it would modify. What about calculations though?

>Okay, I think below should really be "Gameplay Effects have different types" but the key difference here is indeed the effect's duration.

# Chapters?
- [[Common Things to do with Gameplay Effects]]
- [[Using Gameplay Abilities and Gameplay Effects Together]]
- [[Example Gameplay Effects]]


### *Gameplay Effect* Duration Types

| Duration Type | GameplayCue Event | When to use                                                                                                                                                                                                                                 |
| ------------- | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Instant`     | Execute           | For immediate permanent changes to `Attribute's` `BaseValue`. *Gameplay Tags* will not be applied, not even for a frame.                                                                                                                     |
| `Duration`    | Add & Remove      | For temporary changes to `Attribute's` `CurrentValue` and to apply *Gameplay Tags* that will be removed when the *Gameplay Effect* expires or is manually removed. The duration is specified in the `UGameplayEffect` class/Blueprint.       |
| `Infinite`    | Add & Remove      | For temporary changes to `Attribute's` `CurrentValue` and to apply *Gameplay Tags* that will be removed when the *Gameplay Effect* is removed. These will never expire on their own and must be manually removed by an ability or the `ASC`. |
- A *Duration Gameplay Effect*  will call `Add` and `Remove` on the *Gameplay Cue*'s *Gameplay Tags.
- A *Infinite Gameplay Effect* will call `Add` and `Remove` on the *Gameplay Cue*'s *Gameplay Tags*.
- However, an *Instant Gameplay Effect* will call `Execute` on the *Gameplay Cue*'s *Gameplay Tags*.

#### Periodic Effects 
These are useful for damage over time (DOT) type effects. Cannot be [predicted](#concepts-p).

- `Duration` and `Infinite` *Gameplay Effects* have the option of applying `Periodic Effects`.
- These apply `Modifiers` and `Executions` every `X` seconds as defined by its `Period`.
- `Periodic Effects` are treated as `Instant` *Gameplay Effects* when it comes to changing the `Attribute's` `BaseValue` and `Executing` `GameplayCues`. 

>I both want to add this to the table above and also get why it's not there.
>You alter how those Gameplay Effect (duration) types function by enabling Periodic Effects.



---

