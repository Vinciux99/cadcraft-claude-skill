# CADCraft skill for Claude

CADCraft (ArtCraft suite, storytold/cadcraft) is an open-source, pure-Rust 2D CAD: a **clean-room reimplementation of Autodesk AutoCAD** (not Fusion). It has a command line with prompts and `[Keywords]`, object snaps, layers, dimensions, hatches, blocks, layouts/plotting, parametric constraints and AutoLISP. Native **DXF** (read R12–2018 ASCII and binary), **DWG** read/write via the acadrust library; export PDF, SVG, PNG. It is in **early development** (no 3D, no UCS).

This repository packages a **skill** that teaches Claude to drive CADCraft (part of the [ArtCraft suite](https://github.com/storytold/cadcraft) by storytold) and that **knows how to create the MCP server and guide you through it**, on **Windows and macOS**. It is unofficial and not affiliated with storytold.

> Part of the [ArtCraft Claude skills hub](https://github.com/Vinciux99/artcraft-claude-skills): all 13 skills, a picker and a one-command installer.

## What it does
- Operating manual for CADCraft: connection modes, ground rules, every MCP tool, recipes, CLI, control channel, command families and a safe workflow (work on copies, verify, save only with your OK).
- Built-in MCP setup assistant: Claude checks whether the server is connected, then walks you through installing it (automatic script or manual config), the launcher, authentication and verification.
- Troubleshooting table for the usual problems (server "disconnected", ports, macOS quarantine, Windows Store config location...).

## What you can ask Claude
- "Draw a 100 × 60 rectangle with a 4-unit circle at (10,10) and add a linear dimension along the bottom."
- "Open `plan.dxf` (a copy), list the layers and tell me how many polylines are on `WALLS`."
- "Find the text 'kitchen' in this floor plan and render a close-up of that room."
- "Offset this polyline by 0.25 on the inside and trim the corners."
- "Export the drawing as a PDF on A3 and as a PNG of this window."
- "Convert this DWG to DXF and to SVG."
- "Run this AutoLISP routine that places a bolt-circle of 8 holes."
- "Create a new metric drawing with layers Walls, Doors, Dims and set Walls current."

## Highlights
- Command-line style and JSON-style driving (`command_line`, `script`, `execute`), 15 MCP tools.
- Handles/query/render workflow for reading large drawings without dumping all geometry.
- CLI conversion (`convert`, `run --export`), AutoLISP, plot/PDF export, parametric constraints.
- Safety notes on DWG data loss and never overwriting originals.
- Built-in guidance for creating and troubleshooting the MCP server on Windows and macOS.

## App-specific notes
- Kit port **7982** (upstream docs use 7979); live MCP is `mcp --connect 127.0.0.1:7982 --lazy`; headless `mcp`.
- The control channel has no authentication, loopback only; no read/write roots.
- Early development: 2D only (no 3D, no UCS).

## Get the skill

| Where | How |
|---|---|
| **Claude app** (claude.ai / Desktop) | Download [`cadcraft-skill.zip](https://github.com/Vinciux99/cadcraft-claude-skill/releases/latest/download/cadcraft-skill.zip)`, then Customize > Skills > "+" > Create skill > Upload a skill. Needs Settings > Capabilities > "Code execution and file creation" on. |
| **Claude Code, plugin** | `/plugin marketplace add Vinciux99/artcraft-claude-skills` then `/plugin install cadcraft@artcraft` |
| **Claude Code, manual** | copy `skills/cadcraft/` to `~/.claude/skills/cadcraft/` (the setup script below does it) |

## Create the MCP server

Prerequisite: install CADCraft from [upstream releases](https://github.com/storytold/cadcraft/releases) (Windows `.msi`, macOS `.dmg`) and Claude Desktop and/or Claude Code.

**One command (recommended)**

Windows (PowerShell):
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& ([scriptblock]::Create((irm https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.ps1))) -Apps cadcraft"
```
macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.sh | bash -s -- --apps cadcraft
```
Add `-DryRun` (Windows) or `--dry-run` (macOS) to preview, and `-Mode live|headless|both` / `--mode ...` to choose the mode.

**Or from a clone:** double-click `setup-windows.cmd` (Windows) or `setup-macos.command` (macOS; first time right-click > Open).

**Or just ask Claude:** once the skill is installed, say "set up the CADCraft MCP server" and Claude will guide you step by step.

After setup: fully quit and reopen Claude, open CADCraft with the **"CADCraft (Claude)"** launcher, then ask Claude to use it. If the server shows "disconnected", the app was not running when Claude started: open it with the launcher and re-enable the server, or use headless mode.

What the setup changes (all backed up or reversible): Claude Desktop config (`mcpServers.cadcraft`, backup `<file>.backup-<timestamp>`), the launcher, Claude Code's user-scope MCP list and `~/.claude/skills/cadcraft/`.

## Repository layout
```
skills/cadcraft/SKILL.md      the skill
app.json                   machine-readable settings (ports, paths, MCP args) used by the scripts
scripts/                   artcraft-setup.ps1 (Windows), artcraft-setup.sh (macOS), json-tool.js
setup-windows.cmd          double-click wrapper
setup-macos.command        double-click wrapper
.claude-plugin/plugin.json Claude Code plugin manifest
.github/workflows/ci.yml   validation, setup tests on Windows and macOS, release zip
```

## CI and releases
Every push validates `app.json`, `SKILL.md` and the scripts, then runs the setup scripts against a fake install on **Windows** and **macOS** runners. Pushing a tag `v*` builds `cadcraft-skill.zip` and publishes a GitHub release.

## Status and caveats
- Windows paths and arguments come from a working install. The macOS paths (`/Applications/CADCraft.app`, CLI location and release asset name) follow the upstream packaging but have **not** been verified on a Mac; the skill marks unverified items and the setup script reports what it could not find.
- CADCraft is a young open-source project; behaviour can change between releases. `list_commands` / `cadcraft-cli commands` is always the source of truth.
- Unofficial community packaging, MIT licensed (see `LICENSE`).
