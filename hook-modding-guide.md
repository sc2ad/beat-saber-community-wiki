<!-- TITLE: Beginner Hook Modding Guide -->
<!-- SUBTITLE: For Oculus Quest using BeatOn -->

# Introduction

Hopefully you are here because you want to know how to make your own hook mod with emulamer's [BeatOn](https://github.com/emulamer/BeatOn).
While most of what I will cover (or hope to cover) should be applicable to generic hook modding (for any Unity game, really) this guide will be specific to BeatOn and Beat Saber.

**I will not teach you how to program, but rather illustrate specific examples of common desirables and how to acheive them. Please read the warning below!**

>WARNING! THIS IS NOT MEANT TO BE AN ENTIRELY BEGINNER FRIENDLY GUIDE! THIS GUIDE ASSUMES YOU HAVE SOME EXPERIENCE WITH MODDING, WHETHER IT BE THROUGH CODE OR NOT, AND IS NOT FOR THE FAINT OF HEART.
>THIS GUIDE ALSO ASSUMES THAT YOU HAVE MODERATE UNDERSTANDING OF PROGRAMMING IN GENERAL, C++, GITHUB, AND A DECENT UNDERSTANDING OF HOW SOME STUFF IN BEAT SABER IS COMPILED. I WILL NOT BE TEACHING YOU HOW TO CODE!
>YOU HAVE BEEN WARNED!
{.is-danger}

# Setup

Because you haven't left yet, I'm going to assume you are actually interested in making a hook mod. Here's a quick list of everything that you should have for this guide:

* The Beat Saber APK (from the Oculus Quest version). You can get this by doing `adb pull /data/app/com.beatgames.beatsaber-1/base.apk`

