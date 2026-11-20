# Development Notes

## Terminology

There are some concepts referred to by Nexus that `ripperdoc` refers to by another name. These are listed below, and will be referred to as such throughout this file.

- **Component** - what Nexus calls a file (main or optional). This is to distinguish them from individual mod files (e.g. .archive, .reds), which `ripperdoc` also needs to individually understand.
- **Package** - the archive file (.zip, .7z, .rar) downloaded as a mod file (as referred to by Nexus). This is to distinguish them from *game* archive files, ones that have a `.archive` file extension.

## Intended Usage

`ripperdoc` is intended for use primarily in an interactive shell session. Installing a mod usually requires more information than is usually provided at first invocation, so that information has to be supplied by prompting the user for it; thus, an interactive shell session is necessary. An interactive shell session also allows the tool to give feedback directly to the user (e.g. informing of automatic slug updates).

Nevertheless, it is still possible to use `ripperdoc` in a scripting context, and the command-line arguments needed to supply all required information at invocation do exist. However, `ripperdoc` will not be as flexible in that context, and a lot of conveniences will be lost.

### Modes

`ripperdoc` can operate in two modes:

- Interactive mode. This is the interactive shell session, where the tool can ask the user questions and get an immediate answer.
- Scripting mode. This is used when `ripperdoc` detects that stdin is not a TTY. In this mode, if required information is missing, the tool will fail with a "missing information" error.

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

Some mod authors make the source code of their mod available on GitHub, and provide it as another avenue from which to download the mod package. If an identifier contains a single slash that is not at the start or end of the string, and otherwise only contains alphanumeric characters, underscores, and dashes, it will be treated as a GitHub repository identifier.

### Internal Identification

Since slugs can change throughout the lifetime of a mod (version changes, mod renames, or user edits), they are not a stable canonical identifier. Internally, `ripperdoc` uses UUIDs as a static, unique, canonical identifier for any given entity (see "Entities" below). Slugs are simply the user-facing identifier.

## Mod Components

What `ripperdoc` calls "components" are what NexusMods calls "files". These are basically the parts that constitute the mod as a whole, and are shipped by Nexus as individual packages. Generally, a component corresponds to a single package. As on Nexus, there are two kinds of components: main components and optional components.

### Main Components

Main components are the primary component of a mod, corresponding to the "Main Files" section, and are usually the only component.

These components will have slugs generated for them from the name provided by Nexus, or from the GitHub repo name (they can also be user-supplied). For mods with multiple main components, if a newly generated slug clashes with an existing slug (because the component names are too similar), the conflict resolution policy will be to append an integer to the slug. For example, say we're installing a secondary main component for a mod with slug `enhanced-cyberware`, that has a main component with slug `main`. If the generated slug for the secondary main component also comes out as `main`, it will be automatically changed to `main-2`. Main component slugs only have to be unique within their mod. For example, two different mods can have a main component with the slug `main`.

A main component can be referenced by appending `=` to the mod slug, and then adding the component slug. Using the example above, the identifier referencing the secondary main component would be `enhanced-cyberware=main-2`.

Rationale: We use `=` because we already use slashes `/` to detect GitHub repo identifiers, and most other non-alphanumeric characters are shell control characters.

TBD: Disambiguate main components using the MD5 hash (available through the v1 API)?

### Optional Components

Optional components correspond to the "Optional Files" section. These are components that are not necessary for the mod to function correctly (hence "optional", duh).

These components will also have slugs generated for them from the name provided by Nexus, or from the GitHub repo name (they can also be user-supplied). An optional component is then referenced by appending `@` to the identifier, and then its slug. For example, to reference the optional component `timers-only` from the mod `live-a-little`, the notation would be `live-a-little@timers-only`. Similar to main components, optional component slugs only have to be unique within their mod.

