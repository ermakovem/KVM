# KVM Functional Architecture and Design Decisions

Updated: 2026-07-16

This document is the system-level functional authority for the project. It records port roles, functional power and data paths, control ownership, operating sequences, and architectural decisions.

It intentionally does not describe pin numbers, component values, detailed schematic nets, protection-part choices, or PCB implementation. Those details are maintained in [ELECTRICAL_DESIGN_NOTES.md](ELECTRICAL_DESIGN_NOTES.md).

If this document conflicts with an older progress note or control memo, the newest dated functional decision in this document takes precedence.

## Working rule

- Consult this document before answering a question about system behavior, port roles, power or data flow, sequencing, policy, or controller ownership.
- Update this document immediately when a functional or architectural decision changes.
- Update `ELECTRICAL_DESIGN_NOTES.md` instead when a change concerns wiring, pins, component values, protection, datasheet implementation, or PCB layout.
- Mark unresolved architectural choices as **OPEN**.
- A schematic or netlist may temporarily differ from the accepted architecture. Keep the accepted functional decision here and report the implementation mismatch separately.

## Status legend

- **DECIDED** - use this architecture in the schematic and firmware.
- **IN PROGRESS** - the architecture is accepted, but implementation or verification is incomplete.
- **OPEN** - do not treat the current implementation as final.

## Top-level functional diagram

```text
                         DEDICATED POWER INPUT

 USB-C0
   |
   +-- input protection and PD negotiation
   |
   +-- always-on startup power --> MCU + PD0
   |
   +-- protected common power bus
          |
          +--> fixed system +5 V
          |
          +--> USB-C1 programmable VBUS
          |
          +--> USB-C2 programmable VBUS
          |
          +--> USB-C3 programmable VBUS


                            DATA AND VIDEO PATHS

 USB-C1 -- DP Alt Mode --\
                          +--> DP selector/redriver --> DisplayPort output
 USB-C2 -- DP Alt Mode --/

 USB-C1 -- USB 2.0 --\
                       +--> upstream selector --> USB hub
 USB-C2 -- USB 2.0 --/                         |
                                                 +--> USB-A ports
                                                 +--> USB-C3 USB 2.0

 USB-C3: charging + USB 2.0 only; no DisplayPort path.
```

## USB-C port roles

### USB-C0 - dedicated system power input

Status: **DECIDED / IN PROGRESS**

- PD controller: TPS26750 (`PD0`).
- Primary power role: Sink.
- Intended input range: USB PD from the initial 5 V state through EPR profiles, up to the project limit of 48 V.
- Provides the always-on startup power for the MCU and PD0 before the main protected power bus is enabled.
- The main system power bus remains isolated until the negotiated input contract has been validated.
- USB 2.0 D+/D- are reserved for MCU programming and service, not for the KVM hub path.

### USB-C1 and USB-C2 - laptop-facing KVM ports

Status: **DECIDED / IN PROGRESS**

- Shared dual-port PD controller: TPS65988 (`PD1`).
- Primary power role: Source, used to charge connected laptops.
- Data role: USB device/UFP toward the selected laptop.
- DisplayPort role: DP Sink/UFP_D; both ports can receive DP Alt Mode from laptops.
- The port policy may operate as DRP/Try.SRC so a partner that insists on being a power Source can still attach. In that case the KVM acts as a nominal Sink without using the partner as a system power source.
- Each port has an independent programmable VBUS converter and an independent PD-controller power path.
- Only one laptop at a time is selected for the USB hub and DisplayPort output paths.

### USB-C3 - downstream charging and USB 2.0 port

Status: **DECIDED / IN PROGRESS**

- PD controller: TPS65987D (`PD2`).
- Power role: Source.
- Data role: USB Host/DFP through the shared USB 2.0 hub.
- Function: charging and USB 2.0 only.
- DisplayPort Alt Mode: disabled.
- One independent programmable converter generates its VBUS supply.

## Power-up sequence

Status: **DECIDED; final fault timing still requires verification**

```text
1. USB-C0 is connected and initially presents 5 V.
2. The always-on regulator starts the MCU and PD0.
3. The MCU keeps the main protected power bus disabled.
4. The MCU loads the required patch and application configuration into PD0.
5. PD0 negotiates the input USB-PD contract.
6. The source changes the input VBUS to the negotiated voltage.
7. PD0 reports the active contract to the MCU.
8. The MCU validates voltage, current, available power, and policy.
9. The MCU permits the main protected power path to turn on.
10. The MCU waits for the protected bus to become valid.
11. The MCU enables the fixed system rails.
12. Output PD profiles and output converters are enabled according to the
    available input power budget.
```

The main protected power path must not be enabled merely because the initial 5 V is present. The MCU validates the negotiated contract first. PD0 retains the ability to force shutdown during a PD or protection fault.

## Output power-conversion model

Status: **DECIDED / IN PROGRESS**

- Four BQ25756 channels are used: one for the fixed system supply and one for each output USB-C port.
- The three USB-C output channels generate programmable VBUS from the common protected input bus.
- Allowed USB-C output range: 5 V to 20 V.
- Maximum planned output current: 5 A per port, limited by the input-power budget and thermal capability.
- The MCU calculates the total available output budget from the accepted USB-C0 contract.
- PD1 and PD2 advertise only output profiles permitted by the current budget.

### Converter ownership

| Channel | Function | Functional controller |
|---|---|---|
| U30 | Fixed system +5 V | MCU |
| U11 | USB-C1 VBUS | PD1 / TPS65988 |
| U12 | USB-C2 VBUS | PD1 / TPS65988 |
| U1 | USB-C3 VBUS | PD2 / TPS65987D |

