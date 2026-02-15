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

### 3. Extract New Module (from Jam) — **End-to-End Pipeline**

**Goal**: Transform a local folder in a project into a reusable submodule hosted in the organization.

**Pipeline**:
1.  **Analyze**: Identify source folder and intended category.
2.  **Prepare**: Ensure `README.md` and structure are correct.
3.  **Publish**: Create a **new repository** in `godot-shared-modules`.
4.  **Link**: Add this new repo as a submodule to `my_godot_toolbox`.
5.  **Replace**: Replace the original folder in the project with the new submodule.

**Steps for Agent:**

1.  **Identify & Validate**:
    - Source: `<source_path>` (e.g., `systems/footstep_system`).
    - Category: `systems`.
    - Check for `components/`, `resources/`, etc.
    - Generate `README.md` if missing.

2.  **Create & Push to Organization**:
    ```powershell
    # 1. Init local repo
    cd <source_path>
    git init
    git add .
    git commit -m "Init module v1.0.0"

    # 2. PROVISION REMOTE (Hardcoded Organization)
    $org = "godot-shared-modules"
    $repoName = "<module_name>" # e.g. footstep_system
    $fullName = "$org/$repoName"

    # Check existence
    if (gh repo view $fullName 2>$null) {
        Write-Warning "Repo $fullName already exists! Aborting to prevent overwrite."
        exit 1
    }

    # Pre-flight: Check permissions & configure git helper to avoid credential prompts
    if (-not (gh auth status 2>&1 | Select-String "Logged in to github.com")) {
        Write-Error "Not logged in to GitHub CLI! Run 'gh auth login'."
        exit 1
    }

    # Configure git to use gh as credential helper (Prevents prompt hang on push)
    gh auth setup-git

    # Create in Org
    Write-Host "Creating $fullName..."
    try {
        # Using --source=. automatically pushes, so we need credentials ready
        gh repo create $fullName --public --source=. --remote=origin
    } catch {
        Write-Error "Failed to create repo in $org. Check permissions or name availability."
        exit 1
    }

    # Push main branch (if gh didn't fully push)
    git push -u origin main
    ```

3.  **Register in Toolbox (The "Registry")**:
    ```powershell
    # 3. Add to Toolbox
    # STRATEGY: Try local path first. If missing/restricted, CLONE to temp.
    # AUTO-BOOTSTRAP: If remote toolbox missing, create it.

    $localToolbox = "d:\Project\Games\my_godot_toolbox"
    $tempToolbox = "temp_toolbox_$(Get-Random)"
    $isTemp = $false

    if (Test-Path $localToolbox) {
        Write-Host "Using local toolbox at $localToolbox"
        cd $localToolbox
    } else {
        Write-Host "Local toolbox not found. checking remote..."
        $user = (gh api user --jq .login)
        $toolboxRepo = "$user/my_godot_toolbox"
        $toolboxUrl = "https://github.com/$user/my_godot_toolbox.git"

        # 3a. Auto-create if missing on GitHub
        if (-not (gh repo view $toolboxRepo 2>$null)) {
            Write-Host "Toolbox repo not found. Creating $toolboxRepo..."
            gh repo create "my_godot_toolbox" --public --add-readme
            Write-Host "Created new Toolbox registry."
        }

        # 3b. Clone (Safe now)
        Write-Host "Cloning to $tempToolbox..."
        git clone $toolboxUrl $tempToolbox
        cd $tempToolbox
        $isTemp = $true

        # 3c. Ensure Folder Structure (Idempotent)
        $dirs = "systems", "ui", "mechanics"
        $newStructure = $false
        foreach ($d in $dirs) {
            if (-not (Test-Path $d)) {
                mkdir $d | Out-Null
                New-Item "$d/.gitkeep" -Force | Out-Null
                $newStructure = $true
            }
        }
        if ($newStructure) {
            git add .
            git commit -m "Init toolbox structure"
            git push
        }
    }

    $remoteUrl = "https://github.com/$org/$repoName.git"
    $targetPath = "<category>/$repoName" # e.g. systems/footstep_system

    mkdir -p <category>
    git submodule add $remoteUrl $targetPath
    git add .
    git commit -m "Register module: $fullName"
    git push

    if ($isTemp) {
        cd ..
        Remove-Item -Recurse -Force $tempToolbox
    }
    ```

4.  **Integrate back to Project**:
    ```powershell
    # 4. Replace local folder with Submodule
    cd <original_project_root>
    Remove-Item -Recurse -Force <source_path>
    git submodule add $remoteUrl <source_path>
    ```

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
