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

Next, you want to add a submodule with: `https://github.com/sc2ad/beatsaber-hook.git` and initialize it recursively. If you aren't using git for this, you can simply download a .zip and extract it from [this link](https://github.com/sc2ad/beatsaber-hook) and place it in your root project directory.

### IL2CPP Headers



### Syntax Highlighting (Visual Studio Code)

Because I like using Visual Studio Code, I will be covering how to set up syntax highlighting and include paths in order to make sure you can spot some compile errors before you attempt to build.

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
    log(DEBUG, "Called StretchableCube.Awake!");
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
    log(DEBUG, "Called ColorManager.get_colorA!);
    return ColorManager_get_colorA(self);
}
MAKE_HOOK(ColorManager_get_colorB, ColorManager_get_colorB_offset, Color, void* self) {
    log(DEBUG, "Called ColorManager.get_colorB!);
    return ColorManager_get_colorB(self);
}
```

Color is an already defined struct in `../beatsaber-hook/shared/utils/typedefs.h`. Normally, you will have to create the structs yourself, but for some common structs, such as `Color`, `Vector3`, and `Vector2`, they are already defined. You are welcome to look at the `typedefs.h` file to see all of the defined structures, as this guide may not be always up to date.

Next, let's actually _install_ these hooks. Just because we have defined them doesn't mean we have installed them. In fact, we want to install our hooks only once we know that libil2cpp.so has actually been opened. Fortunately, this is as simple as putting some `INSTALL_HOOK` calls in the `__attribute__((constructor)) void lib_main()` call. Go ahead and add this to our main.cpp file:

```cpp
__attribute__((constructor)) void lib_main()
{
    log(INFO, "Installing CustomColors hooks...");
    log(DEBUG, "Installing ColorManager.get_colorA hook!");
    INSTALL_HOOK(ColorManager_get_colorA);
    log(DEBUG, "Installing ColorManager.get_colorB hook!);
    INSTALL_HOOK(ColorManager_get_colorB);
    log(INFO, "Completed installing hooks!");
}
```

Now, we have code that will create a mod that will log whenever we call our colors!




First, I want to start by clarifying what an asset mod _is_ and what makes it different from a hook mod.
An asset mod is called an asset mod because, well, it modifies the assets (who would have guessed?)
However, what this means is that anything dynamic (where it changes during the game) _cannot be modified by an asset mod_. For example, Sc2ad's HitScoreVisualizer is only possible as a hook mod, not as an asset mod. This is because it is _impossible_ to change anything in the assets that lets us change the color of the text that is displayed when we cut a block, because it is simply not loaded from the assets.
When in doubt, if it ever changes in the game, it probably isn't asset moddable. However, if it _doesn't_ change in the game, it likely _is_ asset moddable. Hopefully that makes sense.

## Mod Introduction

Now that I have hopefully cleared up some possible misconceptions, let's begin!
Let's begin! My goal in this guide will be to create a small asset mod that allows me to set my saber colors to specific colors. Think: Custom Colors, but without changing the colors in the background.
Our goal will hopefully look something like this:

![Modding Goal](/uploads/asset-modding/00_custom_colors_goal.png "Custom Colors Goal")

So... how do we even start?

Well, let's break this task down into steps:

1. Get the assets files from the APK

2. Find which assets we want to change

3. Create our changed assets

4. Package our changed assets and create a mod for us to load with BeatOn

Let's start with the 0th step: backups. Because of how annoying it is to pull the APK (and because it is good practice) we should create a backup.
We can do this simply by copying the APK and renaming it from `base.apk` to something like `base_bkp.apk` or anything you want, really. The important thing here is to make sure you **NEVER TOUCH YOUR BACKUP!** Which should go without saying.

## Step 1 - Getting the assets from the APK

The first step is easy! All we have to do is unzip the APK and we have access to our assets!
We can do this by renaming our `base.apk` to `base.zip` and then extracting it like a normal zip file.
This is because APKs are actually just zip files, albeit specifically aligned and signed, but this allows us to get access to the assets!
After extracting, your base folder will probably look like this:

![Unzipped Assets](/uploads/asset-modding/01_unzipped_assets.png "Unzipped APK")

There are a few folders in here, but the main one we are interested in (surprisingly enough) is the `assets` folder.
The assets that we are looking for are located at: `assets/bin/Data`:

![Assets](/uploads/asset-modding/02_assets_folder.png "Assets Folder")

Here we can see quite a few files, some with extensions, some without. That's okay, we don't really mind what they are or what they are named.

**We have completed step 1! Now, let's start the next step: Finding the asset we want to change.**

## Step 2 - Finding which assets to change

This (along with step 3) are the hardest steps because they involve a lot of digging through the assets and moderate understanding of how the game functions.

First up, open up UABE. You can do this by running the AssetBundleExtractor executable located in the release, under the `32bit` folder that you hopefully downloaded at the beginning of this guide.

It will tell you that you haven't opened any files yet. Let's do so.
Click the File --> Open option, navigate to your extracted assets, and select all of them. Yes, ALL OF THEM. It should look something like this:

![Select All](/uploads/asset-modding/03_select_all.png "Select All Assets")

After that, go ahead and open all of the files and let UABE process all 22000 or so assets. Yes, there really is that many. But don't worry! We can find the ones we want without looking through all of these.

After that, a window should open looking similar to this:

![UABE Start](/uploads/asset-modding/04_uabe_start.png "UABE Start")

How are we supposed to read this???
Don't worry, there is a method to understanding this madness (somewhat).
Clicking on the top of any category (Name, Container, Type, etc.) will sort by that category. Clicking again will sort reverse of that category. You can sort by up to two categories at once by clicking two separate categories.
UABE originally sorts by two categories: FileID (from least to greatest) and PathID (from least to greatest).

Now that we understand a little bit about how UABE works, what are we even looking at?
UABE allows us to look at the assets of any Unity game, but in this case we are looking at Beat Saber's assets, which contains anything and everything that might be stored. A few examples include the main menu, text that is displayed, songs (in fact, this is how custom songs are loaded!), custom sabers, and much more. Pretty much anything that remains the same between reboots of the game must be loaded from assets.
An exception to this is local scores, or other stats, which live on the `/sdcard` directory.
So, now we need to find our colors to change. This can be a difficult task, especially if you don't know how it might be stored.

### If you have access to the PC DLLs

Open the PC DLLs in dnSpy and skip to the next section.

### If you do not have access to the PC DLLs

If you _do not_ have access to the PC DLLs, it becomes a little bit more difficult to find what you are looking for, because you can't reproduce any of the code function calls. However, you _can_ reproduce the fields using a tool called IL2CPP Dumper, or you can use my tool, Dumpity.
Because Beat Saber was made in Unity and ported to Android via IL2CPP (a process that converts C# intermediate language calls (IL) into C++ (CPP), hence IL2CPP) we can use these tools in order to convert from our compiled C++ code into C# code we can look at. However, there are no bodies of methods, so the only thing we will use this for is for finding classes and fields that have the `SerializeField` attribute.
We can use IL2CPP Dumper or Dumpity by simply opening the executable, opening our IL2CPP binary file (located at: `lib/armeabi-v7a/libil2cpp.so` in our extracted APK), our global-metadata.dat file (located at: `assets/bin/Data/Managed/Metadata/global-metadata.dat` in our extracted APK), and the Unity version (which is: `2018.3`). Once we enter all of this, IL2CPP will ask you to `Select Mode`. Press `3`, for Auto(Plus) and IL2CPP will run and say something similar to the following:

![IL2CPP Dumper Results](/uploads/asset-modding/05_il2cpp_results.png "IL2CPP Results")

Then, you can open dnSpy and look at the `DummyDll` directory created in the same location as the IL2CPP executable. You can now open these DLLs with dnSpy and continue to the next section.

### Finding the Asset(s) from dnSpy

This is where the PC DLLs come in handy. What I like to do is look for relevant code in dnSpy from the PC DLLs and look for any objects that have fields with a `SerializeField` attribute.
Because I am trying to change colors, I can do a search in dnSpy for `Color`. However, there are far too many results that involve Color. Instead, I'll search only inside `Assembly-CSharp.dll` for Color, which is where most of the code that is Beat Saber specific is stored, and I will only do a search for classes, since a Color object is what we are looking for. I can do this by selecting the `Assembly-CSharp.dll` file and then changing my search. I can see the following results:

![dnSpy Search Results](/uploads/asset-modding/06_dnspy_search_results.png "dnSpy Search Results")

Note: If I didn't find anything useful with my class search, I would have tried a struct search, and would gradually widen my search to different areas in order to find anything that might be useful.
However, it looks like I found something useful right away! `ColorManager` seems very promising. What else could `ColorManager` mean _besides_ being a manager for colors? Seems perfect to me. Let's check to see if it has any Serializable field attributes:

![dnSpy ColorManager Fields](/uploads/asset-modding/07_dnspy_colormanager_fields.png "dnSpy ColorManager Fields")

Hooray! Looks like it does. As a reminder, anything that has a `SerializableField` attribute is likely stored in assets somewhere, so we can be fairly confident that there is at least one object with type: `ColorManager` somewhere in all of the assets.

Let's go ahead and look for that type in UABE! Sadly, we don't know if the ColorManager has a name or not, so for now, let's simply sort by Type and see what we get.

![UABE ColorManager DNE](/uploads/asset-modding/08_uabe_colormanager_dne.png "UABE ColorManager Does Not Exist?")

Wait a minute... There's nothing there! That's because ColorManager is actually a `MonoBehaviour`. What that means is that it is a script that was written instead of a raw, unity asset. UABE does something special with all MonoBehaviours, and prefaces their type with `MonoBehaviour :` which essentially means that we should look for `MonoBehaviour : ColorManager` instead.

![UABE ColorManager Exists](/uploads/asset-modding/09_uabe_colormanager_found.png "UABE ColorManager Found!")

There it is! Now that we have found the asset we are looking for, what do we do with it?

Firstly, we know it has a name. This is helpful to us because it means if we lose it we can find it again later. Specifically, its name is `MonoBehaviour ColorManager`. Secondly, we know there is _exactly one_ color manager. The reason we can be certain of this is because there are no other `MonoBehaviour : ColorManager` types.
From our dnspy exploring, we know that there are three fields in the ColorManager asset object (we know this because the asset object will contain all serialized fields):

* A `PlayerDataModelSO` called `_playerModel`

* A `SimpleColorSO` called `_colorA`

* A `SimpleColorSO` called `_colorB`

So we have two fields that seem interesting and one that seems less so; `_colorA` and `_colorB` are the interesting ones.
Let's do another type search in UABE, this time for types that match `MonoBehaviour : SimpleColorSO`. How did I know that `SimpleColorSO` was a `MonoBehaviour`? I looked in dnSpy. You can follow the hierarchy tree until you reach either `ScriptableObject` or `MonoBehaviour` which indicates that it is a `MonoBehaviour`, or anything else, which indicates that it is a Unity specific asset.
Here's what my type search for `MonoBehaviour : SimpleColorSO` revealed:

![UABE SimpleColorSO Search](/uploads/asset-modding/10_uabe_simplecolorso_search.png "UABE SimpleColorSO Search Results")

Clearly there are several Colors. So the question is, which are the ones we want to modify? Do we want to modify all of them? Some of them? Which color corresponds to left vs. right?
Let's first return back to our ColorManager object that we found. We know that there are two fields of this object that must exist. So which two of the 6 `SimpleColorSO`s are they?
And this is, again, where the PC DLLs help.

### If you have access to the PC DLLs

Simply select the ColorManager object and click `View Data`. This will prompt you to add additional MonoBehaviour information, which you should say yes to.
Then, navigate to your PC DLL directory, and for each DLL, either provide it if you have it, or cancel to continue to the next DLL if you do not. It's okay if you don't have every DLL it asks for. This process may take a while, as it will ask for roughly 100 DLLs, many of which you probably will not have.
By the way, closing UABE will make you have to reenter all of those DLLs, so keeping it open will save you a ton of time!
Once providing the necessary DLLs, the asset window will open, which will appear something like the following:

![UABE ColorManager](/uploads/asset-modding/11_uabe_colormanager.png "UABE ColorManager")

Expanding the + arrow will show you something similar to the following:

![UABE ColorManager Expanded](/uploads/asset-modding/12_uabe_colormanager_expanded.png "UABE ColorManager Expanded")

Before we take a look at what we can see, let's talk about this magical thing that is a `PPtr`!
Put simply: It's a pointer to another asset.
Put longly: It's pairing of an unsigned integer representing the file location (aka FileID), and an unsigned long representing the PathID of another asset.
A PPtr also will provide the type of which it points to, if UABE knows it, which is provided inside the <>. A $ at the front of the type indicates that the type is a MonoBehaviour, and not a Unity specific object.

Now let's take a look:

* `PPtr<GameObject> m_GameObject`: The GameObject referenced by this MonoBehaviour. In cases where there is no particular GameObject, it will reference the GameObject at `(FID: 0, PID: 0)`.

* `UInt8 m_Enabled`: A boolean (unsigned 8 bit integer) representing whether this MonoBehaviour is enabled or not.

* `PPtr<MonoScript> m_Script`: The `MonoScript` that this MonoBehaviour _is_. MonoScripts are Unity specific, that contain a little bit of information about each script. To learn more, follow this pointer and take a look at the fields of the MonoScript.

* `string m_Name`: The name of the MonoBehaviour.

* `PPtr<$PlayerDataModelSO> _playerModel`: A field that we saw in dnSpy

* `PPtr<$SimpleColorSO> _colorA`: A field that we saw in dnSpy

* `PPtr<$SimpleColorSO> _colorA`: A field that we saw in dnSpy

Wait... Why are there more fields than the ones we saw in dnSpy?
That's because Unity decided to add them. Secretly, Unity holds these 4 variables for all MonoBehaviours.
We can actually access the enabled, name, and GameObject of the MonoBehaviour by simply looking up the inheritance tree of our MonoBehaviour, but the Script object is specifically added in by Unity in order to help it understand which MonoBehaviour is which.

Now that we understand what we are looking at, we know that `_colorA` leads us to: `(FID: 0 (121), PID: 60)` which means that the `SimpleColorSO` referenced by the ColorManager is in the same assets file as the ColorManager, which has ID 121, with a PathID of 60.
We can now either do view --> go to asset in UABE, or we can simply click the `[view asset]` option. By viewing the asset, we can see the following:

![UABE ColorManager ColorA](/uploads/asset-modding/13_uabe_colormanager_colora.png "UABE ColorManager ColorA")

We can see that it is called BaseNoteColor1, and it also has a field of type `ColorRGBA` with name `_color`. If we expand `_color`, we can see the RGBA values that make up a Unity color:

![UABE ColorManager ColorA RGBA](/uploads/asset-modding/14_uabe_colormanager_colora_rgba.png "UABE ColorManager ColorA RGBA")

Likewise, you can do the same for `_colorB` and see that it leads us to `(FID: 0 (121), PID: 59)`.
Looks like we have found the assets we want to modify! Let's go on to the next step!

### If you do not have access to the PC DLLs

Oh boy. I hope you have a hex editor installed, because things are about to get super messy.
Because you don't have access to the DLLs, _and_ we are attempting to mod a MonoBehaviour (which only exists in the DLLs, it's not Unity generic) we will need to resort to some rather complicated stuff.

Make sure you have the ColorManager selected in UABE and click `Export Raw`. Save this to a location that you won't forget immediately. You can choose to rename it or simply leave it as its default name, but _don't lose it!_
Now, open up your hex editor and take a look. It'll probably look something similar to this: (I use the Hexdump extension in Visual Studio Code)

![ColorManager HexDump](/uploads/asset-modding/15_colormanager_hexdump.png "ColorManager Hexdump")

So, what are we looking at here?
Unity stores MonoBehaviours in a slightly special way: They have 4 initial fields:

* `PPtr<GameObject> m_GameObject`: The GameObject referenced by this MonoBehaviour. In cases where there is no particular GameObject, it will reference the GameObject at `(FID: 0, PID: 0)`.

* `UInt8 m_Enabled`: A boolean (unsigned 8 bit integer) representing whether this MonoBehaviour is enabled or not.

* `PPtr<MonoScript> m_Script`: The `MonoScript` that this MonoBehaviour _is_. MonoScripts are Unity specific, that contain a little bit of information about each script. To learn more, follow this pointer and take a look at the fields of the MonoScript.

* `string m_Name`: The name of the MonoBehaviour.

Let's talk about this magical thing that is a `PPtr`!
Put simply: It's a pointer to another asset.
Put longly: It's pairing of an unsigned integer representing the file location (aka FileID), and an unsigned long representing the PathID of another asset.
A PPtr also will provide the type of which it points to, if UABE knows it, which is provided inside the <>. A $ at the front of the type indicates that the type is a MonoBehaviour, and not a Unity specific object.
Another thing to note is that a PPtr is exactly 12 bytes long (the first 4 bytes represent the FileID, the last 8 bytes represent the PathID).

A string is not null terminated like in C, instead strings are stored in the assets with their length first, followed by a single byte (UTF8) for each of the characters in the string.
So, let's dissect our .dat file:
The first 12 bytes are taken up by the `m_GameObject` PPtr, and are selected:

![ColorManager Dump GameObject](/uploads/asset-modding/16_colormanager_dump_gameobject.png "ColorManager Hexdump of GameObject PPtr")

The next byte is taken up by the `m_Enabled` UInt8 (or boolean), which is a 0. However, Unity stores all objects with an alignment of 4. What this means is that for any operation that results in the end position not being a multiple of 4 (end position mod 4 != 0), then 0s are added until the position is a multiple of 4.
What this means for the UInt8 is that instead of it taking up 1 byte, as it normally does, it takes up 1 byte + 3 additional 0 bytes. So, the data that represents the `m_Enabled` boolean is actually as follows:

![ColorManager Dump Enabled](/uploads/asset-modding/17_colormanager_dump_enabled.png "ColorManager Hexdump of Enabled bool")

The next 12 bytes represent a PPtr to the `m_Script` field, which is the following:

![ColorManager Dump Script](/uploads/asset-modding/18_colormanager_dump_script.png "ColorManager Hexdump of Script PPtr")

The `m_Script` PPtr is a pointer to: `(FID: 1, PID: 300)` which, when checked in UABE, correctly matches the MonoScript with name: `ColorManager`

The next ? bytes are represented by the string for `m_Name`. I left the number of bytes as a ? because it can vary depending on the length of the string. Remember that to read a string, we first read the first 4 bytes, which represents the length of the string. We then read that many bytes as individual characters that make up the string.
The first 4 bytes represent a length of 12, which means that the entire string takes up 16 bytes:

![ColorManager Dump Name](/uploads/asset-modding/19_colormanager_dump_name.png "ColorManager Hexdump of Name string")

**NOTE: If the string was not exactly a multiple of 4 number of bytes, we would need to align ourselves by adding 0s to the end. For example, in a case where the string is 11 characters long, the string still takes up 16 bytes, however the string itself is only 11 characters long, with an extra 0 byte after it to align to a multiple of 4.**

Now that we have finished the 4 fields that all MonoBehaviours have, what is the rest of the binary data?
Because Unity is, well, Unity, it stores any and all Serializable fields that aren't structs as PPtrs. What this means is that in a MonoBehaviour that contains Serializable fields that are of types that aren't structs (such as our ColorManager, with fields like `_playerModel`, `_colorA`, and `_colorB`) they are stored as PPtrs.
So we know that the remaining 36 bytes are all PPtrs. However, which one is which?
Fortunately, this is as easy as reading from the top, down in dnSpy. This is because Unity uses reflection to know how and what to store in the assets files, so because `_playerModel` comes before `_colorA` which comes before `_colorB`, that is exactly the order in which they are stored.
So, we know that `_colorA` is a PPtr to: `(FID: 0, PID: 60)` and `_colorB` is a PPtr to: `(FID: 0, PID: 59)`
Looks like we have found the assets we want to modify! Let's go on to the next step!

## Step 3 - Creating our changed assets

Now that we know where the assets are that we want to change, we can edit them!
Once again, this is much easier to do if you have access to the PC DLLs.

### If you have access to the PC DLLs

Let's navigate to where `_colorA` and `_colorB` are located. First, let's find out what the file name of our FileID is. Because we know that our PPtrs both share `FID: 121`, we need to figure out what the filename of `FID: 121` is. We can do this by clicking View --> Dependencies and searching for the prefix 121. In Beat Saber 1.1.0, `FID: 121` corresponds to the filename: `sharedassets1.assets`.
Now we can find our assets in UABE. We can do this by following the PPtrs we already found, and by clicking View --> Go to asset. Search for `sharedassets1.assets` and enter the PathID of either color PPtr to find them.
Now that we can see our two colors, we can modify them. First, let's click the `Export Dump` option for each color and dump the colors as UABE text dumps, with whatever names you choose.
Now open these files in whatever text editor you prefer, and modify the RGBA values for the color. Here's my modified `_colorA`:

![UABE Modified ColorA](/uploads/asset-modding/20_uabe_colora_modified.png "UABE Modified ColorA")

After modifying either or both colors, go back to UABE and select the import dump option. **Make sure you select the right object for the file you are importing!**
After importing, you should see a `*` show up in the Modified category of UABE, if there is no `*`, you didn't modify the asset. Make sure you select the asset when you import the dump.
After this, click export raw for both colors, and save them as `ColorA.dat` and `ColorB.dat` specifically.

You are now ready for the next and final step!

### If you do not have access to the PC DLLs

Once again, be prepared for a world of pain.
Because you know where the assets are from the previous step, you need to find the filename the colors are saved in. Because we know the `FID: 0` for both PPtrs from our exploration of the .dat file of our ColorManager, we know that the colors we are looking for are in the same file as the ColorManager.
So, if we look in UABE for the ColorManager, we can see that its FID is 121. Now we can get the filename from this by clicking View --> Dependencies and searching for the prefix 121. In Beat Saber 1.1.0, `FID: 121` corresponds to the filename: `sharedassets1.assets`.
Now we can find our color assets in UABE. We can do this by following the PPtrs we already found, and by clicking View --> Go to asset. Search for `sharedassets1.assets` and enter the PathID of either color PPtr to find them.
Now we need to edit these colors. We can do so by exporting each color as a raw. Call these .dat files `ColorA.dat` and `ColorB.dat` respectively. Because each "color" is actually a `SimpleColorSO`, it is a MonoBehaviour. We can also look in dnSpy to see that `SimpleColorSO` contains one field: A struct for color. This struct contains 4 floats (singles), R, G, B, and A. Taking into account the header bytes because of the fact that this asset is a MonoBehaviour, I have the following RGBA for my `_colorB`:

![Dump ColorB](/uploads/asset-modding/21_dump_colorb.png "Dump of ColoB")

The color struct has the following values:
R: 0.188235
G: 0.619608
B: 1.0
A: 1.0

I can now modify my .dat file for either color in order to contain different R, G, B, or A values. In this case, I decided to change my G to 1.0, which results in the following `ColorB.dat` file:

![Modified ColorB](/uploads/asset-modding/22_dump_colorb_modified.png "Dump of modified ColorB")

After modifying either or both of your colors, you are ready for the next and final step!

## Step 4 - Package our changed assets into a BeatOn mod

We are finally here! All that's left is to package our modified assets into a BeatOn mod and zip it, and you will have created your very own asset mod!

The format of a BeatOn asset mod is simple: It must be a .zip file that contains the following:

* `beatonmod.json`: A JSON file that describes information about the mod, as well as how to install and uninstall it

* Any files that are required by the mod

The format for a beatonmod.json closely resembles the format for manifest.json from BSIPA. Our beatonmod.json will look like this:

```json
{
  "ID": "CustomColors",
  "Name": "Custom Colors",
  "Author": "WHOAMI",
  "Description": [
	  "Template Asset Mod!",
	  "Replaces custom colors with those provided!"
  ],
  "Category": "Other",
  "TargetBeatSaberVersion": "1.1.0",
  "CanUninstall": false,
  "Components": [
    {
      "Type": "AssetsMod",
      "InstallAction": {
        "PreloadFiles": [
			"sharedassets1.assets"
		],
		"Actions": [
			{
				"Type": "ReplaceAsset",
				"StepNumber": 1,
				"Locator": { 
					"TypeIs": "SimpleColorSO",
					"PathIs": {
						"AssetFilename": "sharedassets1.assets",
						"PathID": 60
					}
				},
				"FromDataFile": "ColorA.dat"
			},
			{
				"Type": "ReplaceAsset",
				"StepNumber": 2,
				"Locator": { 
					"TypeIs": "SimpleColorSO",
					"PathIs": {
						"AssetFilename": "sharedassets1.assets",
						"PathID": 59
					}
				},
				"FromDataFile": "ColorB.dat"
			}
		]
      },
	  "UninstallAction": {
        "PreloadFiles": ["sharedassets1.assets"],
		"Actions": [
			{
				"Type": "RestoreAsset",
				"StepNumber": 1,
				"Locator": { 
					"TypeIs": "SimpleColorSO",
					"PathIs": {
						"AssetFilename": "sharedassets1.assets",
						"PathID": 60
					}
				}
			},
			{
				"Type": "RestoreAsset",
				"StepNumber": 2,
				"Locator": { 
					"TypeIs": "SimpleColorSO",
					"PathIs": {
						"AssetFilename": "sharedassets1.assets",
						"PathID": 59
					}
				}
			}
		]
      }
    }
  ]
}
```

Then, package your mod to contain a .zip for the above JSON (saved as `beatonmod.json`) and the ColorA.dat and ColorB.dat files you created. It should look something like this:

![BeatOn Mod](/uploads/asset-modding/23_beatonmod.png "BeatOn Mod")

# Conclusion

My goal with this guide was to walk you through the process of making an asset mod in a fairly generic manner, such that if you were to make other asset mods, you would have an idea of how to approach them.
Fun fact: This is how Custom Colors are currently supported, via an asset replacement that replaces everything the ColorManager uses.
As an extra challenge, try to see if you can figure out how to make the background lights change color too. I'll give you a hint: it must be one of the other 6 colors, take a look at their names and colors to see which it could be, change it, and see what happens.
If you liked what you read, please consider supporting me on [PayPal](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=XQP4GZDKK69DQ&currency_code=USD&source=url), or on [Ko-Fi](https://ko-fi.com/P5P7ZAA9) in order for me to purchase a Quest.
Hopefully you learned something today, feel free to DM me with any questions or concerns. I'll hopefully add an FAQ here later.

Thanks for reading!
Written by [Sc2ad](github.com/sc2ad), or on Discord: Sc2ad#8836
