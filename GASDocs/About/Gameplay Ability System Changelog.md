---
tags:
  - GAS-System
  - iku/draft
---

# *Gameplay Ability System* Changelog

This is a list of notable changes (fixes, changes, and new features) to GAS compiled from the official Unreal Engine upgrade changelog and from undocumented changes that I've encountered. If you've found something that isn't listed here, please make an issue or pull request.

>I only formatted headers for this one. WIP
## Unreal Engine 5

### 5.3
* Crash Fix: Fixed a crash when trying to apply Gameplay Cues after a seamless travel.
* Crash Fix: Fixed a crash caused by GlobalAbilityTaskCount when using Live Coding.
* Crash Fix: Fixed `UAbilityTask::OnDestroy` to not crash if called recursively for cases like `UAbilityTask_StartAbilityState`.
* Bug Fix: It is now safe to call `Super::ActivateAbility` in a child class. Previously, it would call `CommitAbility`.
* Bug Fix: Added support for properly replicating different types of `FGameplayEffectContext`.
* Bug Fix: `FGameplayEffectContextHandle` will now check if data is valid before retrieving "Actors".
* Bug Fix: Retain rotation for Gameplay Ability System Target Data LocationInfo.
* Bug Fix: Gameplay Ability System now stops searching for PC only if a valid PC is found.
* Bug Fix: Use existing GameplayCueParameters if it exists instead of default parameters object in `RemoveGameplayCue_Internal`.
* Bug Fix: GameplayAbilityWorldReticle now faces towards the source Actor instead of the TargetingActor.
* Bug Fix: Cache trigger event data if it was passed in with GiveAbilityAndActivateOnce and the ability list was locked.
* Bug Fix: Support has been added for the FInheritedGameplayTags to update its CombinedTags immediately rather than waiting until a Save.
* Bug Fix: Moved ShouldAbilityRespondToEvent from client-only code path to both server and client.
* Bug Fix: Fixed `FAttributeSetInitterDiscreteLevels` from not working in Cooked Builds due to Curve Simplification.
* Bug Fix: Set CurrentEventData in GameplayAbility.
* Bug Fix: Ensure MinimalReplicationTags are set up correctly before potentially executing callbacks.
* Bug Fix: Fixed `ShouldAbilityRespondToEvent` from not getting called on the instanced *Gameplay Ability*.
* Bug Fix: Gameplay Cue Notify Actors executing on Child Actors no longer leak memory when gc.PendingKill is disabled.
* Bug Fix: Fixed an issue in GameplayCueManager where GameplayCueNotify_Actors could be 'lost' due to hash collisions.
* Bug Fix: WaitGameplayTagQuery will now respect its Query even if we have no Gameplay Tags on the Actor.
* Bug Fix: PostAttributeChange and AttributeValueChangeDelegates will now have the correct OldValue.
* Bug Fix: Fixed FGameplayTagQuery from not showing a proper Query Description if the struct was created by native code.
* Bug Fix: Ensure that the UAbilitySystemGlobals::InitGlobalData is called if the Ability System is in use. Previously if the user did not call it, the Gameplay Ability System did not function correctly.
* Bug Fix: Fixed issue when linking/unlinking anim layers from UGameplayAbility::EndAbility.
* Bug Fix: Updated Ability System Component function to check the Spec's ability pointer before use.
* New: Added a GameplayTagQuery field to FGameplayTagRequirements to enable more complex requirements to be specified.
* New: Introduced FGameplayEffectQuery::SourceAggregateTagQuery to augment SourceTagQuery.
* New: Extended the functonality to execute and cancel Gameplay Abilities & Gameplay Effects from a console command.
* New: Added the ability to perform an "Audit" on Gameplay Ability Blueprints that will show information on how they're developed and intended to be used.
* Change: OnAvatarSet is now called on the primary instance instead of the CDO for instanced per Actor Gameplay Abilities.
* Change: Allow both Activate Ability and Activate Ability From Event in the same Gameplay Ability Graph.
* Change: AnimTask_PlayMontageAndWait now has a toggle to allow Completed and Interrupted after a BlendOut event.
* Change: ModMagnitudeCalc wrapper functions have been declared const.
* Change: FGameplayTagQuery::Matches now returns false for empty queries.
* Change: Updated FGameplayAttribute::PostSerialize to mark the contained attribute as a searchable name.
* Change: Updated GetAbilitySystemComponent to default parameter to Self.
* Change: Marked functions as virtual in AbilityTask_WaitTargetData.
* Change: Removed unused function FGameplayAbilityTargetData::AddTargetDataToGameplayCueParameters.
* Change: Removed vestigial GameplayAbility::SetMovementSyncPoint.
* Change: Removed unused replication flag from Gameplay tasks & Ability system components.
* Change: Moved some gameplay effect functionality into optional components. All existing content will automatically update to use components during PostCDOCompiled, if necessary.

