---
description: Manage Godot Toolbox — install, update, extract, and sync reusable modules between projects using git submodules and feature-based structure
---

# Godot Toolbox Management Skill

## Overview

`my_godot_toolbox` is a central registry for reusable Godot modules.
- **Structure**: Feature-based (e.g., `systems/`, `ui/`).
- **Storage**: Each module is a **separate Git repository** hosted in the `godot-shared-modules` organization.
- **Usage**: Modules are added to projects as **Git Submodules**.

**Agent Context**:
- **Toolbox Root**: The root of the `my_godot_toolbox` repo (where `.gitmodules` lives).
- **Organization**: `godot-shared-modules` (Central Hub).
- **User Context**: derived from `gh api user` or local git config.

---

## Core Principles

### 1. Functional Categorization (MANDATORY)

Modules in the toolbox are organized by **Function/Domain**, not by file type or a flat list. The agent must categorize modules into logical folders:

- `systems/` (e.g., inventory, dialogue, quest_system)
- `ui/` (e.g., menu_framework, debug_console)
- `mechanics/` (e.g., platformer_controller, vehicle_physics)
- `tools/` (e.g., level_editor, behavior_tree_editor)
- `components/` (e.g., health_component, hitbox_component)

**✅ CORRECT — Domain-Driven:**
```
my_godot_toolbox/
├── systems/
│   ├── inventory/          # Submodule -> github.com/godot-shared-modules/inventory
│   └── quest_system/       # Submodule -> github.com/godot-shared-modules/quest_system
├── ui/
│   └── menu_framework/     # Submodule
└── README.md
```

