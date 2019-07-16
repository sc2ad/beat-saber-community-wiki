<!-- TITLE: Asset Modding Guide -->
<!-- SUBTITLE: For Oculus Quest using BeatOn -->

# Introduction

Hopefully you are here because you want to know how to make your own assets mod with emulamer's [BeatOn](https://github.com/emulamer/BeatOn).
While most of what I will cover (or hope to cover) should be applicable to generic asset modding (for any Unity game, really) this guide will be specific to BeatOn and Beat Saber.

>WARNING! THIS IS NOT MEANT TO BE AN ENTIRELY BEGINNER FRIENDLY GUIDE! THIS GUIDE ASSUMES YOU HAVE SOME EXPERIENCE WITH MODDING, WHETHER IT BE THROUGH CODE OR NOT, AND IS NOT FOR THE FAINT OF HEART. YOU HAVE BEEN WARNED!
{.is-danger}

# Setup

Because you haven't left yet, I'm going to assume you are actually interested in making an asset mod. Here's a quick list of everything that you should have for this guide:

* The Beat Saber APK (from the Oculus Quest version). You can get this by doing `adb pull com.beatgames.beatsaber`.

* The latest release of [UABE](https://github.com/DerPopo/UABE/releases). While there are alternatives to UABE (such as emulamer's Assxplorer), it is what I will be showcasing in this guide, so if you want to follow along, I would recommend using UABE.

* (OPTIONAL BUT HIGHLY RECOMMENDED) The PC version of Beat Saber (either Steam or Oculus is fine) somewhere on your computer (specifically the DLLs located under the `BeatSaber_Data/Managed` folder). We want this because it will make reading through the assets' fields much easier.

# Begin

## What is an Asset Mod?

First, I want to start by clarifying what an asset mod _is_ and what makes it different from a hook mod.
An asset mod is called an asset mod because, well, it modifies the assets (who would have guessed?)
However, what this means is that anything dynamic (where it changes during the game) _cannot be modified by an asset mod_. For example, Sc2ad's HitScoreVisualizer is only possible as a hook mod, not as an asset mod. This is because it is _impossible_ to change anything in the assets that lets us change the color of the text that is displayed when we cut a block, because it is simply not loaded from the assets.
When in doubt, if it ever changes in the game, it probably isn't asset moddable. However, if it _doesn't_ change in the game, it likely _is_ asset moddable. Hopefully that makes sense.

__Yuuki:__ Another example to look at is custom sabers. Sabers stay the same throughout the entirety of your play session (they are static). This allows them to be modified via asset modding.
In comparison, text is dynamic and always changing during the game. Your points for each saber cut is dynamic requiring a hook mod.

## Mod Introduction

Now that I have hopefully cleared up some possible misconceptions, let's begin!
__Yuuki:__  ~~Let's begin!~~ My goal in this guide will be to create a small asset mod that allows me to set my saber colors to specific colors. Think: Custom Colors, but without changing the colors in the background.
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

This (along with step 3) are the hardest steps because they involve a lot of digging through the assets and require a moderate understanding of how the game functions.

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

![UABE ColorManager Expanded](/uploads/asset-modding/12_uabe_colormanager_expanded.png, "UABE ColorManager Expanded")

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
