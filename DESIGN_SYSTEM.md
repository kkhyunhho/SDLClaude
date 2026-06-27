# L1 web design system — the shared visual standard

Every device/cell that builds an **L1 web frontend** (the browser UI that
talks to the FastAPI `/v1` server) follows this. It is the visual twin of
the code contract in [ARCHITECTURE.md](ARCHITECTURE.md): same uniformity
argument, applied to the UI so every hardware web feels like one product.

Reference implementation: `SyringeLiquidHandler/web/` (React 19 + Vite +
Tailwind v4 + shadcn/ui on Base UI). New webs start from it.

## Stack

| Layer | Choice |
|-------|--------|
| Framework | React 19 + Vite + TypeScript |
| Styling | Tailwind v4 (`@tailwindcss/vite`), tokens in `src/index.css` `@theme` |
| Components | shadcn/ui on **Base UI** (`@base-ui/react` — `render` prop, not `asChild`) |
| Toasts | sonner (`bottom-right`, `richColors`, `closeButton`) |
| Icons | lucide-react (single-colour line icons only — no emoji) |

## Colour — ISA-101 + Okabe-Ito (colourblind-safe)

Colour is reserved for **abnormal / active** state; idle/normal stays
neutral gray. **Never colour alone** — always pair with an icon + text
label. Status tokens (defined in `index.css` `@theme inline`):

| Token | Hex | Meaning |
|-------|-----|---------|
| `--color-status-ok` | `#0072b2` (blue) | connected / nominal-active |
| `--color-status-warn` | `#e69f00` (amber) | warning / busy / attention |
| `--color-status-fault` | `#d55e00` (vermilion) | fault / alarm |
| `--color-status-idle` | `var(--muted-foreground)` | normal / idle (neutral) |

Neutral base + light/dark themes use the shadcn oklch token set
(`--background`, `--foreground`, `--card`, `--muted`, `--border`, …). Do
not introduce new hues outside the status palette without a reason.

## Typography & shape

- **Font: Geist Variable** (`@fontsource-variable/geist`); `--font-sans`
  and `--font-heading` both map to it. Mono/tabular for numeric readouts
  (`font-mono tabular-nums`).
- **Radius:** `--radius: 0.625rem` with the `sm → 4xl` scale already in
  `index.css`. Bind component corners to it, don't hardcode.

## Units (UI display) — UNIFY, no mixing

| Quantity | Unit | Notes |
|----------|------|-------|
| Length / position | **mm** | never cm/m in the UI |
| Speed | **mm/s** | derive from motor RPM internally, display mm/s |
| Liquid volume | **µL** | |
| Flow rate | **µL/s** | volume÷time, consistent with µL + s |
| Weight | **g** | |
| Time | **s** | |

Internal/native units (RPM, accel codes, pulses/sec) may exist in code but
are **converted to the table above before display**. Speed/accel controls
may use a `%` slider as long as the real max is shown in the unit above
(e.g. "Speed (%) · max 500 mm/s").

## Mandatory layout — every hardware web, regardless of device

Two zones are **always present**; only their hardware-specific contents
change. This is the design twin of the uniform `/v1` contract.

### Header (safety + lifecycle) — always

- Per-device **connection status** pills (ok / fault / idle).
- **Setup** (diagnose + initialize + tare in one), **Diagnose**, and a
  red **STOP** (emergency abort, confirm dialog). STOP is non-negotiable
  on any hardware.

### Body sections — all four mandatory

| Section | Always shows | Hardware-specific part |
|---------|--------------|------------------------|
| **Live** | per-device state (ready/busy/fault) + real-time readouts | which readouts |
| **Visualization** | a schematic/animation area | the device drawing (placeholder if none yet) |
| **Scenario** | build + run a command sequence; per-step replay | which step types |
| **History** | timestamped log of issued commands (newest first) | — |

Device **control tabs** (e.g. Balance / Pump / XZ stage) are the
hardware-specific surface and sit beside the mandatory sections. Every
card/tab title carries a lucide icon for at-a-glance scanning.

## Interaction rules

- **State-gated controls:** commands are disabled unless the relevant
  device is `ready` (not busy, not faulted, initialized as required).
- **Per-device busy:** an operation marks busy **only** the device(s) it
  occupies (a pump prime busies the pump, not the balance/stage).
- **Animations reflect real timing:** motion/fill durations are computed
  from the actual speed/accel profile and hardware spec sheets, so the
  visualization speed tracks the configured speed.

## Figma

This file is the **source of truth**; the Figma library mirrors it (colour
variables, Geist text styles, radius/spacing tokens, the component set, and
a page template encoding the mandatory layout). When code and Figma
disagree, **code wins** and Figma is updated to match.
