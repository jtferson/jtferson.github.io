---
layout: single
title:  "Flecs + UE4 is magic"
date:   2021-01-17 14:58:47 +0300
categories: blog
---
Over the last six months I've been heavily using [Flecs](https://github.com/SanderMertens/flecs) with Unreal. In my opinion, Flecs is one of the most powerful and flexible frameworks that seamlessly brings ECS to Unreal universe.

So, why is that?

- It's easy to add Flecs to the Unreal project. Just put amalgamated flecs files in your source code and compile. No UBT pain at all! It's really surprising how smoothly Flecs can be integrated into Unreal.
- For the complex projects that are usually split into many modules you may consider a way of making Flecs part of the module/plugin Unreal system that is also a simple thing to do.
- Unreal Subsystems are perfect to manage the lifecycle of the Flecs world. It also means that you can get painless access to the Flecs world across the board.
- For some logic related to Unreal runtime objects it's straightforward just to use pointers to your AActors and UObjects inside Flecs components.
- For example, if using C++-based UI and connecting C++ to UMG through BindWidget as I'm doing, then nothing changes for a UI developer - you just need to expose the Flecs world to your widgets and create special UObjects which will be a bridge for data flow between UMG and Flecs. After that it's quite easy to make UI data to be updated from the Flecs side on the event or tick basis.

As an example of what you can make using Unreal with Flecs:

{% include youtube.html id='bvDUuG-esLU' %}
		
It's still WIP, but all game logic including object manager, camera controller, custom Floating Origin system and other are made using Flecs.

Some other things need to know before jumping into Flecs + UE4:

- Because of the hot reloading that is employed by Unreal and other game engines, automatic component registration doesn't work properly for the time being. A workaround, which I found not perfect but passable, is to manually register a list of all your components after recreating the world and before creating Flecs systems. However, that issue is already investigated by the Flecs team.
- If you don't want to define all the initial objects of a scene in the code, you need to make your own authoring system. Personally, I've been working on various options of how to initially populate Flecs worlds, including authoring and conversion from AActors, DataAssets and even LUA scripts.


## Conclusion

Flecs looks very promising, and I believe has a path to becoming a mature fully production-ready ECS framework.

That's all for today. In the next articles I will show:
- How to setup Flecs as part of Unreal module system
- How to manage the Flecs world in Unreal subsystems
- How to expose your custom Flecs modules to make them manageable by a Level Bootstrap object
- And finally how to easily build a bridge between Flecs and UMG to make an efficient system for populating and updating UI data
