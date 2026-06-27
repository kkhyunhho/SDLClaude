# SDLClaude

A Claude Code **skill** that bootstraps a new project's documentation from
the shared conventions used across this workspace's device-driver and
liquid-handling projects. Invoking `/new-project` asks a few questions and
writes three files into a target folder:

- `CLAUDE.md` — the project ruleset (shared core baked in; variable parts
  filled from your answers)
- `ToDo.md` — append-only task log
- `LearnedPatterns.md` — Problem / Cause / Fix / Rule log (seeded empty)

## What the "shared core" is

Extracted from the existing sibling projects (`AutomatedPipette`,
`PrecisionScaleController`, `SyringePumpController`,
`ESP32S3BOX3MotorController`, `SyringeLiquidHandler`,
`PipetteLiquidHandler`), these parts are identical everywhere and are
baked into the templates so you never re-answer them:

- MIT CommLab code style (80-col, 4-space, snake_case / CamelCase)
- Ruff lint + format, line-length 80
- `tests/` vs `claude_test/` debug-file separation
- Testing rules (no magic numbers, no hardcoding to tests)
- Append-only `ToDo.md`; `LearnedPatterns.md` in Problem/Cause/Fix/Rule
- Research-before-coding

The **variable parts** are asked at scaffold time: project name/overview,
language (Python or C+Python), git-workflow strength (Full issue+branch+PR
vs Light ToDo-only), authority (independent vs inherits sibling rulesets),
and the hardware/domain section (left as a stub for a human to fill).

## Install

Place this folder at `<project-or-home>/.claude/skills/new-project/` so
Claude Code discovers it. In this workspace it lives at
`/workspace/.claude/skills/new-project/` (on the persistent disk, so it
survives container recreation).

```
git clone https://github.com/kkhyunhho/SDLClaude.git \
    /workspace/.claude/skills/new-project
```

## Usage

In a Claude Code session, run:

```
/new-project
```

Then answer the prompts. The skill only writes the three files — it does
not run git or create issues.

## Layout

See [`ARCHITECTURE.md`](ARCHITECTURE.md) for the target lab's composition
hierarchy (device → cell → stage → lab) and two-stage / two-NUC deployment
topology.

| Path | Role |
|------|------|
| `ARCHITECTURE.md` | Target lab: composition hierarchy + two-stage/two-NUC topology |
| `DESIGN_SYSTEM.md` | Shared L1 **web** visual standard: colour, font, units, mandatory layout |
| `SKILL.md` | Entry point — questions, assembly, file-writing instructions |
| `templates/CLAUDE.template.md` | Canonical ruleset with placeholders |
| `templates/python-conventions.md` | Python naming table (always inserted) |
| `templates/c-conventions.md` | C naming table (only for C+Python) |
| `templates/full-workflow.md` + `full-git-sections.md` | Full git workflow |
| `templates/light-workflow.md` | Lightweight ToDo-only workflow |
| `templates/ToDo.template.md` | Task-log seed |
| `templates/LearnedPatterns.template.md` | Learned-patterns seed |
