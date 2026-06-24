# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working
with code in the **{{PROJECT_NAME}}** project.

## Overview

{{PROJECT_OVERVIEW}}

This project follows the unified conventions in **CommonClaude**
(`kkhyunhho/CommonClaude`) — the single, self-contained source of truth
for every device-control and integration project in this workspace. The
sections below specialize those conventions for this project; where this
file is silent, CommonClaude governs.

{{AUTHORITY_SECTION}}

## Big picture: one cell, many devices

Every project here either **controls one machine** or **integrates two or
more** into an automated cell. The end goal is a single self-driving
laboratory where all devices compose. To make that composition cheap,
**every project exposes the same shape**:

- a Python **driver library** (`src/<codename>/`) — one class, one device;
  this is the baseline every project ships;
- a thin **FastAPI server** (`server/`) exposing `/v1/*` JSON **where
  remote/web control is actually wanted** — primarily the integration
  module(s) the lab website drives, and any device you want to control
  standalone over HTTP. It is **not mandatory for every device driver**;
  add it per need rather than designing one for each device up front;
- an optional **integration layer** that composes several devices, talking
  to them **over HTTP** for loose coupling, or **in-process** (direct
  import) when real-time safety interlocks demand zero network latency
  (the *hybrid* model).

```
[device /v1]  [device /v1]  [device /v1]    each device: independently controllable
      \            |            /
       \           |           /
      [integration /v1]   composes devices (HTTP + in-process hybrid)
             |
      [lab website / ESP32]   HTTP clients
```

## Environment

| Item    | Detail                                                      |
|---------|-------------------------------------------------------------|
| Runtime | Docker container (`--privileged` — needed for USB hardware) |
| OS      | Ubuntu 24.04 (Noble)                                        |
| Python  | **>= 3.12** (uniform floor across all projects)            |
| Env     | **one shared conda env `elec`** — every project is `pip install -e`'d into it; do NOT use per-project venvs |
| Run as  | **root** — device-node rebuild and `ftdi_sio` detach need `/sys` writes and `os.mknod` |
| Dev tool| Claude Code (CLI / VS Code extension)                       |

All projects share the single conda env **`elec`** (new terminals activate
it automatically). Each driver package is installed editable
(`pip install -e`) into `elec`, so integration layers import siblings
directly — no per-project venv, no `sys.path` bootstrap. Run `ruff`,
`mypy`, `pytest` from the activated env (no `.venv/bin/` prefix).

The container's `/dev` is a private tmpfs, so USB device nodes go stale
after re-enumeration. Device-touching entrypoints rebuild them at startup.

## Repository layout (standard skeleton)

Every project uses the same skeleton. `<codename>` is the short device
codename (see Naming).

```
<ProjectFolder>/
├── CLAUDE.md                 this file
├── README.md                 setup + run
├── DESIGN.md                 architecture + rationale
├── ToDo.md                   append-only task log
├── LearnedPatterns.md        Problem/Cause/Fix/Rule log
├── pyproject.toml            build, deps, ruff + mypy config
├── src/<codename>/           the driver library (one class per device)
├── server/                   FastAPI /v1 bridge — OPTIONAL, add per need
│                             (app.py, routes.py, schemas.py, errors.py)
├── tests/                    production unit tests (CI)
│   └── server/               server tests against a Fake device (if server/)
└── claude_test/              HIL / debug scripts (exempt from style rules)
```

Integration projects add **`workflow.md`** (the operating procedure) and
import the drivers they compose via the documented sibling-import
bootstrap.

## Naming (codename-hybrid)

A short **codename** identifies the device across code; a descriptive
**CamelCase** name identifies the project and class.

| Slot              | Style                       | Example                  |
|-------------------|-----------------------------|--------------------------|
| Repo folder       | `CamelCase`, descriptive    | `SyringePumpController`  |
| Python package    | `lower_codename`            | `src/sy01b/`             |
| Driver class      | `CamelCase` (= folder)      | `SyringePumpController`  |
| Console scripts   | `<codename>-<verb>`         | `sy01b-diagnose`, `sy01b-server` |
| FastAPI server    | `<codename>-server`         | `sy01b-server`           |
| Established codenames | —                       | `sy01b`, `entris_ii`, `picus2`, `mks_motor` |

Within code: variables/functions `lower_case` (nouns vs verbs), classes
`CamelCase`, constants `lower_case`, modules `lowercase`. **Physical
quantities carry a unit suffix** — `travel_z_mm`, `target_uL`,
`speed_rpm` — so cross-device integration cannot silently mix units.

## Driver API contract

So an integration layer can drive any device uniformly, every driver
library exposes:

- **One class per device**, named for the project (`SyringePumpController`).
- **Lifecycle**: `setup()` / `close()` (or a context manager). Open the
  transport once; never auto-initialize motion implicitly.
- **Verbs**: a small set of imperative methods for actions (`aspirate_uL`,
  `move_valve_to_port`, ...).
- **Queries**: read-only `query_*` / `diagnose()` that never move anything
  and are safe to call repeatedly.
- **Device identity by stable USB `VID:PID`**, never a `/dev/ttyUSB*` index
  (which renumbers). Resolve the port at runtime.
