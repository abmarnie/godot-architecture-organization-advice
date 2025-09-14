# Game Project Architecture and Organization Advice for Godot 4.0+

This article summarizes my opinions on structuring a mid-sized Godot 4.0+ project. Most of the opinions stated here are drawn from my own personal and professional experiences working within Godot, discussions with other experienced developers, and practices recommended by the official Godot documentation.

Comments and suggestions are welcome, [start a discussion](https://github.com/abmarnie/godot-architecture-organization-advice/discussions/new/choose) or [raise an issue](https://github.com/abmarnie/godot-architecture-organization-advice/issues/new).

#### Disclaimer

The advice is a summary of observations and not a step-by-step guide. **Tailor architecture to the individual needs of each project.** Consider your goals and follow recomendations selectively. Avoid anything that seems unfun (don't feel compelled to do a massive refactor). Generally be judicious. 

- In a 2 day game jam, you should probably ignore architecture entirely. If your codebase is going to end up under 10k lines-of-code, your project is to expected to only take 6 months, and/or you are working solo, the benefits here are probably marginal at best. The importance of architecture scales with the size (project size, team size) and complexity of your project.
- In some projects, a more "data oriented" approach where folders are seperated by data type (file extension) might make more sense. One advantage of this approach is that you don't have to spend any time thinking about where to add new files. 
- Whether or not to place source code files further away from scenes is mostly subjective. I use VSCode heavily and personally find that having all my source code in one place makes things easier for me. 

#### Contents

- [Directory Structure Advice](#directory-structure-advice)
- [Scene Structure Advice](#scene-structure-advice)
- [Quality of Life Advice](#quality-of-life-advice)

## Directory Structure Advice

This section (mostly) aligns with the [Best Practices for Project Organization](https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html) section of the Godot manual. To avoid [technical issues related to case sensitivity](https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html#style-guide), note the use of `snake_case` for file and folder names, except for `.cs` files, which use `PascalCase` instead. 
 
- **Addons Folder**: Store third-party assets, scenes, and code in `addons/`, including their licenses.
- **Source Code Folder**: Place all source code in a `src/` folder for easy codebase navigation. This is rather subjective; if you prefer, you can treat source code like any other resource (see the "Scene-Based Assets Folder" tip below).
- **Search-Based Navigation**: Prefix names of resources exclusively-used by a specific scene with that scene's name for efficient searching. Example: searching "balls_fish" should locate `balls_fish.tscn` and it's exclusive resources and files `balls_fish.gltf`, `balls_fish_albedo.png`, `balls_fish_fishdata.tres`, `balls_fish.mesh`, etc.
- **Scene-Based Assets Folder**:  Organize scenes and their resources in an `assets/` folder. Each scene should have it's own folder which contains itself and it's exclusive resources. Godot really encourages you to treat scenes as first-class citizens which to structure your entire project around.
    - **Inherited Scene Folders**: Nest folders for inherited scenes within their base scene's folder.
    - **Locally Shared Resources**: Store shared resources for a specific "scene type" in a central folder named after that "scene type", with subfolders for each owning scene. Avoid excessive nesting though.
    - **Globally Shared Resources**: Place general resources used by many different "scene types" in a sibling folder named after that resource's data type. For example, put all globally used `.shader` files in a `shaders/` folder.

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
|   |   |   |-- grass_1.gltf
|   |   |   |-- grass_1.mesh
|   |   |   |-- grass_1.tscn
|   |   |-- grass_2/
|   |   |   |-- grass_2.gltf
|   |   |   |-- grass_2.mesh
|   |   |   |-- grass_2.tscn
|   |   |-- ...
|   |-- shaders/
|   |   |-- generic_shader_1.shader
|   |   |-- generic_shader_2.shader
|   |-- player/
|   |   |-- player.tscn
|   |   |-- player.gltf
|   |   |-- player_albedo.png
|   |-- weapon/
|   |   |-- weapon.tscn
|   |   |-- axe/
|   |   |   |-- axe_weapondata.tres
|   |   |   |-- axe.tscn
|   |   |   |-- axe.gltf
|   |   |   |-- axe_abledo.png
|   |   |-- ...
|   |-- ...
|-- src/ # Alternatively, localize the source code to the scene it controls.
|   |-- Player.cs
|   |-- Weapon.cs
|   |-- weapon_data.gd
|   |-- ...
```

[[Back to top.]](#game-project-architecture-and-organization-advice-for-godot-40)

## Scene Structure Advice

- **Single Controller Script Per Scene**: Attach one main "controller" script to each scene's root node. Name both the controller script and the root node after the scene. This reduces complexity by minimizing unnecessary inter-script communication. Multiple scripts may exist in the SceneTree, but only if a one-to-one correspondence between scripts and (sub)scenes is maintained.
- **Self-Contained Scenes**: Scenes should strive to be self contained, possessing all necessary resources they require. The controller script attached to the root node should only directly reference their children (or descendants); otherwise dependencies need to be externally injected (see next tip). This keeps things modular and loosely coupled.
- **Dependency Injection Techniques**: For scenes with external dependencies, implement dependency injection (ideally from an ancestor) in one of the following ways:
	- Scene controller emits a signal/event for external procedures to run in response to.
	- Scene controller has a public (non-underscore-prefixed) method for externals to directly call.
	- Scene controller has a publically settable (non-underscore-prefixed) field/property for externals to directly inject a reference or value into.
- **Limit Scene Inheritance**: Use scene inheritance sparingly due to its inflexibility. Limit inheritance to one layer if it's too convenient to pass up. It is most useful and necessary when inherting from an imported scene (from a `.blend` or a `.gltf` file).
- **Non-Editable Subscene Children**: Keep subscene children non-editable for encapsulation. Exceptions can be made (e.g., for editing collision shapes), but a design requiring editable children generally indicates that the scene's root node controller script is insufficiently exposing data or functionality.
- **Featureful Scenes**: To reduce clutter, avoid creating scenes with merely  1 or 2 nodes, *unless they have featureful controller scripts*. Often, non-featureful scenes can just be recreated in a few clicks. Some exceptions might naturally be made for editing convenience, e.g., for reusable visuals, reusable static level props (see next tip), or if extending a class with a few pieces of data would seriously help you or is absolutely necessary (for example, if you need to override a specific virtual method, like `RigidBody3D._integrate_forces`).
- **Generality of Scenes**: Design scenes for potential reuse across the game. To reduce clutter, scenes used precisely once and which do not persist across scene loads should be "inlined" (right click in SceneTree -> `Make Local` -> right click in FileSystem -> `View Owners` to double check -> delete in FileSystem). You can always undo this later by doing: right click in SceneTree -> `Save Branch as Scene`.
- **Data Persistence and Sharing**: Prefer to use the `static` keyword for scene-persistent data or instance-independent/shared data (e.g., [flyweight pattern](https://gameprogrammingpatterns.com/flyweight.html)) if possible. Other options include [custom resources](https://docs.godotengine.org/en/stable/tutorials/scripting/resources.html#creating-your-own-resources) (also useful for implementing flyweight and [type object](https://gameprogrammingpatterns.com/type-object.html) patterns) and autoloads (which are useful for cross-cutting concerns such as quest or dialogue systems; see the [official Godot autoload best practice recommendations](https://docs.godotengine.org/en/stable/tutorials/best_practices/autoloads_versus_internal_nodes.html)).
- **Structure SceneTree by Logical Relationship**: Organize the SceneTree relationally rather than spatially. Set `top_level = true` for spatial decoupling in parent-child relationships if needed.
- **Script Member Ordering**: The more consistent things are ordered, the easier it is to navigate and make changes. It is [officially recommended](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html#code-order) to order script members in the following way:
```
01. @tool
02. class_name (PascalCase)
03. extends
04. # docstring

05. signals (snake_case)
06. enums (PascalCase, members are CONSTANT_CASE)
07. constants (CONSTANT_CASE)
08. @export variables (snake_case)
09. public variables (non-underscore-prefixed snake_case)
10. private variables (underscore-prefixed _snake_case)
11. @onready variables (snake_case)

12. optional built-in virtual _init method
13. optional built-in virtual _enter_tree() method
14. built-in virtual _ready method
15. remaining built-in virtual methods (underscore-prefixed _snake_case)
16. public methods (non-underscore-prefixed snake_case)
17. private methods (underscore-prefixed _snake_case)
18. subclasses (PascalCase)
```

The same ordering rules can be applied in C#. Some C# specific ordering considerations include: 

- Put lightweight nested `struct` declarations up at the top, next to nested `enum` declarations.
- Put the backing fields of properties right before the property which uses them, even if they would be placed somewhere else otherwise.
- C# `event`s (usually `Action` or `Func` types) should be placed at the top, where GDScript signals would go.
- `Get`-only properties are basically just methods with non-`void` return types, so they should be grouped with methods.
- Group `interface` implementations together. They should be placed right above `public` methods. You should probably make a comment `/// <see cref="IMyInterface"/>` to indicate the group.

[[Back to top.]](#game-project-architecture-and-organization-advice-for-godot-40)

## Quality of Life Advice

- **Get Node Reference Sanely**: Use the new [scene unique nodes](https://docs.godotengine.org/en/stable/tutorials/scripting/scene_unique_nodes.html) feature to get nodes in a non-fragile way. Using `@export` is fine too, especially on smaller teams.
- **Sharing Scenes across Projects**: Right click on a scene, click `Edit Dependencies`. If the dependencies are local to that Scene's folder, then you can simply drag and drop that folder across Godot projects and things should just work. This opens up new workflows, allowing artists who aren't comfortable with Git to work in seperate / local projects.
- **Eager Assertions**: Proactively assert (e.g., in `_ready`, or upon dependency injection) to ensure that critical node properties are correctly set. A proactive approach helps catch bugs early, reducing the need for excessive safety checks elsewhere.
- **Reduce Git Bloat**: For optimal Git LFS setup and to avoid version control bloat, use the `.gitattributes` and `.gitignore` provided in this repo. Simply download them and place them in the root of your Git repo. I recommend you read [GitHub's documentation on managing large files](https://docs.github.com/en/repositories/working-with-files/managing-large-files).
- **Refactor in the Editor**: Always move or rename files within the Godot editor to avoid Godot's cache from being desynchronized with your local files. If you need to rename an entire folder and doing so naively breaks things (you're using Git, right?), consider first renaming the leaf files before recursively working your way down to the root folder.
- **Commit Frequently While Refactoring**: As of Godot 4.2, refactoring (renaming and moving around files) is still not very stable. Before you attempt a file or folder rename, or moving files, make a commit, so you can `git restore .` in case things go bad.
- **Importing 3D Assets**:`.gltf` is generally recommended for larger teams (`.gltf` is to be preferred over `.glb`). For small teams in which everyone is comfortable having Blender installed, working directly with `.blend` files in the engine is extremely convenient for fast iteration, and is also more convenient for version control purposes (so long as you use the `.gitattributes` and `.gitignore` file provided in this repo). See [this](https://docs.godotengine.org/en/stable/tutorials/assets_pipeline/importing_3d_scenes/available_formats.html#importing-blend-files-directly-within-godot) for detailed instructions for working with `.blend` files in Godot.
- **Prefer `.tres` for Git**: When working with resources, prefer `.tres` over `.res` file extension, except potentially when dealing with large numerical data blobs like meshes. This makes Git history more human-interpretable.
- **Resource File Extension Consistency**: Be consistent with resource file extensions. For example, if you want to use `.mesh` over `.res`/`.tres` for mesh data, do so consistently. The same goes for `.material`, `.shape`, etc.
- **Node Utilization**: Leverage existing Nodes for common functionalities, unless you have a good reason to roll your own.
- **View Owners before Deleting**: Right click -> `View Owners` before deleting scenes or resources, to make sure you won't break anything.
- **Reduce FileSystem Clutter**: Create an empty `.gdignore` file in any folders which shouldn't show up inside the FileSystem dock.
- **Improve Folder Visibility**: Color code project folders with right click -> `Set Folder Color`.
- **Font Size**: Increase font size in Editor Settings to reduce long-term eye strain.
- **Tech Stack Updates**: Consider update Godot, .NET/C#, etc., for improvements depending on the release timeline for your game. If your game is to be release in, say, only 6 months to a year from now, it probably is worth completing locking down all dependencies versions.
- **Static Type Warnings**: As of Godot 4.2+, [enable type warnings](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/static_typing.html#warning-system) for better autocomplete and compile time error detection. If you prefer succinctness, use type inference syntax `:=` for variable initialization.
- **Script Templates**: Consider saving the `default.gd` (or `Default.cs`) file provided in this repo into a `.gdignored` `script_templates/node` folder. Doing so provides you with a nicer default starting template everytime you create a new script.
- **Resetting Cache**: To address errors which happen after git branch switches and pulls, delete the `.godot` folder in the project root to reset the cache (this will force a re-import all assets, so be careful if you have many). Then do `Project` > `Reload Current Project`. This can help resolve various issues such as scene corruption, renaming errors, and other irregularities.

[[Back to top.]](#game-project-architecture-and-organization-advice-for-godot-40)
