# Development Notes

## Identifying a Mod

A mod can be identified by a slug (i.e. a short generated name), its NexusMods ID, or a GitHub repository (`user/repo`).

### Slugs

A slug is a short, unique, memorable name that is generated from the name of the mod (e.g. `enhanced-cyberware`). If the mod comes from Nexus, the name provided by the Nexus API is used. If the mod comes from GitHub, the repository name is used.

Slugs can only contain dashes and lowercase alphanumeric characters. At least one alphabetical character must be present, and the slug cannot have a leading or trailing dash. This is to distinguish them from NexusMods IDs, which are fully numerical, and GitHub repository identifiers, which contain slashes.

Slugs can change over the lifetime of a mod. This can be user-initiated, or automatically (usually from an update). If a slug is automatically changed (e.g. mod version update or rename), `ripperdoc` will print a message reading something like "The slug for this mod has been automatically updated to \<slug\>. You can change this using `mod edit <slug> --slug`.", or ask during the update (Policy TBD).

Slugs are the primary user-facing key for identifying a given mod. As such, several slugs and associated aliases are hardcoded into `ripperdoc`, which will recognize them as core mods. These slugs are:

- `cet`/`cyber-engine-tweaks` - Cyber Engine Tweaks
- `redscript` - RedScript
- `red4ext` - RED4Ext
- `codeware` - Codeware
- `archivexl` - ArchiveXL
- `tweakxl` - TweakXL
- `cybercmd` - cybercmd (used for recognizing when to ignore a dependency)

This hardcoded recognition is generally used when installing mods (to see if the installed mod is one of the above), or for dependency checking. (details TBD) Mods with hardcoded slugs cannot have their slugs changed.

### NexusMods ID

A NexusMods ID is a numerical identifier assigned to a mod when it is uploaded to NexusMods. If a provided identifier is fully numerical, `ripperdoc` will treat it as a NexusMods ID.

### GitHub Repository

Some mod authors make the source code of their mod available on GitHub, and provide it as another avenue from which to download the mod archive. If an identifier contains a single slash that is not at the start or end of the string, and otherwise only contains alphanumeric characters, underscores, and dashes, it will be treated as a GitHub repository identifier.

### Internal Identification

Since slugs can change throughout the lifetime of a mod (version changes, mod renames, or user edits), they are not a stable canonical identifier. Internally, `ripperdoc` uses UUIDs as a static, unique, canonical identifier for any given entity (see "Entities" below). Slugs are simply the user-facing identifier.

### Multiple Main Files

Main files will have slugs generated for them from the name provided by Nexus, or from the GitHub repo name (they can also be user-supplied). For mods with multiple main files, if a newly generated slug clashes with an existing slug (because the file names are too similar), the conflict resolution policy will be to append an integer to the slug. For example, say we're installing a secondary main file for a mod with slug `enhanced-cyberware`, that has a main file with slug `main`. If the generated slug for the secondary main file also comes out as `main`, it will be automatically changed to `main-2`. Main file slugs only have to be unique within their mod. For example, two different mods can have a main file with the slug `main`.

A main file can be referenced by appending `=` to the mod slug, and then adding the file slug. Using the example above, the identifier referencing the secondary main file would be `enhanced-cyberware=main-2`.

Rationale: We use `=` because we already use slashes `/` to detect GitHub repo identifiers, and most other non-alphanumeric characters are shell control characters.

TBD: Disambiguate main files using the MD5 hash (available through the v1 API)?

### Optional Files

Optional files will have slugs generated for them from the name provided by Nexus, or from the GitHub repo name (they can also be user-supplied). An optional file is then referenced by appending `@` to the identifier, and then its slug. For example, to reference the optional file `timers-only` from the mod `live-a-little`, the notation would be `live-a-little@timers-only`. Similar to main files, optional file slugs only have to be unique within their mod.

