---
name: cadcraft
description: Draw, edit, inspect and export DXF/DWG drawings with CADCraft (AutoCAD-style, ArtCraft suite) through MCP, the CLI or the control channel, and set up, connect or troubleshoot its MCP server on Windows and macOS.
---

# CADCraft for Claude

CADCraft (ArtCraft suite, storytold/cadcraft) is an open-source, pure-Rust 2D CAD: a **clean-room reimplementation of Autodesk AutoCAD** (not Fusion). It has a command line with prompts and `[Keywords]`, object snaps, layers, dimensions, hatches, blocks, layouts/plotting, parametric constraints and AutoLISP. Native **DXF** (read R12–2018 ASCII and binary), **DWG** read/write via the acadrust library; export PDF, SVG, PNG. It is in **early development** (no 3D, no UCS).

**Every action is a command** (`CommandSpec`) with an **id = the AutoCAD command name in lower case** (`line`, `circle`, `offset`, ...) or a dotted id for its own/UI commands (`layer.set`, `ui.language`, `document.bytes`, ...), **JSON parameters**, label, menu, shortcut, aliases and `enabled`. The command line, menus, toolbars, scripts, CLI, control channel and MCP reach the same commands (~295 engine commands + ~30 UI-only). **JSON calls never open dialogs**: they use defaults; only menus/toolbars start prompts.

## Quick check: is the MCP server connected?

1. Look at the tools you have. If tools from the MCP server `cadcraft` are available (for example `list_commands`, `execute` or `inspect_*` tools) the server is connected: skip to "Operating knowledge" below and use them directly.
2. If they are missing, check whether the user installed it under the name `cadcraft` or `cadcraft-headless`. In Claude Desktop the server may show as **disconnected**.
3. **Most common cause of "disconnected":** the live server (`mcp --connect 127.0.0.1:{port} --lazy`) exits when CADCraft is not running at the moment Claude starts. Fix: open CADCraft with the **"CADCraft (Claude)" launcher** (it starts the app with its control channel on), then re-enable the server in Claude's settings or fully quit and reopen Claude. It uses `--lazy`, so the server starts even if the app is not open yet and connects when you use it.
4. If the user does not need to watch the app live, **headless mode** always starts (no window, works on files). See "Live, headless or both".
5. If nothing is configured yet, follow "Set up the MCP server" below. Ask the user which OS they use if you do not know.

## Set up the MCP server (guide the user)

Walk the user through this step by step. Ask for the OS (Windows or macOS) first. If you have a shell on the user's machine you may run the steps yourself, but **ask before editing any config file and always back it up first**. Never print tokens or keys in chat.

### 1. Prerequisites
- Install CADCraft from its upstream releases page: https://github.com/storytold/cadcraft/releases (Windows: the `.msi` installer; macOS: the `.dmg`, then drag the app to `/Applications`).
- Install Claude Desktop and/or Claude Code. For Claude Desktop, fully quit and reopen it after any config change.
- The MCP server is the **CLI** that ships with the app: `cadcraft-cli`. Windows installs it next to the app (`C:\Program Files\CADCraft\cadcraft-cli.exe`, not on `PATH`). On macOS the CLI is a separate release download (asset `cadcraft-cli-<version>-macos-universal.zip`) unless it is bundled in the app (unverified): the setup script below fetches it into `~/.artcraft/bin/` and clears the macOS quarantine flag.
- Upstream docs use port 7979; this kit uses **7982** so that all 13 ArtCraft apps can run together without clashing.

### 2. Option A: automatic setup (recommended)
The setup script finds the app and CLI, creates the launcher, backs up and updates Claude Desktop's config, registers the server in Claude Code (if the `claude` command exists) and installs this skill. Preview first with a dry run: nothing is changed.

