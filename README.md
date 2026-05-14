---

![https://raw.githubusercontent.com/hrabanazviking/Seidr-Smidja/refs/heads/development/2D66H.jpg](https://raw.githubusercontent.com/hrabanazviking/Seidr-Smidja/refs/heads/development/2D66H.jpg)

---

# Seiðr-Smiðja

> *An agent-only VRM avatar smithy. No human hands. Only the fire, the spec, and the eye that sees.*

![Status: Genesis Phase — Vertical Slice Not Yet Forged](https://img.shields.io/badge/status-genesis%20phase-darkred)
![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue)
![License: TBD](https://img.shields.io/badge/license-TBD-lightgrey)
![Branch: development](https://img.shields.io/badge/branch-development-green)

> **Status banner (2026-05-06, current):** Genesis + Hardening + **Brúarhönd v0.1** all complete. **446 non-Blender, non-VRoid-host pytest tests passing, 3 skipped, 42 Blender/VRoid deselected.** 10 ADRs accepted (D-001..D-010). Three full Mythic Engineering rituals closed in one evening. The forge is built, hardened, and now grew a hand long enough to reach across Tailscale into a VRoid Studio session on a different machine. The "Genesis Phase" badge above is preserved for historical record per the additive-only rule.

---

## What Is This?

**Seiðr-Smiðja** (Seething-Forge) is a headless, programmatic forge that lets AI agents — Hermes, OpenClaw, Claude Code, and others not yet named — design, build, render, critique, and export fully realized VRM avatars without any human clicking through a GUI.

An agent submits a YAML specification describing an avatar: height, proportions, hair, outfit, expressions, license metadata. The forge loads a VRoid Studio base template, opens Blender headlessly, applies every parametric choice from the spec, validates the result against VRChat and VTube Studio compliance rules, renders a set of preview PNG images, and returns the finished `.vrm` file and all renders to the calling agent.

The agent sees what it has made. It critiques the renders. It revises the spec. It submits again. This is the forge cycle. It runs entirely without human hands inside it.

**Three things to understand:**

First, the forge is built around a *vision feedback loop*. Every build that produces a `.vrm` also produces rendered preview images through the Oracle Eye — front view, three-quarter, side, face close-up, T-pose, signature expressions. An agent that cannot see its creation cannot refine it. The renders are not optional.

Second, every output is compliance-checked before delivery. If an avatar fails VRChat polygon budgets or VTube Studio blendshape requirements, the forge returns a structured failure report — never a silent pass. A blade that cannot cut has not been made.

Third, the forge speaks through four doors — MCP (Mjöll), CLI (Rúnstafr), REST (Straumur), and skill manifests for Hermes/OpenClaw/Claude Code. All four doors lead to the same fire. What works through one works through all.

---

## What It Is NOT

- **Not a human GUI tool.** There is no user interface for clicking. All access is programmatic and agent-driven. If you want to use this forge as a human, you do so through the CLI or by writing a script — not through a graphical application.
- **Not a generic 3D pipeline.** Seiðr-Smiðja is purpose-built for VRM avatars using VRoid Studio base meshes and the VRM Add-on for Blender. It will not export arbitrary 3D formats, process non-VRM meshes, or serve as a general Blender automation layer.
- **Not for non-VRM avatars.** The Gate validates VRChat and VTube Studio compliance. Everything about the forge's design — the spec schema, the base mesh strategy, the compliance rules — is shaped around the VRM ecosystem. Other avatar formats are out of scope.

---

![https://raw.githubusercontent.com/hrabanazviking/Seidr-Smidja/refs/heads/development/QRAO5.jpg](https://raw.githubusercontent.com/hrabanazviking/Seidr-Smidja/refs/heads/development/QRAO5.jpg)

---

## Two Capabilities — The Headless Forge and the Reaching Hand

Seiðr-Smiðja exposes two complementary modes for AI agents:

**1. The Headless Forge (Genesis v0.1)** — agent submits a YAML Loom spec, the forge loads a VRoid base from the Hoard, runs Blender headlessly to apply parametric changes, validates the result at the Gate, renders preview images through the Oracle Eye, and returns a fully VRChat-ready and VTube-Studio-ready `.vrm` plus all renders. No human touches anything. This is the original capability — Mode B in the dispatch model.

**2. The Reaching Hand — Brúarhönd v0.1** — for the operations the headless forge cannot reach (live VRoid Studio session manipulation, manual asset editing, GUI-only flows), Seiðr-Smiðja can drive a real VRoid Studio session running on the same machine OR on a different machine connected via **Tailscale**. A small daemon (Horfunarþjónn) listens on the VRoid host; a client (Hengilherðir) in the forge process speaks to it over HTTPS through Tailscale's encrypted overlay. Bearer-token authentication plus Tailscale ACL provides defense in depth. The agent drives screenshots, clicks, types, hotkeys, and the high-level VRoid menu actions (open project, save project, export VRM). Every action is followed (when the agent wishes) by a screenshot returned through the same Oracle Eye vision channel, so the agent sees what its hand has done — Mode A in the dispatch model.

Both modes can run in the same dispatch — Mode C — where a single `run_id` correlates the two arms across the two-Annáll telemetry topology.

---

## Setup and Operation (Authoritative Guide — 2026-05-06)

> *This section is the current authoritative setup and operation reference for v0.1 of Seiðr-Smiðja. The older "Quickstart for an AI Agent" section below this one is preserved for historical record per the additive-only rule, but several of its claims (e.g. "Phase 5 — coming") are now superseded by the live implementation documented here.*

### 1. Install the Base Package

Requires Python 3.10 or newer. Blender (with the [VRM Add-on for Blender by saturday06](https://github.com/saturday06/VRM-Addon-for-Blender)) is required for the headless forge path; the Brúarhönd path does not require Blender on the forge machine.

```bash
# Clone the repo
git clone https://github.com/hrabanazviking/Seidr-Smidja.git
cd Seidr-Smidja
git checkout development

# Install with dev tools (recommended for development)
pip install -e ".[dev]"

# Or install the base package only (production forge use)
pip install -e .
```

### 2. Optional Install — Brúarhönd Daemon Dependencies

Install this on the machine where **VRoid Studio runs** (the daemon host). The forge machine itself does NOT need these.

```bash
pip install -e ".[brunhand-daemon]"
```

This installs the GUI automation extras: `pyautogui`, `mss`, `pygetwindow`, plus platform-conditional `pywinauto` (Windows) or `pyobjc-framework-Quartz` (macOS).

### 3. Verify the Install

```bash
python tools/verify_install.py
```

Reports Python version, dependency status, Blender executable resolution, and forge readiness.

### 4. Bootstrap the Hoard (one-time)

The Hoard ships empty by design. Bootstrap a permissively-licensed seed VRM into `data/hoard/bases/`:

```bash
python tools/bootstrap_hoard.py
# or via CLI:
seidr bootstrap-hoard
```

This downloads, SHA-256 verifies, and registers the seed VRM in the catalog. Mismatched hashes are deleted; absent hashes log a WARNING.

### 5. Generate Your First Avatar (Headless Forge Path)

This requires Blender installed and discoverable (in `PATH`, or via `SEIDR_BLENDER_PATH` env var, or via `config/user.yaml`).

```bash
# Build an avatar from the minimal example spec
seidr build examples/spec_minimal.yaml --out ./out/

# Inspect a produced VRM (Gate compliance check)
seidr inspect ./out/<avatar_id>.vrm

# List available base assets in the Hoard
seidr list-assets

# Print version
seidr version
```

`./out/` will contain `<avatar_id>.vrm` plus a `renders/` directory with PNG previews for the standard Oracle Eye views.

### 6. Drive a Live VRoid Studio Session (Brúarhönd Path)

Set this up on **two machines** (or one — both work):

#### A. On the VRoid host machine

```bash
# Generate and export a bearer token
export BRUNHAND_TOKEN="$(python -c 'import secrets; print(secrets.token_urlsafe(32))')"

# Or save it to a secrets file the daemon can read at startup
```

Edit `config/user.yaml` on the VRoid host (gitignored — never committed):

```yaml
brunhand:
  daemon:
    bind_address: 100.x.y.z   # the host's Tailscale IP — find via `tailscale ip -4`
    port: 8848
    allow_remote_bind: true   # required for non-localhost binds — explicit operator opt-in
    project_root: ~/Documents/VRoidProjects
    export_root: ~/Documents/VRoidExports
    tls:
      enabled: false   # acceptable if daemon is bound to a Tailscale interface (overlay encrypts)
```

Start the daemon:

```bash
python -m seidr_smidja.brunhand.daemon
# or via the operator launcher:
python tools/brunhand_daemon.py
```

The daemon prints a startup banner with the bind address. Health probe (no auth required):

```bash
curl http://100.x.y.z:8848/v1/brunhand/health
# → {"status": "ok", "version": "0.1.0", "os": "Windows", "screen": [1920, 1080], ...}
```

#### B. On the forge machine

Edit `config/user.yaml` on the forge:

```yaml
brunhand:
  hosts:
    - name: vroid-workstation
      address: vroid-workstation.tailnet.ts.net   # MagicDNS or Tailscale IP
      port: 8848
      token_env: BRUNHAND_TOKEN_VROID_WORKSTATION   # forge-side env var name holding the token
```

Set the matching token:

```bash
export BRUNHAND_TOKEN_VROID_WORKSTATION="<the same token>"
```

Verify:

```bash
seidr brunhand health vroid-workstation
seidr brunhand capabilities vroid-workstation
seidr brunhand screenshot vroid-workstation --out /tmp/probe.png
```

Drive VRoid:

```bash
# Open a project
seidr brunhand vroid-open vroid-workstation /path/to/project.vroid

# Click somewhere (e.g. a UI element at coordinates 800, 450)
seidr brunhand click vroid-workstation 800 450

# Type into the focused field
seidr brunhand type vroid-workstation "MyAvatar"

# Send a hotkey (Ctrl+S to save)
seidr brunhand hotkey vroid-workstation ctrl+s

# Take a screenshot to see the result
seidr brunhand screenshot vroid-workstation --out /tmp/after.png

# Export VRM (drives File→Export VRM, types validated path, verifies file on disk)
seidr brunhand vroid-export vroid-workstation /path/to/output.vrm
```

For full operator setup including Tailscale ACL examples, see [`docs/features/brunhand/TAILSCALE.md`](docs/features/brunhand/TAILSCALE.md). For the architecture, see [`docs/features/brunhand/ARCHITECTURE.md`](docs/features/brunhand/ARCHITECTURE.md). For the full data flow including all seven failure modes, see [`docs/features/brunhand/DATA_FLOW.md`](docs/features/brunhand/DATA_FLOW.md).

### 7. Run the Test Suite

```bash
# Default: all tests except those requiring Blender or a live VRoid host
pytest -m "not requires_blender and not requires_vroid_host"
# → 446 passed, 3 skipped, 42 deselected (Blender/VRoid)

# Include Blender integration tests (requires Blender installed)
pytest -m "not requires_vroid_host"

# Include live VRoid host tests (requires SEIDR_VROID_HOST env var pointing to a daemon)
pytest

# With coverage
pytest --cov=seidr_smidja -m "not requires_blender and not requires_vroid_host"
# → 82% aggregate
```

### 8. Configuration Model

- `config/defaults.yaml` — shipped defaults. **Never edit this directly.**
- `config/user.yaml` — your overrides. **Gitignored. Created by you on first setup.**
- Environment variables — highest precedence:
  - `SEIDR_BLENDER_PATH` — Blender executable
  - `BRUNHAND_TOKEN` — daemon-side bearer token
  - `BRUNHAND_TOKEN_<HOST>` — forge-side token per registered host
  - `SEIDR_STRAUMUR_HOST` — REST bind address override
  - `SEIDR_BRUNHAND_BIND` — Brúarhönd daemon bind override

---

## Full Operational Reference

### CLI Commands (Rúnstafr)

| Command | Purpose |
|---|---|
| `seidr build <spec> [--out <dir>] [--config <yaml>] [--no-telemetry]` | Build a VRM from a Loom spec via the headless forge. |
| `seidr inspect <vrm_path> [--targets ...] [--json] [--config <yaml>]` | Run Gate compliance check on an existing `.vrm`. |
| `seidr list-assets [--type <t>] [--tag <t>] [--json] [--config <yaml>]` | List available Hoard base assets. |
| `seidr bootstrap-hoard [--force] [--config <yaml>]` | Download and register the Hoard seed VRM(s). |
| `seidr version` | Print version. |
| `seidr brunhand health <host>` | Health probe a registered Brúarhönd daemon. |
| `seidr brunhand capabilities <host>` | List the daemon's available primitives. |
| `seidr brunhand screenshot <host> [--out <path>] [--region x,y,w,h] [--monitor <n>]` | Capture a screenshot via the daemon. |
| `seidr brunhand click <host> <x> <y> [--button left/right/middle] [--mods ctrl,shift]` | Mouse click on the daemon's host. |
| `seidr brunhand type <host> <text>` | Type text into the daemon's focused field. |
| `seidr brunhand hotkey <host> <combo>` | Press a hotkey combination (e.g. `ctrl+s`). |
| `seidr brunhand vroid-open <host> <project_path>` | Open a `.vroid` project on the daemon's host. |
| `seidr brunhand vroid-export <host> <vrm_path>` | Export to VRM via VRoid Studio's File→Export VRM. |
| `seidr brunhand vroid-save <host> <project_path>` | Save the active VRoid project to a path. |

### REST Endpoints (Straumur)

The Straumur REST bridge runs as `python -m seidr_smidja.bridges.straumur` (uvicorn). Defaults to bind `127.0.0.1:8801`; non-localhost requires `straumur.allow_remote_bind: true`.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/avatars` | Submit a Loom spec, receive `{run_id, vrm_path, render_paths, warnings}`. |
| `GET` | `/v1/avatars/{run_id}` | Retrieve a prior build's Annáll record. |
| `POST` | `/v1/inspect` | Gate compliance check on an existing `.vrm` (path validated against allow-list). |
| `GET` | `/v1/assets` | List Hoard base assets. |
| `POST` | `/v1/brunhand/dispatch` | Forward a Brúarhönd dispatch request to a registered daemon (operator-side proxy). |

### MCP Tools (Mjöll)

The Mjöll MCP bridge runs as `python -m seidr_smidja.bridges.mjoll`. Requires the optional `[mcp]` install group — falls back to a clear error message if `mcp` is not installed.

| Tool | Purpose |
|---|---|
| `seidr.build_avatar` | Submit a Loom spec, return `{vrm_path, render_paths, warnings}`. |
| `seidr.inspect_vrm` | Gate compliance check, return `ComplianceReport`. |
| `seidr.brunhand_screenshot` | Capture a screenshot via a registered daemon. |
| `seidr.brunhand_click` | Click via a registered daemon. |
| `seidr.brunhand_vroid_export` | Export VRM via VRoid Studio on a registered daemon. |

### Skill Manifests

- **Hermes:** `src/seidr_smidja/bridges/skills/hermes/manifest.yaml` — wraps the CLI for use as a Hermes skill.
- **OpenClaw:** `src/seidr_smidja/bridges/skills/openclaw/manifest.yaml` — wraps the CLI for use as an OpenClaw skill.
- **Claude Code:** `src/seidr_smidja/bridges/skills/claude_code/SKILL.md` — Claude Code skill markdown documenting CLI invocation.

---

## Quickstart for an AI Agent

> **Historical marker (2026-05-06 — Phase 7 + Brúarhönd close):** The block below preserves the original Genesis-era Quickstart for archival continuity. **For current authoritative commands, refer to the "Setup and Operation" section above.** The CLI rename (`seidr inspect` is canonical, ratified by D-008) and the `seidr list-assets` implementation (D-009) are now both live.

### Through Rúnstafr (CLI)

```bash
# Install the package (requires Python 3.11+ and Blender in PATH or BLENDER_PATH set)
pip install -e ".[dev]"

# Build an avatar from a spec file
seidr build examples/spec_minimal.yaml --output ./out/

# Check compliance on an existing .vrm
seidr check ./out/avatar.vrm

# List available base meshes in the Hoard
seidr list-assets --type vrm_base
```

> **Implementation note (2026-05-06 — AUDIT-003):** The compliance command is registered as `seidr inspect` in the implementation, not `seidr check`. The `seidr list-assets` CLI command is not yet implemented in v0.1 — the equivalent endpoint is `GET /v1/assets` on the Straumur REST bridge. See `src/seidr_smidja/bridges/INTERFACE_AMENDMENT_2026-05-06.md`.

> **Update 2026-05-06 (post-Brúarhönd close):** D-009 ratified `seidr list-assets` as implemented and `seidr bootstrap-hoard` as documented. Both are now live. See [`docs/DECISIONS/D-009-list-assets-and-bootstrap-hoard-cli.md`](docs/DECISIONS/D-009-list-assets-and-bootstrap-hoard-cli.md).

On success, `./out/` will contain `avatar.vrm` and a `renders/` directory with PNG previews.

### Through Mjöll (MCP)

> **Update 2026-05-06:** Mjöll is now LIVE (no longer "Phase 5 — coming"). Run as `python -m seidr_smidja.bridges.mjoll`. Five MCP tools registered: `seidr.build_avatar`, `seidr.inspect_vrm`, `seidr.brunhand_screenshot`, `seidr.brunhand_click`, `seidr.brunhand_vroid_export`. See the **Full Operational Reference → MCP Tools (Mjöll)** table above.

*(Original genesis-era note follows for archival continuity:)*

*(Phase 5 — coming. The MCP server will be started as a long-running process and register the `seidr_build` tool. The tool input/output schema is defined in [`src/seidr_smidja/bridges/INTERFACE.md`](src/seidr_smidja/bridges/INTERFACE.md).)*

```json
{
  "tool": "seidr_build",
  "input": {
    "spec": { "avatar_id": "my_avatar", "base_asset_id": "vroid/tall_feminine_v1" },
    "output_dir": "/path/to/output"
  }
}
```

### Through Straumur (REST)

> **Update 2026-05-06:** Straumur is now LIVE (no longer "Phase 5 — coming"). Run as `python -m seidr_smidja.bridges.straumur`. Five endpoints live: `POST /v1/avatars`, `GET /v1/avatars/{run_id}`, `POST /v1/inspect` (allow-list-validated), `GET /v1/assets`, `POST /v1/brunhand/dispatch`. Defaults bind 127.0.0.1; non-localhost requires `straumur.allow_remote_bind: true` config flag (D-005-style discipline; see [`AUDIT_GENESIS.md`](docs/AUDIT_GENESIS.md) for the H-005 hardening fix). See the **Full Operational Reference → REST Endpoints (Straumur)** table above.

*(Original genesis-era note follows for archival continuity:)*

*(Phase 5 — coming. FastAPI server at `seidr_smidja.bridges.straumur.app:app`.)*

```http
POST /build
Content-Type: application/json

{ "spec_source": "...", "base_asset_id": "vroid/tall_feminine_v1", "output_dir": "..." }
```

---

## Reading Order for a New Arrival

Walk this path in order. Each scroll opens the next.

### Project-level scrolls

1. **[docs/SYSTEM_VISION.md](docs/SYSTEM_VISION.md)** — Read the Central Image (the Hermes völva story). Then the Unbreakable Vows. Five minutes here prevents hours of misunderstanding.
2. **[docs/PHILOSOPHY.md](docs/PHILOSOPHY.md)** — The Five Sacred Principles and Ten Sacred Laws. These are structural invariants, not guidelines.
3. **[docs/DOMAIN_MAP.md](docs/DOMAIN_MAP.md)** — Who owns what. The Dependency Law. Every domain defined in one place. (Includes additive Brúarhönd addendum at the bottom.)
4. **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** — The four-layer model. The Shared Anvil pattern. The Blender subprocess design.
5. **[docs/DATA_FLOW.md](docs/DATA_FLOW.md)** — How a build request flows through all nine steps, with failure paths and the vision feedback loop.
6. **[docs/REPO_OVERVIEW.md](docs/REPO_OVERVIEW.md)** — The living terrain map. "Where do I look for X?" answered in a table.
7. **The relevant INTERFACE.md** — Whichever domain you are working in, read its contract before touching its code.

### Brúarhönd feature scrolls (added 2026-05-06)

8. **[docs/features/brunhand/VISION.md](docs/features/brunhand/VISION.md)** — The Skald's vision scroll for the bridge that grew a hand. Central Image of the agent reaching across Tailscale. Primary Rite. Unbreakable Vows specific to Brúarhönd.
9. **[docs/features/brunhand/PHILOSOPHY_ADDENDUM.md](docs/features/brunhand/PHILOSOPHY_ADDENDUM.md)** — Three new sacred principles: The Hand Asks Permission · The Distant Eye Sees as Clearly as the Near Eye · The Hand Has Honest Limits.
10. **[docs/features/brunhand/ARCHITECTURE.md](docs/features/brunhand/ARCHITECTURE.md)** — Daemon (Horfunarþjónn) and client (Hengilherðir) split. Lateral dispatch seam. Authentication. Capabilities probe. Per-OS implementation table. Optional dependency strategy.
11. **[docs/features/brunhand/DATA_FLOW.md](docs/features/brunhand/DATA_FLOW.md)** — Mode A/B/C dispatch. 15-step Mode A walkthrough. 8 Mermaid diagrams. All 7 failure flows. Two-Annáll asymmetry. Concurrency model.
12. **[docs/features/brunhand/README.md](docs/features/brunhand/README.md)** — Operator quickstart. Three modes of use. Reading order specific to Brúarhönd.
13. **[docs/features/brunhand/TAILSCALE.md](docs/features/brunhand/TAILSCALE.md)** — Tailscale ACL setup (tagged + single-user). Daemon bind config. TLS configuration. Failure modes specific to Tailscale topology.
14. **[docs/features/brunhand/AUDIT_BRUNHAND_2026-05-06.md](docs/features/brunhand/AUDIT_BRUNHAND_2026-05-06.md)** — Audit report. 18 findings. Verdict PASS WITH CONCERNS. Full remediation history.
15. **[src/seidr_smidja/brunhand/INTERFACE.md](src/seidr_smidja/brunhand/INTERFACE.md)** + **[daemon/INTERFACE.md](src/seidr_smidja/brunhand/daemon/INTERFACE.md)** + **[client/INTERFACE.md](src/seidr_smidja/brunhand/client/INTERFACE.md)** — Public contracts.

### Decision records (cross-cutting)

16. **[docs/DECISIONS/](docs/DECISIONS/)** — All ten ratified ADRs. D-001..D-007 are genesis decisions. D-008 ratified `seidr inspect` as canonical. D-009 closed the deferred D-008 sub-items. D-010 ratified the entire Brúarhönd feature.

---

## Project Protocol

This repository is built and maintained using the **Mythic Engineering** protocol — six named AI roles, a living document system, and a clear daily practice for keeping code and documentation in alignment. Read [`MYTHIC_ENGINEERING.md`](MYTHIC_ENGINEERING.md) before contributing.

The full phase-by-phase progress tracker lives in [`TASK_seidr_smidja_genesis.md`](TASK_seidr_smidja_genesis.md).

---

## Current Status

> **Update 2026-05-06 (Brúarhönd v0.1 close):** **Three full Mythic Engineering rituals completed in one evening — Genesis, Hardening, and Brúarhönd v0.1.** **446 non-Blender, non-VRoid-host pytest tests passing, 3 skipped, 42 Blender/VRoid deselected** (Genesis 159 → Hardening 286 → Brúarhönd 430 → Brúarhönd remediation 489). **Aggregate coverage 82%** on the forge subtree, **68%** on the new brunhand subtree. **10 ADRs accepted** (D-001..D-010). All 10 genesis findings + all 23 hardening findings + all 18 Brúarhönd audit findings closed (one Low residual H-V-001 deferred from hardening; B-013/B-014 documented as v0.2 candidates in the Brúarhönd interface amendment). See [`docs/DEVLOG.md`](docs/DEVLOG.md) for the full session log including the closing entry "Brúarhönd v0.1 Ritual Closed (Phase 7 — Scribe close)."
>
> Open work for v0.1.x — none blocking; H-V-001 cleanup, real Blender-enabled CI smoke, real VRoid-host smoke, then a v0.1.0 tag. v0.2 candidates: Loom→VRoid translation layer, Annáll streaming replication, Tailscale dynamic discovery, layout-aware hotkey allow-list, inline-token-refusal flag, parallel Mode C.

*(Original Genesis-era status follows for archival continuity:)*

**Genesis complete — vertical slice forged, 159 non-Blender tests green.**

The full Mythic Engineering genesis ritual (Phases 0–7) has run. All seven ADRs ratified; all 10 audit findings from `docs/AUDIT_GENESIS.md` closed. The `seidr build` pipeline is wired end-to-end: Loom validates specs, Hoard resolves assets, Forge and Oracle Eye call Blender headlessly (requires Blender), Gate runs VRChat/VTube Studio compliance, Annáll records every session. Run `pytest -m "not requires_blender"` for the 159-test non-Blender suite; the full forge cycle requires Blender in PATH or `BLENDER_PATH` set.

> **Note (2026-05-06):** The badge above still reads "genesis phase" — it will be updated when the first Blender-enabled CI run completes and v0.1 is tagged.

See the progress tracker in [`TASK_seidr_smidja_genesis.md`](TASK_seidr_smidja_genesis.md), the Brúarhönd-specific tracker in [`TASK_brunarhond_v0_1.md`](TASK_brunarhond_v0_1.md), and `docs/DEVLOG.md` for the full record across all three rituals.

---

## Repository Structure

```
Seiðr-Smiðja/
├── README.md                          ← You are here.
├── MYTHIC_ENGINEERING.md              ← How to work in this repo.
├── TASK_seidr_smidja_genesis.md       ← Phase inventory and progress tracker.
├── pyproject.toml                     ← Package definition, entry points, markers.
│
├── docs/
│   ├── SYSTEM_VISION.md               ← Soul, Primary Rite, Unbreakable Vows, Central Image.
│   ├── PHILOSOPHY.md                  ← Five Sacred Principles, Ten Sacred Laws.
│   ├── DOMAIN_MAP.md                  ← All domains, ownership, Dependency Law.
│   ├── ARCHITECTURE.md                ← Layers, Shared Anvil, subprocess pattern, config.
│   ├── DATA_FLOW.md                   ← Nine-step Primary Rite, failure flows, feedback loop.
│   ├── REPO_OVERVIEW.md               ← Living terrain map and navigation table.
│   ├── DEVLOG.md                      ← Daily session log.
│   └── DECISIONS/                     ← Architectural decision records (ADRs).
│
├── config/
│   ├── defaults.yaml                  ← Shipped defaults. Never edit directly.
│   └── user.yaml                      ← Your overrides. Gitignored.
│
├── src/seidr_smidja/
│   ├── loom/                          ← Spec schema, validation, serialization.
│   ├── hoard/                         ← Base .vrm catalog and asset resolution.
│   ├── forge/                         ← Headless Blender build subprocess.
│   ├── oracle_eye/                    ← Headless Blender render subprocess; vision feedback.
│   ├── gate/                          ← VRChat + VTube Studio compliance validation.
│   ├── annall/                        ← Session tracking, event logging, build history.
│   ├── _internal/                     ← Shared Blender subprocess runner (per D-003).
│   ├── brunhand/                      ← Brúarhönd — cross-machine VRoid Studio remote control (added 2026-05-06).
│   │   ├── INTERFACE.md               ← Top-level domain contract.
│   │   ├── exceptions.py              ← Typed exception hierarchy.
│   │   ├── models.py                  ← Pydantic v2 request/response models.
│   │   ├── daemon/                    ← Horfunarþjónn — Watching-Daemon runs on VRoid host.
│   │   │   ├── INTERFACE.md           ← HTTP API contract for daemon endpoints.
│   │   │   ├── app.py                 ← FastAPI app + middleware + concurrent-session lock.
│   │   │   ├── auth.py                ← Gæslumaðr — bearer-token guard (constant-time).
│   │   │   ├── capabilities.py        ← Sjálfsmöguleiki — per-OS capability registry.
│   │   │   ├── runtime.py             ← PyAutoGUI/MSS/pygetwindow shim layer.
│   │   │   ├── config.py              ← Daemon config loader.
│   │   │   ├── __main__.py            ← `python -m seidr_smidja.brunhand.daemon`
│   │   │   └── endpoints/             ← health, capabilities, primitives, vroid handlers.
│   │   └── client/                    ← Hengilherðir — Reaching Client runs in forge.
│   │       ├── INTERFACE.md           ← Python API contract for the client.
│   │       ├── client.py              ← BrunhandClient — primitive methods + auto-timeout.
│   │       ├── session.py             ← Tengslastig — session container + owns_client.
│   │       ├── factory.py             ← make_client_from_config / make_session_from_config.
│   │       └── oracle_channel.py      ← Ljósbrú — Oracle Eye integration channel.
│   └── bridges/                       ← Mjöll (MCP), Rúnstafr (CLI), Straumur (REST), Skills.
│       ├── core/                      ← Shared Anvil — dispatch() AND brunhand_dispatch() (lateral).
│       ├── mjoll/                     ← MCP server — 5 live tools.
│       ├── runstafr/                  ← CLI — 14 live commands (5 forge + 8 brunhand + version).
│       ├── straumur/                  ← REST — FastAPI app with 5 live endpoints.
│       └── skills/                    ← Hermes/OpenClaw/Claude Code skill manifests.
│
├── data/                              ← Compliance rule YAML files, Hoard catalog.
├── tests/                             ← Pytest suite. 446 non-Blender, non-VRoid-host tests passing (3 skipped, 42 Blender/VRoid deselected).
│   └── brunhand/                      ← 144 + 59 = 203 Brúarhönd-specific tests.
├── tools/                             ← bootstrap_hoard.py, verify_install.py, brunhand_daemon.py, verify_brunhand.py.
├── examples/                          ← Example spec files.
│
└── docs/features/brunhand/            ← Brúarhönd feature documentation suite (added 2026-05-06).
    ├── VISION.md                      ← [Skald] Soul, Primary Rite, Unbreakable Vows.
    ├── PHILOSOPHY_ADDENDUM.md         ← [Skald] Three new sacred principles.
    ├── ARCHITECTURE.md                ← [Architect] Daemon/client split, dispatch seam.
    ├── DATA_FLOW.md                   ← [Cartographer] Mode A/B/C, 8 diagrams, 7 failure flows.
    ├── README.md                      ← [Scribe] Operator quickstart and three modes.
    ├── TAILSCALE.md                   ← [Scribe] ACL setup and bind configuration.
    └── AUDIT_BRUNHAND_2026-05-06.md   ← [Auditor] Bug hunt + remediation history.
```

---

![https://raw.githubusercontent.com/hrabanazviking/Seidr-Smidja/refs/heads/development/Viking_Apache_V2_1.jpg](https://raw.githubusercontent.com/hrabanazviking/Seidr-Smidja/refs/heads/development/Viking_Apache_V2_1.jpg)

---

## License

Copyright (c) 2026 Volmarr Wyrd

Seiðr-Smiðja is licensed under the **Apache License, Version 2.0**. See the [LICENSE](LICENSE) file for the full license text and [NOTICE](NOTICE) for the project attribution.

Unless required by applicable law or agreed to in writing, this project is distributed on an "AS IS" BASIS, without warranties or conditions of any kind, either express or implied.

---

## Distribution and Privacy Position

Seiðr-Smiðja is published here as source code and project material.

The author does not require users to provide age, identity, government ID, biometric data, or similar personal information in order to access or use the source code in this repository.

The author may decline to provide official binaries, installers, hosted services, app-store releases, or other official distribution channels where doing so would require age verification, identity verification, or similar personal-data collection.

Any third party who forks, packages, redistributes, deploys, hosts, or otherwise makes this software available does so independently and is solely responsible for compliance with applicable law, platform policy, and distribution requirements in their own jurisdiction and context.

See [LEGAL-NOTICE.md](LEGAL-NOTICE.md) for details.

---

*Front door written by Eirwyn Rúnblóm, Scribe — 2026-05-06.*
*For Volmarr Wyrd and Runa Gridweaver Freyjasdóttir.*

---

---

## RuneForgeAI

**RuneForgeAI** is my AI research, development, and creative systems forge: a Norse Pagan cyber-Viking workshop for building mythic AI architectures, memory systems, world engines, companion intelligence, and structured vibe coding tools.

RuneForgeAI exists at the crossroads of:

- **Mythic Engineering**
- **AI memory and continuity systems**
- **Viking-themed simulation and worldbuilding**
- **AI companions with stable identity**
- **small-model enhancement through architecture**
- **retrieval, grounding, and truth-verification systems**
- **cyber-Heathen software design**
- **human + AI co-creation**

The core idea is simple:

> AI should not be treated as a disposable text generator.  
> It should be shaped into structured, memory-bearing, meaning-aware systems that can preserve continuity, deepen creativity, and help humans build living worlds.

RuneForgeAI is where I explore architectures that make AI more coherent, more persistent, and more useful: not through hype, but through structure. Memory, retrieval, world state, personality, routing, verification, symbolic logic, and mythic design language all become part of the same forge.

This work connects directly to my larger ecosystem of projects, including the **Norse Saga Engine**, **Mythic Engineering**, **WYRD Protocol**, **Mímir-Vörðr**, cyber-Viking philosophy, AI companion design, and the broader vision of spiritually meaningful technology.

### What RuneForgeAI Builds

- AI-native memory frameworks
- persistent personality and companion systems
- Viking and mythic world simulation tools
- roleplay and RPG intelligence architectures
- structured prompt and documentation protocols
- retrieval-augmented truth systems
- small-model orchestration patterns
- cyber-Viking AI aesthetics and interfaces
- open frameworks for human-AI creative collaboration

### Guiding Principle

> Build AI like a living system, not a pile of prompts.

RuneForgeAI is my digital forge for turning myth, memory, code, and consciousness into working architecture.

---

![https://raw.githubusercontent.com/hrabanazviking/Seidr-Smidja/refs/heads/development/image-23-RuneForgeAI.jpg](https://raw.githubusercontent.com/hrabanazviking/Seidr-Smidja/refs/heads/development/image-23-RuneForgeAI.jpg)

---

![https://raw.githubusercontent.com/hrabanazviking/Seidr-Smidja/refs/heads/development/IMG_0407.jpeg](https://raw.githubusercontent.com/hrabanazviking/Seidr-Smidja/refs/heads/development/IMG_0407.jpeg)

---

