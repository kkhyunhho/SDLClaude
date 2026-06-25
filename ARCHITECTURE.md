# Lab architecture — composition hierarchy & deployment topology

This is the target system the CommonClaude projects compose into: a
self-driving electrochemistry lab. It records the agreed structure so each
device/cell is built to slot in. (Counts are the current plan; TBD items
will firm up.)

## Composition hierarchy — build small, compose up

Each level is a "device-shaped unit" the next level composes. Build and
HIL-verify each level standalone, then compose — never big-bang.

```
L0  Device driver   one class per device TYPE (sy01b, mks_motor, entris_ii,
                    picus2, linear_motor, arm, heater, camera). Instantiable
                    per physical unit.
        | compose
L1  Cell / station  composes HETEROGENEOUS device instances for one
                    co-located function (e.g. 3 motors + 1 pump). Owns that
                    cell's real-time safety interlocks IN-PROCESS.
        | compose
L2  Stage system    composes the cells on ONE NUC into a working stage.
        | network
L3  Lab (future)    coordinates stages across NUCs over a network substrate.
```

- **One USB port per device unit** (expanded via USB hubs). No RS-485 /
  CAN multidrop or shared-bus addressing — chosen for debugging simplicity
  (each unit isolatable on its own port). Trade-off: more adapters/ports.
  Units are identified by USB identity (`VID:PID`, or `VID:PID:SERIAL` /
  port when several share a `VID:PID`).
- **Real-time safety stays in-process within a cell** (e.g. paired-motor
  interlock: one motor faults → stop the whole gantry now). Never route a
  safety interlock over the network.
- **A cell is composable**: it may expose a `/v1` server (or substrate
  node) and be driven by the stage above, exactly like a single device.

## Deployment topology — two stages, two NUCs

The system is built in two stages, each on its **own NUC**. Cross-stage
coordination (L3) is networked.

### Stage 1 (NUC A)

| Cell | Devices |
|------|---------|
| Dispensing station ×3 | each = 3 MKS motors (XYZ cartesian) + 1 syringe pump |
| Weighing station ×1   | linear motor + electronic balance |
| Robot arm ×1          | **stage-level (shared)** — transfers between stations |

Totals: 9 MKS motors, 3 syringe pumps, 1 linear motor, 1 balance, 1 arm.
The 4 cells + arm compose into the Stage-1 orchestrator.

### Stage 2 (NUC B)

1 MKS motor, 1 syringe pump, 1 robot arm, plus 3–4 further devices (TBD).

## Coordination substrate (OPEN decision)

- **Within a cell** → in-process Python calls (real-time, safe).
- **Across cells / across NUCs** → a networked substrate. Candidates:
  - **ROS 2** — best fit for arms + multi-axis motion + cameras; Actions
    give long-running goals with feedback/cancel (pipetting, arm moves);
    node-per-device; real-time capable. Heavier / learning curve.
  - **MQTT broker** — lighter pub/sub + last-will; telemetry + commands
    are easy, long-running-action semantics must be built by hand.
- Decide before building L2/L3. Until then, compose cells in-process on a
  single NUC.

## Why incremental + uniform contract

Building one device/cell at a time is safe **because** every unit follows
the same CommonClaude contract (`/v1` shape, error envelope, lifecycle).
Uniformity is what keeps the later "compose into one class" cheap; the one
risk of incremental work — interface drift — is prevented by the standard.
