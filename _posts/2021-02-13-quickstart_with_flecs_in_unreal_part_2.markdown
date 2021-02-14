---
layout: single
title:  "Quickstart with Flecs in Unreal. Part II"
excerpt: "How to start quckly with Flecs in Unreal."
date:   2021-02-12 17:26:00 +0300
categories: blog
toc: true
toc_sticky: true
tags: unreal flecs c++
---

In [the previous part of the article]({% post_url 2021-02-12-quickstart_with_flecs_in_unreal_part_1 %}) we've created a basic infrastructure to bring Flecs to Unreal. Now we are ready to construct some basic gameplay foundation using Unreal rendering and Flecs-based game logic.

In this post I will cover the following things:

- ISM-based rendering and its integration with Flecs
- Gameplay mechanics (without covering details, but the source code is always available)
- GameplayConfig and authoring initial entities

## ISM-based rendering

We want to have many thousands of ships and projectiles, so that's obvious to select `InstancedStaticMesh` as the main tool for rendering. It's definitely possible to use **Niagara** to render a insanely huge amount of mesh instances, but that seems that Niagara systems are not so easy to setup from C++ side yet. So, we'll stop a choice on `InstancedStaticMesh`for now.

Firstly, let's create a standard `AActor` class with a `UInstancedStaticMeshComponent`:

**AISMController.h**
```cpp
UCLASS()
class MAINGAMEPLAY_API AISMController : public AActor
{
	GENERATED_BODY()
public:
	AISMController();
	// ISM component
	UPROPERTY(EditAnywhere)
	UInstancedStaticMeshComponent* InstancedStaticMeshComponent = nullptr;

	// Some future API functions
	//...
private:
	// Each instance of ISM has its transform stored in this array
	TArray<FTransform> transforms;
};
```

**AISMController.cpp**
```cpp
AISMController::AISMController()
{
	PrimaryActorTick.bCanEverTick = false;

	InstancedStaticMeshComponent = CreateDefaultSubobject<UInstancedStaticMeshComponent>(TEXT("InstancedStaticMeshComponent"));
	SetRootComponent(InstancedStaticMeshComponent);
}
```

`AISMController` will be created once and automatically for every unique combination of a material and mesh. For that purpose, a special component will contain a map container with keys representing a combined hash of a material and mesh:

**MainGameplay_Components.h**
```cpp
struct ISM_Map
{
	TMap<uint32, AISMController*> ISMs;
};
```

An example of creating `AISMController` with adding it to the `ISM_Map` component:
```cpp
//MainGameplay_Inits.cpp
int32 CreateISMController(UWorld* InWorld, UStaticMesh* InMesh, UMaterialInterface* InMaterial, ISM_Map& InMap)
{
	auto hash = HashCombine(GetTypeHash(InMaterial), GetTypeHash(InMesh));

	auto find = InMap.ISMs.Find(hash);
	if (find == nullptr)
	{
		FVector Location = FVector::ZeroVector;
		FRotator Rotation = FRotator::ZeroRotator;
		FActorSpawnParameters SpawnInfo;

		auto controller = Cast<AISMController>(
			InWorld->SpawnActor(AISMController::StaticClass(), &Location, &Rotation, SpawnInfo));
		controller->Initialize(InMesh, InMaterial);

		InMap.ISMs.Add(hash, controller);
	}
	return hash;
}

//AISMController.cpp
void AISMController::Initialize(UStaticMesh* InMesh, UMaterialInterface* InMaterial) const
{
	InstancedStaticMeshComponent->SetStaticMesh(InMesh);
	InstancedStaticMeshComponent->SetMaterial(0, InMaterial);

	//...
}
```


