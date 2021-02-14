---
layout: single
title:  "Quickstart with Flecs in Unreal. Part I"
excerpt: "How to start quckly with Flecs in Unreal."
date:   2021-02-12 17:25:00 +0300
categories: blog
toc: true
toc_sticky: true
tags: unreal flecs c++
---

> Relevance: Unreal 4.26 and Flecs 2.3

Hi everyone! Today is a long article about how to start quickly with [Flecs](https://github.com/SanderMertens/flecs) in Unreal. The article consists of two parts:

- Making Flecs part of the Unreal module system (this post)
- Creating Flecs-based Gameplay in Unreal ([the next post]({% post_url 2021-02-13-quickstart_with_flecs_in_unreal_part_2 %}))

Although posts are written in a tutorial manner, they're rather my personal experiences with Flecs in Unreal and don't pretend to be an optimal way of integrating Flecs with Unreal. 

And it seems that an implementation of ECS Space Battle in Unreal is like 'Hello World'. Let's make it a good tradition. So, at the end of two parts of the article we will have a battle between thousands of moving ships with projectiles that are dynamically instantiated. The source code is also included.

Let's begin.

## Adding Flecs to Unreal module system


> If you don't have plans to use [UBT modules](https://docs.unrealengine.com/en-US/ProductionPipelines/BuildTools/UnrealBuildTool/ModuleFiles/index.html) in your project, it will be enough to put flecs.h and flecs.c inside your project. Otherwise you'll need some small additional manipulations to run Flecs with UBT modules correctly.

UBT Modules are very useful for separating different functionalities and providing it for other modules.

In this post we'll create the following additional UBT Modules:

- **FlecsLibrary.** This module consists of only two Flecs source code files and nothing more.
- **UnrealFlecs.** This module is related to managing the lifecycle of the Flecs world and bootstrapping Flecs components and systems.
- **MainGameplay**. This module is for gameplay data and logic.

If you want to know how to create UBT modules easy and fast, check out [my other tutorial]({% post_url 2021-02-12-creating-ubt-module-fast %}).

## FlecsLibrary Module

First, let's setup the `FlecsLibrary` module. Put [flecs.h](https://raw.githubusercontent.com/SanderMertens/flecs/master/flecs.h) and [flecs.c](https://raw.githubusercontent.com/SanderMertens/flecs/master/flecs.c) in Public and Private folders respectively. 

Take a look at the top of `flecs.h` :
```c
#define flecs_STATIC

// Some macros in between

#ifndef flecs_STATIC
#if flecs_EXPORTS && (defined(_MSC_VER) || defined(__MINGW32__))
  #define FLECS_API __declspec(dllexport)
#elif flecs_EXPORTS
  #define FLECS_API __attribute__((__visibility__("default")))
#elif defined _MSC_VER
  #define FLECS_API __declspec(dllimport) // <-- What we prefer if we use Microsoft C/C++ compiler
#else
  #define FLECS_API // <--- What will be right now
#endif
#else
  #define FLECS_API
#endif
```

As you can see above, the compiler will replace `FlECS_API` with nothing, but we want to see Flecs functions to be accessible to other UBT modules. Let's do it.

Firstly, we should comment or delete macro `flecs_STATIC` in the very top of `flecs.h`.
```c
//#define flecs_STATIC
```

Then because UBT has no idea what `flecs_EXPORTS` is, we define this macro in `build.cs` file.

In the end `FlecsLibrary.build.cs` will look like this:
```csharp
using UnrealBuildTool;

public class FlecsLibrary : ModuleRules
{
	public FlecsLibrary(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

		PublicDependencyModuleNames.AddRange(new string[] {
			"Core",
			"CoreUObject",
			"Engine",
			"UFlecsQuickstart",
		});

		//The path for the header files
		PublicIncludePaths.AddRange(new string[] {"FlecsLibrary/Public"});
		//The path for the source files
		PrivateIncludePaths.AddRange(new string[] {"FlecsLibrary/Private"});

		//Important part because UBT has no idea what 'flecs_EXPORTS' is
		AppendStringToPublicDefinition("flecs_EXPORTS", "0");
	}
}
```

At last the compiler will replace macro `FLECS_API` with `__declspec(dllimport)` and Flecs will be accessed from other UBT modules.

## UnrealFlecs Module

Purposes of this module are as follows:

- Make the lifecycle of the Flecs world part of a certain `Unreal Subsystem`.  For example, if we subclass `UGameInstanceSubsystem`,  it will be quite simple to make the Flecs world alive only when Game Instance exists. It's also means we could have an easy access to the Flecs world in any Unreal scene.
- Encapsulate a set of Flecs components and system in UObject classes in order to expose them.
- Create a Bootstrap base class which will provide a start point for initializing flecs components and systems.

### Managing Flecs worlds by Unreal Subsystem

Let's create a new class inherited from `UGameInstanceSubsystem` and call it `UUnrealFlecsSubsystem` :

**UnrealFlecsSubsystem.h**
```cpp
UCLASS()
class UNREALFLECS_API UUnrealFlecsSubsystem : public UGameInstanceSubsystem
{
	GENERATED_BODY()
public:
	virtual void Initialize(FSubsystemCollectionBase& Collection) override;
	virtual void Deinitialize() override;

	// Getting the Flecs world if we need it
	flecs::world* GetEcsWorld() const;
protected:
	// UnrealSubsystem doesn't have a Tick function; instead
	// we use `FTickerDelegate` 
	FTickerDelegate OnTickDelegate;
	FDelegateHandle OnTickHandle;

	// UUnrealFlecsSubsystem contains a pointer to the current Flecs world
	flecs::world* ECSWorld = nullptr;
	
private:
	//Our custom Tick function used by `FTickerDelegate`
	bool Tick(float DeltaTime);
};
```

**UnrealFlecsSubsystem.cpp**
```cpp
void UUnrealFlecsSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
	OnTickDelegate = FTickerDelegate::CreateUObject(this, &UUnrealFlecsSubsystem::Tick);
	OnTickHandle = FTicker::GetCoreTicker().AddTicker(OnTickDelegate);

	//Create the Flecs world immediately after initialization of Game Instance
	ECSWorld = new flecs::world();
	
	UE_LOG(LogTemp, Warning, TEXT("UUnrealFlecsSubsystem has started!"));
	
	Super::Initialize(Collection);
}

void UUnrealFlecsSubsystem::Deinitialize()
{
	FTicker::GetCoreTicker().RemoveTicker(OnTickHandle);

	//Destroying the Flecs world as part of the deinitialization of Game Instance
	if(!ECSWorld) delete(ECSWorld);

	UE_LOG(LogTemp, Warning, TEXT("UUnrealFlecsSubsystem has shut down!"));
	
	Super::Deinitialize();
}

flecs::world* UUnrealFlecsSubsystem::GetEcsWorld() const
{
	return ECSWorld;
}

bool UUnrealFlecsSubsystem::Tick(float DeltaTime)
{
	// Every Tick we progress Flcs system 
	// and pass the value of Delta Time to the Flecs World
	if(ECSWorld) ECSWorld->progress(DeltaTime);
	return true;
}
```

### Flecs Bootstrap

Now that we've created **`UnrealFlecsSubsystem`,** we can use it to obtain the Flecs world for our Bootstrap class.

I'm purposely making the Bootstrap base class is very light, so that in the subclasses it would be possible declare Flecs systems right inside a `Bootstrap` function or use a bit different approach. It's up to you.

**FlecsBootstrap.cpp**
```cpp
**// Called when the game starts or when spawned
void AFlecsBootstrap::BeginPlay()
{
	UGameInstance* gameInstance = UGameplayStatics::GetGameInstance(GetWorld());
	UUnrealFlecsSubsystem* subSystem = gameInstance->GetSubsystem<UUnrealFlecsSubsystem>();

	Bootstrap(*subSystem->GetEcsWorld());
	
	Super::BeginPlay();
}

void AFlecsBootstrap::Bootstrap(flecs::world& ecs)
{
	// Here we can initialize flecs systems and components, if we'd like to
}**
```

### FlecsModuleBase

I want to separate sets of functionality into different modules and enable/disable them without any code modification. For that purpose, I'm going to create `UFlecsModuleBase` derived from simple `UObject`. In an `Initialize` function I will pass the current Flecs world to use it in the child classes for component registrations or system declarations:

**FlecsModuleBase.h**
```cpp
UCLASS(Abstract)
class UNREALFLECS_API UFlecsModuleBase : public UObject
{
	GENERATED_BODY()
public:
	virtual void Initialize(flecs::world& ecs);
};
```

## Gameplay Module

From now Flecs is integrated into Unreal, and we can have an easy access to the Flecs world. It's time to subclass our Bootstrap base class and expand its functionality in order to take the first step towards building gameplay.

In `AMainGameplayBootstrap`  I'm adding a `FlecsModules` field that is simply an array of any types inherited from `UFlecsModuleBase`:

**MainGameplayBootstrap.h**
```cpp
UCLASS()
class MAINGAMEPLAY_API AMainGameplayBootstrap : public AFlecsBootstrap
{
	GENERATED_BODY()

public:
	// Sets in a blueprinted class
	UPROPERTY(EditAnywhere)
	TArray<TSubclassOf<UFlecsModuleBase>> FlecsModules;
protected:
	virtual void Bootstrap(flecs::world& ecs) override;
};
```

Then in the cpp I'm iterating over all the types of the Flecs modules. Inside the loop I create each Module and call a `Initialize` function:

**MainGameplayBootstrap.cpp**
```cpp
void AMainGameplayBootstrap::Bootstrap(flecs::world& ecs)
{
	for(auto moduleType : FlecsModules)
	{
		auto module = NewObject<UFlecsModuleBase>(this, moduleType);
		module->Initialize(ecs);
	}
}
```

That's it. With that approach we can freely attach different sets of Flecs functionality to the scene and reorder them if we need. 

### Custom Flecs Modules

Now we can create our custom Flecs modules. For our purposes, we'll limit ourselves to just three modules:

- **Components Module**. Declaring components and their registrations.
- **Inits Module**. Creating initial entities. And in the next post also authoring Unreal data into complex Flecs entities.
- **Systems Module**. Containing all the logic related to gameplay.

**MainGameplay_Components.h**
```cpp
struct Transform
{
	FTransform Value;
};

UCLASS()
class MAINGAMEPLAY_API UMainGameplay_Components : public UFlecsModuleBase
{
	GENERATED_BODY()

	virtual void Initialize(flecs::world& ecs) override;
};
```


As it was said, this module is clearly just about declaring components in the header and their registrations in the cpp. It's quite important don't forget to register every component which you declared ealier in the header file. Flecs component auto-registration doesn't work in Unreal, for the time being. So, if a component is used in systems and don't have a proper registration, it can even cause a crash.

**MainGameplay_Components.cpp**
```cpp
void UMainGameplay_Components::Initialize(flecs::world& ecs)
{
	//Register Transform component
	ecs.component<Transform>();
}
```

 

**MainGameplay_Inits.cpp**
```cpp
void UMainGameplay_Inits::Initialize(flecs::world& ecs)
{
	//Create our first entity with Transform component
	ecs.entity().set<Transform>({FTransform{FVector(10, 20, -20)}});
}
```

**MainGameplay_Systems.cpp**
```cpp
 
void SystemTestFlecs(flecs::iter& It)
{
	auto cTransform = It.column<Transform>(1);
	for(auto i : It)
	{
		//Just want to see a message every tick in the output log
		UE_LOG(LogTemp, Warning, TEXT("Time %s. Entity %s with Position %s"),
			*FString::SanitizeFloat(It.world_time()),
			*FString::FormatAsNumber(It.entity(i).id()),
			*cTransform[i].Value.ToString());
	}
}

void UMainGameplay_Systems::Initialize(flecs::world& ecs)
{
	//Initialize our first Flecs system
	flecs::system<>(ecs, "SystemTestFlecs")
		.signature("Transform")
		.iter(SystemTestFlecs);
}
```

We've created our Modules and now we can add them to our blueprinted Bootstrap on the scene:

![Screenshot](/assets/images/posts/part1_image1.png)

As a rule of thumb, I always put the modules which contains component registrations **BEFORE** any other modules. It prevents any issues related to executing systems with unregistered components.

Finally, we are ready to push the Play button and see the result. If everything is OK, something like this will appear in the Unreal's output log:

![Screenshot](/assets/images/posts/part1_image2.png)

And after that the most interesting part begins. [Let's move on]({% post_url 2021-02-13-quickstart_with_flecs_in_unreal_part_2 %}).