Windows (PowerShell):
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& ([scriptblock]::Create((irm https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.ps1))) -Apps cadcraft"
```
macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.sh | bash -s -- --apps cadcraft
```
Or clone/download this repository and double-click `setup-windows.cmd` (Windows) or `setup-macos.command` (macOS; first time: right-click, Open). Useful options: `-Mode live|headless|both` / `--mode ...` (default `live`; `both` registers `cadcraft` live and `cadcraft-headless`), `-DryRun` / `--dry-run`, `-NoLauncher`, `-NoClaudeDesktop`, `-NoClaudeCode`, `-NoSkill` (macOS: `--no-launcher`, `--no-claude-desktop`, `--no-claude-code`, `--no-skill`).

What it changes: Claude Desktop config (`%APPDATA%\Claude\claude_desktop_config.json` plus the Microsoft Store/MSIX copy under `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\`; macOS `~/Library/Application Support/Claude/claude_desktop_config.json`; backup saved next to it as `<file>.backup-<timestamp>`), the launcher "CADCraft (Claude)" (Windows: Desktop shortcut; macOS: `~/Applications/CADCraft (Claude).app` plus a Desktop alias), Claude Code's user-scope MCP list, and `~/.claude/skills/cadcraft/`.

### 3. Option B: manual setup
**Windows** - edit `%APPDATA%\Claude\claude_desktop_config.json` (if Claude was installed from the Microsoft Store, also the copy under `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\`). Keep other servers; add this inside `mcpServers`. Backslashes must be doubled in JSON.
```json
{
  "mcpServers": {
    "cadcraft": {
      "command": "C:\\Program Files\\CADCraft\\cadcraft-cli.exe",
      "args": [
        "mcp",
        "--connect",
        "127.0.0.1:7982",
        "--lazy"
      ]
    },
    "cadcraft-headless": {
      "command": "C:\\Program Files\\CADCraft\\cadcraft-cli.exe",
      "args": [
        "mcp"
      ]
    }
  }
}
```
**macOS** - edit `~/Library/Application Support/Claude/claude_desktop_config.json`. Use the absolute CLI path (replace `YOUR_NAME`; JSON does not expand `~`). The CLI path is unverified if your install differs: check with `ls ~/.artcraft/bin /Applications/CADCraft.app/Contents/MacOS`.
```json
{
  "mcpServers": {
    "cadcraft": {
      "command": "/Users/YOUR_NAME/.artcraft/bin/cadcraft-cli",
      "args": [
        "mcp",
        "--connect",
        "127.0.0.1:7982",
        "--lazy"
      ]
    },
    "cadcraft-headless": {
      "command": "/Users/YOUR_NAME/.artcraft/bin/cadcraft-cli",
      "args": [
        "mcp"
      ]
    }
  }
}
```
**Launcher** (starts the app with the control channel; needed before the live server can connect):
- Windows: create a Desktop shortcut named "CADCraft (Claude)" with target `"C:\Program Files\CADCraft\cadcraft.exe" --control 7982`
- macOS: run `open -n -a "/Applications/CADCraft.app" --args --control 7982`, or save it as an app with Script Editor/`osacompile` (the setup script does this for you).

**Claude Code** (any OS; run once):
Windows (PowerShell):
```powershell
claude mcp add --scope user cadcraft -- "C:\Program Files\CADCraft\cadcraft-cli.exe" mcp --connect 127.0.0.1:7982 --lazy
claude mcp add --scope user cadcraft-headless -- "C:\Program Files\CADCraft\cadcraft-cli.exe" mcp
```
macOS (Terminal):
```sh
claude mcp add --scope user cadcraft -- "$HOME/.artcraft/bin/cadcraft-cli" mcp --connect 127.0.0.1:7982 --lazy
claude mcp add --scope user cadcraft-headless -- "$HOME/.artcraft/bin/cadcraft-cli" mcp
```
Check with `claude mcp list`.

### 4. Authentication
CADCraft needs no token: the control channel listens on loopback (`127.0.0.1`) only and has no authentication. Anyone on the machine can drive the app while it is open, so close it when you are done.

### 5. Finish and verify
1. **Fully quit and reopen Claude** (Desktop: quit from the tray/menu bar, not just close the window).
2. Open CADCraft with the **"CADCraft (Claude)" launcher** before asking Claude to use it.
3. In Claude, confirm the server `cadcraft` is connected and call its `list_commands` tool (or equivalent). A non-empty command list means it works.

### Live, headless or both
- **Live** (`cadcraft`): drives the open app; the user sees every change and screenshots/UI tools work. Needs the launcher first.
- **Headless** (`cadcraft-headless`): in-process engine, no window; always starts; works on files; UI/screenshot tools are not available.
- **Both**: register both and pick per task. Switch with `-Mode` / `--mode`, or by editing the config entries above.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Server shows "disconnected" right after Claude starts | Live server exits when CADCraft is not running | Open CADCraft with the "CADCraft (Claude)" launcher, then re-enable the server or restart Claude; or use headless mode |
| "Port already in use" / app opens but Claude cannot connect | Another app or an old CADCraft instance holds port 7982 | Quit all CADCraft windows, check the launcher uses the kit port, make sure no other ArtCraft app uses the same port |
| `command not found` / spawn error | CLI path wrong or the CLI is not installed (Windows: not on `PATH`; macOS: separate download) | Use the full absolute path in the config; macOS: run the setup script (it downloads the CLI to `~/.artcraft/bin/`) |
| macOS: "cannot be opened because the developer cannot be verified" | Gatekeeper/quarantine on the app or CLI | Right-click the app, Open; for the CLI: `xattr -dr com.apple.quarantine ~/.artcraft/bin/cadcraft-cli` |
| Anyone else can drive the app | The control channel has no authentication (loopback only) | Start the app with the control channel only when needed and quit it when done |
| Config edited but nothing changed (Windows) | Claude installed from the Microsoft Store reads the MSIX copy of the config | Edit/update `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\claude_desktop_config.json` too |
| JSON config error / server not listed | Unescaped backslashes or trailing comma | Double the backslashes on Windows; validate the JSON; restore the `.backup-*` file if needed |
| App crashes on start or window is blank | Graphics backend problem | Windows: set `WGPU_BACKEND=vulkan` (or `dx12`) for one launch; avoid `gl` on some AMD drivers |
| Changes do not appear in the window | You are on the headless server | Use the live server (launcher first) |
| Quotes disappear in PowerShell JSON arguments | Windows PowerShell 5.1 quoting for `.exe` | Use `--script`/files where the CLI supports them, or single-quote the whole JSON |

## Operating knowledge

## Settings, flags, environment
- **App flags documented:** `--control <port>`, `--sample` (opens the sample drawing), `--version`. Nothing else is documented (no `--empty`, `--data-dir`, ...).
- **Settings folder:** Windows `%APPDATA%\CADCraft`; macOS `~/Library/Application Support/CADCraft` (unverified). Contents: `workspace.json` (panel layout), `cadcraft.prefs` (theme/language), `support/` for startup `acad.lsp`/`acaddoc.lsp`. Move it with `CADCRAFT_CONFIG_DIR`.
- **Log:** `<settings>/logs/cadcraft.log` (previous → `cadcraft.1.log`, then `cadcraft.2.log`; max 16 MiB). Attach it to bug reports.

| Variable | Effect |
|---|---|
| `CADCRAFT_CONTROL_PORT` | Same as `--control <port>`; `--control <port>` wins, a bare `--control` uses the variable's port |
| `CADCRAFT_CONFIG_DIR` | Alternative settings/log folder (isolated profile); empty = no `workspace.json` reads/writes |
| `RUST_LOG` | Log levels (default `info` for CADCraft, `warn` the rest); `cadcraft_engine=debug` logs every command-line input |
| `WGPU_BACKEND` | Graphics backend; on Windows the default is DirectX 12 only. `vulkan` for troubleshooting; avoid `gl` (AMD driver crash `atio6axx.dll`); `metal` on macOS |
| `CADCRAFT_SYSTEM_CURSOR=1` | Show the system cursor over the crosshair (magnifier, screen recordings) |
| `CADCRAFT_FONTALT` / `CADCRAFT_FONTFALLBACK` | Substitute / fallback font (like `SETVAR FONTALT` / `FONTFALLBACK`) |
| `CADCRAFT_LOCALE` | Force the system language for one run (e.g. `en-GB`); a saved choice wins |

If the app flashes in Task Manager and closes (Windows), from PowerShell in the install folder: `$env:WGPU_BACKEND = "vulkan"; & .\cadcraft.exe`, then `Remove-Item Env:WGPU_BACKEND`.

The **"CADCraft (Claude)"** launcher starts the app with `--control 7982`. It is the normal way to open the app when an agent has to drive it. (The installer's regular "CADCraft" shortcut does not open the channel.)

## Connecting

| Mode | Start | What it drives |
|---|---|---|
| MCP headless | `cadcraft-cli mcp` | An in-process drawing session, no window |
| MCP attached to the app | app with `--control 7982`, then `cadcraft-cli mcp --connect 127.0.0.1:7982` (+ `--lazy`) | The drawing open in the app, live; enables `screenshot`, `ui_inspect`, `ui_click` |
| CLI | `cadcraft-cli info/convert/run/commands/sample/perf ...` | One file per call, headless (does not talk to the app) |
| Control channel | `cadcraft --control 7982` (or `CADCRAFT_CONTROL_PORT=7982`) | JSON lines over TCP `127.0.0.1:7982` |

- **Port:** upstream docs use **7979**; this kit uses **7982** to avoid colliding with VectorCraft (7979).
- **Authentication:** **none** documented (no token, no token file). The channel listens only on `127.0.0.1`, but anyone on the machine who opens the port controls the app.
- **Read/write roots:** **none** documented. `open`/`save`/`--save`/`--export` use any path the user can reach: choose carefully.
- **`--connect` is strict:** without `HOST:PORT` (or with an unknown argument) `cadcraft-cli mcp` exits with an error instead of falling back to headless; if the app does not answer, startup fails. With **`--lazy`** the bridge should start anyway and connect on the first call (in practice a registered server has still been seen to exit when the app was off). **`--launch`** also opens the app, but the docs describe it **only for macOS** (unverified here).

### Choosing the channel
1. **A CADCraft MCP server is available** → use it (tools below).
2. **A shell on the user's machine** → `cadcraft-cli` with its full path for file work; for the open app, JSON lines on port 7982 (see "Control channel").
3. **A cloud sandbox** cannot run the user's app or reach its ports; it can only prepare files (`.scr` scripts, `.lsp` routines, command files).
4. **Last resort: computer use on the GUI.** Type into the command line at the bottom (`line`, `circle`, `zoom e`, ...): more reliable than icons. Non-US keyboards: shortcuts with `[` `]` may misfire. `Cmd` in the docs is `Ctrl` on Windows.

## Ground rules
- **Units and coordinates:** f64 in **drawing units** ("world units"), all 2D in the **WCS** (no UCS, no 3D). `new_drawing {}` creates an **imperial** drawing by default; `{"metric": true}` (CLI `run --metric`) for metric. Command-line input: absolute `x,y`, relative `@dx,dy`, polar `@d<a`, `#x,y`, direct distance; also `5'6-1/2"`, `45d30'`, `N45dE`. Paper, margins and plot offsets are in **mm**; plot PNG resolution in dpi (50–2400).
- **Object references:** every entity has a hexadecimal string **handle** (e.g. `"10A"`), returned by `inspect_drawing {"entities": true}`, `query_entities`, `find_text`. JSON commands take it in `handles` (e.g. `move {"handles":["10A"],"delta":[5,0]}`, `plotstyle {name, handles?}`). Alternatively `select {"window":[[x0,y0],[x1,y1]]}` then a command acting on the selection. Layers, styles, blocks, layouts are referenced by **name**.
- **Undo:** each command/routine is one step; undo = whole-drawing snapshot (cheap), max 2,000. `UNDO` supports only the count (no Mark/Back/BEgin/End). Use `command_line {"text":"u"}` or `undo` (AutoCAD names; check `list_commands {"filter":"undo"}`). An entire AutoLISP routine is one step.
- **Parameters:** MCP validates type and presence of its tools' arguments (`-32602`: `command_line {}` never presses Enter by accident). **Extra properties are not rejected** by the MCP schema; whether a command's `params` are strict or tolerant is not documented: read the command's `params doc` in `list_commands`.
- **Errors:** a failing command gives `isError: true` (MCP) or `ok:false` (control). But a rejected line in `cmdline.input` answers **`"ok": true` with `error`**: always check `error`.
- **Batch / `$N`:** there is no batch tool and no `$N` references to earlier results. For multiple steps: `script` (multi-line `.scr` style), several `--cmd` in the CLI, or separate calls re-reading handles with `query_entities`.
- **Open prompts:** after `command_line` look at `prompt`, `running`, `keywords`, `accept`. If a command is left half-done, `cancel` (Esc) before continuing.
- **Long jobs:** the channel waits for the app up to **60 s**; then `timeout` with `state`: `not-run` (can be resent) or `may-have-run` (first `drawing.inspect`, then possibly repeat). A long AutoLISP loop runs in slices per frame; Esc stops it.
- **Known traps:** saving **DWG loses** whatever CADCraft does not model (custom entities, some xdata, color books): never overwrite the original DWG. SHX fonts are not read (text with its own stroke fonts has different metrics). Ellipses/splines are tessellated in intersections/offset. Save format: the file-format-parity doc says DXF/DWG 2000–2018 (2018 default, `saveas` with `version` and `binary`), the README still says "DXF write R2000": check the saved file. No editable dynamic blocks, no REFEDIT on xrefs. `ui.screenshot` needs a presented frame (locked screen → use `render`/`ui.render`).

## MCP tools
15 tools: 12 in both modes, the last 3 only when attached to the app.

| Tool | Arguments / notes |
|---|---|
| `command_line` | `{text}` (required). Types a line like a person: starts commands by name/alias and answers prompts; spaces act as Enter, empty text = Enter. Returns prompt, keywords, output, `error` |
| `script` | `{text}` multi-line `.scr` style (blank line = Enter) |
| `execute` | `{command, params?}` any command with JSON, never dialogs (→ `engine.execute`) |
| `list_commands` | `{filter?}` substring of id, label or alias → id, label, menu, alias, params doc |
| `inspect_drawing` | `{entities?: bool, limit?}` counts, extents, layers, styles, blocks, layouts, selection, undo history, entities |
| `query_entities` | `{type?, layer?, window?, crossing?, limit?, offset?}`; `type` e.g. `Line`, `Circle`, `Polyline`; `window` `[[x0,y0],[x1,y1]]` inside or crossing (`crossing:false` = fully inside only); `count` total, `returned` page (default 500). It is the `entities` command |
| `render` | `{width?, height?, fit?, window?}` PNG as an image; the whole drawing, or only `window` (1600 px wide with the window's shape if unsized); min 16 px |
| `find_text` | `{contains?, layer?, window?, limit?, offset?}` readable texts (TEXT, MTEXT without codes, attributes, multileaders) with `at`, height, rotation, style, layer, handle; default 200. Read-only (the `texts` command) |
| `new_drawing` | `{metric?: bool}` new drawing, imperial by default (the `new` command) |
| `open` | `{path}` open a drawing (→ `app.open`) |
| `save` | `{path?}` saves DXF or DWG by extension; `.svg`/`.png`/`.pdf` **export**: the drawing keeps its name and unsaved changes |
| `cancel` | `{}` = Esc: cancels the running command or clears the selection |
| `screenshot` | `{path?}` PNG of the window (app only) |
| `ui_inspect` | `{}` UI state: panels, canvas, view, cursor, snaps (app only) |
| `ui_click` | `{x, y, space?, button?}` click in the **drawing in world coordinates** (`space:"screen"` for screen points); right button = Enter (→ `ui.pointer`, app only) |

Resources: `cadcraft://drawing` (inspect JSON) and `cadcraft://commands` (catalog). No MCP prompts documented. The other channel methods (`ui.set`, `ui.elements`, `ui.key`, ...) have no MCP tool: use the control channel, or `execute` for UI commands (`ui.*`).

## Recipes

**Typical session (MCP):**
```text
command_line {"text": "rectang 0,0 100,60"}
execute {"command": "circle", "params": {"center": [10, 10], "radius": 4}}
command_line {"text": "dimlinear 0,0 100,0 50,-8"}
inspect_drawing {}            → check counts and geometry
render {}                     → look at the result
```

**Command line like a draftsperson:** `line 0,0 @10,0 @0,5 c` (closed triangle), `circle 5,2 1`, `offset 0.25` (then pick object and side), `zoom e`.

**JSON (examples for the `execute` tool):** `line {"points":[[0,0],[10,0]]}`, `circle {"center":[0,0],"radius":5}`, `layer.new {"name":"Walls","color":"red","current":true}`, `move {"handles":["10A"],"delta":[5,0]}`, `zoom {"mode":"extents"}`, `style {"name":"FA","font":"Vazirmatn"}` (text style with an installed TTF).

**Reading a big floor plan without dumping all geometry:** `find_text {"contains": "kitchen"}` → take `at` → `render {"window": [[x0,y0],[x1,y1]]}` on that room → `query_entities {"window": [[x0,y0],[x1,y1]], "layer": "..."}` for handles.

**Framing a selection:** `select {"window":[[0,0],[20000,15000]]}` then `zoom {"mode":"object"}`.

**Export without touching the original:** `save {"path": "/Users/me/Drawings/sheet.pdf"}` exports and leaves the drawing as is. To plot from a layout: `pagesetup {layout, device, paper?}` (e.g. `paper: "A3"`), `plot {device?, paper?, ...}` / `exportpdf` (options: `plotStyleTable`, `plotTransparency`, `center`, `offset` in mm). The output-path parameter of `plot`/`exportpdf` is not documented: look it up in `list_commands {"filter":"plot"}`.

**AutoLISP:** `execute {"command":"lisp","params":{"code":"(command \"circle\" '(0 0) 5)"}}`; `inputs` answers `get...` calls in order (numbers, strings, points `[x,y]`). Load a file with `appload {"path": ".../routine.lsp"}` or `{name, text}`. Reply `{value, output, error?}`.

**App language/theme (UI commands):** `ui.language {"lang":"en"}`, `ui.theme {"theme":"dark"}`, `ui.animations {"animations":"off"}`. Channel replies stay in English.

## CLI
`cadcraft-cli` is headless: it works on files, not on the open app (except `mcp --connect`).
```text
cadcraft-cli info FILE.dxf                                 JSON summary
cadcraft-cli convert IN.dxf OUT.(dxf|dwg|svg|png|pdf) [FRAMING]
cadcraft-cli run [FILE | --sample | --metric] [--script TEXT] [--script-file F.scr]
                 [--cmd 'id {json}']... [--save OUT.dxf] [--export OUT.(png|svg|pdf)] [FRAMING]
cadcraft-cli commands [FILTER]                             command catalog (JSON)
cadcraft-cli mcp [--connect HOST:PORT [--lazy] [--launch]]
cadcraft-cli sample (bracket|floorplan) OUT.(dxf|dwg|svg|png)
cadcraft-cli perf [N | FILE]                               timings
cadcraft-cli --version
FRAMING: --window x0,y0,x1,y1 (drawing units, lower-left corner first) | --view extents|current
```
- Images: PNG 2400×1600, SVG, PDF, always model space. `convert` fits the extents; `run --export` shows the current view if the script/commands changed it (`zoom`, `pan`, `view.set`, ...), otherwise the extents. `--window` and `--view` are alternatives.
- In `--script` the sequence `\n` becomes a newline. Unknown options → error.
- A reader that closes stdout early (`| head`) does not stop `--save`/`--export`; the exit status still reports failures.
```sh
cadcraft-cli info ~/Drawings/plan.dxf
cadcraft-cli convert plan.dwg detail.png --window 0,0,20000,15000
cadcraft-cli run plan.dwg --cmd 'zoom {"mode":"window","p1":[0,0],"p2":[20000,15000]}' --export detail.png
cadcraft-cli run --sample --script 'CIRCLE 22,3 1\n' --save out.dxf --export out.png
cadcraft-cli commands dim                                  # commands containing "dim"
```
(Windows PowerShell: `& "C:\Program Files\CADCraft\cadcraft-cli.exe" ...`; single quotes keep the JSON quotes of `--cmd` intact.)

## Control channel
- **Transport:** TCP on `127.0.0.1:<port>` (here 7982), **JSON lines**: one request per line, one response per line. No authentication. Server-side timeout 60 s (see `state` above).
- **Format:** `{"id":1,"method":"cmdline.input","params":{"text":"circle 0,0 5"}}` → `{"id":1,"ok":true,"result":{"prompt":"Command:","output":[...],...}}`; error `{"id":...,"ok":false,"error":"..."}`. An unknown method points to `control.methods`.
- **Methods:** `cmdline.input {text}`, `cmdline.script {text}`, `cmdline.key {key:"enter"|"escape"}`, `cmdline.state`, `engine.execute {command, params}` (alias `command`), `engine.commands`, `drawing.inspect {entities?, limit?}`, `ui.inspect` (also `closePrompt`: the drawing of an open "Save changes?", and `focus.id/label/role`), `ui.elements {label?, role?, focused?}`, `ui.menu.list` / `ui.menu.invoke {command}`, `ui.pointer {x, y, space?, button?, action?}`, `ui.click` / `ui.move {x, y, button?, shift?, ctrl?}` (screen points), `ui.drag {x, y, to, button?}`, `ui.key {key, cmd?, shift?, alt?}` / `ui.text {text}`, `ui.set {...}` (palette, toolbar, `theme`, `animations`, `saveFormat`, `saveVersion`, `interfaceLanguage`, `systemCursor`), `ui.resize {width, height}`, `ui.focus`, `ui.screenshot {path?}`, `ui.render {path?, width?, height?, fit?}` → `{pngBase64, width, height, bytes, path?}`, `app.open {path}` / `app.save` / `app.quit`, `control.methods`, `ping` → `"pong"`.
- **Catalog commands** are called **only** via `engine.execute`: `{"id":4,"method":"engine.execute","params":{"command":"open","params":{"path":"..."}}}`. `open` or `document.bytes` as a top-level `method` give `unknown method`. `document.bytes` returns the DXF in base64 in the `data` field.
- **UI elements:** `ui.elements` gives the real egui `id`, label, role, rectangle `[x,y,w,h]`, state; filters are exact. `ui.key` only queues: query `ui.inspect`/`ui.elements` before the next action. Docking panels: `toolSets`, `layers`, `properties`, `externalReferences` (protected center `canvas`) via `engine.execute` `ui.dock {operation, panel, ...}` and `ui.dock.reset`.
- **From PowerShell** (generic TCP client):
```powershell
$c = [System.Net.Sockets.TcpClient]::new("127.0.0.1", 7982)
$s = $c.GetStream(); $w = [IO.StreamWriter]::new($s); $w.AutoFlush = $true; $r = [IO.StreamReader]::new($s)
$w.WriteLine('{"id":1,"method":"ping","params":{}}'); $r.ReadLine()
$w.WriteLine('{"id":2,"method":"cmdline.input","params":{"text":"zoom e"}}'); $r.ReadLine()
$c.Close()
```

## Commands by family
Id = AutoCAD name in lower case (AGENTS.md rule); those listed come from the docs. For parameters and everything else: `list_commands {"filter": "..."}` / `cadcraft-cli commands <filter>`.

- **Drawing:** `line`, `pline`, `circle`, `arc`, `rectang`, `polygon`, `ellipse`, `spline`, `point`, `xline`, `ray`, `donut`, `text`, `mtext`, `revcloud`, `wipeout`.
- **Modify:** `erase`, `move`, `copy`, `rotate`, `scale`, `mirror`, `stretch`, `offset`, `trim`, `extend`, `fillet`, `chamfer`, `break`, `join`, `explode`, `array` (+ `arrayedit`, `arrayclose`, `arraypath`), `overkill`.
- **Annotation:** `dimlinear` and the other `dim*`, `dimstyle`, `mleader` (+ `mleaderedit`, `mleaderalign`, `mleadercollect`), `table`, `style`, `texts` (read texts).
- **Hatches and blocks:** `hatch`, `block`, `insert`, `refedit`/`refset`/`refclose`, `xattach`, `xbind`, `xopen`, `blocks.dynamic`, `blocks.reference`, `blocks.reference.set`, `blocks.reference.grips`, `blocks.reference.grip`.
- **Layers and properties:** `layer.new {name, color, current}`, `layer.set {name, plotStyle}`, `plotstyle`, `setvar` (also by typing the variable name as a command).
- **View and selection:** `zoom {mode: "extents"|"window"(p1,p2)|"object"}`, `pan`, `view.set`, `select {window}`, `selectsimilar`, `isolateobjects`/`hideobjects`/`unisolateobjects`, `isodraft`, `isoplane`.
- **Parametric:** `autoconstrain`, `parameters`, `constraintinfer`, `constraints.inspect`, `constraints.dof`.
- **Files, layouts, plotting:** `new`, `open`, `saveas {version, binary}`, `document.bytes`, `drawing.inspect`, `entities`, `pagesetup`, `plot`, `exportpdf`, `preview`, `stylesmanager {action}`, `plottermanager {action}`, `audit`, `recover`.
- **AutoLISP:** `lisp {code, inputs?}`, `appload {path} | {name, text | data}`.
- **UI (app only):** `ui.language`, `ui.theme`, `ui.animations`, `ui.saveformat`, `ui.saveversion`, `ui.dock`, `ui.dock.reset`, `ui.panel.quickcalc`.
- **Undo:** `u`/`undo`, `redo`, `mredo` (check exact ids with the filter).

## Recommended workflow
1. **Orient:** `ping` / `ui_inspect` (app) or `inspect_drawing {}` (headless): extents, units, layers, layouts, any running command (`cmdline.state`). If a prompt is open, `cancel`.
2. **Open or create on a copy:** copy the user's DXF/DWG first (e.g. `plan_copy.dxf`) and open that; for new drawings `new_drawing {"metric": true}` if the user works in mm/m.
3. **Find commands:** `list_commands {"filter": "..."}` for ids and params doc; do not guess JSON keys.
4. **Edit in small steps:** one command per call (`execute` or `command_line`), check `error`/`isError`, re-read with `query_entities` / `find_text` / `inspect_drawing`.
5. **Verify visually:** `render` (also with `window`) or `screenshot` with the app; if wrong, `u`/`undo` and retry.
6. **Save/export only with the user's OK**, always under a new name; never overwrite the original (especially DWG, which loses unmodeled data). A `.pdf/.png/.svg` export via `save` does not change the drawing file.
7. **Close when done:** the channel has no authentication, so close the app opened with the "CADCraft (Claude)" launcher (or `app.quit`, checking `closePrompt` in `ui.inspect`) so port 7982 does not stay open.
