# Lab architecture — Levels, Composition, and Phases

The target system the CommonClaude projects build into: a self-driving
electrochemistry lab. Three distinct concepts; keep them separate.
(Counts are the current plan; TBD items will firm up.)

## Level (L) — control-code depth of a device/subsystem

How deep the control stack is built for a given device. Built **per need**
— not every device reaches L1 or L2.

| Level | What | Example |
|-------|------|---------|
| **L0** | Hardware-driving **driver** library | `sy01b`, `entris_ii`, `picus2`, `mks_motor` |
| **L1** | **Server** — expose the driver over HTTP | FastAPI `/v1` (sy01b-server) |
| **L2** | **ESP touchscreen / physical UI** | ESP32-S3-BOX-3 LVGL touch UI |

L0 is the baseline every device ships. L1/L2 are added only where remote /
web / on-device control is actually wanted.

## Composition — how units combine (a SEPARATE axis from Level)

Build small, compose up; each tier is itself a "device-shaped unit." This
is orthogonal to Level — it is about wiring multiple things together, not
about control depth.

```
device instance   one physical unit (its own USB port)
      | compose
cell               heterogeneous instances for one co-located function
                   (e.g. 3 motors + 1 pump); owns its real-time safety
                   interlocks IN-PROCESS
      | compose
Phase-system       the cells/devices on one NUC, composed into a working
                   stage (see Phase below)
      | network
whole SDL          Phases coordinated across NUCs (networked substrate)
```

- **One USB port per device unit** (expanded via USB hubs). No RS-485 /
  CAN multidrop or shared-bus addressing — chosen for debugging simplicity
  (each unit isolatable on its own port). Trade-off: more adapters/ports.
  Units are identified by USB identity (`VID:PID`, or `VID:PID:SERIAL` when
  several share a `VID:PID`).
- **Real-time safety stays in-process within a cell** (e.g. paired-motor
  interlock: one motor faults → stop the whole gantry now). Never route a
  safety interlock over the network.

## Phase — SDL hardware stage

Which hardware, on which NUC. The whole SDL is built in two phases, each on
its **own NUC**; cross-phase coordination is networked.

### Phase 1 (NUC A)

| Cell | Devices |
|------|---------|
| Dispensing cell ×3 | each = 3 MKS motors (XYZ cartesian) + 1 syringe pump |
| Weighing cell ×1   | linear motor + electronic balance |
| Robot arm ×1       | **Phase-level (shared)** — transfers between cells |

Totals: 9 MKS motors, 3 syringe pumps, 1 linear motor, 1 balance, 1 arm.
The 4 cells + arm compose into the Phase-1 orchestrator (one NUC).

### Phase 2 (NUC B)

1 MKS motor, 1 syringe pump, 1 robot arm, plus 3–4 further devices (TBD).

### Robot arms (external / senior-owned)

Two arms, added later by a different owner: arm #1 is used **before Phase 1
starts**, arm #2 **at the end of Phase 2**. Their driver code follows a
different coding style. **Mitigation: wrap each arm behind a thin adapter
that satisfies the standard cell / `/v1` interface.** The foreign style
stays isolated behind the adapter; conversion happens only at that
boundary, so neither side has to adopt the other's style.

## Coordination substrate (OPEN decision)

- **Within a cell** → in-process Python calls (real-time, safe).
- **Across cells / across NUCs** → a networked substrate. Candidates:
  - **ROS 2** — best fit for arms + multi-axis motion + cameras; Actions
    give long-running goals with feedback/cancel (pipetting, arm moves);
    node-per-device; real-time capable. Heavier / learning curve.
  - **MQTT broker** — lighter pub/sub + last-will; telemetry + commands
    are easy, long-running-action semantics must be built by hand.
- Decide before wiring cross-cell / cross-NUC composition. Until then,
  compose cells in-process on a single NUC.

## Why incremental + uniform contract

Building one device/cell at a time is safe **because** every unit follows
the same CommonClaude contract (`/v1` shape, error envelope, lifecycle).
Uniformity is what keeps the later "compose into one class" cheap; the one
risk of incremental work — interface drift — is prevented by the standard.