Since main components generally do not own optional ones (at least I've never seen any), component identifiers cannot be stacked. For example, `live-a-little=game-2-5@timers-only` is not allowed, although this might change if I do find a case of this happening. However, optional components can be incompatible with certain main components (for example, if a mod has one main component for game v2.31 and another for game v2.5, and the optional component only supports game v2.5), so optional components will support incompatibility rules.

### Suppression

Components of a mod can be suppressed. This is basically a mod-level disable, and suppressed components will not be enabled when their parent mod is enabled. As always, when suppressing a component, `ripperdoc` performs dependency resolution, and additionally suppresses any components depending on the component being suppressed.

Enabled components cannot be suppressed, and `ripperdoc` will fail if suppression of an enabled component is attempted. This ensures that `enable` and `disable` are the final gatekeepers of consistency; suppression just builds on that.

### Component Handling

NexusMods distinguishes between main files (components), which contain the primary functionality of the mod, and optional files (components), which usually provide add-on functionality. Mod authors can also upload multiple different main components (e.g. to target different game versions), and multiple optional components. Furthermore, optional components can be freestanding (i.e. they can be enabled without enabling any main components). Therefore, `ripperdoc` must account for this. (details TBD)

#### Installing

When installing a component and the mod is already installed, `ripperdoc` will ask the user if the package provided is for an optional component (maybe use edit distance to only ask if package file name similarity is below a certain threshold, detection algorithm TBD), and proceed accordingly (implementation TBD).

## Subcommands

Where `TARG` is used, it means the command can target any entity (see "Entities" below). Where `MOD` is used, mods are the only valid target entity.

### `config`

**Syntax:** `config [OPTIONS]`

Edits `ripperdoc`'s configuration.

**Options and Subcommands:**

TBD

### `install`

**Syntax:** `install [OPTIONS] <PACKAGE>`

Extracts a package, pulls mod and component information from the Nexus API, resolves dependencies, adds the mod files to managed state, optionally enabling it as well. If mod is already installed, performs an update instead (Definition of update TBD).

Automatic unpacking relies on the package's directory structure matching what `ripperdoc` expects (see Features). If `ripperdoc` cannot recognize the directory structure, the user can extract the package manually into a directory with the correct structure, and run `install` pointing at this directory. `ripperdoc` will detect this automatically. Since `ripperdoc` pulls information based on what it can gather from the package filename, installing this way means the user has to provide information that `ripperdoc` can use to pull what it needs. This can be done with the options from `mod edit`, or the `--package` option.

**Options and Subcommands:**

- `--package/-p <PATH>` - if `PACKAGE` is a directory (manual extraction), uses `PATH` as the source package file (todo: elaborate)
- `--enable/-e` - enables the mod after installing, ignored if updating
- `--use-nexus-tags/-T` - use the tags specified by Nexus (also a config key `use-nexus-tags`), ignored if updating
- `--use-nexus-category/-C` - use the category specified by Nexus (also a config key `use-nexus-category`), ignored if updating
- all options from `mod edit` can also be used here, and can be used while updating.

### `uninstall`

**Syntax:** `uninstall [OPTIONS] <TARG ...>`

Disables the target(s) if enabled, then deletes from managed state, optionally recursing to its dependencies.

**Options and Subcommands:**

- `--recursive/-r` - also uninstalls the target's dependencies that no other entities depend on
- `--preserve-package/-p [PATH]` - places the package file at `PATH`, current directory if unspecified

### `enable`

**Syntax:** `enable [OPTIONS] [TARG ...]`

Checks for incompatibilities and conflicts, performs dependency resolution, then enables the target and its dependencies (see Core Concepts for definition of "enable").

**Options and Subcommands:**

- `--staging-pool/-s` - target the staging pool; if provided, then `TARG` must not be present, required otherwise
- `--force/-f` - skip dependency and incompatibility checks

### `disable`

**Syntax:** `disable [OPTIONS] [TARG ...]`

Checks dependencies, then disables the target (see Core Concepts for definition of "disable").

**Options and Subcommands:**

- `--staging-pool/-s` - target the staging pool; if provided, then `TARG` must not be present, required otherwise
- `--recursive/-r` - also disables the target's dependencies that no other mods depend on
- `--force/-f` - skip dependency checks

### `outdated`

**Syntax:** `outdated [OPTIONS]`

Pulls latest mod info from the Nexus API and displays mods that need updating.

**Options and Subcommands:**

- `--since/-d <DATE>` - instead show mods that were updated after the (d)ate `DATE` (date format TBD).

### `stage`

**Syntax:** `stage [OPTIONS] <TARG ...>`

Adds the specified targets and any disabled dependencies to the staging pool, and then enable it (by default).

**Options and Subcommands:**

- `--no-enable/-x` - do not also enable the targets
- `--nonrecursive/-n` - do not stage dependencies; if present, implies `-x`

### `unstage`

**Syntax:** `unstage [OPTIONS] [TARG ...]`

Removes the specified targets and any eligible (definition TBD) dependencies from the staging pool, and then disables them (by default).

**Options and Subcommands:**

- `--no-disable/-x` - do not also disable the targets
- `--all/-a` - unstage all targets and clear the staging pool; if provided, then `TARG` must not be present, required otherwise
- Option to skip dependencies TBD

### `deploy`

**Syntax:** `deploy [OPTIONS]`

Deploy all targets in the staging pool (see Core Concepts for definition of "deploy").

**Options and Subcommands:**

- `--unstage-disabled/-d` - unstages all (d)isabled targets in the staging pool before deploying

### `mod`

**Syntax:** `mod <SUBCOMMAND>`

Performs operations on a target, e.g. updating compatibility info, opening the modpage, etc.

**Options and Subcommands:**

- `info <TARG>` - show target info
- `edit <TARG> <OPTIONS ...>` - edit target info (validity of `TARG` depends on options specified)
  - `--slug/-s <SLUG>` - Set the target's slug identifier
  - `--nexus-id/-N <ID>` - Add a Nexus mod ID (`MOD` only)
  - `--github-id/-G <REPO>` - Add a GitHub repo (`MOD` only)
  - `--author/-A <NAME>` - Set the mod author name. Author name is usually inferred from GitHub repo name, or pulled from the Nexus API.
  - `--category/-c [CATEGORY]` - Set the mod's category, unsets the category if no category is provided (`MOD` only)
  - `--tag/-t <TAG ...>` - Add one or more tags to the mod, tags prefixed with `del:` (e.g. `del:realistic`) will remove them (`MOD` only)
  - `--mark-core/-k` - Mark mod as (k)ore (`MOD` only)
  - `--dependency/-d <TARG ...>` - Add one or more dependencies. Prepend `TARG` with `del:` to remove it as a dependency instead.
  - `--optional/-o <DEP>` - Mark a dependency as optional - warns if specified dependency is a core mod
  - `--incompatible/-x <TARG ...>` - Mark mod as incompatible with one or more mods, prefix `TARG` with `del:` to unmark
  - `--incompatible-game/-g <VER>` - Mark mod as incompatible with a particular game version, prefix `VER` with `del:` to unmark
  - `--load-after/-a <TARG>` - Configure mod to load its archive files after `TARG`
  - `--load-before/-b <TARG>` - Configure mod to load its archive files before `TARG`
  - `--load-first/-f` - Configure mod to load its archive files before any others
  - `--load-last/-l` - Configure mod to load its archive files after all others
  - `--remove-load-rule/-r [TARG]` - Remove the load constraint with `TARG`. If `TARG` is not specified, removes any absolute load rules (e.g. load first, load last).
- `suppress <COMPONENT>` - suppresses the specified component.
- `unsuppress <COMPONENT>` - unsuppresses the specified component.
- `open <MOD> [OPTIONS]` - Open the mod's Nexus page or GitHub page (prioritizes Nexus, GitHub as fallback)
  - `--github/-g` - Open the mod's GitHub repo page, if any

### `list`

**Syntax:** `list [OPTIONS]`

Lists installed mods.

**Options and Subcommands:**

- `--category/-c <CATEGORY>` - only lists mods of the given category
- `--enabled/-e` - only list enabled mods.
- `--disabled/-d` - only list disabled mods.
- `--tags/-t <TAGS> ...` - only lists mods with the given tags
- `--sort/-s <PRED>` - sort the list order by a given predicate. Prepend `rev:` to the predicate (e.g. `rev:name`) to reverse the order. Valid predicates are:
  - `name` - sort by slugs' alphabetical order.
  - `author` - sort by mod author name in alphabetical order, slug alphabetical order within author clusters.
  - `category` - sort by category alphabetical order, slug alphabetical order within categories, uncategorized mods last.
  - `last-updated` - sort by date of last update.
  - `install-date` - sort by date of install.

### `bisect`

**Syntax:** TBD

Controls the bisection process.

**Options and Subcommands:**

TBD

### `snapshot`

**Syntax:** `snapshot <SUBCOMMANMD>`

Manages snapshots, exact details TBD.

**Options and Subcommands:**

- `take` - Take a snapshot (Full syntax TBD)
- `delete` - Delete a snapshot (Full syntax TBD)
- `apply` - Applies a snapshot (Full syntax TBD)

### `doctor`

**Syntax:** `doctor [OPTIONS]`

Validates the current mod state, prints any inconsistencies and errors found, and suggests fixes. In interactive mode, the user can select which fixes to apply. In scripting mode, it simply prints out the inconsistencies and errors, and exits.

**Options and Subcommands:**

- `--apply-all/-a` - automatically applies all suggested fixes. Useful for scripting mode.
- TBD

### `load-order`

**Syntax:** TBD

Lists or edits the load order of archive files and their corresponding entities.

**Options and Subcommands:**

TBD

### `method-scan`

**Syntax:** `method-scan [OPTIONS]`

Scans redscript and CET files for engine method wrappers and replacers.

**Options and Subcommands:**

- `--operation/-o <OP>` - show only method modifiers corresponding to `OP` (accepted values for `OP` are only `wrap` or `replace`)
- `--class/-c <CLASS>` - show only method modifiers which target `CLASS`
- `--method/-m <METHOD>` - show only method modifiers which target `METHOD`
- `--wraps/-w <ARG>` - Shorthand convenience option (TODO)
- `--replaces/-r <ARG>` - Shorthand convenience option

## Load Order

*Important note about managing load order:* `mod` can only manage load order down to the component level. Managing load order at the file level is done using the `load-order` subcommand.

Load order is controlled by the `--load-*` flags on the `mod` subcommand, which specifies load constraints on entities. This uses `modlist.txt` (check the modding wiki for the correct name) in `archive/pc/mod` to apply the load order. Redscript and CET (afaik) don't provide a method to control their load order, so load order rules only apply to `.archive` files.

There are two kinds of load constraints: relative and absolute. Relative constraints specify an order relative to another target, and are specified using `--load-before` and `--load-after`. Absolute constraints specify an absolute load order (very first or very last), and are specified using `--load-first` and `--load-last`. Constraints can be removed using the `--remove-load-rule` option, which removes any relative load constraints to a specified target, or removes any absolute constraint if no target is specified.

Load order is stored as a list of rules in the mod or component manifest. The final load order is determined using topological sort, and then modlist.txt is written to. Any cycles will cause an error, and the operation will fail. Additionally, any attempt to make a target load after its dependencies should also be rejected (first to load wins conflicts).

One invariant that is naturally enforced by the disallowing of cycles is that only one absolute constraint can exist on a mod at any given time. We do not need to explicitly check for this, as cycle detection will catch it automatically, given the correct encoding. However, we can, as a convenience to the user, catch it in the specific case where the user is adding the opposite absolute constraint on a target that already has one (e.g. `mod edit some-mod --load-last` where `some-mod` already has load first), and offer to automatically delete the existing absolute constraint.

### Mod Splitting

Since modlist.txt handles load order at the individual archive file level, it is possible to split archive files from the same mod across a load order (e.g. load `modA-1.archive`, then `modB.archive`, then `modA-2.archive`). This can be managed with the `load-order` subcommand.

### REDmod

Support for REDmod extends to supporting the `mods` directory, but load order will remain unsupported for now, since very few mods actually use REDmod, and those that do almost always also supply a non-REDmod version. REDmod load order support will *definitely* come at some point (assuming this tool doesn't get abandoned eventually), but the ETA for now is basically indefinite.

## Tags

Mods can have one or more tags, which can be used to filter or group related mods. Only mods can be tagged; components cannot. Tags follow the same naming rules as slugs: Only lowercase alphanumeric characters and dashes, dashes cannot be trailing or leading.

In a command like `mod edit --tag`, prepending `del:` to a tag (e.g. `del:realistic`) will delete the tag from the mod.

## Core Concepts

### Management

To install or uninstall a mod does not mean that the mod is installed to the game directory. In this case, installation means that a mod has been placed under the management of this tool. It is similar in concept to something like `chezmoi manage`.

Similarly, to uninstall a mod means that the mod has been removed from the management of this tool, similar to something like `chezmoi forget`.

The operations that actually make a mod active or inactive within the game files are the enable and disable operations. These are what copy the mod files to the game install directory. Unlike general-purpose mod managers, enabling and disabling a mod in `ripperdoc` is dependency and compatibility aware, and can fail if, for example, other mods depend on a mod to be disabled, or if a mod to be enabled is incompatible with a currently-enabled mod.

### Entities

Mods are not the only things that can be enabled and disabled. Individual components (main and optional components) can be too. An entity is the generic representation of anything that can be enabled or disabled. An entity is either a mod or a component. Components are further split into main and optional components.

Every entity has three attributes:

- A UUID - the internal, permanent identifier. It never changes, even if the slug does.
- A slug - the user-facing identifier (see "Identifying a Mod").
- A DirTree - the files contained within that entity (see "Architecture").

Every entity supports two core operations: enable and disable. What these do depends on the kind of entity. A mod owns components (entities owned by another entity), so enabling a mod enables all of its owned, unsuppressed components.

Ownership (at the entity level) is only one level deep: mods own components, and components never own other components (but do own files - see below). No new kinds of entity are expected beyond mods, main components, and optional components.

#### Exclusive Main

A mod can be marked as exclusive main. Only one main component of an exclusive-main mod can be enabled at once. Enabling a second main component either disables the currently enabled main component, or fails. (Policy TBD, possibly chosen with a CLI flag.)

#### Freestanding

An optional component can be marked as freestanding. A freestanding optional component can be enabled without any main component. Enabling a non-freestanding optional component fails if its main component is not enabled.

### Files

A file is an individual file with an extension (e.g. .archive, .reds, .lua). Files are owned by components, and form permanent leaf nodes in a DirTree. Important to note is that files are **not** entities; they cannot be individually enabled or disabled; they do not have a UUID, or a slug, or a DirTree. The only attribute that files share with entities is load order, and even then that is only defined for archive files.

Since load order can be specified at the archive file level, `ripperdoc` can track load order at the file level. Load order is the only attribute that exists on both entities and files. A load order attribute on a mod applies to all its components as a whole; a load order attribute on a component applies to all its archive files as a whole; a load order attribute on an archive file applies to it only. (TBD: load order conflicts between levels)

`ripperdoc` can recognize a file type (e.g. Archive file, Redscript code, CET Lua code) by its extension and path. For example YAML files in `r6/tweaks` are recognized as Tweak files that require TweakXL. This file recognition is used to infer core dependencies (e.g. a mod containing Tweak files requires TweakXL; a mod containing Lua files in `bin/x64/plugins/cyber_engine_tweaks/mods` requires CET), among other things (exactly what is still TBD).

### File Ownership and Overwrites

Different mods must never write to the same file. In practice, this does not happen between well-formed mods. If it does, something has gone very wrong. If a mod would overwrite another mod's file, the two mods are immediately marked as incompatible, and the enable fails.

Overwrites should also never happen during an update. An update always disables the mod being updated first, so its files are no longer in the game directory.

Shared directories are normal. Many mods write into the same directories (e.g. `archive/pc/mod`). Only file collisions are a problem.

There are two exceptions:

- Optional components - an optional component can replace a file from its own mod's main component. This is the only legitimate overwrite, and it is rare.
- Bundled dependencies - a mod package might include a copy of one of its dependencies. If an overwrite would hit a file owned by one of the mod's dependencies, the file is skipped silently. Otherwise, the operation bails.

### Stage and Deploy

While enable and disable (generally) operate on individual entities, there is also the option for a batch-based workflow - stage and deploy. These are operations that build on the core enable and disable operations, and come in handy when testing mod compatibility. Staging works similarly to `git` operations, like `git add` and `git commit`.

The stage and deploy operations revolve around a staging pool. This is a pool of entities that is maintained between invocations, and (as the name suggests) is a staging area, similar to staged files in Git. There is only one staging pool at any given time.

Stage and unstage are individual entity operations. Staging an entity enables it (by default, can be left disabled with `--no-enable`), and adds it to the staging pool. This state persists between invocations, similar to `git` staging. Similarly, unstaging an entity disables it and removes it from the staging pool. `unstage --all` unstages all currently staged entities and clears the staging pool. `enable` and `disable` operate on the staging pool when supplied with the `--staging-pool` argument, enabling or disabling all the entities in the pool in one operation. Note that enabled and disabled state is still an entity-specific state, and does not apply to the staging pool as a whole. It is possible for some entities within a staging pool to be disabled, and others to be enabled.

Deploy is a staging pool operation. Conceptually, it merges all staged entities into the main enabled pool. Basically, it clears the staging pool without disabling the staged entities. Deploying by default requires all entities in the staging pool to be enabled, but this can be sidestepped by supplying the `--unstage-disabled` flag, which unstages all disabled entities in the pool before deploying.

### Caveats

While dependencies are pulled automatically from the Nexus API, the requirements section of a modpage is often abused by mod authors to advertise their other mods. This is often seen in Virtual Atelier modpages, where mod authors list their clothing mods whose items are sold by this atelier. Additionally, other dependencies are often optional. As such, when installing a mod, `ripperdoc` will have to ask the user for additional information on dependency optional-ness. To cut down on the number of questions, if a requirement is already installed and enabled, or is a known core mod, it is skipped and automatically marked as a dependency.

Compatibility information is listed on the modpage, and is often human-readable and cannot be parsed by a program. As such, compatibility information will have to be entered manually by the user using the `mod` subcommand.

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
  - Able to recognize if the root directory of a mod package is mergeable with the game install root
  - If not, able to handle some divergent cases
  - Can recognize superfluous files (e.g. item code files, license files) and ignore them automatically
- Dependency resolution - builds a dependency graph, and ensures the state is clean before enabling/disabling a mod
  - Implicit dependency inference - Recognizes file extensions and paths in the mod package that correspond to core mods
  - Bisection - recognizes dependencies during bisect and partitions the mod pool accordingly
- Incompatibility and conflict detection
  - When enabling a mod, detects if it is incompatible with any currently enabled mods
  - Incompatibility information is symmetric (A incompatible with B => B incompatible with A)

## Architecture

This section covers low-level implementation details. The sections above cover the concepts.

### Entity Representation

Entity can be implemented as an enum or a trait. (TBD)

- Trait - each kind of entity gets its own enable and disable implementation, without a `match` expression.
- Enum - the set of entities is closed (mod, main component, optional component), and components never own other components, so an enum fits. Enums also serialize directly with serde, and can be stored without boxing. Separate implementations are still possible: each variant wraps its own struct with its own `impl`, and the enum delegates to it (manually, or with the `enum_dispatch` crate).

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

`FileOp::Overwrite` is rare, because overwrites should only happen for optional components (see "File Ownership and Overwrites"). If the diff produces an `Overwrite` whose target file is owned by a dependency of the mod, the file is skipped. Otherwise, the operation bails.

### Open Questions

These came up in brainstorming, but are not decided yet.

- Where enable and disable live - most enable and disable checks look at other entities (is the main component enabled, does anything depend on this). An `entity.enable()` method cannot see other entities unless it is given the whole state. An alternative is `state.enable(uuid)`, with each entity only supplying its own data.
- Where enabled state is stored - storing an enabled flag on both a mod and its components lets them disagree. An alternative is to store the flag on components only, and derive the mod's state (enabled, disabled, or partial).
- Freestanding and exclusive main as graph edges - a non-freestanding optional component could be modelled as a dependency on a main component. Exclusive main could be modelled as mutual incompatibility between main components. This would reuse dependency and incompatibility checks, bisect partitioning, and `--force`.
- Bundled dependency check at merge time - the collision is visible during the merge, before the diff. A "dependency always wins" rule at merge time also works no matter which mod was enabled first. Consider only skipping silently when the file hashes match, and warning when they differ.
- Enforcing the FileOp invariant with types - e.g. separate `DirOp` and `FileOp` enums, or separate `FilePath` and `DirPath` newtypes.
- File leaf metadata - each file leaf could store its source path, its owning entity's UUID, and a content hash. The diff could then compare leaves without reading files from disk.
- Crash recovery - in-memory rollback does not run if the process dies mid-apply. A journal written before the apply and removed after it would allow recovery on the next run.
- Case normalization - Linux is case-sensitive, but mod packages are not consistent about case. Normalizing DirTree keys would make merge and diff handle this.
