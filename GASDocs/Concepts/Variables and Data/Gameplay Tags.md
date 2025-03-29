---
tags:
  - iku/draft
---
# Gameplay Tags
Unreal Documentation: [*FGameplayTags*](https://docs.unrealengine.com/en-US/API/Runtime/GameplayTags/FGameplayTag/index.html) 
>"These tags are incredibly useful for classifying and describing the state of an object." - Tranek

## Overview
----
- *Gameplay Tags* run most of GAS, and are used to describe almost everything in GAS that isn't a number.
- They are hierarchical names in the form of `Parent.Child.Grandchild`, and are registered with the *Gameplay Tag Manager*. 
	- For example, if a character is stunned, we could give it a `State.Debuff.Stun` *Gameplay Tag*.
	- This tag would live for the duration of the stun.
- Designed to replace *Booleans* or *Enums* with *Gameplay Tags*.
	- For example, you'd consider using *Boolean* logic on whether or not objects have certain *Gameplay Tags*.

>Tags can have more layers than the example above
>You ideally want to be sure of your tags and their structure. 
>One of the first concepts I think is really important.

### Option: *Loose Gameplay Tags*
The *ASC* allows you to add *Loose Gameplay Tags* that are not replicated and must be managed manually. 
## Defining Gameplay Tags
----
*Gameplay Tags* must be defined ahead of time in the `DefaultGameplayTags.ini`. 
- You can edit this file with an IDE or Text Editor
- You can edit this file within the Unreal Engine Editor.
### Learning from the Sample .ini File
The Sample Project extensively uses *Gameplay Tags*.
>because it uses GAS.  Here is the ini from that project.

### Editing  *Gameplay Tags* In-Engine
The *Gameplay Tag* editor can create, rename, search for references, and delete *Gameplay Tags*.
- Searching for *Gameplay Tag* references will bring up the familiar *Reference Viewer* graph in the Editor.
- This will show all the assets that reference the *Gameplay Tag*.
-  This will not however show any C++ classes that reference the *Gameplay Tag*.

[![GameplayTag Editor in Project Settings](https://github.com/tranek/GASDocumentation/raw/master/Images/gameplaytageditor.png)](https://github.com/tranek/GASDocumentation/raw/master/Images/gameplaytageditor.png)
### Renaming Existing (Re-Defining) *Gameplay Tags*
Renaming *Gameplay Tags* creates a redirect so that assets still referencing the original *Gameplay Tag* can redirect to the new *Gameplay Tag*.
#### Avoiding a Tag Redirect
I prefer if possible to:
- instead create a new *Gameplay Tag*, 
- update all the references manually to the new *Gameplay Tag*,
- and then delete the old *Gameplay Tag* to avoid creating a redirect.
## Using Gameplay Tags
----
Each Section here is something you'd probably or almost have to do while making thing happen for your game. In other words, this section is less about design and more about technique.
### Replicating Gameplay Tags
- *Gameplay Tags* are replicated if they're added from a *Gameplay Effect*. 
- [ ] Figure out Tag Map Count
- [ ] Figure out Fast Replication vs Replication
#### Replicating with *Fast Replication*
*Fast Replication* requires that the server and the clients have the same list of *Gameplay Tags*.
- This generally shouldn't be a problem so you should enable this option. 
- *Gameplay Tag Containers* can also return a `TArray<FGameplayTag>` for iteration.
- *Gameplay Tags* stored in `FGameplayTagCountContainer` have a *TagMap* that stores the number of instances of that *GameplayTag*. 
- A `FGameplayTagCountContainer` may still have the *Gameplay Tag* in it but its `TagMapCount` is zero. 
	- You may encounter this while debugging if an ASC still has a *Gameplay Tag*. 
Any of the `HasTag()` or `HasMatchingTag()` or similar functions will check the `TagMapCount` and return false if the *Gameplay Tag* is not present or its `TagMapCount` is zero.
In addition to *Fast Replication*, the *Gameplay Tag* editor has an option to fill in commonly replicated *Gameplay Tags* to optimize them further.


>Again, FWhat is going on? I gotta verify these

> [!NOTE] Sample Project
> 
> The Sample Project uses a *Loose Gameplay Tag* for `State.Dead` so that the owning clients can immediately respond to when their health drops to zero. 

#### Replicating with *Tag Map Count*
>Okay so what is this? Guessing it's in the code and just mentions it
- Respawning manually sets the `TagMapCount` back to zero. 
- Only manually adjust the `TagMapCount` when working with *Loose Gameplay Tags*. 

It is preferable to use the `UAbilitySystemComponent::AddLooseGameplayTag()` and `UAbilitySystemComponent::RemoveLooseGameplayTag()` functions than manually adjusting the `TagMapCount`.



### Referencing Gameplay Tags:
Getting a reference to a *Gameplay Tag* in C++:

```c
FGameplayTag::RequestGameplayTag(FName("Your.GameplayTag.Name"))
```

### Managing *Gameplay Tags* (with the *Gameplay Tag Manager*)
For advanced *Gameplay Tag* manipulation like getting the parent or children *Gameplay Tags*, look at the functions offered by the *Gameplay Tag Manager*. 

- To access the *Gameplay Tag Manager*, include `GameplayTagManager.h` and call it with `UGameplayTagManager::Get().FunctionName`. 
- The *Gameplay Tag Manager* actually stores the *Gameplay Tags* as relational nodes (parent, child, etc) for faster processing than constant string manipulation and comparisons.
### Setting *Gameplay Tags* to be *Gameplay Cue* only
>There are two methods in the original documentation.

#### Method: Via UPROPERTY Only
*Gameplay Tags* and *Gameplay Tag Containers* can have the optional *UPROPERTY* specifier `Meta = (Categories = "GameplayCue")` that filters the tags in the Blueprint to show only *Gameplay Tags* that have the parent tag of *Gameplay Cue*. 

- This is useful when you know the *Gameplay Tag* or *Gameplay Tag Container* variable should only be used for *Gameplay Cues*.
#### Method: Via Struct Usage / Editing Unreal Engine Code
Alternatively, there's a separate structure in code called `FGameplayCueTag` that encapsulates a `FGameplayTag`.
- This also automatically filters *Gameplay Tags* in Blueprint to only show those tags with the parent tag of *GameplayCue*.
- If you want to filter a *Gameplay Tag* parameter in a function, use the *UFUNCTION* specifier `Meta = (GameplayTagFilter = "GameplayCue")`.
- However, *Gameplay Tag Container* parameters in functions can not be filtered. 

##### Editing Unreal Engine Itself for Allowing This
If you would like to edit your engine to allow this, follow the steps below.

Within `Engine\Plugins\Editor\GameplayTagsEditor\Source\GameplayTagsEditor\Private\SGameplayTagGraphPin.cpp` , and specifically (?) the `SGameplayTagGraphPin::GetListContent()` function:
- look at how `SGameplayTagGraphPin::ParseDefaultValueData()` calls `FilterString = UGameplayTagsManager::Get().GetCategoriesMetaFromField(PinStructType);` 
- and how it passes `FilterString` to `SGameplayTagWidget`.


The *Gameplay Tag Container* version of these functions are in `Engine\Plugins\Editor\GameplayTagsEditor\Source\GameplayTagsEditor\Private\SGameplayTagContainerGraphPin.cpp` .
- For this, do not check for the meta field properties and pass along the filter.

## Giving *Gameplay Tags* to an Object
---

When giving tags to an object, it should go to the owner's *Ability System Component.*
- In code, `UAbilitySystemComponent` implements the `IGameplayTagAssetInterface`, giving functions to access its owned *Gameplay Tags*.

### Dealing with non GAS Objects and Gameplay
"if it has one so that GAS can interact with them." 
> Ideally I'd find out how you deal with non GAS stuff, because the idea seems to be use GAS for all gameplay.
> Probably why Tom Looman teaches you to roll your own.

### Using a *Gameplay Tag Container* to hold many *Gameplay Tags*
Multiple *Gameplay Tags* can be stored in an *Gameplay Tag Container*, called in code as `FGameplayTagContainer`. 
- It is preferable to use a  over a `TArray<FGameplayTag>`  since *Gameplay Tag Containers* add some efficiency magic. 

#### In `FTerms` instead
>Why is it formatted this FWay? Are the all code or are the docs messing with me?

While tags are standard `FNames`, they can be efficiently packed together in `FGameplayTagContainers` for replication if *Fast Replication* is enabled in the project settings. 

### Responding to Gameplay Tags Changes
The *ASC* provides a delegate for when *GameplayTags* are added or removed. 

- It takes in a *EGameplayTagEventType* that can specify only to fire when the *Gameplay Tag* is added/removed
- Or for any change in the *Gameplay Tag*'s *Tag Map Count*.

```c
AbilitySystemComponent->RegisterGameplayTagEvent(FGameplayTag::RequestGameplayTag(FName("State.Debuff.Stun")), EGameplayTagEventType::NewOrRemoved).AddUObject(this, &AGDPlayerState::StunTagChanged);
```

The callback function has a parameter for the *GameplayTag* and the new *TagCount*.

```c
virtual void StunTagChanged(const FGameplayTag CallbackTag, int32 NewCount);
```

### Loading *Gameplay Tags* from Plugins and their Own .ini Files
If you create a plugin with its own .ini files with *Gameplay Tags*, you can load that plugin's *Gameplay Tag* .ini directory in your plugin's `StartupModule()` function.
#### Example of *Loading Gameplay Tags* from a *Engine Plugin* .ini File
For example, in the *Common Conversation Plugin* that comes with Unreal Engine:
- The example code would look for the directory `Plugins\CommonConversation\Config\Tags`. 
- Then it would load any .ini files with *Gameplay Tags* in them into your project.
- Just to be sure, make sure your Plugin is enabled at Engine startup.


```c
void FCommonConversationRuntimeModule::StartupModule()
{
	TSharedPtr<IPlugin> ThisPlugin = IPluginManager::Get().FindPlugin(TEXT("CommonConversation"));
	check(ThisPlugin.IsValid());
	
	UGameplayTagsManager::Get().AddTagIniSearchPath(ThisPlugin->GetBaseDir() / TEXT("Config") / TEXT("Tags"));

	//...
}
```
