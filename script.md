# STF Presentation Video Script
I create 3d avatars for social VR applications like VRChat and for V-Tubing applications like VSeeFace.

An avatar like that is a 3d model with added information about things like where is the viewport, how to animate it based on motion tracking, what animation should be played based on which input, how its facial features should move based on speech or facial tracking.

Most avatars have even more advanced features, like audio-reactive emissions, a bunch of clothing and asset toggles, which change how the avatar might behave.

The only way to distribute such an avatar currently is a .unitypackage, with a scene with the avatar in there, somewhere.

End-users are expected to install a professional game-engine, import all dependencies and then the avatar's.unitypackage. Then end-users are required to know how to adapt the avatar to become their character.

Unity, and all game engines for that matter, are not character creation tools. Game-engines are tools you can create a character editor with. While blendshapes have somewhat nice sliders, they are hardly suitable by themselves. To scale the ears of this avatar, bone transforms are far better suited. Best would be a combination of bone transforms and a corrective blendshape, perhaps.

To give end-users a usable way to change the ear size would be a slider which blends between two animations... in a blendtree.

To create a fully stand-alone and easy-to-use character editor, you would need to associate blendtrees with a display-name, category, tooltip, perhaps an icon, and perhaps a camera view.
Any arbitrary avatar could be setup with this information and made easy to adapt for end-users if such an application existed.

Optimally, target applications would allow users to directly upload such a file.

As a hobby avatar creator, but professional software engineer, I thought creating such a character editor would not be too hard.
All I needed was a fileformat which could carry this information.

## glTF
glTF is a open standardized fileformat for 3d models & scenes. It consists of a scene definition in .json and a bunch of binary buffers for things like meshes, images and such.

According to the specification, pretty much everything is extensible. Perfect for me, i can just serialize my avatar & character editor information in .json and just create a character editor app which can work with that.

YAAAY

### VRM Tangent
There is already a glTF extension format called VRM which supposedly is already that.

öööhhmmm

* It supports 5 visemes, the rest of social VR supports 15 as well as facial tracking. And all of that can be arbitrarily animated in a state machine.
* VRM, as well as glTF, supports only a few hard-coded shaders. VRC and such support custom shaders.
* VRM doesn't support animations, only blendshape poses. There is no information about how to activate them or any sort of animation logic.

VRM is fully irrelevant for social VR and VTubing and should not be used.

### Avatar Format Requirements
Ok, so lets make my own. Its just chugging .json around, pretty much the easiest thing in programming.

This format has to fulfill a few requirements.
* It must support the basic avatar features, but be further extensible. To support all target applications it will have to support mutually exclusive features. This is solved by having target specific components.
* It must preserve unrecognized and unsupported parts.
* Everything must be uniquely identifiable, even after modification, to support addon assets.
* Its objects must support being overridden and extended.
	For example, you can have a generic bone physics component, as well as more advanced but application specific ones like VRC phys bones.
	In that case, if you import the avatar into VRChat, the VRC phys bone definition must 'override' the generic one.
* It must support arbitrary materials independent of the specific target shader.

### Bumping into glTF
So, i tried to actually implement that, for both Unity and Godot 4.

For Unity i choose to work with UnityGLTF, as GLTFast, the newer one is nowhere near finished.
And i hit a hard wall. You can't extend it. But its open source, so i forked it and made it work anyway. The codebase reminds me of spaghetti code and i had to do ugly things like exporting private methods as function pointers to my custom code to add the simplest things to it.

I also noticed that blendshapes had no names on my models. I found out that Blender puts them on the 'extras' field of the mesh. UnityGLTF expects them to be on the first mesh primitive for some reason.

Wait. Did they forget to actually put blendshape names into the glTF spec?
Yip. So it's not standardized and applications will put it where ever they feel like. Autsch.

It's slightly better on the Godot side. Godot has an interface to implement glTF extensions. So I used it and bumped into a looooot of issues with it. A lot of them got fixed by now so I won't get into detail about them.

### glTF's shitness
Doing all of this work with and around glTF i noticed a few more issues with glTF.
All the models i exported from Blender had a ridiculous on disk size and a absolutely stupid VRAM size.

TODO: sparse accessors and buffer system fuckery

* hard coded resource types
* mesh and mesh instance
* material system


All of this in mind, in order to make glTF work for my avatar format, or for anything at all, i would have to reimplement half of glTF in the glTF 'extensions' fields.

At that point my files would not be compatible with most glTF implementations whatsoever, and even if, in order for them to support my avatar extensions, i have to fork most of them and ask my avatars' users to import this custom glTF library.

As much as it pains me saying that about an open & standardized format, i can only advise against using glTF, for any usecase.

## STF
So i made my own from scratch. It's very much based on the concept of glTF, but diverts in significant ways.

### core format

### MTF

### addons

### application targets

### avatar extensions

### future (no blender)

---
Almost all relevant target applications for my avatars are based on Unity. Each target application has its own set of components and expects for the avatar to be setup in its own special way, so i have to create a duplicate setup, mostly from scratch, for each application.