* IL2CPP Dumper, either [mine](https://github.com/sc2ad/Il2CppDumper/releases/) (which supports proper dumping of struct field offsets) or [Perfare's](https://github.com/Perfare/Il2CppDumper/releases) (which is the main repo).

* [NDK](https://developer.android.com/ndk/downloads) Which you will use for compiling your mod, download the latest release.

* [ADB](https://developer.android.com/studio/releases/platform-tools.html) Which you will use for viewing logs, among other various functions

* [dnSpy](https://github.com/0xd4d/dnSpy/releases) Which you will use for analyzing and decompiling the game code to more legible C#

* [Ghidra](https://ghidra-sre.org/) Which you will use for analyzing the game code

* (OPTIONAL BUT HIGHLY RECOMMENDED) The PC version of Beat Saber (either Steam or Oculus is fine) somewhere on your computer (specifically the DLLs located under the `BeatSaber_Data/Managed` folder). We want this because it will make reading through the assets' fields much easier.

* (OPTIONAL BUT HIGHLY RECOMMENDED) [This mod template](https://github.com/sc2ad/beat-saber-community-wiki/releases/download/v0.0.0/template.zip) This is used to help create your project, but is NOT a replacement, nor is it definitely a valid BeatOn mod.

# Begin

## What is a Hook Mod?

First, I want to start by clarifying what makes a hook mod different from an asset mod. Asset mods, well, mod assets, whereas hook mods are for modding pretty much everything else. While hook mods are more powerful than asset mods (you can do more with them) I strongly suggest that you check to make sure that what you are attempting to make cannot be done by asset modding before attempting to make it a hook mod. Things like transparent walls, song loading, and custom sabers (to name a few) are much, much easier as asset mods than as hook mods.

If this still sounds appealing to you, or you can't make your idea work without modifying things that happen dynamically, keep reading.

A hook mod is called such because it does something called "hooking", which is the process of taking an existing function and running custom code instead of that function whenever that function is called. This is extremely powerful (with the right skillset) because it means you can do pretty much anything code related, so long as you find the right function to hook into.

The downside of this, for now, is that you must program your hooks (and your entire mod, for that matter) in C/C++ (or with [leo60228's rust port](https://github.com/leo60228/rust-saber), which is missing many features), which are not exactly the easiest of languages to learn.

## Installation and Setup

### Ghidra

I put this step first because it will take quite awhile for this step to complete. What we want to do is open Ghidra and import the `libil2cpp.so` from our APK. This will allow us to take a closer look at functions, their parameters, and how they are called.

Open Ghidra. Once you do, you will probably see something like this screen:

![Ghidra Screen](/uploads/hook-modding/03_ghidra_start_screen.png "Ghidra Start Screen")

Click File --> New Project and then press I for Import File. Navigate to where your libil2cpp.so is, and it should now look something like this (I renamed my libil2cpp.so to include the version)

![Ghidra Project Screen](/uploads/hook-modding/04_ghidra_project_screen.png "Ghidra Project Screen")

Next, double click `libil2cpp.so` (or in my case, `libil2cpp_1.2.0.so`) It will first ask you what type of file it is. Press enter to just use the default. Next it will ask you to auto analyze. Say yes, and leave the settings as-is. This process will take around 2-3 hours or more (depending on how good your computer is), so feel free to follow the rest of the introduction and then take a walk, get a lunch break, etc. until it finishes.

### NDK

So, there is quite a few things that need to be done in order to properly get yourself ready for making building hook mods. Let me briefly explain what it is we are actually making. At a high level, we want to write code that we know will let us interact with the game in some way. Narrowing it down some, we want to create an executable application that can be loaded at the same time as beat saber itself, allowing us to hook into function calls from within beat saber. What executable type are we making and how are we doing it? [NDK](https://developer.android.com/ndk/downloads). This will let us build a .so file, which we can then load via jakibaki's modloader, which is what BeatOn uses behind the scenes.

So to actually be able to do what I just described, we need to download a few things and set up a few things. Firstly, download the latest release of [NDK](https://developer.android.com/ndk/downloads). At the time of me writing this guide, the latest version is r20. This guide will most likely hold true for many versions to come, and is known to work on NDK versions >= r12.

Next, place ndk in a safe place, and add it to your PATH environment variables. You can either do this via some command line args, or just do it through the user interface. Here I show myself doing it through the user interface:

![Environment Variables](/uploads/hook-modding/00_env_vars.png "Enviornment Variables UI")

Go ahead and click the environment variables option (you can get to the dialog shown in the screenshot by typing environment variables into your start menu), find your `Path` key, click edit, and add a new line to it which will point to your inner directory, which is the directory that contains the `CHANGELOG` file, among `ndk-build` and a few others.

To verify ndk set up properly, type `ndk-build` into your cmd. It should result in something like the following:

![ndk-build result](/uploads/hook-modding/02_ndk_result.png "ndk-build Result")

### Template Project

If you haven't already, I _strongly_ suggest downloading [this mod template](https://github.com/sc2ad/beat-saber-community-wiki/releases/download/v0.0.0/template.zip) which provides you with a few different files, which I will explain.

* `Android.mk`
* `Application.mk`
* `beatonmod.json`
* `main.cpp`
* `build.sh`
* `build.bat`

Now, let me explain what each of these files is used for.

The easiest to explain is the `main.cpp` file. This holds the code for mod, and is almost always the main thing you will be modifying when making changes to your mod.

Next, we have `beatonmod.json`, which is simply the .json that allows for this mod to be installed via BeatOn. You may familarize yourself with the format if you so desire, but it probably won't make _too_ much sense just yet. I will talk more about this fomat once we have completed our mod.

Moving on, we have our next two files: `build.sh` and `build.bat`. These are hopefully self-explanatory: they build the .so for you, so when you want to build your mod, you can simply `cd` to your mod directory and run `build`.

And now the final two files, which are perhaps the hardest to understand, so I will take a little time to explain what is going on in each.

`Android.mk` this file contains various configuration and flags we want to set when building with NDK for Android. `Application.mk` contains similar configurations, however, they contain more generic settings instead of specific flags and modules, like `Android.mk` contains.

Here is a dump of the template's `Android.mk` file:

```mk
LOCAL_PATH := $(call my-dir)

TARGET_ARCH_ABI := armeabi-v7a

include $(CLEAR_VARS)
LOCAL_LDLIBS := -llog
LOCAL_CFLAGS    := -DMOD_ID='"TemplateMod"' -DVERSION='"0.0.1"'
LOCAL_MODULE    := templatemod
LOCAL_CPPFLAGS := -std=c++2a
LOCAL_SRC_FILES := main.cpp ../beatsaber-hook/shared/utils/utils.cpp ../beatsaber-hook/shared/inline-hook/inlineHook.c ../beatsaber-hook/shared/inline-hook/relocate.c
include $(BUILD_SHARED_LIBRARY)
```

So, let's talk about each line. First off, `LOCAL_PATH`: pretty self-explanatory, no need to explain.

Next, `TARGET_ARCH_ABI`: this specifies which architecture ABI we want to target. Specifically for beat saber, we want to target the armeabi-v7a architecture, as opposed to any others (such as x86, for example) because this is what architecture the game is run in.

Next, we have an `include` call to `$(CLEAR_VARS)` which, you guessed it, clears the variables. This allows us to start our build by being clear.

Now we have a `LOCAL_LDLIBS` which is set to include `-llog`, which is actually a shortened form of `liblog.so` (dictated by the -l). So, we will load the `liblog.so` when building this next module.

And now we have a key known as `LOCAL_CFLAGS`, which is incredibly important. Without this line, you will get errors when attempting to build informing you that you have not defined `MOD_ID` and `VERSION`. Whenever you make a versioning change, it is suggested to change your version on both the `beatonmod.json` and the `VERSION` tag here. The `-D` indicates that something is being defined (in this case, `MOD_ID` and `VERSION` are both being defined and set to strings).

`LOCAL_MODULE` is fairly straightforward; it's just the name of the library you are building. Once built, NDK will automatically prefix your .so with `lib`, so in this case, after a successful build, the resultant .so would be: `libtemplatemod.so`

`LOCAL_CPPFLAGS` incidate what C++ flags to use. In this case, we are telling the C++ compiler to compile using C++20.

`LOCAL_SRC_FILES` is the 'meat' of the mod, which tells NDK which files to actually compile into a .so. You may notice some files that we haven't talked about yet, such as any of the `../beatsaber-hook` files. We will talk about these files (and how to obtain them) in the next step.

Lastly, we tell NDK to actually build the shared library.

And next, `Application.mk`:

```mk
APP_ABI := armeabi-v7a
APP_PIE:= true
APP_STL := c++_static
```

`APP_ABI` is the ABI we want our application to target. The same as specified in our `Android.mk` file.

`APP_PIE` is ...

`APP_STL` is the definition of what compiler/libraries we want to use. In this case, we use the C++ static libraries (static because we want the C++ code to be within each mod)

This was just a brief sharing of my extremely limited knowledge on NDK build files. You are welcome to read more about them on NDK's official documentation page: [Android.mk](https://developer.android.com/ndk/guides/android_mk), [Application.mk](https://developer.android.com/ndk/guides/application_mk)

### IL2CPP Dumper

Now, we need to download and run IL2CPP Dumper, which we will need in order to help us understand how each class/struct is defined, as well as where their methods are in memory (offsets). IL2CPP Dumper is a powerful tool that is _extremely_ useful, because it is able to decompile code from `libil2cpp.so` into C# header information (does not decompile the insides of each method).

We can use IL2CPP Dumper or Dumpity by simply opening the executable, opening our IL2CPP binary file (located at: `lib/armeabi-v7a/libil2cpp.so` in our extracted APK), our global-metadata.dat file (located at: `assets/bin/Data/Managed/Metadata/global-metadata.dat` in our extracted APK), and the Unity version (which is: `2018.3`). Once we enter all of this, IL2CPP will ask you to `Select Mode`. Press `3`, for Auto(Plus) and IL2CPP will run and say something similar to the following:

![IL2CPP Dumper Results](/uploads/asset-modding/05_il2cpp_results.png "IL2CPP Results")

Now, in the directory that you ran IL2CPP Dumper, there should be a folder called `DummyDll` and a file called `dump.cs`. Both of these will prove to be useful. Open up `dump.cs` in a text editor of your choice and leave it open, it's a large file, but it contains a large portion of the header information that you might want.

### DnSpy

Next, make sure you have dnSpy downloaded and open up all of the assemblies in the `DummyDll` folder from IL2CPP Dumper. Also, if you have access to the PC DLLs, open those too. You can find the PC DLLs under `BeatSaber_Data/Managed` under your main Beat Saber directory.

The reason we want both `DummyDll` and `dump.cs` open is because `dump.cs` does not properly write out each attribute, whereas `DummyDll` does.

### Utils

Next, let's go ahead and set up our utilities for actually hooking and whatnot. To do this, I _strongly_ recommend setting up a git repo where your template `Android.mk`, `Application.mk`, `main.cpp` and build scripts are within a folder on the root of the repo (call it, say, `templatemod`).

Next, you want to add a submodule with: `https://github.com/sc2ad/beatsaber-hook.git` and [initialize it recursively](https://www.vogella.com/tutorials/GitSubmodules/article.html#submodules_trackbranch) (`git submodule update --init --recursive`). If you aren't using git for this, you can simply download a .zip and extract it from [this link](https://github.com/sc2ad/beatsaber-hook) and place it in your root project directory.

### IL2CPP Headers



### Syntax Highlighting (Visual Studio Code)

Because I like using [Visual Studio Code](https://code.visualstudio.com/), I will be covering how to set up syntax highlighting and include paths in order to make sure you can spot some compile errors before you attempt to build.

Inside VSC, press ctrl-shift-p to bring up the command window and find C/C++ edit configurations (JSON). Make sure you have the C/C++ extension installed. This will open a JSON window. You will want to add your NDK to the `includePath` directory, `path/to/my/ndk-bundle/**`. Next, change the `cppStandard` value to `"c++17"`, or if it doesn't exist, make it and set it to that. For `intelliSenseMode`, change it to `"gcc-x64"`. This will hopefully show you compile errors now.

You can double check by opening `main.cpp` which should have no compile errors, except for `log`. To resolve the `log` compile error, you can add to definitions _before_ including `../beatsaber-hook/shared/utils/utils.h`:

```cpp
#define MOD_ID "TemplateMod"
#define VERSION "1.0.0"
```

by now, you should be able to build your template mod without any issue!

## Mod Introduction

Because you weren't scared off by me saying **The mods need to be programmed in C/C++** then clearly you are insane enough to complete this guide. Let's do something simple: Custom Colors (because why not?)
Here's what we want it to look like:

![Modding Goal](/uploads/asset-modding/00_custom_colors_goal.png "Custom Colors Goal")

Let's break this task down into steps:

1. Look at the game code in order to determine how the colors of each block are determined
2. Find function(s) that we would want to hook into
3. Hook into the functions we found and make them instead show our colors!

### Step 1, 2 - Look at the game code in order to determine how the colors of each block are determined

As always, there is an _invisible_ step 0, which is **making sure you backup your APK!** Please do this by copying it somewhere and making sure you won't delete it on accident or in a fit of range if your mod isn't working.

Now to get the game code, hopefully you already followed the setup procedure (if not, go do it now). Go ahead and take a look at your `dump.cs` and `DummyDlls`. What we want to find is something pertaining to `color`. We can do a search in either `dump.cs` or in `DummyDlls` using dnSpy, but the best and easiest way is to use the PC DLLs. Sadly, for this guide, not having the PC DLLs will make your life _much_ more difficult. The best option for you would be to extensively research all of the search and find matches in `dump.cs` but it will take awhile, and will still be mostly guesswork based off of class names.

Because I am trying to change colors, I can do a search in dnSpy for `Color`. However, there are far too many results that involve Color. Instead, I'll search only inside `Assembly-CSharp.dll` for Color, which is where most of the code that is Beat Saber specific is stored, and I will only do a search for classes, since a Color object is what we are looking for. I can do this by selecting the `Assembly-CSharp.dll` file and then changing my search. I can see the following results:

![dnSpy Search Results](/uploads/asset-modding/06_dnspy_search_results.png "dnSpy Search Results")

Note: If I didn't find anything useful with my class search, I would have tried a struct search, and would gradually widen my search to different areas in order to find anything that might be useful.

However, it looks like I found something useful right away! `ColorManager` seems very promising. What else could `ColorManager` mean _besides_ being a manager for colors? Seems perfect to me. Let's take a closer look at this class:

![dnSpy ColorManager Properties](/uploads/hook-modding/05_dnspy_colormanager_props.png "dnSpy ColorManager Properties")

It looks like there are two properties, which are of type `Color`, that seem useful to us. In order to take a look into this in more detail, let me explain a little about how properties work when compiled via IL2CPP:

Put simply, properties end up making a function for their `get` and `set` calls. These functions are universally similar: All instance getters take in the instance of the object and return the property, and all instance setters take in the instance of the object, as well as a value of the same type of the property in order to set, and return void.

What this means is that we have the ability to hook into property getters and setters, just like any other function call. This is extremely powerful, because we can essentially forcefully return whatever values we want when these values our gotten. Take our example, every time `colorA` or `colorB`'s get methods are called, we can override what the normal method does and instead return our own color!

### Step 3 - Hook into the functions we found and make them instead show our colors

#### Hook Creation Introduction

Now that we have found some functions that we are interested in (specifically, the `ColorManager.colorA` and `ColorManager.colorB` getters) we can start to create our hooks. There are several things to be wary of when creating hooks:

##### Parameters

This can sometimes be the hardest to diagnose, because it can be very hard to remember to include all the parameters you need. Specifically, for all instance methods, such as this one:

```csharp
[Address(RVA = "0x130C5A4", Offset = "0x130C5A4")]
public void RefreshColors()
{
}
```

There is actually _one_ parameter, which is a pointer to the instance of the class, or the object.

##### Understanding the difference between value types and reference types

Value types in C# represent types that are passed literally, stored literally, and returned literally, whereas reference types are stored via pointer. What that means is that in a function, for example, a value type will be passed as-is, whereas a reference type will be passed via pointer.

Take our `ColorManager.get_colorA()` example. Because all C# structs are value types, the return of this function will not be a pointer to an object, but instead a struct representing the color. What does this struct look like? Well, you can find it in dnSpy by clicking on the `Color` type, but I'll save you that trouble by telling you that it is 4 floats, each for R, G, B, and A values. So, the return of this function call is actually 4 floats, or 16 bytes. Likewise, in any case where a struct is passed into a function, it is again treated literally. As well as when structs are defined in fields, they are stored literally.

Value types include the primitives (such as `int` and `double`), enums (which are actually just ints), and any C# structs.

Reference types are anything that don't fall into the above, such as strings, or instances of classes. They are treated and stored as pointers. For example, for a function that returns an instance of a class, it simply returns a pointer to the class, which is 4 bytes long (or a standard unsigned integer). And in cases where reference types are defined in fields, or passed as parameters, they are always 4 bytes long, because they are always pointers.

An important thing to note is that when hooking functions, it isn't important that the types of the parameters are the same (in fact, it doesn't matter _at all_) instead it matters that the _size_ of each parameter is the same.

#### Hook Creation

Now that you hopefully have a better understanding, this code should make _some_ sense, but I'll walk you through it anyways. This is from the template.

```cpp
#define StretchableCube_Awake_offset 0x12F05D4

MAKE_HOOK(StretchableCube_Awake, StretchableCube_Awake_offset, void, void* self) {
    log("Called StretchableCube.Awake!");
    StretchableCube_Awake(self);
}
```

First, we define an offset which we will need for making our hook. This is good practice, because when a new version is released, offsets will most likely be outdated and require changing. Having them at the top, or somewhere consistent will help you change these and avoid headaches.

Next, we call the MAKE_HOOK macro, which requires us to speccify a name of the function we are hooking (can be any name we desire), the offset of the function, the return value (can be void), and the arguments (or none if there are no arguments)

So, what we have done above is create a hook called `StretchableCube_Awake` at the offset `0x12F05D4`, with a `void` return type, and one parameter `void* self`. Recall that the thing that matters when making hooks with parameters is that the _size_ of the parameters must match the original function. In this case, we are simply saying: "I want to hook a function at offset 0x12F05D4, which has a return type of void, and a single parameter which is 4 bytes long. I want to call this original function StretcableCube_Awake".

Inside of the hook, we can see that we log before calling `StretchableCube_Awake` and then we call the original method. When running this mod in game, every time a wall (which uses StretchableCube) is 'awoken' we will see some output in our log, which we can view by doing: `adb shell "logcat |grep 'QuestHook'"`

Now, let's write something similar that will hook into the `get_colorA` of ColorManager, as well as the `get_colorB` of ColorManager:

```cpp
#include <android/log.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <dirent.h>
#include <linux/limits.h>
#include <sys/sendfile.h>
#include <sys/stat.h>

#define MOD_ID "CustomColorsMod"
#define VERSION "0.0.1"

#include "../beatsaber-hook/shared/inline-hook/inlineHook.h"
#include "../beatsaber-hook/shared/utils/utils.h"

using namespace std;

#define ColorManager_get_colorA_offset 0x130C350
#define ColorManager_get_colorB_offset 0x130C4A8

MAKE_HOOK(ColorManager_get_colorA, ColorManager_get_colorA_offset, Color, void* self) {
    log("Called ColorManager.get_colorA!);
    return ColorManager_get_colorA(self);
}
MAKE_HOOK(ColorManager_get_colorB, ColorManager_get_colorB_offset, Color, void* self) {
    log("Called ColorManager.get_colorB!);
    return ColorManager_get_colorB(self);
}
```

Color is an already defined struct in `../beatsaber-hook/shared/utils/typedefs.h`. Normally, you will have to create the structs yourself, but for some common structs, such as `Color`, `Vector3`, and `Vector2`, they are already defined. You are welcome to look at the `typedefs.h` file to see all of the defined structures, as this guide may not be always up to date.

Next, let's actually _install_ these hooks. Just because we have defined them doesn't mean we have installed them. In fact, we want to install our hooks only once we know that libil2cpp.so has actually been opened. Fortunately, this is as simple as putting some `INSTALL_HOOK` calls in the `__attribute__((constructor)) void lib_main()` call. Go ahead and add this to our main.cpp file:

```cpp
__attribute__((constructor)) void lib_main()
{
    log("Installing CustomColors hooks...");
    log("Installing ColorManager.get_colorA hook!");
    INSTALL_HOOK(ColorManager_get_colorA);
    log("Installing ColorManager.get_colorB hook!);
    INSTALL_HOOK(ColorManager_get_colorB);
    log("Completed installing hooks!");
}
```

Now, we have code that will create a mod that will log whenever we call our colors!


If you liked what you read, please consider supporting me on [PayPal](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=XQP4GZDKK69DQ&currency_code=USD&source=url), or on [Ko-Fi](https://ko-fi.com/P5P7ZAA9) in order for me to purchase a Quest.
Hopefully you learned something today, feel free to DM me with any questions or concerns. I'll hopefully add an FAQ here later.

Thanks for reading!
Written by [Sc2ad](github.com/sc2ad), or on Discord: Sc2ad#8836
