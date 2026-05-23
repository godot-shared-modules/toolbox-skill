# Godot Toolbox Agent Skill

This repository hosts the **Toolbox Management Skill** for AI Agents (Cursor, Windsurf, etc.).
It automates the creation, extraction, and management of Godot modules as Git Submodules.

## 📦 Quick Install

To add this skill to your Godot project, run the command for your OS in your project root:

### Windows (PowerShell)
```powershell
New-Item -ItemType Directory -Force .agent/workflows; Invoke-WebRequest https://raw.githubusercontent.com/godot-shared-modules/toolbox-skill/main/toolbox.md -OutFile .agent/workflows/toolbox.md
```

### Linux / macOS (Bash)
```bash
mkdir -p .agent/workflows && curl -o .agent/workflows/toolbox.md https://raw.githubusercontent.com/godot-shared-modules/toolbox-skill/main/toolbox.md
```

## Features

- **Extract Modules**: Turn a local folder into a shared Git Submodule (`/toolbox extract`).
- **Auto-Bootstrap**: Automatically creates and clones the Toolbox registry if missing.
- **Portable**: Works in any workspace (even without access to the main toolbox folder).

## Usage

Once installed, use the slash command in your agent:
`/toolbox`
