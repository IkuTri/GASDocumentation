---
tags:
  - iku/fine
---
# Concepts
This page is a table that tries to highlight GAS concepts. 

Note that the documentation was designed with some readability in mind - so using something like:
- Obsidian (with it's graph view feature) 
- Other markdown tools
### Components
This table includes related components. Some from third party plugins that interact with GAS, the others 

| Acronyms | Documentation                | Literal Object in Code | Conceptual Name            | Tranek Docs Name               | Description | Context                |
| -------- | ---------------------------- | ---------------------- | -------------------------- | ------------------------------ | ----------- | ---------------------- |
| ASC      | [[Ability System Component]] |                        | *Ability System Component* | AbilitySystemComponent         |             | Core Component for GAS |
| MC       | Mover Component              |                        |                            |                                |             |                        |
| CMC      |                              | Movement Component     | CharacterMovementComponent | *Character Movement Component* |             |                        |

### Primary Concepts

| Acronyms | Documentation          | Literal Object in Code | Conceptual Name      | Tranek Docs Name | Description |
| -------- | ---------------------- | ---------------------- | -------------------- | ---------------- | ----------- |
| GE       | [[Gameplay Effects]]   | `UGameplayEffect`      | *Gameplay Effect*    | GameplayEffect   |             |
| GA       | [[Gameplay Abilities]] |                        | *Gameplay Abilities* | GameplayAbility  |             |
| AT       | [[Ability Tasks]]      |                        | *Ability Task*       | AbilityTask      |             |
### Secondary Concepts (Sub or Underlying Objects)
Tags drive or validate things  (ability activations, effects, etc.)

| Acronyms | Documentation     | Literal Object in Code | Conceptual Name | Tranek Docs Name | Description                                                   |
| -------- | ----------------- | ---------------------- | --------------- | ---------------- | ------------------------------------------------------------- |
| Tag, GT  | [[Gameplay Tags]] |                        | *Gameplay Tag*  | GameplayTag      | Tags are used to define almost everything you design with GAS |
| GC       | [[Gameplay Cues]] |                        | *Gameplay Cue*  | GameplayCue      |                                                               |

### Variables and Variable Storage / Structs
How you store the named statistics / variables for your game's rules or *Ruleset*.

| Acronyms | Documentation           | Literal Object in Code | Conceptual Name  | Tranek Docs Name | Description |
| -------- | ----------------------- | ---------------------- | ---------------- | ---------------- | ----------- |
| GAB      | [[Attributes]] |                        | *Attributes*     |                  |             |
| GABS     | [[Attribute Sets]]      |                        | *Attribute Sets* |                  |             |
## Operations

- [[Gameplay Ability Operations]]
### Calculation Utilities
>Flow?

| Acronyms        | Documentation                      | Literal Object in Code | Conceptual Name                  | Tranek Docs Name             | Description |
| --------------- | ---------------------------------- | ---------------------- | -------------------------------- | ---------------------------- | ----------- |
| CAR?            | [[Custom Application Requirement]] |                        |                                  |                              |             |
| ModMagCalc, MMC | [[Modifier Magnitude Calculation]] |                        | *Modifier Magnitude Calculation* | ModifierMagnitudeCalculation |             |

### World Objects
| Acronyms | Documentation              | Literal Object in Code | Conceptual Name | Tranek Docs Name | Description |
| -------- | -------------------------- | ---------------------- | --------------- | ---------------- | ----------- |
| ASG?     | [[Ability System Globals]] |                        |                 |                  |             |

## Replication
[[Gameplay Ability Replication]]
