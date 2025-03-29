---
tags:
  - iku/draft
---



Honestly I think the documentation really needs this to make GAS more effective. Tom Looman's course covers this by avoiding GAS and teaching it to you, but I don't like the stream of consciousness / video format.
- Realize what your Gameplay is
- Decide where the ASC is going.
- Understand GAS well enough to load all core concepts (cognitive load) into your working headspace.
- ...Then you can decide what you need.

## Tools
- Something to Diagram with
- A way to test number calculations out
- A note taking application / documentation method


## Goals
- Make sure your game rules are coherent enough that you can decide if GAS is the right fit
- Design the system that enforces your rules
- Be mindful of the gameplay implications (networking for multiplayer, etc)

# Example

## ASC Placement
- I've decided to put it on the Player State.

[[IkuSoft MMC]]
[[IkuSoft Gameplay Effects]]
[[IkuSoft Gameplay Abilities]]
## Not Gameplay Abilities

### GMC Movement Component Related

| Category               | Action            | Primary Input | Alternative | Activation Type | System Integration          |
| ---------------------- | ----------------- | ------------- | ----------- | --------------- | --------------------------- |
| **Locomotion Primary** |                   |               |             |                 |                             |
|                        | Standard Movement | WASD          | -           | Analog          | Enhanced Movement Component |


### IkuSoft Interface (UI / UX)

| **Interface Control**  |                   |               |             |                 |                             |
| ---------------------- | ----------------- | ------------- | ----------- | --------------- | --------------------------- |
|                        | Inventory         | I             | -           | Toggle          | UI State Manager            |
|                        | Map               | M             | -           | Toggle          | UI State Manager            |
|                        | Weapon Inspect    | T             | -           | Toggle          | Equipment Inspector         |
|                        | Quick Menu        | Tab           | -           | Toggle          | UI State Manager            |
|                        | Ping System       | B             | -           | Press           | Communication Manager       |


- Remember that ArcInventory wants GA's as part of what it sends to the ASC.
- It also wants tags via the item definitions so I think it aligns well?


## Attribute Sets


The grouping of Attribute Sets is designed to 
- Establish Placeholder
This is for [[Iku Page Placeholder]]


#### Core
##### Character Attributes

| Literal Name in Code       | Math Context |     | Description |     |     |
| -------------------------- | ------------ | --- | ----------- | --- | --- |
| UMinCharacterAttributeSet  | Minimum      |     |             |     |     |
| UBaseCharacterAttributeSet | Base         |     |             |     |     |
| UMaxCharacterAttributeSet  | Max          |     |             |     |     |

##### Current Status



#### For Items

##### Reciever / Full Gun Maybe?

| Literal Name in Code       | Math Context |     | Description |     |     |
| -------------------------- | ------------ | --- | ----------- | --- | --- |
| UMinCharacterAttributeSet  | Minimum      |     |             |     |     |
| UBaseCharacterAttributeSet | Base         |     |             |     |     |
| UMaxCharacterAttributeSet  | Max          |     |             |     |     |

