---
layout: single
title:  "Creating UBT Module Fast"
categories: blog
tags: unreal
excerpt: "Useful templates to avoid manual boilerplate."
---

Since I love to separate different sets of functionality, using UBT modules for that purpose is an expected choice. Unfortunately, there is still no way to create UBT Modules in Unreal automatically, as opposed to creating Plugins. We should do it manually writing a boring boilerplate.

So, here is a list of the code templates that can be copy-pasted into the project. After that you need only replace `ModuleName` and `ProjectName` with the names of your module and project respectively.

First, create a Module folder inside the Source directory and add **`build.cs`** to that folder:

**ModuleName.build.cs**

```cpp
using UnrealBuildTool;

public class ModuleName : ModuleRules
{
	public ModuleName(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

		PublicDependencyModuleNames.AddRange(new string[] {
			"Core",
			"CoreUObject",
			"Engine",
			"ProjectName",
		});

		//The path for the header files
		PublicIncludePaths.AddRange(new string[] {"ModuleName/Public"});
		//The path for the source files
		PrivateIncludePaths.AddRange(new string[] {"ModuleName/Private"});
	}
}
```

Then create a subclass that is derived from `IModuleInterface`:

**ModuleNameModule.h**

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Modules/ModuleInterface.h"

DECLARE_LOG_CATEGORY_EXTERN(ModuleName, All, All);

class FModuleNameModule : public IModuleInterface
{
public:
 
	/* This will get called when the editor loads the module */
	virtual void StartupModule() override;
 
	/* This will get called when the editor unloads the module */
	virtual void ShutdownModule() override;
};
```

**ModuleNameModule.cpp**

```cpp
#include "ModuleName/Public/ModuleNameModule.h"
#include "Modules/ModuleManager.h"
#include "Modules/ModuleInterface.h"

IMPLEMENT_GAME_MODULE(FModuleNameModule, ModuleName);

DEFINE_LOG_CATEGORY(ModuleName);
 
#define LOCTEXT_NAMESPACE "ModuleName"
 
void FModuleNameModule::StartupModule()
{
	UE_LOG(ModuleName, Warning, TEXT("ModuleName module has started!"));
}
 
void FModuleNameModule::ShutdownModule()
{
	UE_LOG(ModuleName, Warning, TEXT("ModuleName module has shut down"));
}
 
#undef LOCTEXT_NAMESPACE
```

And the last thing you should do is adding your Module to Target.cs and the uproject file:

**ProjectName.Target.cs**

```cpp
ExtraModuleNames.AddRange( new string[] { "ProjectName", "ModuleName" } );
```

**ProjectNameEditor.Target.cs**

```cpp
ExtraModuleNames.AddRange( new string[] { "ProjectName", "ModuleName" });
```

**ProjectName.uproject**

```cpp
{
	"Modules": [
		{
			"Name": "ModuleName",
			"Type": "Runtime",
			"LoadingPhase": "Default"
		},
	]
}
```

After all of this you can finally recompile you project.