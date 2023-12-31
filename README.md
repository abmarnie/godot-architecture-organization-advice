# Game Project Architecture and Organization Advice for Godot 4.0+

This article summarizes [my opinions](https://twitter.com/abmarnie) on structuring a mid-sized Godot 4.0+ project. It is drawn from my recent experience developing Ettare with [cdbunker2](https://twitter.com/cdbunker2), analyzing other Godot projects, reading the practices recommended by the Godot manual, and having discussions with other experienced Godot devs.

**Disclaimer**:
Be judicious: don't take things too far, only follow advice that you see the wisdom of, and avoid anything that seems unfun. In particular **architecture should always be tailored to the individual needs of a project.** For example: 
- In a 2 day game jam, you should probably ignore architecture completely.
- In some projects, a flatter directory structure, or a more "data oriented" approach where folders are seperated by data type (file extension) makes more sense.

**Contents**:
- [Directory Structure Advice](#directory-structure-advice)
- [Scene Structure Advice](#scene-structure-advice)
- [Quality of Life Advice](#quality-of-life-advice)

## Directory Structure Advice

This section aligns with the [Best Practices for Project Organization](https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html) section of the Godot manual. To avoid [technical issues related to case sensitivity](https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html#style-guide), note the use of `snake_case` for file and folder names, except for .cs files, which use `PascalCase`. Also note that using `.gltf` is generally recommended for larger teams, but for small teams in which everyone is comfortable using Blender, working directly with `.blend` files is very convenient.

- **Addons Folder**: Store third-party assets, scenes, and code in `addons/`, including their licenses.
- **Source Code Folder**: Place all source code in a `src/` folder for easy IDE navigation. *If you 
- **Search-Based Navigation**: Prefix scene filenames with their exclusive resources for efficient searching. Example: searching "balls_fish" should locate `balls_fish.tscn` and it's exclusive resources and files `balls_fish.gltf`, `balls_fish_albedo.png`, `balls_fish.mesh`, etc. In general, consider searchability when naming new files.
- **Scene-Based Assets Folder**:  Organize scenes and their resources in an `assets/` folder. Each scene should have it's own exclusive folder which contains itself and it's exclusive resources.
    - **Inherited Scene Folders**: Nest folders for inherited scenes within their base scene's folder.
    - **Scene Type Specific Resources**: Store commonly used resources for a specific scene type in a central folder, with subfolders for each owning scene. Avoid excessive nesting though.
    - **Globally Shared Resources**: Place general resources used by many different types of scenes in sibling folders, like `shaders/`.

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
|-- src/
|   |-- Player.cs
|   |-- Weapon.cs
|   |-- weapon_data.gd
|   |-- ...
```

While the Godot documentation suggests placing source code near associated scenes, using a separate `src/` folder can improve IDE navigation. The choice between the two approaches depends on personal preference and project needs.

[[Back to top.]](#game-project-architecture-and-organization-advice-for-godot-40)

## Scene Structure Advice

- **Single Controller Script Per Scene**: Attach one main "controller" script to each scene's root node. Name both the controller script and the root node after the scene. This reduces complexity by minimizing unnecessary inter-script communication. Multiple scripts may exist in the SceneTree, but only if a one-to-one correspondence between scripts and (sub)scenes is maintained.
- **Self-Contained Scenes**: Scenes should strive to be self contained, possessing all necessary resources they require. The controller script attached to the root node should only directly reference their children; otherwise dependencies need to be externally injected (see next tip). This keeps things modular and loosely coupled.
- **Dependency Injection Techniques**: For scenes with external dependencies, implement dependency injection (ideally from an ancestor) in one of the following ways:
	- Scene controller emits a signal/event for external procedures to run in response to.
	- Scene controller has a public (non-underscore-prefixed) method for externals to directly call.
	- Scene controller has a publically settable (non-underscore-prefixed) field/property for externals to directly inject a reference or value into.
- **Limited Scene Inheritance**: Use scene inheritance sparingly due to its inflexibility. Limit inheritance to one layer if it's too convenient to pass up. It is most useful and necessary when inherting from an imported scene (from a `.blend` or a `.gltf` file), or when a scene needs to share everything with a base scene except for some resource references.
- **Non-Editable Subscene Children**: Keep subscene children non-editable for encapsulation. Exceptions can be made (e.g., for editing collision shapes), but a design requiring editable children generally indicates that the scene's root node controller script is insufficiently exposing data or functionality.
- **Featureful Scenes**: To reduce clutter, avoid creating scenes with merely  1 or 2 nodes, *unless they have featureful controller scripts*. Often, non-featureful scenes can just be recreated in a few clicks. Some exceptions will naturally have to be made, e.g., for reusable visuals, or static level props. 
- **Generality of Scenes**: Design scenes for potential reuse across the game. To reduce clutter, scenes used precisely once and which do not persist across scene loads should be "inlined" (right click in SceneTree -> `Make Local` -> right click in FileSystem -> `View Owners` to double check -> delete).
- **Data Persistence and Sharing**: Use the `static` keyword for persistent data or shared information and functionality. Consider using [custom resources](https://docs.godotengine.org/en/stable/tutorials/scripting/resources.html#creating-your-own-resources) if inspector serialization is needed. [Use autoloads sparingly](https://docs.godotengine.org/en/stable/tutorials/best_practices/autoloads_versus_internal_nodes.html) for larger projects. Autoloads should generally be used only if they have no dependencies and they never have their state externally mutated.  
- **Structure SceneTree by Logical Relationship**: Organize the SceneTree relationally rather than spatially. Set `top_level = true` for spatial decoupling in parent-child relationships if needed.
- **Get Node Reference Sanely**: Use the new [scene unique nodes](https://docs.godotengine.org/en/stable/tutorials/scripting/scene_unique_nodes.html) feature to get nodes in a non-fragile way. Using `@export` is fine too, especially on smaller teams.
- **Eager Assertions**: Proactively assert (e.g., in `_ready`, or upon dependency injection) to ensure that critical node properties are correctly set. A proactive approach helps catch bugs early, reducing the need for excessive safety checks elsewhere.

[[Back to top.]](#game-project-architecture-and-organization-advice-for-godot-40)

## Quality of Life Advice

- **Refactor in the Editor**: Always move or rename files within the Godot editor to avoid Godot's cache from being desynchronized from your local files. If you need to rename an entire folder, consider first renaming the leaf files before recursively working your way down to the root folder.
- **Reduce Git Bloat**: For optimal Git LFS setup and to avoid version control bloat, use the `.gitattributes` and `.gitignore` provided in this repo.
- **Prefer `.tres` for Git**: When working with resources, prefer `.tres` over `.res` file extension, except potentially when dealing with large numerical data blobs like meshes. This makes Git history more human-interpretable.
- **Node Utilization**: Leverage existing Nodes for common functionalities, unless you have a good reason to roll your own.
- **View Owners before Deleting**: Right click -> `View Owners` before deleting scenes or resources, to make sure you won't break anything.
- **Reduce FileSystem Clutter**: Create an empty `.gdignore` file in any folders which shouldn't show up inside the FileSystem dock.
- **Improve Folder Visibility**: Color code project folders with right click -> `Set Folder Color`.
- **Font Size**: Increase font size in Editor Settings to reduce long-term eye strain.
- **Tech Stack Updates**: Regularly update Godot, .NET/C#, etc., for improvements. Be cautious about updating near release, or once the project becomes very large.
- **Static Type Warnings**: As of Godot 4.2+, [enable type warnings](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/static_typing.html#warning-system) for better autocomplete and compile time error detection. If you prefer succinctness, use type inference syntax `:=` for variable initialization.
- **Script Templates**: Save customized [script templates](https://docs.godotengine.org/en/stable/tutorials/scripting/creating_script_templates.html) in `.gdignore`'d `script_templates/` folder. Adjust Project Setting's `editor/script_templates_search_path` to `script_templates` folder. Consider using the `default.gd` and `Default.cs` provided in this repo for some saner defaults. To do so, simply save those files into a new `script_templates/node/` folder.

[[Back to top.]](#game-project-architecture-and-organization-advice-for-godot-40)
