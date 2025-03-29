---
tags:
  - iku/check
  - iku/fine
---

# Gameplay Effect Execution Calculation
Unreal Documentation: [*GameplayEffectGameplay Effect Execution Calculations*](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UGameplayEffectExecutionCalculat-/index.html) 

## Overview
*Gameplay Effect Execution Calculations* are the most powerful way for *Gameplay Effects* to make changes to an ASC.

Calculating damage based on a complex formula from many attributes on the *Source* and the *Target* is the most common example of an *Execute Calc*. 

Like *Modifier Magnitude Calculations*, these can capture *Attributes* and optionally snapshot them. 
- Anything with the word 'Execute' in it typically refers to these two types of *Gameplay Effects*.
- Can only be used with *Instant* and *Periodic* *Gameplay Effects*. 

Unlike MMCs, these can change more than one *Attribute* and essentially do anything else that the programmer wants. 
- They can not be *predicted*
- They must be implemented in C++.
### Aliases
- *Execution Calculation*
-  *Execution* (you will often see this term in the plugin's source code), or *Exec Calc*) 
- Editor prefers *Execute Calc* / *Calculation*


### Snapshotting (Capturing Attributes from *Spec*)
*Snapshotting* deals with when you want to grab the value from the *Attribute*.
- *Snapshotting* captures the *Attribute* when the *Gameplay Effect Spec* is created.
- Not *Snapshotting* captures the *Attribute* when the *Gameplay Effect Spec* is applied. 

#### Capturing and *Current Value*
Capturing *Attributes* recalculates their *Current Value* from existing mods (*Modifiers*?) on the *ASC*. 
- This recalculation will not run `PreAttributeChange()` in the *Ability Set* so any clamping must be done here again.

#### Table with Capturing

| Snapshot | Source or Target | Captured on *Gameplay Effect Spec* |
| -------- | ---------------- | ---------------------------------- |
| Yes      | Source           | Creation                           |
| Yes      | Target           | Application                        |
| No       | Source           | Application                        |
| No       | Target           | Application                        |
#### Setting up your Capture Process by Defining a *Struct*
To set up *Attribute* capture, we follow a pattern set by Epic's *ActionRPG Sample Project*.

- This involves defining a *struct*.
- It will hold and define how we capture the *Attributes*.
- It will also create a single copy of it in the *struct*'s constructor.

##### Naming Conventions
Each struct needs a unique name as they share the same namespace. You will have a struct like this for every *Execute Calc*. 

- Using the same name for the structs will cause incorrect behavior in capturing your *Attributes*.
- This will involve mostly capturing the values of the wrong *Attributes*.

##### Prediction and Replication
For *Local Predicted*, *Server Only*, and *Server Initiated* *Gameplay Abilities*, the *Execute Calc* only calls on the Server.



## Overview Example: Total Damage Calculation
As mentioned before, "Calculating damage based on a complex formula from many attributes on the *Source* and the *Target* is the most common example of an *Execute Calc*. "

The included Sample Project has a simple *Execute Calc* for this.

- It reads the value of damage from the *Gameplay Effect Spec's* `SetByCaller`.
- Then mitigates that value based on the "Armor" *Attribute* captured from the *Target*.
- See `GDDamageExecCalculation`'s header and implementation file for how this works.

- [ ] Fix link to this code

## Additional Methods for *Execution Calculations*
### Sending Data to Execution Calculations
There are a few ways to send data to an *Execution Calculation* in addition to capturing *Attributes*.

##### SetByCaller
Any `SetByCallers` set on the *Gameplay Effect Spec* can be directly read in the *Execution Calculation*.

```c++
const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();
float Damage = FMath::Max<float>(Spec.GetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag(FName("Data.Damage")), false, -1.0f), 0.0f);
```


### Backing Data Attribute Calculation Modifier
If you want to hardcode values to a *Gameplay Effect*, you can pass them in using a *Calculation Modifier* that uses one of the captured *Attributes* as the backing data.

In this screenshot example, we're adding 50 to the captured Damage *Attribute*. You could also set this to *Override* to just take in only the hardcoded value.

![Backing Data Attribute Calculation Modifier](https://github.com/tranek/GASDocumentation/raw/master/Images/calculationmodifierbackingdataattribute.png)

The *Execution Calculation* reads this value in when it captures the *Attribute*.

```c++
float Damage = 0.0f;
// Capture optional damage value set on the damage GE as a CalculationModifier under the Execution Calculation
ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().DamageDef, EvaluationParameters, Damage);
```

### Backing Data Temporary Variable Calculation Modifier
If you want to hardcode values to a *GameplayEffect*, you can pass them in using a *Calculation Modifier* that uses a *Temporary Variable* or *Transient Aggregator* as it's called in C++. The *Temporary Variable* is associated with a *GameplayTag*.

In this screenshot example, we're adding 50 to a *Temporary Variable* using the *Data.Damage* *GameplayTag*.

![Backing Data Temporary Variable Calculation Modifier](https://github.com/tranek/GASDocumentation/raw/master/Images/calculationmodifierbackingdatatempvariable.png)

Add backing *Temporary Variables* to your *Execution Calculation*'s constructor:

```c++
ValidTransientAggregatorIdentifiers.AddTag(FGameplayTag::RequestGameplayTag("Data.Damage"));
```

The *Execution Calculation* reads this value in using special capture functions similar to the *Attribute* capture functions.

```c++
float Damage = 0.0f;
ExecutionParams.AttemptCalculateTransientAggregatorMagnitude(FGameplayTag::RequestGameplayTag("Data.Damage"), EvaluationParameters, Damage);
```


### Gameplay Effect Context
You can send data to the *Execution Calculation* via a custom [*GameplayEffectContext* on the *GameplayEffectSpec*](#concepts-ge-context).

In the *Execution Calculation* you can access the *Effect Context* from the `FGameplayEffectCustomExecutionParameters`.

```c++
const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();
FGSGameplayEffectContext* ContextHandle = static_cast<FGSGameplayEffectContext*>(Spec.GetContext().Get());
```

If you need change something on the *GameplayEffectSpec* or the *EffectContext*:

```c++
FGameplayEffectSpec* MutableSpec = ExecutionParams.GetOwningSpecForPreExecuteMod();
FGSGameplayEffectContext* ContextHandle = static_cast<FGSGameplayEffectContext*>(MutableSpec->GetContext().Get());
```

Use caution if modifying the *Gameplay Effect Spec* in the *Execution Calculation*. See the comment for `GetOwningSpecForPreExecuteMod()`.

```c++
/** Non const access. Be careful with this, especially when modifying a spec after attribute capture. */
FGameplayEffectSpec* GetOwningSpecForPreExecuteMod() const;
```

