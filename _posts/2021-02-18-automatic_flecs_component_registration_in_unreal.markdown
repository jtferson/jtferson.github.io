---
layout: single
title:  "Automatic registration of Flecs components"
excerpt: "How to register Flecs component automatically."
date:   2021-02-17 23:00:00 +0300
categories: blog
tags: unreal flecs c++
---
Hi everyone! Today we'll implement the automatic registration of Flecs components in Unreal. 

As you might know, there is a problem with the current approach to component registration. We have to have a long boring list of all the components that aren't set before initializing systems/queries but might be used by them. The number of components is growing, and therefore it's growing a chance of forgetting to register some component and get a crash. 

Let's try to create a simple solution with a minimal boilerplate. The main idea of this solution is to keep registration functions somewhere and call them only when the time comes.

For that purpose, we need a global array which contains pointers to all registration functions. The only argument of these functions is the Flecs world.

**FlecsRegistration.h**
```cpp
namespace FlecsGlobals
{
	//You probably don't need a module API macro if you don't use UBT modules
	UNREALFLECS_API
	extern TArray<void (*)(flecs::world&)> FlecsRegs;
}
```
**FlecsRegistration.cpp**
```cpp
namespace FlecsGlobals
{
	TArray<void (*)(flecs::world&)> FlecsRegs;
}
```

Next, let's declare a template class with static functions. These functions will create pointers to the registration functions and will be indirectly called right after component declarations:

**FlecsRegistration.h**
```cpp
template<class T>
class FlecsComponentRegistration
{
	//the name of a component to register
	static const FString Name;
	
	static bool IsReg;
	//we will call this function indirectly and immediately after the component's declaration  
	static bool Init() 
	{
		//make a pointer to the registration function
		void (*func)(flecs::world&);
		func = &FuncReg;
		//add the pointer to the global array
		FlecsGlobals::FlecsRegs.Add(func);
		return true;
	}
	//our registration function
	static void FuncReg(flecs::world& InWorld)
	{
		InWorld.component<T>();
		UE_LOG(LogTemp, Warning, TEXT("FlecsComponent %s is registered"), *Name);
	}
};

//Define static variables
template<class T>
const FString FlecsComponentRegistration<T>::Name = FString("ComponentName");
template<class T>
bool FlecsComponentRegistration<T>::IsReg = FlecsComponentRegistration<T>::Init();
```

Then, let's create a macro which allows us to ease using all this boilerplate above:

**FlecsRegistration.h**
```cpp
#define FLECS_COMPONENT(Type) \
	const FString FlecsComponentRegistration<Type>::Name = FString(#Type); \
	bool FlecsComponentRegistration<Type>::IsReg = FlecsComponentRegistration<Type>::Init();
```

Lastly, we should actually call registration functions somewhere in the code after creating the world and before constructing systems/queries. 
I'm doing this in `UnrealFlecsSubsystem` only once:

**UUnrealFlecsSubsystem.cpp**
```cpp
void UUnrealFlecsSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
	ECSWorld = new flecs::world();

	UE_LOG(LogTemp, Warning, TEXT("Total Component Registrations %s"), *FString::FromInt(FlecsGlobals::FlecsRegs.Num()));
	for (auto reg : FlecsGlobals::FlecsRegs)
	{
		reg(*ECSWorld);
	}
}
```

So from now we need only to add a simple macro after declaring a component to register it automatically. For example:
```cpp
struct SpaceshipTarget
{
	flecs::entity Entity;
	FVector Position;
};
FLECS_COMPONENT(SpaceshipTarget)
```

I added this solution to the [UnrealFlecsQuickstart project](https://github.com/jtferson/UnrealFlecsQuickstart). Now after hitting Play I'm getting these lovely messages about component registrations:

![Screenshot](/assets/images/posts/component_registrations.png)

Farewell to a long list of the explicit component registrations!


The full source code of the solution:

**FlecsRegistration.h**
```cpp
namespace FlecsGlobals
{
	UNREALFLECS_API
	extern TArray<void (*)(flecs::world&)> FlecsRegs;
}

template<class T>
class FlecsComponentRegistration
{
	static const FString Name;
	static bool IsReg;
	static bool Init() 
	{
		void (*func)(flecs::world&);
		func = &FuncReg;
		FlecsGlobals::FlecsRegs.Add(func);
		return true;
	}
	static void FuncReg(flecs::world& InWorld)
	{
		InWorld.component<T>();
		UE_LOG(LogTemp, Warning, TEXT("FlecsComponent %s is registered"), *Name);
	}
};

template<class T>
const FString FlecsComponentRegistration<T>::Name = FString("ComponentName");
template<class T>
bool FlecsComponentRegistration<T>::IsReg = FlecsComponentRegistration<T>::Init();

#define FLECS_COMPONENT(Type) \
	const FString FlecsComponentRegistration<Type>::Name = FString(#Type); \
	bool FlecsComponentRegistration<Type>::IsReg = FlecsComponentRegistration<Type>::Init();

```
**FlecsRegistration.cpp**
```cpp
namespace FlecsGlobals
{
	TArray<void (*)(flecs::world&)> FlecsRegs;
}
```

**UUnrealFlecsSubsystem.cpp**
```cpp
void UUnrealFlecsSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
	ECSWorld = new flecs::world();

	UE_LOG(LogTemp, Warning, TEXT("Total Component Registrations %s"), *FString::FromInt(FlecsGlobals::FlecsRegs.Num()));
	for (auto reg : FlecsGlobals::FlecsRegs)
	{
		reg(*ECSWorld);
	}
}
```