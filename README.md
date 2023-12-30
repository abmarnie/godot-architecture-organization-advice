# Game Project Architecture and Organization Advice for Godot 4.0+

This article summarizes my (https://twitter.com/abmarnie) opinions on structuring a mid-sized Godot 4.0+ project, drawn from my three months of experience developing [Ettare](https://twitter.com/cdbunker2) in Godot, analyzing other Godot projects, discussing with other experienced Godot devs, and my previous two years of experience in game development in Unity.It goes without saying, be judicious, don't take things too far, only follow advice that you see the wisdom of, and most of all, don't do anything that would make things less fun.

**Contents**::
- [Directory Structure Advice](#directory-structure-advice)
- [Scene Structure Advice](#scene-structure-advice)
- [QoL Advice](#qol-advice)
- [Git Advice](#git-advice)

## Directory Structure Advice

This section is mostly in-line with the philosophy advocated for in the [Best Practices for Project Organization](https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html#style-guide) section of the manual. Note that all file and foldernames should use `snake_case`, with the exception of `.cs` files, which should use `PascalCase`, otherwise, you will run into headaches later.

- **Addons Folder**: Third party resources, scenes, and code are placed in an `addons/` folder, alongside their licenses.
- **Source Code Folder**: Source code is placed in a single `src/` folder, for ease of navigation while working in IDEs.
- **Search-Based Navigation**: Scene filenames prefix their exclusive resources, making them searchable together. For instance, searching "balls_fish" finds `balls_fish.tscn` and its exclusive resources like `balls_fish.blend`, `balls_fish.png`, etc. In general, when naming things, put thought into how searchable the name is.
- **Scene-Based Assets Folder**: Scenes and their resources are placed in an `assets/` folder. Each scene is in its own folder with its exclusive resources. 
    - **Inherited Scene Folders**: Folders for inherited scenes are nested within their base scene's folder.
    - **Shared Resources**: Common resources used by multiple scenes are stored in a central folder with subfolders for each owning scene. It is easy to take this too far (too much folder nesting), so be judicious.
    - **Unowned Resources**: General resources not specific to any scene are placed in sibling folders, like `shaders/` for unowned `.shader` files.

Here is an ASCII art example:
```bash
project_root/
|-- .gitignore
|-- .gitattributes
|-- README.md
|-- addons/
|   |-- third_party_asset_1/
|   |   |-- license.md
|   |-- third_party_asset_2/
|   |   |-- license.md
|   |-- ...
|-- assets/
|   |-- foliage/
|   |   |-- foliage.material
|   |   |-- foliage_albedo.png
|   |   |-- foliage.shader
|   |   |-- grass_1/
|   |   |   |-- grass_1.blend
|   |   |   |-- grass_1.tscn
|   |   |-- grass_2/
|   |   |   |-- grass_2.blend
|   |   |   |-- grass_2.tscn
|   |   |-- ...
|   |-- shaders/
|   |   |-- generic_shader_1.shader
|   |   |-- generic_shader_2.shader
|   |-- player/
|   |   |-- player.tscn
|   |   |-- player.blend
|   |   |-- player_albedo.png
|   |-- weapon/
|   |   |-- weapon.tscn
|   |   |-- axe/
|   |   |   |-- axe_weapondata.tres
|   |   |   |-- axe.tscn
|   |   |   |-- axe.blend
|   |   |   |-- axe_abledo.png
|   |   |-- ...
|   |-- ...
|-- src/
|   |-- Player.cs
|   |-- Weapon.cs
|   |-- weapon_data.gd
|   |-- ...
```

The Godot documentation suggests placing source code near its associated scenes for consistency, but this is less familiar to programmers and may impact IDE navigation. Instead, I recommend identifying a script's corresponding scene through the fact that it's name matches the aligned scene's name.

[[Back to top.]](#game-project-architecture-and-organization-advice-for-godot-40)

## Scene Structure Advice

- **Limited Scene Inheritance**: Avoid scene inheritance due to its complexity and poor documentation. If it's too convenient to pass up, limit to one layer of inheritance. The main case where it is "too convenient to pass up" is when the inherited scene needs to share absolutely everything from a base scene, except for resource references, as this makes it more convenient to instance things through code.
- **Non-Editable Subscene Children**: Subscenes' children should be non-editable to maintain encapsulation and a clean SceneTree. Exceptions are allowed (editing children collision shapes), but in most cases having a design that requires so suggests that the root node is not selectively exposing enough (allowing editable children exposes *everything*, which breaks encapsulation).
- **Single Controller Script Per Scene**: Each scene should have one main "controller" script, named after the scene and its root node, attached to the root. This reduces complexity by minimizing script communication and "scriptitis" (too many small scripts). Multiple scripts can exist in the SceneTree, but only if there is a one-to-one correspondence between scripts and (sub)scenes.
- **Self-Contained Scenes**: Scenes should strive to be self contained, possessing all necessary resources they require. The controller script attached to the root node should only directly reference their children; otherwise dependencies need to be externally injected (see next tip). This keeps things modular and loosely coupled.
- **Dependency Injection Techniques**: For scenes with external dependencies, which will inevitably happen, implement dependency injection (ideally, from an ancestor) in one of the following ways:
	- Scene controller emits a signal/event for external procedures to run in response to.
	- Scene controller has a public (non-underscore-prefixed) method for externals to directly call.
	- Scene controller has a publically settable (non-underscore-prefixed) field/property for externals to directly inject a reference or value into.
- **Featureful Scenes**: Scenes should be deep / featureful. Scenes containing merely 1 or 2 nodes are to be avoided to reduce clutter and complexity. Often, simple 1 or 2 node scenes, especially ones without featureful controller scripts, can simply be recreated in just a few clicks. Exceptions should be made for scenes with deep / featureful controller scripts. 
- **Generality of Scenes**: Scenes should be generally useful, potentially being instanced and reused across many other scenes, or throughout the the game/application lifecycle. A scene that is instanced precisely once and does not persist across scene loads should probably be "inlined" (right click in SceneTree -> `Make Local` -> right click in FileSystem -> `View Owners` to double check -> delete) to reduce clutter and complexity.
- **Data Persistence and Sharing**: Use `static` for data persisting between scenes or shared across objects. For inspector serialization, consider [custom resources](https://docs.godotengine.org/en/stable/tutorials/scripting/resources.html#creating-your-own-resources). Prefer `static` classes over [autoloads](https://docs.godotengine.org/en/stable/tutorials/best_practices/autoloads_versus_internal_nodes.html), unless inheritance from a `Godot` type is needed. Use `static` classes or autoloads only if they  have no dependencies and self-manage their own state. 
- **Structure SceneTree by Relationship**: The SceneTree should be organized in relational terms rather than spatial terms. In order to spatially decouple nodes in a parent-child relationship, simply set `top_level = true` in the child.
- **Get Node Reference Sanely**: Use the wonderful new [scene unique nodes](https://docs.godotengine.org/en/stable/tutorials/scripting/scene_unique_nodes.html) feature to get nodes in a non-fragile way. Using `@export` is fine as well, especially if the team is small.
- **Eager Assertions**: Proactively assert (e.g., in `_ready` or upon dependency injection) in scene controllers to ensure that critical node properties are correctly set. A proactive approach helps catch bugs early, reducing the need for excessive safety checks elsewhere.

[[Back to top.]](#game-project-architecture-and-organization-advice-for-godot-40)

## QoL Advice

- Regularly update your tech stack, including Godot, .NET / C# (if applicable), etc., to benefit from fixes and improvements. It does not take very long. As always, be judicious, if you are nearing release or your project gets rather large, it's probably a good idea to lock in your tech stack versions unless a new must-have fix is released.
- Always move files and rename them within the Godot editor, or you will suffer.
- For GDScript in Godot 4.2, enable type warnings for better autocomplete and to convert runtime errors into compile-time errors. If you don't like writing types, use type inference syntax `:=` for all variable initialization.
- Use existing Nodes for common functionalities. Less code is better.
- Right click -> `View Owners` before deleting scenes or resources to ensure safety. 
- Create an empty `.gdignore` file in any folders which shouldn't show up inside the Godot FileSystem Dock. 
- Color-code project folders for better organization by right click -> `Set Folder Color`.
- Increase font size in Editor Settings for eye comfort.
- Set saner defaults for new scripts by saving a template in a new `.gdignore`'d folder`script_templates/node/` and adjusting the Project Setting's `editor/script_templates_search_path` to point to the following new `.gd` and `.cs` files saved in said folder: 

```
# meta-name: Default
# meta-description: Custom base template for Node
# meta-default: true

extends  _BASE_

func  _ready()  ->  void:
	 pass

func  _physics_process(delta:  float)  ->  void:
	 pass
```
```cs
// meta-name: Default
// meta-description: Custom base template for Node
// meta-default: true

using _BINDINGS_NAMESPACE_;
using System;

namespace MyDefaultNamespace;

public partial class _CLASS_ : _BASE_
{
    public override void _Ready()
    {
	    
    }

    public override void _PhysicsProcess(double delta)
    {
	    
    }
}
```

[[Back to top.]](#game-project-architecture-and-organization-advice-for-godot-40)

## Git Advice

Use the following `.gitattributes` file to set up Git LFS properly to avoid version control bloat:
```bash
# Normalize EOL for all files that Git considers text files.
* text=auto eol=lf

# Godot specific binaries
*.res filter=lfs diff=lfs merge=lfs -text

# Image formats
*.bmp filter=lfs diff=lfs merge=lfs -text
*.dds filter=lfs diff=lfs merge=lfs -text
*.exr filter=lfs diff=lfs merge=lfs -text
*.hdr filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.jpeg filter=lfs diff=lfs merge=lfs -text
*.png filter=lfs diff=lfs merge=lfs -text
*.tga filter=lfs diff=lfs merge=lfs -text
*.svg filter=lfs diff=lfs merge=lfs -text
*.webp filter=lfs diff=lfs merge=lfs -text

# Audio and Video formats
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.wav filter=lfs diff=lfs merge=lfs -text
*.ogg filter=lfs diff=lfs merge=lfs -text
*.ogx filter=lfs diff=lfs merge=lfs -text
*.ogv filter=lfs diff=lfs merge=lfs -text

# 3D formats
*.gltf filter=lfs diff=lfs merge=lfs -text
*.blend filter=lfs diff=lfs merge=lfs -text
*.blend1 filter=lfs diff=lfs merge=lfs -text
*.glb filter=lfs diff=lfs merge=lfs -text
*.dae filter=lfs diff=lfs merge=lfs -text
*.obj filter=lfs diff=lfs merge=lfs -text
*.fbx filter=lfs diff=lfs merge=lfs -text

# Build
*.dll filter=lfs diff=lfs merge=lfs -text

# Packaging
*.zip filter=lfs diff=lfs merge=lfs -text
*.7z filter=lfs diff=lfs merge=lfs -text
*.gz filter=lfs diff=lfs merge=lfs -text
*.rar filter=lfs diff=lfs merge=lfs -text
*.tar filter=lfs diff=lfs merge=lfs -text
*.file filter=lfs diff=lfs merge=lfs -text
*.dylib filter=lfs diff=lfs merge=lfs -text
```

Use the following `.gitignore` file to avoid further version control bloat:
```bash
nupkg/

.godot/
bin/
obj/
.generated/
.vs/
.DS_Store

*.translation
*.blend1
``` 

Saving resources as `.tres` extension (instead of `.res`) is generally advisable for most resource types, because it makes Git history more interpretable. The main exceptions are resources which are just huge numerical data blobs, like meshes.

[[Back to top.]](#game-project-architecture-and-organization-advice-for-godot-40)
