<div align="center">

# ⚒️ FORGE

**One prompt in. A finished shot out.**

Unified MCP orchestrator for **Blender**, **Unreal Engine 5**, and **Adobe Premiere Pro** —
model, render, and edit from a single natural-language prompt.

[![CI](https://github.com/YOUR_USERNAME/forge/actions/workflows/ci.yml/badge.svg)](https://github.com/YOUR_USERNAME/forge/actions)
![Tests](https://img.shields.io/badge/tests-36%2F36%20passing-brightgreen)
![Python](https://img.shields.io/badge/python-3.11%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)

</div>

---

Say this to Claude:

> *"Build a campfire at night in a forest clearing — warm flickering light, embers drifting up. Render a 4-second loop at 1080p, then import it into Premiere and assemble it with a gentle fade-in."*

...and walk away. FORGE models the scene in Blender, renders it (or hands it to Unreal for a photoreal pass), watches the disk until the frames land, imports them into Premiere, applies the edit, and drops the final file in your delivery folder with a manifest. No app-switching. No dragging files between programs. No babysitting render progress bars.

## Why this exists

Blender, Unreal, and Premiere each have excellent MCP servers — but they don't know about each other. The moment your idea spans two apps ("render this, *then* edit it"), you're back to being the human glue: waiting for renders, hunting for output folders, importing by hand. FORGE is that glue, automated. It doesn't replace the app MCPs — it **conducts** them.

```
Claude Desktop / Claude Code
        │  MCP stdio
        ▼
forge_server.py  (FastMCP orchestrator)
  router · state store · render watcher (watchdog)
        │
        ├── adapters/blender.py   → blender-mcp addon           TCP :9876
        ├── adapters/unreal.py    → UE Remote Control (native)  HTTP :30010
        └── adapters/premiere.py  → premiere-pro-mcp      MCP stdio + CEP bridge
        ▼
D:/forge/renders/   ← shared pipeline folder (the handoff point)
  blender/  ue5/  premiere/  finals/
```

The trick that makes it work: **the filesystem is the contract.** Every render lands in a per-job subfolder of one shared pipeline directory. A watchdog-based watcher notices when files stop growing (renderers write incrementally), signals the orchestrator, and the next step fires with `"$latest"` automatically pointing at those fresh frames.

---

## 🚀 Quickstart

### 1. Install FORGE

```bash
git clone https://github.com/YOUR_USERNAME/forge.git
cd forge
pip install -r requirements.txt        # fastmcp, watchdog, pyyaml
cp forge.example.yaml forge.yaml       # then edit paths/ports if needed
```

Default pipeline folder is `D:/forge/renders/` — change `pipeline.render_output` in `forge.yaml` if you want it elsewhere.

### 2. Connect the apps (only the ones you'll use — FORGE runs with any subset)

**🟠 Blender**
Install the [blender-mcp](https://github.com/ahujasid/blender-mcp) addon (`addon.py`) in Blender ≥ 3.0, then in the 3D View sidebar (press `N`): **BlenderMCP → Connect to MCP server**. It listens on port 9876.

**🔵 Unreal Engine 5** *(optional — but its Lumen renders are fast and gorgeous)*
No third-party plugin needed. In your project, enable two stock Epic plugins — **Remote Control API** and **Python Editor Script Plugin** — restart, then press `` ` `` and run:
```
WebControl.StartServer
```
(Or enable auto-start in *Project Settings → Web Remote Control*.) Keep it on localhost — that's Epic's own advice. Community MCP plugins on port 55557 also work via `unreal.transport: tcp` in `forge.yaml`.

**🟣 Premiere Pro** *(optional)*
```bash
npm i -g premiere-pro-mcp
```
Install its CEP panel (`%APPDATA%\Adobe\CEP\extensions\MCPBridgeCEP` on Windows), then in Premiere: **Window → Extensions → MCP Bridge → Start Bridge**.

> **No Node.js? No problem.** Set `premiere.mode: jsx` in `forge.yaml` and FORGE writes ready-to-run ExtendScript import files instead — run them via *File → Scripts* in Premiere.

### 3. Verify everything

```bash
python forge_server.py doctor
```

Every failing check prints exactly what to fix:

```
FORGE doctor — Phase 0 connectivity check

[ok] pipeline folder: D:/forge/renders
[ok] Blender MCP addon reachable on localhost:9876
[--] Unreal Engine (Remote Control) NOT reachable on localhost:30010
     -> In UE: enable 'Remote Control API' + 'Python Editor Script Plugin'...
[ok] Premiere adapter in jsx fallback mode (writes import scripts)
```

---

## 🎬 Three ways to drive it

### A. `forge_desktop.py` — recommended (your Claude subscription, no API key)

Uses Claude Code headless as the brain — same account as your Claude app, zero extra billing.

```bash
npm install -g @anthropic-ai/claude-code
claude                      # log in once, then exit
python forge_desktop.py
```

It asks two questions — a delivery folder, and *what to make* — then streams Claude's progress live and sweeps the finished files (plus a `forge_manifest.json`) into your folder when done. Only FORGE's own tools are pre-approved (`--allowedTools mcp__forge`); nothing else runs unattended.

### B. Claude Desktop — conversational mode

Merge `claude_desktop_config.example.json` into your `claude_desktop_config.json` (fix the paths), restart Claude Desktop, and FORGE's tools appear. Now you can iterate like you're talking to a technical director:

> "Render a preview frame of the open Blender scene."
> "Nice — bump the samples, make it 4K, and this time do the full animation."
> "Import everything into Premiere on track V2 and show me what the watcher has seen."

### C. `forge_run.py` — Anthropic API mode, with a safety net

```bash
pip install anthropic
set ANTHROPIC_API_KEY=sk-ant-...
python forge_run.py          # add --yes to skip approvals
```

Same prompt-to-deliverables loop, but **every Blender/Unreal code block is shown to you before it runs** — press Enter to approve, `a` to auto-approve the rest, `s` to skip a step. Great for watching how the agent thinks the first few times.

---

## ✨ Creative recipes

Real prompts, and what FORGE actually does with them.

### 🔥 The one-prompt short film
> *"Create a lighthouse on a rocky cliff at dusk, stormy sky, waves below. Render a 6-second establishing shot at 1080p/24fps, then a close-up still of the lantern. Import both into Premiere, put the wide shot first, and export an MP4 into finals."*

Under the hood: FORGE builds geometry → materials → lighting → camera in Blender in small verified chunks (checking `get_scene_info` between stages), renders the animation to `renders/blender/job_<id>/`, the watcher waits for all 144 frames to stabilize, then chains `"$latest"` straight into the Premiere import and assembly.

### ⚡ The Blender → Unreal photoreal handoff
> *"Model a low-poly desert temple in Blender, export it as FBX, import it into Unreal, place it in the level with a cine camera slow push-in, set up Lumen lighting with warm directional sun and height fog, and render the sequence at 4K/30."*

This is FORGE's signature move: `blender.command (execute_code)` → FBX export into the pipeline folder → `ue5.command (import_fbx)` → UE Python spawns the asset, builds a LevelSequence and lighting → Movie Render Queue fires → the watcher catches the PNG sequence as MRQ writes it. Blender for modeling speed, Unreal for final-pixel beauty.

### 📱 The vertical social cut
> *"Render the current scene as a 5-second vertical (1080×1920) loop and drop it into Premiere in a bin called 'Reels'."*

`resolution: "vertical"` is a built-in preset (so are `1080p`, `4k`, `720p`, and any `WIDTHxHEIGHT`). Perfect for turning one 3D scene into feed-ready content without touching an export dialog.

### 🎨 The iteration loop (Claude Desktop mode)
> "Give me a quick 720p preview frame." → *look at it* → "Warmer light, camera lower, more fog." → "Good — now the full animation at 1080p while I get coffee."

`render_scene` returns a `job_id` immediately for long renders; poll `get_render_status` whenever you're curious. Nothing blocks the conversation.

### 🤖 The fully autonomous batch (forge_desktop.py)
> *"Create three different product-shot environments for a matte black water bottle — studio softbox, sunlit concrete, and neon night. One 3-second turntable each at 1080p. Assemble all three into one sequence with cuts, and deliver the MP4."*

The agent works through all three scenes, falls back gracefully if an app is down (UE offline → finishes in Blender Cycles and says so; Premiere in jsx mode → the import scripts become part of the deliverable), and **always** ends by calling `deliver_files` — even after partial failure, you get whatever exists plus notes.

### 🧩 Power-user mode: hand-written pipelines
Skip natural language entirely and pass explicit steps to `run_pipeline`:

```json
[
  {"app": "blender", "action": "render",
   "params": {"resolution": "1080p", "animation": true,
              "frame_start": 1, "frame_end": 96, "fps": 24}},
  {"app": "premiere", "action": "import",
   "params": {"files": "$latest", "bin": "Campfire", "sequence": "Main", "track": 2}}
]
```

Render outputs chain into the next step automatically via `"$latest"`. Any of premiere-pro-mcp's 269 tools stay reachable through the `premiere.tool` passthrough — transitions, color, exports, captions, all of it.

---

## 🧰 Tool reference (what Claude sees)

| Tool | What it does |
|---|---|
| `render_scene` | Render in Blender (open scene) or UE5 (`level_sequence=/Game/...`). Returns a `job_id`; long renders run in the background. |
| `get_render_status` | Job status + output paths so far. |
| `run_pipeline` | Ordered multi-app steps; outputs chain via `"$latest"`. |
| `get_pipeline_status` | Live connectivity of all three apps + recent jobs. |
| `import_render` | Push the latest (or named) renders into Premiere. |
| `list_render_outputs` | Everything the watcher has seen, newest first. |
| `set_output_path` | Repoint the pipeline folder (jailed to `pipeline.allowed_root`). |
| `deliver_files` | Copy finished files into the session's delivery folder, with a manifest. |

## ⚙️ Configuration at a glance (`forge.yaml`)

```yaml
pipeline:
  render_output: D:/forge/renders/   # the shared handoff folder
  default_format: PNG                # EXR | PNG | MP4
unreal:
  transport: remote_control          # Epic-native (recommended) | tcp
  port: 30010                        # 30010 remote_control, 55557 tcp
premiere:
  mode: mcp                          # mcp | jsx (offline import scripts)
watcher:
  stabilize_delay_ms: 1000           # wait after a file stops growing
  force_polling: false               # true for network drives
```

## 🔒 Safety posture

FORGE lets an LLM drive real applications on your machine, so the boundaries are strict:

- **No raw string interpolation into app code. Ever.** Blender/UE scripts are generated only from range-checked ints, allowlisted enums, and JSON-escaped paths — hostile inputs like `/Game/Shot; rm -rf /` are rejected before any network call (and there are tests proving it).
- Premiere imports, `set_output_path`, and `deliver_files` are **jailed to the pipeline folder** — the agent cannot read or ship files from anywhere else on your disk.
- Router step **allowlist**, 25-step pipeline limit, 64 MB response cap, timeouts everywhere.
- Unreal Remote Control stays bound to **localhost** (Epic's own guidance). Never expose ports 9876 / 30010 / 55557 beyond your machine.
- JSX fallback scripts JSON-escape every value — no ExtendScript breakout.

## ✅ Tests

```bash
python -m pytest tests/ -q     # 36 tests
```

The end-to-end suite runs the **real adapters and real watcher** against fake app servers that speak the actual wire protocols (blender-mcp TCP JSON, UE Remote Control HTTP) — including hostile-input injection tests and a scripted full agent run that ends in a delivery manifest.

## 🩹 Troubleshooting

Every error message tells you its own fix — that's a design feature — but the common ones:

| Symptom | Fix |
|---|---|
| `cannot reach localhost:9876` | In Blender's sidebar: BlenderMCP → Connect to MCP server |
| `cannot reach Remote Control on :30010` | In UE console: `WebControl.StartServer` |
| `could not start premiere-pro-mcp` | `npm i -g premiere-pro-mcp`, or set `premiere.mode: jsx` |
| Render finished but no files detected | Confirm all apps write to the same `render_output` path |
| Watching a NAS/network drive | Set `watcher.force_polling: true` |

## 🗺️ Roadmap

**v0.2** — logging layer, coverage in CI, real-hardware demo recording · **v0.3** — job persistence across restarts, render progress estimation · **v0.4** — parallel steps, pipeline presets, DaVinci Resolve adapter · **v0.5** — PyPI release (`pipx install forge-orchestrator`).

## 🤝 Contributing

FORGE is built spec-first — read [`PLAN.md`](PLAN.md) for architecture and [`CLAUDE.md`](CLAUDE.md) for the workflow, then see [`CONTRIBUTING.md`](CONTRIBUTING.md). Every change ships with a test; `pytest` green + `bandit` clean are the merge gates.

## 📜 License & credits

MIT — see [LICENSE](LICENSE).

FORGE stands on excellent open-source shoulders: [blender-mcp](https://github.com/ahujasid/blender-mcp) by Siddharth Ahuja (MIT), [premiere-pro-mcp](https://github.com/leancoderkavy/premiere-pro-mcp), [FastMCP](https://github.com/PrefectHQ/fastmcp), and [watchdog](https://github.com/gorakhargosh/watchdog). FORGE drives them as sub-clients over their public interfaces — it bundles none of their code.

*FORGE is an independent open-source project. Blender, Unreal Engine, and Adobe Premiere Pro are trademarks of their respective owners and are not affiliated with this project.*