Since main files generally do not own optional files (at least I've never seen any), file identifiers cannot be stacked. For example, `live-a-little=game-2-5@timers-only` is not allowed, although this might change if I do find a case of this happening. However, optional files can be incompatible with certain main files (for example, if a mod has one main file for game v2.31 and another for game v2.5, and the optional file only supports game v2.5), so optional files will have incompatibility rules on them.

## Main File and Optional File Handling

NexusMods distinguishes between main files, which contain the primary functionality of the mod, and optional files, which usually provide add-on functionality. Mod authors can also upload multiple different main files (e.g. to target different game versions), and multiple optional files. Furthermore, optional files can be freestanding (i.e. they can be enabled without enabling any main file). Therefore, `ripperdoc` must account for this. (details TBD)

### Installing

When installing a file and the mod is already installed, `ripperdoc` will ask the user if the file provided is an optional file (maybe use edit distance to only ask if file similarity is below a certain threshold), and proceed accordingly.

## Subcommands

- `install <FILE>` - unpacks a mod archive, pulls its information from the Nexus API, performs dependency resolution, adds the mod files to managed state; if mod is already installed, performs an update instead
  - `--enable/-e` - enables the mod after installing, ignored if updating
  - `--use-nexus-tags/-T` - use the tags specified by Nexus (also a config key `use-nexus-tags`), ignored if updating
  - `--use-nexus-category/-C` - use the category specified by Nexus (also a config key `use-nexus-category`), ignored if updating
  - all options from `mod edit` can also be used here, and can be used while updating.
- `uninstall <MOD ...>` - disables the mod if enabled, then deletes from managed state, optionally recursing to its dependencies
  - `--recursive/-r` - also uninstalls the mod's dependencies that no other mods depend on
  - `--preserve-archive/-a [PATH]` - places the archive file at `PATH`, current directory if unspecified
- `enable <MODS ...>` - checks for incompatibilities and conflicts, then enables it (see section below for what "enable" entails)
  - `--staging-pool/-s` - target the staging pool; if provided, then `MODS` must not be present, required otherwise
  - `--force/-f` - skip dependency and incompatibility checks
- `disable <MODS ...>` - checks for dependencies, then disables the mod (see section below for what "disable" entails)
  - `--staging-pool/s` - target the staging pool; if provided, then `MODS` must not be present, required otherwise
  - `--recursive/r` - also disables the mod's dependencies that no other mods depend on
  - `--force/-f` - skip dependency checks
- `outdated` - pulls updated mod info from the Nexus API, displays mods that need updating
  - Options and subcommands TBD
- `stage <MODS ...>` - add a mod and any disabled dependencies to the staging pool
  - `--no-enable/-x` - do not also enable the mod
  - `--nonrecursive/-n` - do not stage dependencies; if present, implies `-x`
- `unstage [MODS ...]` - remove mods from the staging pool
  - `--all/-a` - unstage all mods and clear the staging pool; if provided, then `MODS` must not be present, required otherwise
- `deploy` - deploy all mods in the staging pool
  - `--unstage-disabled/-d` - unstages all disabled mods in the staging pool before deploying
- `mod <SUBCOMMAND>` - operations on a mod, e.g. updating compatibility info, opening the modpage, etc.
  - `info <MOD>` - show mod info
  - `edit <MOD> <OPTIONS ...>` - edit mod info
    - `--mark-core/-k` - Mark mod as (k)ore
    - `--incompatible-mod/-m <MOD ...>` - Mark mod as incompatible with one or more mods, inverse `--compatible-mod`
    - `--incompatible-game/-g <VER>` - Mark mod as incompatible with a particular game version, inverse `--compatible-game`
    - `--load-after/-a <MOD>` - Configure mod to load its archive files after `MOD`
    - `--load-before/-b <MOD>` - Configure mod to load its archive files before `MOD`
    - `--load-first/-f` - Configure mod to load its archive files before any others
    - `--load-last/-l` - Configure mod to load its archive files after all others
    - `--remove-load-rule/-r [MOD]` - Remove the load constraint with `MOD`. If `MOD` is not specified, removes any absolute load rules (e.g. load first, load last).
    - `--optional <DEP>` - Mark a dependency as optional - warns if specified dependency is a core mod
    - `--nexus-id/-N <ID>` - Add a Nexus mod ID
    - `--github-id/-G <REPO>` - Add a GitHub repo
    - `--slug/-s <SLUG>` - Set the mod's slug identifier
    - `--category/-c [CATEGORY]` - Set the mod's category, unsets the category if no category is provided
    - `--tag/-t` - Add one or more tags to the mod, tags prefixed with `del:` (e.g. `del:realistic`) will remove them
  - `open [OPTIONS]` - Open the mod's Nexus page or GitHub page (prioritizes Nexus, GitHub as fallback)
    - `--github/-g` - Open the mod's GitHub repo page, if any
- `list [OPTIONS]` - lists installed mods
  - `--category/-c CATEGORY` - only lists mods of the given category
  - `--tags/-t TAGS ...` - only lists mods with the given tags
  - `--sort/-s` - sort the list order by a given predicate (TBD)
- `bisect` - starts the bisect process
  - Options and subcommands TBD
- `rollback` - rolls back to a previous mod state
  - Options and subcommands TBD
- `method-scan` - Scans redscript and CET files for method wrappers and replacers
  - `--operation/-o OP` - show only method modifiers corresponding to `OP` (accepted values for `OP` are only `wrap` or `replace`)
  - `--target TARGET` - show only method modifiers which target `TARGET`
  - `--method METHOD` - show only method modifiers which target `METHOD`

## Load Order

Load order is controlled by the `--load-*` flags on the `mod` subcommand. This uses `modlist.txt` (check the modding wiki for the correct name) in `archive/pc/mod` to apply the load order. Redscript and CET (afaik) don't provide a method to control their load order, so load order rules only apply to `.archive` files.

Load order is stored as a list of rules in the mod or file manifest. The final load order is determined using topological sort, and then modlist.txt is written to. Any cycles will cause an error, and the operation will fail.

### REDmod

Support for REDmod extends to supporting the `mods` directory, but load order will remain unsupported for now, since very few mods actually use REDmod, and those that do almost always also supply a non-REDmod version. REDmod load order support will *definitely* come at some point (assuming this tool doesn't get abandoned eventually), but the ETA for now is basically indefinite.

## Tags

Mods can have one or more tags, which can be used to filter or group related mods. Only mods can be tagged; files cannot. Tags follow the same naming rules as slugs: Only lowercase alphanumeric characters and dashes, dashes cannot be trailing or leading.

In a command like `mod edit --tag`, prepending `del:` to a tag (e.g. `del:realistic`) will delete the tag from the mod.

## Mod Management Concepts

### Core Concepts

To install or uninstall a mod does not mean that the mod is installed to the game directory. In this case, installation means that a mod has been placed under the management of this tool. It is similar in concept to something like `chezmoi manage`.

Similarly, to uninstall a mod means that the mod has been removed from the management of this tool, similar to something like `chezmoi forget`.

The operations that actually make a mod active or inactive within the game files are the enable and disable operations. These are what copy the mod files to the game install directory. Unlike general-purpose mod managers, enabling and disabling a mod in `ripperdoc` is dependency and compatibility aware, and can fail if, for example, other mods depend on a mod to be disabled, or if a mod to be enabled is incompatible with a currently-enabled mod.

### Entities

Mods are not the only things that can be enabled and disabled. Individual files (main files and optional files) can be too. An entity is the generic representation of anything that can be enabled or disabled. An entity is either a mod or a file. Files are further split into main files and optional files.

Every entity has three attributes:

- A UUID - the internal, permanent identifier. It never changes, even if the slug does.
- A slug - the user-facing identifier (see "Identifying a Mod").
- A DirTree - the files contained within that entity (see "Architecture").

Every entity supports two core operations: enable and disable. What these do depends on the kind of entity. A mod owns files (entities owned by another entity), so enabling a mod enables all of its owned files. Files to be ignored can be specified.

Ownership is only one level deep: mods own files, and files never own other files. No new kinds of entity are expected beyond mods, main files, and optional files.

#### Exclusive Main

A mod can be marked as exclusive main. Only one main file of an exclusive-main mod can be enabled at once. Enabling a second main file either disables the currently enabled main file, or fails. (Policy TBD, possibly chosen with a CLI flag.)

#### Freestanding

An optional file can be marked as freestanding. A freestanding optional file can be enabled without any main file. Enabling a non-freestanding optional file fails if its main file is not enabled.

### File Ownership and Overwrites

Different mods must never write to the same file. In practice, this does not happen between well-formed mods. If it does, something has gone very wrong. If a mod would overwrite another mod's file, the two mods are immediately marked as incompatible, and the enable fails.

Overwrites should also never happen during an update. An update always disables the mod being updated first, so its files are no longer in the game directory.

Shared directories are normal. Many mods write into the same directories (e.g. `archive/pc/mod`). Only file collisions are a problem.

There are two exceptions:

- Optional files - an optional file can replace a file from its own mod's main file. This is the only legitimate overwrite, and it is rare.
- Bundled dependencies - a mod archive might include a copy of one of its dependencies. If an overwrite would hit a file owned by one of the mod's dependencies, the file is skipped silently. Otherwise, the operation bails.

### Stage and Deploy

While enable and disable (generally) operate on individual mods, there is also the option for a batch-based workflow - stage and deploy. These are operations that build on the core enable and disable operations, and come in handy when testing mod compatibility. Staging works similarly to `git` operations, like `git add` and `git commit`.

The stage and deploy operations revolve around a staging pool. This is a pool of mods that is maintained between invocations, and (as the name suggests) is a staging area, similar to staged files in Git. There is only one staging pool at any given time.

Stage and unstage are individual mod operations. Staging a mod enables it (by default, can be left disabled with `--no-enable`), and adds it to the staging pool. This state persists between invocations, similar to `git` staging. Similarly, unstaging a mod disables it and removes it from the staging pool. `unstage` without an argument unstages all currently staged mods and clears the staging pool. `enable` and `disable` operate on the staging pool when supplied with the `--staging-pool` argument (short form TBD), enabling or disabling all the mods in the pool in one operation. Note that enabled and disabled state is still a mod-specific state, and does not apply to the staging pool as a whole. It is possible for some mods within a staging pool to be disabled, and others to be enabled.

Deploy is a staging pool operation. Conceptually, it merges all staged mods into the main enabled pool. Basically, it clears the staging pool without disabling the staged mods. Deploying by default requires all mods in the staging pool to be enabled, but this can be sidestepped by supplying the `--unstage-disabled` flag, which unstages all disabled mods in the pool before deploying.

### Caveats

While dependencies are pulled automatically from the Nexus API, the requirements section of a modpage is often abused by mod authors to advertise their other mods. This is often seen in Virtual Atelier modpages, where mod authors list their clothing mods whose items are sold by this atelier. Additionally, other dependencies are often optional. As such, when installing a mod, `ripperdoc` will have to ask the user for additional information on dependency optional-ness. To cut down on the number of questions, if a requirement is already installed and enabled, or is a known core mod, it is skipped and automatically marked as a dependency.

Compatibility information is listed on the modpage, and is often human-readable and cannot be parsed by a program. As such, compatibility information will have to be entered manually by the user using a subcommand like `mod`.

## Features

- Non-Nexus support - supports management of mods not installed from Nexus (e.g. RedHotTools)
- Mod install state tracking
  - Tracks enables/disables and deployments, and can roll back to a previous state if needed
  - Can associate install state with a save file, and load the install state associated with that save file
- Core mod awareness
  - Aware of core mods in the ecosystem (e.g. CET, redscript, ArchiveXL, etc.) and accounts for them automatically
  - Mods such as modder's libraries and other modder resources can be marked as core
  - Disregard `cybercmd` requirement - superseded by RED4ext
- Game version awareness
  - Checks if the last update time of a mod was before the release date of the installed game version, and asks for user action
  - When updating a mod, if the last update time is after the release date of the installed game version, auto marks the mod as compatible
- Root directory awareness
  - Able to recognize if the root directory of a mod archive is mergeable with the game install root
  - If not, able to handle some divergent cases
  - Can recognize superfluous files (e.g. item code files, license files) and ignore them automatically
- Dependency resolution - builds a dependency graph, and ensures the state is clean before enabling/disabling a mod
  - Implicit dependency inference - Recognizes file extensions and paths in the mod archive that correspond to core mods
  - Bisection - recognizes dependencies during bisect and partitions the mod pool accordingly
- Incompatibility and conflict detection
  - When enabling a mod, detects if it is incompatible with any currently enabled mods
  - Incompatibility information is symmetric (A incompatible with B => B incompatible with A)

## Architecture

This section covers low-level implementation details. The sections above cover the concepts.

### Entity Representation

Entity can be implemented as an enum or a trait. (TBD)

- Trait - each kind of entity gets its own enable and disable implementation, without a `match` expression.
- Enum - the set of entities is closed (mod, main file, optional file), and files never own other files, so an enum fits. Enums also serialize directly with serde, and can be stored without boxing. Separate implementations are still possible: each variant wraps its own struct with its own `impl`, and the enum delegates to it (manually, or with the `enum_dispatch` crate).

### DirTree

DirTree is a hand-rolled data structure that represents a directory tree. It must work purely in memory (e.g. merge two DirTrees just to see the result), and it must also drive changes on disk.

The main operation of a DirTree is to merge another DirTree. This is the fundamental operation for applying mod files to the game directory. Directories that exist in both trees are merged normally. Files that exist in both trees are a collision (see "File Ownership and Overwrites").

#### Leaf Nodes

A leaf node is any node that has no children, regardless of its type (file or directory), as such, an empty directory is considered a leaf node.

This means that a node's status as a leaf can change throughout an operation. For example, when a directory is freshly created, it is a leaf node, but as soon as subdirectories or files get added in the same operation, it is no longer a leaf node.

### Applying Changes

All filesystem changes are calculated first, then applied all at once. If any operation fails, the applied changes are rolled back. Changes are never calculated while they are being made.

Applying a change happens in three steps:

1. Merge - merge the DirTrees of all enabled entities into a target tree. This is purely in memory.
2. Diff - compare the target tree with the tree that is currently deployed. The result is a list of FileOps (the plan). This is purely in memory.
3. Apply - perform each FileOp in the plan.

DirTree stays a pure data type. The plan is the only link between DirTrees and the disk.

### FileOp

A FileOp is an enum that represents a single file operation (e.g. `CreateDir`, `Copy`, `Overwrite`).

Invariant: only `FileOp::CreateDir` or `FileOp::RemoveDir` can target a directory (the directory must not yet exist in the former case, and must be empty in the latter). All other operations must target normal files. An operation on a directory would have to be broken down recursively into smaller FileOps anyway, so FileOps only ever target leaf nodes (at the time the op runs) of a DirTree. Additionally, `FileOp::RemoveDir` must only operate on directories owned by `ripperdoc`. Attempting to `RemoveDir` a directory that doesn't appear anywhere in `ripperdoc`'s state should fail.

`FileOp::Overwrite` is rare, because overwrites should only happen for optional files (see "File Ownership and Overwrites"). If the diff produces an `Overwrite` whose target file is owned by a dependency of the mod, the file is skipped. Otherwise, the operation bails.

### Open Questions

These came up in brainstorming, but are not decided yet.

- Where enable and disable live - most enable and disable checks look at other entities (is the main file enabled, does anything depend on this). An `entity.enable()` method cannot see other entities unless it is given the whole state. An alternative is `state.enable(uuid)`, with each entity only supplying its own data.
- Where enabled state is stored - storing an enabled flag on both a mod and its files lets them disagree. An alternative is to store the flag on files only, and derive the mod's state (enabled, disabled, or partial).
- Freestanding and exclusive main as graph edges - a non-freestanding optional file could be modelled as a dependency on a main file. Exclusive main could be modelled as mutual incompatibility between main files. This would reuse dependency and incompatibility checks, bisect partitioning, and `--force`.
- Bundled dependency check at merge time - the collision is visible during the merge, before the diff. A "dependency always wins" rule at merge time also works no matter which mod was enabled first. Consider only skipping silently when the file hashes match, and warning when they differ.
- Enforcing the FileOp invariant with types - e.g. separate `DirOp` and `FileOp` enums, or separate `FilePath` and `DirPath` newtypes.
- File leaf metadata - each file leaf could store its source path, its owning entity's UUID, and a content hash. The diff could then compare leaves without reading files from disk.
- Crash recovery - in-memory rollback does not run if the process dies mid-apply. A journal written before the apply and removed after it would allow recovery on the next run.
- Case normalization - Linux is case-sensitive, but mod archives are not consistent about case. Normalizing DirTree keys would make merge and diff handle this.
