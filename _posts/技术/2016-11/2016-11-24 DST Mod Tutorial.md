---
layout: post
title: Learn Mod Of Don't Starve
category: 技术
tags: game
keywords: mod, don't starve
description: create mod for the game, Don't starve
---

## Creature Mod Tutorial 1 - "Creating a Mod"


In this tutorial we're going to learn how to create our very own mod for 'Don't Starve'.  Mods are a way for players to add their own content to the game.  Creating a mod is really easy.  All we need is two files and a folder and we're good to go.  Let's get started!

-  **Step 1 - Creating a Mod Folder**  
	The very first thing we need to do is create a folder where our mod is going to live.  You can do this by browsing to your 'Don't Starve' directory and inside of this, there should already be a 'mods' folder.  Now create a new folder in the 'mods' directory and give it a name representative of your mod.

-  **Step 2 - Creating 'modinfo.lua'**  
	For the next step, we need to setup a description for this mod.  We do that by adding a file to our mod folder you need to create a file in your mod folder called 'modinfo.lua'.  The easiest way to create this file is to copy it from another mod such as the one found in this tutorial.  Once you've copied 'modinfo.lua' into your mod folder, open it with any text editor and fill out all the information about your mod.  This info will show up in the 'mods' menu next time you restart the game.

-  **Step 3 - Creating 'modmain.lua'**  
	For the final step, we need to create a file called 'modmain.lua'.  For this tutorial, we're going to leave this file empty but in future tutorials, we'll see how this file is used to hook your mod into the game.

And that's it!  Now next time you restart the game, you should see your mod listed when clicking on the 'mods' menu on the right side of the main menu.  Our mod doesn't do anything interesting just yet but we'll get to that in the next tutorial.

## Creature Mod Tutorial 2 - "Spawning a Creature"

In this tutorial we're going to learn how to spawn an in game creature at the player's position.  We're doing this so that later on in future tutorials when we're creating our own creatures, we can easily test them in game.  Let's get started!

-  **Step 1 - Writing a spawn function.**  
	We need to write ourselves a function which will spawn our creature.  I've chosen to spawn a 'Beefalo' but feel free to pick your own creature.  To see how this is done, check out the function 'SpawnCreature' found in 'modmain.lua'.

- **Step 2 - Triggering our function.**  
	Writing our spawn function isn't enough.  We need some way of telling the game to run our function.  Luckily for us, 'Don't Starve' let's us do that by calling 'AddSimPostInit'.  This function tells the game to run our function once the game has started and the player has spawned in the world.  By adding this call at the end of modmain.lua, we're telling the game to call 'SpawnCreature' once we've entered the game.

And that's it!  By enabling your mod in the main menu, you should now have a shiny new 'Beefalo' spawn right on top of you any time you jump into the game!

## Creature Mod Tutorial 3 - "Importing animation from Spriter"

In this tutorial we're going to import a brand new creature from 'Spriter'.  Spriter is a 2D animation tool that comes packaged with the Don't Starve Mod tools.  Let's get started!

-  **Step 1 - Importing a Spriter Project**  
	Spriter saves it's data to '.scml' files.  To import a Spriter project, all we need to do is copy the Spriter '.scml' file and all of it's images to a subfolder in our mod called 'exported'.  In this tutorial's case, the spriter file is called 'tut03.scml'.

-  **Step 2 - Creating a Prefab.**  
	All objects in 'Don't Starve' are spawned from prefabs.  A prefab is a way of defining an object that can be spawned in the game.  So to spawn our newly imported Spriter creature, we need to create a new prefab.  To add a new prefab, you need to create a subfolder for your mod called 'scripts' and inside that folder, create another subfolder called 'prefabs'.  This is where we put new prefabs create by our mod.  For this tutorial, our prefab file is called 'tut03.lua'.

-  **Step 3 - Registering our Prefab.**  
	Everything our mod contains needs to be registered within 'modmain.lua'.  In this case we want to register a new prefab so at the top of 'modmain.lua' we need to list our prefab files.

-  **Step 4 - Spawning our Prefab.**  
	We now need to spawn an instance of our prefab in game.  So again in 'modmain.lua' we need to change our 'SpawnCreature' function to spawn our new prefab instead of the Beefalo.

And that's it!  If you've followed all the steps correctly, you should now see your creature spawn in game and play it's idle animation!

## Creature Mod Tutorial 4 - "Locomotion"

In this tutorial we're going to learn how to move our creature around the world.  To do this we need to add two components to our creature.  A locomotor component which handles movement and a physics component which allows the locomotor component to do collision detection with the world.  Let's get started!

-  **Step 1 - Adding physics**  
	The first thing we need to do is add physics to our creature.  Luckily for us, most object types in 'Don't Starve' have helper functions to setup their physics. You can see how we use the 'MakeCharacterPhysics' function in our prefab 'scripts/prefabs/tut04.lua'.

- **Step 2 - Attaching a locomotor.**  
	The locomotor component is what let's our creature find it's way through the world.  It handles setting up paths and avoiding obstacles which is why it needs our physics component to be set up.  Similar to the physics component, you can also see how this is setup in 'scripts/prefabs/tut04.lua'.

-  **Step 3 - Run.**  
	In future tutorials, we will see how we can choose to move based on behaviours but for now, we're simply going to tell our creature to run forward at the bottom of *'scripts/prefabs/tut04.lua'*.

And that's it!  Now our creature can move around the world.  Right now it looks more like sliding across the world but in our next tutorial, we'll look at how we can setup a stategraph to animate our creature based on it's movement.