The prefab system is one of the most powerful things that are provided by Flecs. Thanks to Flecs prefabs we can easily have the universal logic for creating an entity and all the needed components and then adding the entity as an instance to the ISM component:
```cpp
// MainGameplay_Components.h

//Every instance has a pointer to its ASIMController
struct ISM_ControllerRef
{
	AISMController* Value;
};
//Every instance has the hash of its AISMController
struct ISM_Hash
{
	int32 Value;
};
//Every instance gets a unique index inside AISMController
struct ISM_Index
{
	int Value;
};

//Event structure for adding a new instance
//Contains the hash of AISMController, prefab's Id, and initial FTransfom
struct ISM_AddInstance
{
	int32 Hash;
	flecs::entity Prefab;
	FTransform Transform;
};

// MainGameplay_Systems.cpp
void SystemAddInstance(flecs::iter& It)
{
	auto ecs = It.world();
	auto cAdd = It.column<ISM_AddInstance>(1);
	auto cMap = It.column<ISM_Map>(2);

	for (auto i : It)
	{
		auto controller = *(cMap->ISMs.Find(cAdd[i].Hash));
		if (controller != nullptr)
		{
			//Return a newly created or released index
			auto instanceIndex = controller->AddInstance();
			ecs.entity()
			   .add_instanceof(cAdd[i].Prefab)
			   .set<ISM_ControllerRef>({controller})
			   .set<ISM_Index>({instanceIndex})
			   .set<ISM_Hash>({cAdd[i].Hash})
			   .set<Transform>({cAdd[i].Transform});
		}
		It.entity(i).destruct();
	}

	for (auto& data : cMap->ISMs)
	{
		// After adding all instances, 
		// create or expand the transform array
		data.Value->CreateOrExpandTransformArray();
	}
}

// AISMController.cpp
int32 AISMController::AddInstance(FVector location)
{
	int32 instanceIndex;
	if(indexPool.IsEmpty())
	{
		FTransform transform{location};
		instanceIndex = InstancedStaticMeshComponent->AddInstance(transform);
	}
	else
	{
		// Getting an index from the pool of reserve indices
		// More on this later
		indexPool.Dequeue(instanceIndex);
		InstancedStaticMeshComponent->SetCustomDataValue(instanceIndex, 0, 0);
	}
	return instanceIndex;
}

void AISMController::CreateOrExpandTransformArray()
{
	if(GetInstanceCount() != transforms.Num())
	{
		transforms.AddUninitialized(GetInstanceCount() - transforms.Num());
		for (auto i = 0; i < transforms.Num(); i++)
		{
			transforms[i] = FTransform{FVector::ZeroVector};
		}
	}
}

```

Since we potentially can have thousands of instance transforms we need a performant and flexible way of updating them:
```cpp
// MainGameplay_System.cpp
// Before updating instance transforms calling ISM functions,
// we copy a new value of transform in the temporary array inside AISMController
void SystemCopyInstanceTransforms(flecs::iter& It)
{
	auto cTransform = It.column<Transform>(1);
	auto cISMIndex = It.column<ISM_Index>(2);
	auto cISMController = It.column<ISM_ControllerRef>(3);

	for (auto i : It)
	{
		auto index = cISMIndex[i].Value;
		cISMController[i].Value->SetTransform(index, cTransform[i].Value);
	}
}

// AISMController.cpp
void AISMController::SetTransform(int32 instanceIndex, const FTransform& transform)
{
	transforms[instanceIndex] = transform;
}

```

```cpp
// MainGameplay_Systems.cpp
// Now at the end of frame we update all the transforms in one batch for each ISM
void SystemUpdateTransformsInBatch(flecs::iter& It)
{
	auto cMap = It.column<ISM_Map>(1);
	for (auto& data : cMap->ISMs)
	{
		data.Value->BatchUpdateTransform();
	}
}

// AISMController.cpp
void AISMController::BatchUpdateTransform() const
{
	if(transforms.Num() > 0)
	{
		InstancedStaticMeshComponent->BatchUpdateInstancesTransforms(0, transforms, true, true);
	}
}
```

One more thing. As you could see above, I've been using some pool of indices to get an index for a new instance. It means that an instance which we want to remove isn't really being removed inside `AISMController`.  Quite opposite thing is happening instead; an index is being added to the pool of reserve indices. Then that index can be obtained from this pool to be used by a new instance. Otherwise we'd need to recalculate the indices of all active instances that are stored in `ISM_Index` components. 