**❌ WRONG — Type-Based or Flat:**
`scripts/inventory.gd`, `addons/inventory/` (avoid generic `addons` unless it's an editor plugin)

### 2. Feature-Based Module Design

Every module **MUST** be fully self-contained. It should not depend on project-specific global classes or autoloads unless they are also modules.

**Structure of a Module:**
```
<category>/<module_name>/
├── components/          # Node scripts/scenes (single responsibility)
├── resources/           # Custom Resource classes
├── scenes/              # Pre-built scenes
├── <module_name>.gd     # Facade / Main script
├── plugin.cfg           # Optional (for Editor Plugins)
└── README.md            # Metadata (REQUIRED)
```

### 3. Module README.md (REQUIRED — Single Source of Truth)

Every module **MUST** contain a `README.md` at its root. This replaces `module.json`.

**Template:**

```markdown
# Module Name

One-line description.

## Module Info
- **Category**: systems (e.g., systems, ui, mechanics)
- **Version**: 1.0.0
- **Godot**: 4.3+
- **Tags**: rpg, inventory
- **Dependencies**: ui/menu_framework (or "none")
- **Compatible Games**: 2D, 3D, Both
- **Autoloads**: Manager → manager.gd (or "none")

## Configuration
- `@export var target: Node` — description **(required)**
- `@export var speed: float` — description *(optional)*

## Signals
- `died()` — description

## Structure
- `components/`...
- `resources/`...

## Quick Start
1. Add node...
```

**Parsing Rules:**
- **Version/Tags/Deps/Autoloads**: Parse from `## Module Info`.
- **Configuration/Signals**: Parse for context/setup instructions.

---

## Workflows (Usage Vectors)

### 1. Install Module
**"Install inventory into this project"**

1.  Identify project root.
2.  Read `my_godot_toolbox/.gitmodules` to find the module's remote URL and path.
3.  Ask: **Copy** (stable) or **Submodule** (dev/sync)?
    *   **Destination**: Suggest installing to `addons/<module_name>` (standard) OR matching the toolbox category structure `systems/<module_name>` (if user prefers).
    *   **Copy**:
        ```bash
        git clone --depth 1 <url> <destination_path>
        rm -rf <destination_path>/.git
        ```
    *   **Submodule**:
        ```bash
        git submodule add <url> <destination_path>
        ```
4.  Read `README.md`: Install dependencies recursively. Register autoloads.

### 2. Update Module
**"Update inventory"**

1.  Check `README.md` version.
2.  **Submodule**: `cd <module_path> && git pull origin main`
3.  **Copy**: Re-clone & overwrite (warn first).
4.  Compare new version (Warn if MAJOR change). Update parent repo pointer.

### 3. Extract New Module (from Jam) — **Hardlink Mirror Pipeline**

**Goal**: Transform scattered files in a project into a clean module while keeping them in their original locations for live-sync development.

**Pipeline**:
1.  **Identify**: List all candidate files (scripts, scenes, assets) and their intended subdirectory in the module.
2.  **Consolidate (Mirror)**: Create a mirror folder in `.toolbox_sync/<module_name>/`.
3.  **Hardlink**: Create hard links from the scattered files into the mirror folder.
4.  **Protect Godot**: Add a `.gdignore` file to `.toolbox_sync/` to prevent duplicate asset importing.
5.  **Metadata**: Generate `README.md` and standard module structure inside the mirror.
6.  **Publish**:
    - Push the mirror folder (real files) to `godot-shared-modules`.
    - Register in `my_godot_toolbox`.
    - Replace the `.toolbox_sync/<module_name>` folder with a Git submodule.

**Steps for Agent:**

1.  **Identify & Map**:
    - Build a mapping of `scattered_path` -> `mirror_path`.
    - Example: `scripts/health.gd` -> `.toolbox_sync/health_system/health.gd`

2.  **Initialize Mirror Area**:
    ```powershell
    $mirrorRoot = ".toolbox_sync"
    mkdir -p "$mirrorRoot/<module_name>"

    # Standard Godot ignore for the whole sync area
    if (-not (Test-Path "$mirrorRoot/.gdignore")) {
        New-Item "$mirrorRoot/.gdignore" -Value "*" | Out-Null
    }

    # Ensure project git ignores the sync area
    if (Test-Path ".gitignore") {
        $content = Get-Content ".gitignore"
        if ($content -notcontains ".toolbox_sync/") {
            Add-Content ".gitignore" "`n.toolbox_sync/"
        }
    }
    ```

3.  **Create Hardlinks**:
    The agent must create hardlinks from the scattered project files into the mirror folder.

    **Windows (PowerShell)**:
    ```powershell
    New-Item -ItemType HardLink -Path <mirror_path> -Target <scattered_path> -Force
    ```

    **Windows (CMD)**:
    ```cmd
    mklink /H <mirror_path> <scattered_path>
    ```

    **Linux / macOS / Unix**:
    ```bash
    ln <scattered_path> <mirror_path>
    ```

    *Note: Hardlinks must reside on the same partition/volume as the source files.*

4.  **Publish to Organization**:
    ```powershell
    # 1. Init local repo in the mirror folder
    cd .toolbox_sync/<module_name>
    git init
    git add .
    git commit -m "Init module v1.0.0"

    # 2. PROVISION REMOTE (Hardcoded Organization)
    $org = "godot-shared-modules"
    $repoName = "<module_name>"
    $fullName = "$org/$repoName"

    gh auth setup-git
    gh repo create $fullName --public --source=. --remote=origin
    git push -u origin main
    ```

5.  **Register in Toolbox (The "Registry")**:
    ```powershell
    $localToolbox = "d:\Project\Games\my_godot_toolbox"
    cd $localToolbox
    $remoteUrl = "https://github.com/godot-shared-modules/$repoName.git"
    mkdir -p <category>
    git submodule add $remoteUrl <category>/$repoName
    git add .
    git commit -m "Register module: $fullName"
    git push
    ```

6.  **Consolidate Sync**:
    - The mirror folder is now a separate Git repo (or submodule).
    - You can edit files in the main Godot project, and they change in the mirror instantly.
    - You can `git push` from the mirror to update the shared module.

### 4. Sync Development (Jam)
**"Fix bug in inventory"**

1.  **Check**: Is it a submodule?
2.  **Sync**:
    ```bash
    cd <module_path>
    git checkout main   # CRITICAL: Fix detached HEAD
    git add . && git commit -m "fix" && git push
    cd <project_root>
    git add <module_path> && git commit -m "update ref"
    ```

### 5. List Modules
**"List modules"**
1.  Parse `README.md` of all toolbox submodules.
2.  Table: Category, Name, Version, Description, Tags.

### 6. Status Check
**"Check status"**
1.  Compare local `README.md` versions vs toolbox.
2.  `git status` in submodules for uncommitted changes.

### 7. Remove Module
**"Remove inventory"**
1.  Check dependencies.
2.  **Submodule**: `git submodule deinit`, `git rm`.
3.  **Copy**: `rm -rf`.
4.  Remind to remove Autoloads.

### 8. Scaffold Module
**"Create empty mechanics/healing"**

1.  Create structure: `<category>/<module_name>/`.
2.  Generate `README.md` (template with Category).
3.  Generate `.gitignore` (`*.import`, `.godot/`).
4.  Create starter script.

---

## Common Issues & Solutions
1.  **Repo Creation Hangs**: Run `gh auth setup-git` to use GH CLI as git credential helper.
2.  **Detached HEAD**: Always `git checkout main` in submodule before committing.
3.  **`res://` Paths**: Use relative paths (`preload("icon.png")`). Agent scans/warns.
4.  **`.import` Files**: **INCLUDE** `*.import` files (they contain UIDs). Ignore `.godot/`.
5.  **Dependencies**: Agent recursively installs deps from README.
6.  **Conflicts**: Agent warns on Major version mismatch.
7.  **Binary Assets**: Suggest `git lfs` if >10MB.

## Git Cheatsheet

```bash
# Update all
git submodule update --remote --merge

# Add to toolbox
git submodule add <url> <category>/<name>

# Fix changes
cd <path> && git checkout main && git push
```
