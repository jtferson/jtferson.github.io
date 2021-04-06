---
layout: single
title:  "Automatic registration of Flecs components"
excerpt: "How to register Flecs component automatically."
date:   2021-02-17 23:00:00 +0300
categories: blog
tags: unreal flecs c++
---

| **2021-04-06 Update**
| I've replaced the previous solution with the one that's based on the local static array. The issue with the previous one was relying on global variables and therefore on their unspecific (in some cases) order of initializing. It worked well for the UnrealFlecs project but might not work at all for other projects. From now this solution uses the local static variable, and it guarantees that the array of component registrations is initialized in the first call and only once. Descriptions in this post and the source code were changed accordingly.

Hi everyone! Today we'll implement the automatic registration of Flecs components in Unreal. 

As you might know, there is a problem with the current approach to component registration. We have to have a long boring list of all the components that aren't set before initializing systems/queries but might be used by them. The number of components is growing, and therefore it's growing a chance of forgetting to register some component and get a crash. 

Let's try to create a simple solution with a minimal boilerplate. The main idea of this solution is to keep registration functions somewhere and call them only when the time comes.

For that purpose, we need a local static array which contains pointers to all registration functions. The only argument of these functions is the Flecs world.

**FlecsRegistration.h**
```cpp
class FlecsRegContainer
{
public:
	//You probably don't need a module API macro if you don't use UBT modules
	UNREALFLECS_API
    static TArray<void (*)(flecs::world&)>& GetFlecsRegs();
};
```
**FlecsRegistration.cpp**
```cpp
TArray<void(*)(flecs::world&)>& FlecsRegContainer::GetFlecsRegs()
{
	static TArray<void(*)(flecs::world&)> instance;
	return instance;
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
		//make a function pointer to the registration function
		void (*func)(flecs::world&);
		func = &FuncReg;
		//add the pointer the registation array
		TArray<void (*)(flecs::world&)>& regs = FlecsRegContainer::GetFlecsRegs();
		regs.Add(func);
		return true;
	}
	//our registration function
	static void FuncReg(flecs::world& InWorld)
	{
		InWorld.component<T>();
		UE_LOG(LogTemp, Warning, TEXT("FlecsComponent %s is registered"), *Name);
	}
};

//Define static templated variables
template<class T>
const FString FlecsComponentRegistration<T>::Name = FString("ComponentName");
template<class T>
bool FlecsComponentRegistration<T>::IsReg = FlecsComponentRegistration<T>::Init();
```

Then, let's create a set of macros which allows us to ease using all this boilerplate above:

**FlecsRegistration.h**
```cpp
#define FLECS_COMPONENT(Type) \
struct Type; \
const FString FlecsComponentRegistration<Type>::Name = FString(#Type); \
bool FlecsComponentRegistration<Type>::IsReg = FlecsComponentRegistration<Type>::Init(); \
struct Type
```
But since UHT parses header before compiler replaces macro, we can't really use the above macro with components that are wrapped by Unreal macros (USTRUCT, for example). So, we need to use a similar macro but only after declaring such component:

**FlecsRegistration.h**
```cpp
#define REG_COMPONENT(Type) \
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

    auto& regs = FlecsRegContainer::GetFlecsRegs();
    for (auto reg : regs)
    {
    	reg(*ECSWorld);
    }
	UE_LOG(LogTemp, Warning, TEXT("Total Component Registrations %s"), *FString::FromInt(regs.Num()));
}
```

So from now we need only to add simple macros before or after declaring a component to register it automatically. Some use cases:
```cpp
//Use FLECS_COMPONENT macro for regular components which are not wrapped by USTRUCT
FLECS_COMPONENT(Spaceship)
{
     float MaxVelocity;
};

//Use REG_COMPONENT macro for USTRUCT components after their declaring
USTRUCT(BlueprintType)
struct FSpaceship {
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere)
    float MaxVelocity;
};
REG_COMPONENT(FSpaceship)
```

I added this solution to the [UnrealFlecsQuickstart project](https://github.com/jtferson/UnrealFlecsQuickstart). Now after hitting Play I'm getting these lovely messages about component registrations:

![Screenshot](/assets/images/posts/component_registrations.png)

Farewell to a long list of the explicit component registrations!


The full source code of the solution:

**FlecsRegistration.h**
```cpp
class FlecsRegContainer
{
public:
	UNREALFLECS_API
    static TArray<void (*)(flecs::world&)>& GetFlecsRegs();
};

template<class T>
class FlecsComponentRegistration
{
	static const FString Name;
	static bool IsReg;
	static bool Init() 
	{
		void (*func)(flecs::world&);
		func = &FuncReg;
		TArray<void (*)(flecs::world&)>& regs = FlecsRegContainer::GetFlecsRegs();
		regs.Add(func);
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
struct Type; \
const FString FlecsComponentRegistration<Type>::Name = FString(#Type); \
bool FlecsComponentRegistration<Type>::IsReg = FlecsComponentRegistration<Type>::Init(); \
struct Type

#define REG_COMPONENT(Type) \
const FString FlecsComponentRegistration<Type>::Name = FString(#Type); \
bool FlecsComponentRegistration<Type>::IsReg = FlecsComponentRegistration<Type>::Init();
```
**FlecsRegistration.cpp**
```cpp
TArray<void(*)(flecs::world&)>& FlecsRegContainer::GetFlecsRegs()
{
	static TArray<void(*)(flecs::world&)> instance;
	return instance;
}
```

**UUnrealFlecsSubsystem.cpp**
```cpp
void UUnrealFlecsSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
	ECSWorld = new flecs::world();

	auto& regs = FlecsRegContainer::GetFlecsRegs();
    for (auto reg : regs)
    {
    	reg(*ECSWorld);
    }
	UE_LOG(LogTemp, Warning, TEXT("Total Component Registrations %s"), *FString::FromInt(regs.Num()));
}
```