https://docs.unrealengine.com/5.3/en-US/unreal-engine-5.3-release-notes/

<a name="changelog-5.2"></a>
### 5.2
* Bug Fix: Fixed a crash in the `UAbilitySystemBlueprintLibrary::MakeSpecHandle` function.
* Bug Fix: Fixed logic in the Gameplay Ability System where a non-Controlled Pawn would be considered remote, even if it was spawned locally on the server (e.g. Vehicles).
* Bug Fix: Correctly set activation info on predicted instanced abilities that were rejected by the server.
* Bug Fix: Fixed a bug that would cause GameplayCues to get stuck on remote instances.
* Bug Fix: Fixed a memory stomp when chaining calls to WaitGameplayEvent.
* Bug Fix: Calling the AbilitySystemComponent `GetOwnedGameplayTags()` function in Blueprint no longer retains the previous call's return values when the same node is executed multiple times.
* Bug Fix: Fixed an issue with GameplayEffectContext replicating a reference to a dynamic object that would never be replicated.
  * This prevented GameplayEffect from calling `Owner->HandleDeferredGameplayCues(this)` as `bHasMoreUnmappedReferences` would always be true.
* New: The [Gameplay Targeting System](https://docs.unrealengine.com/en-US/gameplay-targeting-system-in-unreal-engine/) is a way to create data-driven targeting requests.
* New: Added custom serialization support for GameplayTag Queries.
* New: Added support for replicating derived FGameplayEffectContext types.
* New: Gameplay Attributes in assets are now registered as searchable names on save, allowing for references to attributes to be seen in the reference viewer.
* New: Added some basic unit tests for the AbilitySystemComponent.
* New: Gameplay Ability System Attributes now respect Core Redirects. This means you can now rename Attribute Sets and their Attributes in code and have them load properly in assets saved with the old names by adding redirect entries to DefaultEngine.ini.
* Change: Allow changing the evaluation channel of a Gameplay Effect Modifier from code.
* Change: Removed previously unused variable `FGameplayModifierInfo::Magnitude` from the Gameplay Abilities Plugin.
* Change: Removed the synchronization logic between the ability system component and Smart Object instance tags.

https://docs.unrealengine.com/5.2/en-US/unreal-engine-5.2-release-notes/

<a name="changelog-5.1"></a>
### 5.1
* Bug Fix: Fixed issue where replicated loose gameplay tags were not replicating to the owner.
* Bug Fix: Fixed AbilityTask bug where abilities could be blocked from timely garbage-collection.
* Bug Fix: Fixed an issue when a gameplay ability listening to activate based on a tag would fail to be activated. This would happen if there were more than one Gameplay Ability listening to this tag, and the first one in the list was invalid or didn't have authority to activate.
* Bug Fix: Fixed GameplayEffects that use Data Registries correctly from warning on load and improved the warning text.
* Bug Fix: Removed code from UGameplayAbility that was incorrectly only registering the last instanced ability with the Blueprint debugger for breakpoints.
* Bug Fix: Fixed Gameplay Ability System Ability getting stuck if EndAbility was called during the lock inside ApplyGameplayEffectSpecToTarget.
* New: Added support for Gameplay Effects to add blocked ability tags.
* New: Added WaitGameplayTagQuery nodes. One is based off of the UAbilityTask and the other is of UAbilityAsync. This node specifies a TagQuery, and will trigger its output pin when the query becomes true or false, based on configuration.
* New: Modified AbilityTask debugging in Console Variables to enable debug recording and printing to log by default in non-shipping builds (with ability to hotfix on/off as needed).
* New: You can now set AbilitySystem.AbilityTask.Debug.RecordingEnabled to 0 to disable, 1 to enable in non-shipping builds, and 2 to enable all builds (including shipping).
* New: You can use AbilitySystem.AbilityTask.Debug.AbilityTaskDebugPrintTopNResults to only print the top N results in log (to avoid log spam).
* New: STAT_AbilityTaskDebugRecording can be used to test perf impact from these on-by-default debugging changes.
* New: Added a debug command to filter GameplayCue events.
* New: Added new debug commandsAbilitySystem.DebugAbilityTags, AbilitySystem.DebugBlockedTags, andAbilitySystem.DebugAttribute to the Gameplay Ability System.
* New: Added a Blueprint function to get a debug string representation of a Gameplay Attribute.
* New: Added a new Gameplay Task resource overlap policy to cancel existing tasks.
* Change: Now Ability Tasks should make sure to call Super::OnDestroy only after they do anything needed to the Ability pointer, as it will be nulled out after calling it.
* Change: Converted FGameplayAbilitySpec/Def::SourceObject to be a weak reference.
* Change: Made a Ability System Component reference in the Ability Task a weak pointer so Garbage Collection can delete it.
* Change: Removed redundant enum EWaitGameplayTagQueryAsyncTriggerCondition.
* Change: GameplayTasksComponent and AbilitySystemComponent now support the registered subobject API.
* Change: Added better logging to indicate why Gameplay Abilities failed to be activated.
* Change: Removed AbilitySystem.Debug.NextTarget and PrevTarget commands in favor of global HUD NextDebugTarget and PrevDebugTarget commands.

https://docs.unrealengine.com/5.1/en-US/unreal-engine-5.1-release-notes/

<a name="changelog-5.0"></a>
### 5.0

https://docs.unrealengine.com/5.0/en-US/unreal-engine-5.0-release-notes/

<a name="changelog-4.27"></a>

## Unreal Engine 4

### 4.27
* Crash Fix: Fixed a root motion source issue where a networked client could crash when an Actor finishes executing an ability that uses a constant force root motion task with a strength-over-time modifier.
* Bug Fix: Fixed a regression in Editor loading time when using GameplayCues.
* Bug Fix: GameplayEffectsContainer's `SetActiveGameplayEffectLevel` method will no longer dirty FastArray if setting the same EffectLevel.
* Bug Fix: Fixed an edge case in GameplayEffect mixed replication mode where Actors not explicitly owned by the net connection but who utilize that connection from `GetNetConnection` will not received mixed replication updates.
* Bug Fix: Fixed an endless recursion occuring in GameplayAbility's class method `EndAbility` which was called by calling `EndAbility` again from `K2_OnEndAbility`.
* Bug Fix: GameplayTags Blueprint pins will no longer be silently cleared if they are loaded before tags are registered. They now work the same as GameplayTag variables, and the behavior for both can be changed with the ClearInvalidTags option in the Project Settings.
* Bug Fix: Improved thread safety of GameplayTag operations.
* New: Exposed SourceObject to GameplayAbility's `K2_CanActivateAbility` method.
* New: Native GameplayTags. Introducing a new `FNativeGameplayTag`, these make it possible to do one off native tags that are correctly registered and unregistered when the module is loaded and unloaded.
* New: Updated `GiveAbilityAndActivateOnce` to pass in FGameplayEventData parameter.
* New: Improved ScalableFloats in the GameplayAbilities plugin to support dynamic lookup of curve tables from the new Data Registry System. Added a ScalableFloat header for easier reuse of the generic struct outside the abilities plugin.
* New: Added code support for using the GameplayTag UI in other Editor customizations via GameplayTagsEditorModule.
* New: Modified UGameplayAbility's PreActivate method to optionally take in trigger event data.
* New: Added more support to filter GameplayTags in the Editor using a project-specific filter. `OnFilterGameplayTag` supplies the referencing property and the tag source, so you can filter tags based on what asset is requesting the tag.
* New: Added option to preserve the original captured SourceTags when GameplayEffectSpec's class method `SetContext` is called after initialization.
* New: Improved UI for registering GameplayTags from specific plugins. The new tag UI now lets you select a plugin location on disk for newly added GameplayTag sources.
* New: A new track has been added to Sequencer to allow for triggering notify states on Actors built using the GameplayAbiltiySystem. Like notifies, the GameplayCueTrack can utilize range-based events or trigger-based events.
* Change: Changed the GameplayCueInterface to pass GameplayCueParameters struct by reference.
* Optimization: Made several performance improvements to loading and regenerating the GameplayTag table were implemented so that this option would be optimized.

https://docs.unrealengine.com/en-US/WhatsNew/Builds/ReleaseNotes/4_27/

<a name="changelog-4.26"></a>
### 4.26 - Beta to Release
* GAS plugin is no longer flagged as beta.
* Crash Fix: Fixed a crash when adding a gameplay tag without a valid tag source selection.
* Crash Fix: Added the path string arg to a message to fix a crash in UGameplayCueManager::VerifyNotifyAssetIsInValidPath.
* Crash Fix: Fixed an access violation crash in AbilitySystemComponent_Abilities when using a ptr without checking it.
* Bug Fix: Fixed a bug where stacking GEs that did not reset the duration on additional instances of the effect being applied.
* Bug Fix: Fixed an issue that caused CancelAllAbilities to only cancel non-instanced abilities.
* New: Added optional tag parameters to gameplay ability commit functions.
* New: Added StartTimeSeconds to PlayMontageAndWait ability task and improved comments.
* New: Added tag container "DynamicAbilityTags" to FGameplayAbilitySpec. These are optional ability tags that are replicated with the spec. They are also captured as source tags by applied gameplay effects.
* New: GameplayAbility IsLocallyControlled and HasAuthority functions are now callable from Blueprint.
* New: Visual logger will now only collect and store info about instant GEs if we're currently recording visual logging data.
* New: Added support for redirectors on gameplay attribute pins in blueprint nodes.
* New: Added new functionality for when root motion movement related ability tasks end they will return the movement component's movement mode to the movement mode it was in before the task started.

https://docs.unrealengine.com/en-US/WhatsNew/Builds/ReleaseNotes/4_26/

<a name="changelog-4.25.1"></a>
### 4.25.1
* Fixed! UE-92787 Crash saving blueprint with a Get Float Attribute node and the attribute pin is set inline
* Fixed! UE-92810 Crash spawning actor with instance editable gameplay tag property that was changed inline

<a name="changelog-4.25"></a>
### 4.25
* Fixed prediction of `RootMotionSource` `AbilityTasks`
* [`GAMEPLAYATTRIBUTE_REPNOTIFY()`](#concepts-as-attributes) now additionally takes in the old `Attribute` value. We must supply that as the optional parameter to our `OnRep` functions. Previously, it was reading the attribute value to try to get the old value. However, if called from a replication function, the old value had already been discarded before reaching SetBaseAttributeValueFromReplication so we'd get the new value instead.
* Added [`NetSecurityPolicy`](#concepts-ga-netsecuritypolicy) to `UGameplayAbility`.
* Crash Fix: Fixed a crash when adding a gameplay tag without a valid tag source selection.
* Crash Fix: Removed a few ways for attackers to crash a server through the ability system.
* Crash Fix: We now make sure we have a GameplayEffect definition before checking tag requirements.
* Bug Fix: Fixed an issue with gameplay tag categories not applying to function parameters in Blueprints if they were part of a function terminator node.
* Bug Fix: Fixed an issue with gameplay effects' tags not being replicated with multiple viewports.
* Bug Fix: Fixed a bug where a gameplay ability spec could be invalidated by the InternalTryActivateAbility function while looping through triggered abilities.
* Bug Fix: Changed how we handle updating gameplay tags inside of tag count containers. When deferring the update of parent tags while removing gameplay tags, we will now call the change-related delegates after the parent tags have updated. This ensures that the tag table is in a consistent state when the delegates broadcast.
* Bug Fix: We now make a copy of the spawned target actor array before iterating over it inside when confirming targets because some callbacks may modify the array.
* Bug Fix: Fixed a bug where stacking GameplayEffects that did not reset the duration on additional instances of the effect being applied and with set by caller durations would only have the duration correctly set for the first instance on the stack. All other GE specs in the stack would have a duration of 1 second. Added automation tests to detect this case.
* Bug Fix: Fixed a bug that could occur if handling gameplay event delegates modified the list of gameplay event delegates.
* Bug Fix: Fixed a bug causing GiveAbilityAndActivateOnce to behave inconsistently.
* Bug Fix: Reordered some operations inside FGameplayEffectSpec::Initialize to deal with a potential ordering dependency.
* New: UGameplayAbility now has an OnRemoveAbility function. It follows the same pattern as OnGiveAbility and is only called on the primary instance of the ability or the class default object.
* New: When displaying blocked ability tags, the debug text now includes the total number of blocked tags.
* New: Renamed UAbilitySystemComponent::InternalServerTryActiveAbility to UAbilitySystemComponent::InternalServerTryActivateAbility.Code that was calling InternalServerTryActiveAbility should now call InternalServerTryActivateAbility.
* New: Continue to use the filter text for displaying gameplay tags when a tag is added or deleted. The previous behavior cleared the filter.
* New: Don't reset the tag source when we add a new tag in the editor.
* New: Added the ability to query an ability system component for all active gameplay effects that have a specified set of tags. The new function is called GetActiveEffectsWithAllTags and can be accessed through code or blueprints.
* New: When root motion movement related ability tasks end they now return the movement component's movement mode to the movement mode it was in before the task started.
* New: Made SpawnedAttributes transient so it won't save data that can become stale and incorrect. Added null checks to prevent any currently saved stale data from propagating. This prevents problems related to bad data getting stored in SpawnedAttributes.
* API Change: AddDefaultSubobjectSet has been deprecated. AddAttributeSetSubobject should be used instead.
* New: Gameplay Abilities can now specify the Anim Instance on which to play a montage.

https://docs.unrealengine.com/en-US/WhatsNew/Builds/ReleaseNotes/4_25/

<a name="changelog-4.24"></a>
### 4.24
* Fixed blueprint node `Attribute` variables resetting to `None` on compile.
* Need to call [`UAbilitySystemGlobals::InitGlobalData()`](#concepts-asg-initglobaldata) to use [`TargetData`](#concepts-targeting-data) otherwise you will get `ScriptStructCache` errors and clients will be disconnected from the server. My advice is to always call this in every project now whereas before 4.24 it was optional.
* Fixed crash when copying a `GameplayTag` setter to a blueprint that didn't have the variable previously defined.
* `UGameplayAbility::MontageStop()` function now properly uses the `OverrideBlendOutTime` parameter.
* Fixed `GameplayTag` query variables on components not being modified when edited.
* Added the ability for `GameplayEffectExecutionCalculations` to support scoped modifiers against "temporary variables" that aren't required to be backed by an attribute capture.
  * Implementation basically enables `GameplayTag`-identified aggregators to be created as a means for an execution to expose a temporary value to be manipulated with scoped modifiers; you can now build formulas that want manipulatable values that don't need to be captured from a source or target.
  * To use, an execution has to add a tag to the new member variable `ValidTransientAggregatorIdentifiers`; those tags will show up in the calculation modifier array of scoped mods at the bottom, marked as temporary variables—with updated details customizations accordingly to support feature
* Added restricted tag quality-of-life improvements. Removed the default option for restricted `GameplayTag` source. We no longer reset the source when adding restricted tags to make it easier to add several in a row. 
* `APawn::PossessedBy()` now sets the owner of the `Pawn` to the new `Controller`. Useful because [Mixed Replication Mode](#concepts-asc-rm) expects the owner of the `Pawn` to be the `Controller` if the `ASC` lives on the `Pawn`.
* Fixed bug with POD (Plain Old Data) in `FAttributeSetInitterDiscreteLevels`.

https://docs.unrealengine.com/en-US/WhatsNew/Builds/ReleaseNotes/4_24/

 