The next step is to somehow hide a "removed" instance because `ISM` will render it anyway. That's a job for `SetCustomDataValue` - a ISM function that allows to pass float data to instances.

That's exactly how I'm doing this:

**AISMController.cpp**
```cpp
void AISMController::Initialize(UStaticMesh* InMesh, UMaterialInterface* InMaterial) const
{
	//... Some unrelevant logic here

	//Define that we have only one custom data for instances right now
	InstancedStaticMeshComponent->NumCustomDataFloats = 1;
}

void AISMController::RemoveInstance(int32 instanceIndex)
{
	//Add the index of a removed instanced to the pool
	indexPool.Enqueue(instanceIndex);
	
	//Pass the instance's index, custom data index, and custom data value
	//Value == 0 - an instance is shown (default value)
	//Value == 1 - an instance is hidden
	InstancedStaticMeshComponent->SetCustomDataValue(instanceIndex, 0, 1);
}
```

Then we need to modify our materials adding special nodes in which we obtain custom data on a per-instance basis. For the sake of simplicity in order to hide an instance we simply set its Opacity equal to 0: 

![Screenshot](/assets/images/posts/part2_image1.png)

## Gameplay Mechanics

OK, after implementing ISM-based rendering, we are ready to build some gameplay. Actually, I won't really focus on all the gameplay logic details because covering it is not a purpose of this post, and you can always check out the source code if you'd like to. However, here's some important points:

- Boid simulation is in use to move ships of both teams in a visually natural way. In fact, I has been invented nothing new in this matter. Moreover, I partly include some code originally made by Bogdan Codreanu: [https://github.com/BogdanCodreanu/ECS-Boids-Murmuration_Unity_2019.1](https://github.com/BogdanCodreanu/ECS-Boids-Murmuration_Unity_2019.1)
- Subdividing space into hashed cells could be implemented better. For example, I've been using two different systems for computing cells for enemy searching and boid movements instead of just one.
- Beams are made in the simplest way as meshes that are non-uniformly scaled in one direction
- It's possible instead of beams to use moving projectiles, but today I love beams more.
- All algorithms are far from a perfect state.  The application is single-threaded. There is a room for optimizations.

## Gameplay Config, Models and Inits

`GameplayConfig` is a complex `DataAsset` that will consist of other nested `DataAssets` including the following things:

- Weapon Types
- Spaceship Types
- Teams

Additionally, `GameplayConfig` contains some properties related to a Boid simulation and Arena settings.

From a code architecture perspective `GameplayConfig` is part of `MainGameplayBootstrap` and needs for authoring the initial entities.

### GameplayConfig

Let's start with creating `DataAssets`:

**SpaceshipWeapon.h**
```cpp
UENUM(BlueprintType)
enum class EWeaponType : uint8
{
	Bolt = 0,
	Beam = 1,
};
UCLASS()
class MAINGAMEPLAY_API USpaceshipWeapon : public UDataAsset
{
	GENERATED_BODY()
public:
	UPROPERTY(EditAnywhere)
	UStaticMesh* Mesh = nullptr;
	UPROPERTY(EditAnywhere)
	UMaterialInterface* Material = nullptr;
	UPROPERTY(EditAnywhere)
	EWeaponType WeaponType;
	UPROPERTY(EditAnywhere)
	float Cooldown;
	UPROPERTY(EditAnywhere, Category=Projectile)
	float Lifetime;
	UPROPERTY(EditAnywhere, Category=Projectile)
	float Speed;
	UPROPERTY(EditAnywhere, Category=Projectile)
	float ProjectileScale;
	//Some value that needs to calculate the beam scale correctly
	UPROPERTY(EditAnywhere, Category=Projectile)
	float BeamMeshLength;
};
```

**SpaceshipType.h**
```cpp
UCLASS()
class MAINGAMEPLAY_API USpaceshipType : public UDataAsset
{
   GENERATED_BODY()
public:
   UPROPERTY(EditAnywhere)
   UStaticMesh* Mesh = nullptr;
   UPROPERTY(EditAnywhere)
   UMaterial* Material = nullptr;
   UPROPERTY(EditAnywhere)
   USpaceshipWeapon* Weapons = nullptr;
   UPROPERTY(EditAnywhere)
   float MaxSpeed;
};
```

**BattleTeam.h**
```cpp
UCLASS()
class MAINGAMEPLAY_API UBattleTeam : public UDataAsset
{
	GENERATED_BODY()
public:
	UPROPERTY(EditAnywhere)
	USpaceshipType* SpaceshipType = nullptr;
	UPROPERTY(EditAnywhere)
	int32 NumShips;
};
```

Now it's a turn to create `GameplayConfig`:

**GameplayConfig.h**
```cpp
UCLASS()
class MAINGAMEPLAY_API UGameplayConfig : public UDataAsset
{
	GENERATED_BODY()
public:
	UPROPERTY(EditAnywhere)
	TArray<UBattleTeam*> Teams;
	UPROPERTY(EditAnywhere)
	FVector2D SpawnRange;
	UPROPERTY(EditAnywhere)
	float ShootingCellSize;
	
	UPROPERTY(EditAnywhere, Category=BoidSettings)
	float SeparationWeight;
	UPROPERTY(EditAnywhere, Category=BoidSettings)
	float CohesionWeight;
	UPROPERTY(EditAnywhere, Category=BoidSettings)
	float AlignmentWeight;
	UPROPERTY(EditAnywhere, Category=BoidSettings)
	float CageAvoidWeight;
	UPROPERTY(EditAnywhere, Category=BoidSettings)
	float CellSize;
	UPROPERTY(EditAnywhere, Category=BoidSettings)
	float CageSize;
	UPROPERTY(EditAnywhere, Category=BoidSettings)
	float CageAvoidDistance;
};
```

And that's how it's finally looking inside Editor:

![Screenshot](/assets/images/posts/part2_image2.png)

![Screenshot](/assets/images/posts/part2_image3.png)

![Screenshot](/assets/images/posts/part2_image4.png)

![Screenshot](/assets/images/posts/part2_image5.png)

### Spaceship Models

I used [Magica Voxel](https://ephtracy.github.io/) to prototype my ugly spaceship models, then [MeshLab](https://www.meshlab.net/) to smooth edges, and before the final exporting also [Blender](https://www.blender.org/) to fix pivots:

![Screenshot](/assets/images/posts/part2_image6.png)

![Screenshot](/assets/images/posts/part2_image7.png)

Specially for beams I've made a cube with the shifted pivot to scale this mesh only in one direction:

![Screenshot](/assets/images/posts/part2_image8.png)

### Game Authoring

Finally, we have data, logic and models. It's time to expand our `MainGameplayBootstrap`:

**MainGameplayBootstrap.h**
```cpp
//Add a interface to use UGameplayConfig inside our custom Flecs module
UINTERFACE(MinimalAPI)
class UGameplayConfigSet : public UInterface
{    
	GENERATED_BODY()
};
class MAINGAMEPLAY_API IGameplayConfigSet
{
	GENERATED_BODY()
public:
	virtual void SetConfig(UGameplayConfig* InConfig);

	UGameplayConfig* Config = nullptr;
};

UCLASS()
class MAINGAMEPLAY_API AMainGameplayBoostrap : public AFlecsBootstrap
{
	GENERATED_BODY()

public:
	//Add UGameplayConfig property
	UPROPERTY(EditAnywhere)
	UGameplayConfig* Config;
	UPROPERTY(EditAnywhere)
	TArray<TSubclassOf<UFlecsModuleBase>> FlecsModules;
protected:
	virtual void Bootstrap(flecs::world& ecs) override;
};
```

**MainGameplayBootstrap.cpp**
```cpp
void IGameplayConfigSet::SetConfig(UGameplayConfig* InConfig)
{
	Config = InConfig;
}

void AMainGameplayBoostrap::Bootstrap(flecs::world& ecs)
{
	for(auto moduleType : FlecsModules)
	{
		auto module = NewObject<UFlecsModuleBase>(this, moduleType);

		//If the current module implement UGameplayConfigSet, 
		//we'll set GameplayConfig to it
		auto bImplementConfigSet = module->Implements<UGameplayConfigSet>();
		if(bImplementConfigSet)
		{
			auto configSet = Cast<IGameplayConfigSet>(module);
			configSet->SetConfig(Config);
		}
	
		module->Initialize(ecs);
	}
}
```

**MainGameplay_Inits.h**
```cpp
//MainGameplay_Inits inherits from IGameplayConfigSet as well
UCLASS()
class MAINGAMEPLAY_API UMainGameplay_Inits : public UFlecsModuleBase, public IGameplayConfigSet
{
	GENERATED_BODY()

	virtual void Initialize(flecs::world& ecs) override;
};
```

Now we can add `GameplayConfig` right to the blueprinted `MainGameplayBootstrap` :

![Screenshot](/assets/images/posts/part2_image9.png)

And that's it. Afterward, we should be able to get up and run with Flecs creating prefabs and spaceship creation commands for both teams:

**MainGameplay_Inits.cpp**
```cpp
void UMainGameplay_Inits::Initialize(flecs::world& ecs)
{
	ISM_Map ismMap{TMap<uint32, AISMController*>()};

	for (auto team : Config->Teams)
	{
		auto teamEntity = ecs.entity();

		auto spaceshipHash = CreateISMController(World, team->SpaceshipType->Mesh, team->SpaceshipType->Material,
		                                         ismMap);
		auto weaponHash = CreateISMController(World, team->SpaceshipType->Weapons->Mesh,
		                                      team->SpaceshipType->Weapons->Material, ismMap);

		auto projectilePrefab = ecs.prefab()
		                           .set<ProjectileLifetime>({team->SpaceshipType->Weapons->Lifetime})
		                           .add_owned<ProjectileLifetime>()
		                           .add_owned<ProjectileInstance>();;

		if (team->SpaceshipType->Weapons->WeaponType != EWeaponType::Beam)
		{
			projectilePrefab.set<Speed>({team->SpaceshipType->Weapons->Speed});
		}

		auto spaceshipPrefab = ecs.prefab()
		                          .set<SpaceshipWeaponData>({
			                          projectilePrefab, weaponHash,
			                          team->SpaceshipType->Weapons->WeaponType == EWeaponType::Beam,
			                          team->SpaceshipType->Weapons->ProjectileScale,
			                          team->SpaceshipType->Weapons->BeamMeshLength
		                          })
		                          .set<SpaceshipWeaponCooldownTime>({team->SpaceshipType->Weapons->Cooldown, 0.f})
		                          .set<SpaceshipTarget>({flecs::entity::null()})
		                          .set<Speed>({team->SpaceshipType->MaxSpeed})
		                          .set<Transform>({FTransform(FVector::ZeroVector)})
		                          .set<BattleTeam>({teamEntity})
		                          .add_owned<SpaceshipWeaponCooldownTime>()
		                          .add_owned<SpaceshipTarget>()
		                          .add_owned<Transform>()
		                          .add_owned<BoidInstance>();

		ecs.entity().set<BatchInstanceAdding>(
			{team->NumShips, spaceshipHash, spaceshipPrefab});
	}

	ecs.entity("Game")
	   .set<UWorldRef>({World})
	   .set<BoidSettings>({
		   Config->SeparationWeight, Config->CohesionWeight, Config->AlignmentWeight, Config->CageAvoidWeight,
		   Config->CellSize, Config->CageSize, Config->CageAvoidDistance
	   })
	   .set<GameSettings>({Config->SpawnRange, Config->ShootingCellSize})
	   .set<ISM_Map>(ismMap)
	   .set<TargetHashMap>({TMap<FIntVector, TArray<Data_TargetInstance>>{}});
}
```

## Final Result

Look, we finally have thousands of ships and beams!

{% include youtube.html id='MYy-FICVKpU' %}




> All of the code for this two-part article can be found [on Github](https://github.com/jtferson/UnrealFlecsQuickstart)