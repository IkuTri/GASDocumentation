---
tags:
  - iku/draft
---


# Commonly Implemented Abilities
[[Sprint]

[[Aiming Down Sights]]]
## Movement<a name="cae-sprint"></a>
### Sprint
The Sample Project provides an example of how to sprint - run faster while (Left Shift) is held down.

The faster movement is handled predictively by the *Character Movement Component* by sending a flag over the network to the server.
See `GDCharacterMovementComponent.h/cpp` for details.

The GA handles responding to the (Left Shift) input, tells the *CMC* to begin and stop sprinting, and to predictively charge stamina while (Left Shift) is pressed. 
- See `GA_Sprint_BP` for details.


<a name="cae-ads"></a>
### Aim Down Sights
The Sample Project handles this the exact same way as sprinting but decreasing the movement speed instead of increasing it.

See *GDCharacterMovementComponent.h/cpp* for details on predictively decreasing the movement speed.

See *GA_AimDownSight_BP* for details on handling the input. There is no stamina cost for aiming down sights.


## Interaction
### One Button Interaction System
[GASShooter](https://github.com/tranek/GASShooter) implements a one button interaction system where the player can press or hold 'E' to interact with interactable objects like reviving a player, opening a weapon chest, and opening or closing a sliding door.

