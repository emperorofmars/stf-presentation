# STF Presentation Video Script
I create 3d avatars bases for social VR applications like VRChat and for V-Tubing applications like VSeeFace as a hobby. These get adapted by end users into their characters and used as their representation of themselves.

An avatar like that is a 3d model with added information about things like where is the viewport, what animation should be played based on which input, how its facial features should move based on speech or camera based facial tracking.

Most avatars have even more advanced features, like audio-reactive emissions, a bunch of clothing and asset toggles, settings to change how the avatar might behave.

The only way to distribute such an avatar base currently is a `.unitypackage`, with a scene with the avatar in there, somewhere.

End-users are expected to install Unity, a professional game-engine, import all dependencies and then the avatar's `.unitypackage`. Then end-users are required to know how to adapt the avatar to become their character, in this professional tooling.

Unity, and all game engines for that matter, are not character creation tools. Game-engines are tools you can create a character editor with.
Apart from changing the textures, blendshapes are the most common way to offer customization for avatars, as they can be changed by somewhat nice sliders. However, they are hardly suitable by themselves. For example, to scale the ears of this avatar, bone transforms are far better suited. Best would be a combination of bone transforms and a corrective blendshape.

To give end-users a usable way to change the ear size, would be a slider which blends between two animations... in a blendtree.

To create a fully stand-alone and easy-to-use character editor, you would need to associate blendtrees with a display-name, category, tooltip, perhaps an icon and a camera view.
Any arbitrary avatar, could be setup with this information and made easy to adapt for end-users, if such an application existed.

Optimally, target applications would allow users to directly upload a file exported from such an application.

As a hobby avatar creator, but professional software engineer, I thought creating such a character editor would not be too hard.
All I needed was a fileformat which could carry this extra information of avatar functionality and customization.

## glTF
glTF is an open standardized fileformat for 3d models & scenes. It consists of a scene definition in .json and a bunch of binary buffers for things like meshes and images.

According to the glTF specification, pretty much everything is extensible. Perfect for me, as I can just serialize my avatar & character editor information in .json and just create a character editor app which can parse this data.

YAAAY

### VRM Tangent
There is already a glTF extension format to support avatars, called VRM.
It supports 5 visemes, no animations, only blendshape poses, and only a few hardcoded shaders.
VRM is fully irrelevant for social VR and V-Tubing, and should not be used.

### Avatar Format Requirements
Ok, so let's make my own. Its just chugging .json around, pretty much the easiest thing in programming.

This format has to fulfill a few requirements, the most important are:
* Everything must be uniquely identifiable, even after modification, to support addon assets.
* Between import and export, the file must not change unless manually done so by the user.
* It must support the basic avatar features, and be further extensible to support all possible target applications to 100%.
* It will have to support mutually exclusive and redundant features. This is solved by having target specific components.
* Its objects must support being overridden and extended.
	For example, you can have a generic bone physics component, as well as more advanced but application specific ones, like VRC phys bones.
	In that case, if you import the avatar into VRChat, the VRC-physbone definition must 'override' the generic one.
* It must support arbitrary materials, independent of the specific target shader.

So, I tried to actually implement that, for both Unity and Godot 4, over a year ago.

### Bumping into glTF
For Unity I choose to work with UnityGLTF, as GLTFast the newer library was nowhere near finished.
Aaand I hit a hard wall. Both hardcode support for specific extensions and don't allow custom ones to be added.
I had to fork UnityGLTF and modify it to support my additional extensions as well.

I also noticed that blendshapes had no names on my models. I found out that Blender puts them on the 'extras' field of the mesh object. UnityGLTF expects them to be on the first mesh primitive for some reason.

Wait. Did they forget to actually put blendshape names into the glTF spec?
Yip. So it's not standardized and applications will put it where ever they feel like. Autsch.

It's slightly better on the Godot side. Godot has an interface to implement glTF extensions. So I used it and bumped into a looooot of issues with it. A lot of them got fixed by now, so I won't get into detail about them.

### glTF's shitness
Doing all of this work with and around glTF, I noticed a few more issues.
All the models I exported from Blender had a ridiculous on disk size and an absolutely ridiculous size in VRAM.

Diving deeper into the glTF spec and descending waay too deep the madness thinking about 3d fileformats, I think identified the 6 most significant issues with glTF:
- glTF doesn't really have the concept of mesh instances. Material references and morphtarget values sit on the mesh object.
  https://github.com/KhronosGroup/glTF/issues/1249
  https://github.com/KhronosGroup/glTF/issues/1036
- In glTF everything is addressed by its array-index. Indices are very likely to break between import and export. (If an extension is not supported by an application and gets stored as raw JSON, that references other objects by index, it will break. Addon assets like supported by STF would also break.)
- Limited animation support. Only transforms and morphtarget values (per mesh, not per mesh-instance) can be animated.
  The [KHR_animation_pointer](https://github.com/KhronosGroup/glTF/pull/2147) extension proposal would fix that partially.
- glTF only supports specific hard-defined material properties.
- The buffer system is convoluted and a lot of implementations don't seem to bother with it. As such blendshapes store values for every vertex, even if a vertex not included in the blendshape. Typical VR avatars have multiple hundred blendshapes, which leads to comical file sizes and VRAM use. This is the reason for the insane file sizes from Blender, and also Godot. The Unity VRM tooling has an option to do this correctly, which is off by default.
- glTF in its specification is supremely extensible, however implementing additional extensions in often not supported or accompanied by severe issues. Implementations have to be forked and modified at the core. The way glTF does extensions, does not naturally lead to a good design in its implementations.


All of this in mind, in order to make glTF work for my avatar format, or for anything at all for that matter, I would have to reimplement half of glTF in the glTF 'extensions' fields, and fork every implementation I want it to work with.

As much as it pains me saying that about an open & standardized format, I can only advise against using glTF, for any usecase.

## STF
So I made my own extensible 3d fileformat from scratch, called STF - Scene Transfer Format. It's very much based on the concept of glTF, but diverts in significant ways.

### Core Format
The .json definition has only assets, nodes, resources, as well as some meta information.
Both, nodes and resources can have components on them.
Each of these objects is addressed by a UUID and has a `type` property.

That's it.

The piece of code that imports and exports an object is matched based on the object's 'type' property.
By default, a set of supported standard types is included, but support for additional ones can be easily implemented and hot loaded.
This forces a good design in STF implementations, a plugin based architecture.

### MTF
In order to support arbitrary materials & shaders, STF materials consist of a set of arbitrary properties. A core set of standard ones will be defined, like albedo, metallicity, roughness, etc.
Arbitrary ones can be just added, for example audiolink base emissions, or fur density.

### Addons
Assets also have a type, as such I created the STF.addon_asset type. It requires for all of its root nodes to be addon nodes. Thanks to everything in STF being addressed by a unique ID, these addons can be applied even if a target asset has already been modified by a user.

### Application Targets
Multiple mutually exclusive features, like application specific bone-physics libraries, can be supported at the same time. They simply declare to which application they are specific and which other component to disable.

### Avatar Extensions
Additionally to STF and its core types, I created a proof of concept set of hot loaded extensions to express functionality that makes up VR and V-Tubing avatars in a subproject called AVA.

### Future

godot.
no blender


