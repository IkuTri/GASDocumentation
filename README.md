
# Readme - Fork of GASDocs for Obsidian Rewrite

This is my rewrite to better understand Gameplay Ability System. I rewrote it in Obsidian.

## Using this Git Repo
Since this is a fork, I put the original uproject in it's own folder.

### Installation Steps

1. Clone this repo.
2. Install *Obsidian*.
3. Launch the application, and choose to open an existing folder as a *vault*.
4. Choose the "GASDocs" folder.
5. Allow / install plugins as you require (optional)

### How to Use Markdown
- Step by step: [Markdown Tutorial](https://www.markdowntutorial.com/)
- Video in English with Cheat Sheet: [The Only Markdown Crash Course You Will Ever Need - YouTube](https://youtu.be/_PPWWRV6gbA)
	- This tutorial uses another program

### How to Use Obsidian

#### Full Tutorials
[Obsidian: The King of Learning Tools (FULL GUIDE + SETUP) - YouTube](https://youtu.be/hSTy_BInQs8)
- I like how this tutorial is self aware about how complicated it can get
- Also talks about how to gain traction with writing / thinking with more advanced topics

#### Core Concepts
>Enough hopefully to get you started.
- A *root folder* is the base folder's "inside", where files and more folders are stored.
- An *Obsidian Vault* is the *root folder* for whatever you are working on.
- You can have as many Vaults as you want
	- You can Sync them with their paid service
	- Or you can use the *Git Plugin* (Community)
	- Apple Devices have the option to use *iCloud* (Core)
- A *Workspace* is saved Tabs / Windows on your Computer (Desktop)
#### Core Plugins
- This Vault uses *File Properties* to deal with tags.
- I use the *Graph View* to help organize with 
- Personally, I'd disable *Page Preview* 
- *Workspaces* are from the *Workspace Plugin*

#### Community Plugins (Recommended)
I don't want to link these, since you should just go to Settings and install them there. 

They all seem to be git repos if you do have an issue you'd like to file -- or if you want to help with their codebase.

- *Vertical Tabs* - My preferred workflow / workspace management
- *Tag Wrangler* - Allows mass renaming of tags, etc.
- *Advanced Tables* - Smoother dealing with tables. 
	- Handles to move things around are really nice.
- *Slash Commander* - Allows commands from `/`, very nice for quick spawning of tables, callouts, etc.
- *Git* - for using a *Git Repository*

# Tranek's Original GASDocumentation Intro
My understanding of Unreal Engine 5's GameplayAbilitySystem plugin (GAS) with a simple multiplayer sample project. This is not official documentation and neither this project nor myself are affiliated with Epic Games. I make no guarantee for the accuracy of this information.

The goal of this documentation is to explain the major concepts and classes in GAS and provide some additional commentary based on my experience with it. There is a lot of 'tribal knowledge' of GAS among users in the community and I aim to share all of mine here.

The Sample Project and documentation are current with **Unreal Engine 5.3** (UE5). There are branches of this documentation for older versions of Unreal Engine, but they are no longer supported and are liable to have bugs or out of date information. Please use the branch that matches your engine version.

[GASShooter](https://github.com/tranek/GASShooter) is a sister Sample Project demonstrating advanced techniques with GAS for a multiplayer FPS/TPS.

The best documentation will always be the plugin source code.