## Control and communication architecture

### MCU and PD0

Status: **DECIDED**

```text
MCU
  +--> configures and monitors PD0
  +--> controls the fixed system converter
  +--> calculates the shared output-power budget
```

### MCU and PD1

Status: **DECIDED**

```text
MCU --> PD1 TPS65988
          +--> controls USB-C1 converter
          +--> controls USB-C2 converter
          +--> negotiates both laptop-port PD contracts
```

The two output converters use independent physical control buses because they have the same fixed device address.

### MCU and PD2

Status: **DECIDED; final MCU peripheral assignment remains OPEN**

```text
MCU --> PD2 TPS65987D
          +--> controls USB-C3 converter
          +--> negotiates the USB-C3 source contract
```

## Control responsibilities

### MCU

- Boots from the always-on supply.
- Loads patches and application configurations into the PD controllers.
- Reads the accepted input contract from PD0.
- Calculates the total available output-power budget.
- Enables the main protected power path only after validating the input contract.
- Controls the fixed system converter and switched system rails.
- Supplies permitted PDO and power-policy information to PD1 and PD2.
- Coordinates USB 2.0 and DisplayPort KVM selection.

### PD0

- Negotiates the dedicated input power contract.
- Reports the active PDO/RDO and fault state to the MCU.
- Participates in main power-path shutdown control.
- Does not directly enable the main load before the input contract is validated.

### PD1

- Independently negotiates USB-PD contracts for USB-C1 and USB-C2.
- Advertises only profiles allowed by the MCU power budget.
- Programs the USB-C1 and USB-C2 converters in response to attach, detach, reset, and negotiated-profile events.
- Enables a port power path only after the corresponding converter output is valid.
- Participates in the laptop selection policy for USB and DisplayPort.

### PD2

- Negotiates the source contract for USB-C3.
- Programs the USB-C3 converter.
- Provides charging and USB 2.0 policy only.
- Does not enter DisplayPort Alt Mode.

## USB 2.0 architecture

Status: **ARCHITECTURE DECIDED; implementation incomplete**

- USB-C1 and USB-C2 are alternative upstream connections to the USB2514B hub.
- FSUSB42 selects which laptop is connected to the hub upstream port.
- USB-C3 is a downstream USB 2.0 port from the hub.
- USB-C0 is not part of the switched KVM USB path.
- USB-C1, USB-C2, and USB-C3 data paths pass through their protection devices before reaching the selector or hub.

## DisplayPort architecture

Status: **PARTIALLY DECIDED**

- Only USB-C1 and USB-C2 participate in DP Alt Mode.
- Their DP inputs are selected by TMUXHS4612 and routed through PI2DPX2020 to the full-size DisplayPort output.
- USB-C3 has no DP lanes and no HPD connection.
- TMUXHS4612 port A is assigned to USB-C1 and port B is assigned to USB-C2.
- The TMUXHS4612 sideband switch routes the monitor HPD signal only to the selected laptop-port PD controller input.
- Selection of the USB upstream port, DP lanes, and HPD path must describe the same active laptop.

## Fault and power-budget behavior

Status: **PARTIALLY DECIDED**

- The MCU is the system-level power-budget authority.
- A port must not advertise power that exceeds the currently available input budget.
- A converter or PD fault must disable the affected output path without depending on normal host software timing.
- Loss or reduction of the input contract requires the MCU to recalculate permitted output profiles and reduce or disable loads safely.
- Exact fault ownership, timing, and recovery behavior remain open for final definition.

## Open architectural decisions

1. Final output PDO tables for USB-C1, USB-C2, and USB-C3.
2. Behavior when the available input power changes while output contracts are active.
3. Complete USB hub, upstream selector, DP selector, and redriver integration.
4. Final coordination sequence for changing the active laptop.
5. Final fault ownership and recovery policy across the MCU, PD controllers, protection devices, and converters.
6. User-interface behavior for port status, selection, and fault reporting.

## Functional decision log

### 2026-07-16 - PD1 owns both laptop-port converters

TPS65988 controls the independent USB-C1 and USB-C2 converters. The MCU communicates with PD1 as the system host and supplies the permitted power policy.

### 2026-07-16 - PD2 owns the USB-C3 converter

TPS65987D controls the USB-C3 converter and remains a target of the MCU host.

### 2026-07-16 - USB-C3 has no DisplayPort function

USB-C3 is only a charging and USB 2.0 downstream port. DisplayPort Alt Mode is disabled.

### 2026-07-16 - Laptop ports may support a nominal Sink role

USB-C1 and USB-C2 may use a DRP/Try.SRC policy so a DP-capable partner that insists on sourcing power can still attach. The KVM does not use that partner as its system power source.

### 2026-07-16 - TMUXHS4612 selects laptop DisplayPort and HPD paths

TMUXHS4612 port A belongs to USB-C1 and port B belongs to USB-C2. DP lane selection and HPD sideband selection follow the same active-laptop state.

### 2026-07-16 - Functional and electrical documentation are separated

This document contains architecture and behavior only. Electrical implementation is maintained in `ELECTRICAL_DESIGN_NOTES.md`.

## Updating this document

When a functional decision is accepted:

1. update the relevant architecture section;
2. add a dated functional decision entry when the change affects future behavior;
3. mark replaced decisions explicitly;
4. update `PROJECT_PLAN.md` and `PROJECT_PROGRESS.md` at the next milestone;
5. commit documentation with the corresponding project snapshot only after user approval.