- **Exception hierarchy**: a single `DeviceError` base with specific
  subclasses (e.g. `NotInitializedError`, `PlungerOverloadError`). Attach
  `command_sent`, `raw_reply`, `error_code` so the server can serialize a
  stable error envelope. This is what the FastAPI layer maps to HTTP.

## FastAPI server standard (`/v1`)

The `server/` package is a **thin** bridge — all device logic stays in the
driver. Mirror the reference implementation (`sy01b-server`):

| File              | Role                                                          |
|-------------------|---------------------------------------------------------------|
| `app.py`          | `create_app(device_factory=None, *, config=None)` factory + lifespan. Inject a real driver in prod, a Fake in tests. `app.state.{device, config, lock, last_diagnose}`. |
| `routes.py`       | `APIRouter(prefix="/v1")`. Each handler acquires `app.state.lock`; blocking driver calls run via `run_in_threadpool`. |
| `schemas.py`      | Pydantic request/response models (units in field names). |
| `errors.py`       | `register_exception_handlers` — driver exception to HTTP status + stable JSON `{error, code, command, raw_reply_hex, message}`. No traceback leaks. |
| `<codename>.toml` | Bench config (`.example` committed, real file gitignored). |
| `__main__.py`     | `python -m server --config ...` entrypoint. |

API rules:

- Everything under `/v1`. OpenAPI tags **Discovery** (read-only, repeat-safe)
  / **Motion** (state-changing) / **Low-level (deprecated)**.
- Mandatory endpoints: `GET /v1/health`, `GET /v1/diagnose`,
  `GET /v1/status`; motion as `POST`.
- **Single in-flight**: `--workers 1` + `asyncio.Lock`. Commands never
  interleave; a long op holds the lock for its whole duration.
- **No auto-init**: the server forwards calls and never starts motion the
  client did not request.
- Driver opened once at startup (lifespan), closed at shutdown.

Standard error to status map: `400` invalid arg/command; `409` wrong state
(not-initialized, overflow); `500` device fault (overload, init-failed);
`502` protocol; `503` transport-closed / diagnostic; `504` timeout.

## Integration & safety model (hybrid)

- An integration layer is itself a device-shaped unit: it MAY expose its
  own `/v1` server and be composed by a higher layer.
- **Loose / human / remote control uses HTTP.** The website and ESP32 are
  just HTTP clients of a `/v1` server.
- **Real-time cross-device safety uses in-process calls.** When one
  device's fault must stop another *now* (e.g. a motor `ConnectionError`
  must halt the pump), route through direct import, not HTTP — network
  latency/timeout is unacceptable for an interlock.
- **Fail-safe is mandatory.** A comms fault stops all motion first, then
  aborts. Document the exact policy per project in Hardware notes.

## Code style

MIT CommLab style. **80-column** lines, **4-space** indent (no tabs), one
statement per line. Comment the *why*, never restate the *what*; delete
stale comments. **English only** in code, comments, docstrings, commits,
issues, PRs. Public functions/classes get **Google-style docstrings**
(`Args:`/`Returns:`/`Raises:`), stating what and why. **No magic numbers**
— name them (module constants / an `OPERATOR CONFIGURATION` block).

{{LANGUAGE_NAMING}}

## Debug file management

`tests/` = production CI tests; `claude_test/` = HIL / debug / one-off
scripts (exempt from the 80-col limit and mandatory docstrings).
`claude_test/README.md` indexes the directory — add a row per script.
Promote stable logic into `src/` or `tests/` and delete the scratch copy.

## Testing strategy

- **Pure logic** (frame builders, parsers, unit conversion, status decode)
  is unit-tested in `tests/` — fast, no hardware.
- **Motion / I/O** is **HIL-verified** via `claude_test/` against the real
  device; it is intentionally not unit-tested.
- The `server/` is tested against an injected **Fake device** in
  `tests/server/`.
- No magic numbers, no hardcoding to match test inputs; fix logic, not the
  branch.

## Linting & types

Ruff (`line-length = 80`) + mypy, configured in `pyproject.toml`. Before
considering a change done: `ruff check`, `ruff format --check`, `mypy`.

## Task management

{{TASK_WORKFLOW}}

## Learned Patterns

Treat [`LearnedPatterns.md`](LearnedPatterns.md) as part of the workflow.
Read relevant sections before drafting a `ToDo.md` entry (cite `(see LP §X)`);
after a task, append new gotchas in **Problem / Cause / Fix / Rule** form
(§1 Recurring / §2 Gotchas / §3 Library quirks / §4 Workflow / §5
Environment / §99 Uncategorized). Promote stable patterns into this file.

## Research before coding

Verify a real interface before calling it: read the driver source and the
device manual, search the repo for prior usage, trust docs over memory.
Hardware quirks are real and documented — do not guess.

## Hardware / domain notes

<!-- TODO: Fill in this project's device(s), protocol, USB VID:PID table,
     units, fault/fail-safe policy, and front-panel prerequisites. Never
     invent hardware facts — they come from real specs and bench
     observation. Cross-link gotchas to LearnedPatterns.md. -->
