# DPUSB2TypeC KVM

An open-source hardware project for a desktop KVM/docking device with two laptop USB-C inputs, one full-size DisplayPort output, a shared USB 2.0 hub, and managed USB-C Power Delivery.

> **Status:** active development. The project is not yet ready for manufacturing or PCBA ordering.

## Project goals

The device is intended to:

- accept DisplayPort Alt Mode from two laptops over USB-C;
- switch the selected video stream to one full-size DisplayPort output;
- switch a shared USB 2.0 hub between the laptops;
- charge the connected laptops through USB-C Power Delivery;
- receive power from a dedicated USB-C PD input;
- manage the available power budget;
- provide a local interface using an STM32 MCU, a round GC9A01 display, and an encoder.

The upper target for the video path is DisplayPort 2.1/UHBR20. Achieving it depends on the final stackup, controlled impedance, channel loss, and signal-integrity verification.

## Main components

| Function | Component |
|---|---|
| MCU and system control | STM32G474RBT6 |
| USB-C PD input sink | TPS26750 |
| Two laptop USB-C PD ports | TPS65988 |
| Additional downstream USB-C PD port | TPS65987D |
| Programmable four-switch buck-boost converters | BQ25756 |
| Always-on 3.3 V supply | LMR51610Y |
| Input eFuse / power path | TPS26630 |
| DP/USB4 high-speed mux | TMUXHS4612 |
| DP/USB4 linear redriver | PI2DPX2020 |
| USB 2.0 hub | USB2514B |
| USB 2.0 2:1 mux | FSUSB42 |
| CC/SBU protection | TPD4S480 |

## Repository structure

```text
docs/
  FUNCTIONAL_ARCHITECTURE.md — functional diagram and accepted design decisions
  ELECTRICAL_DESIGN_NOTES.md — accepted schematic and electrical implementation notes
  PROJECT_PLAN.md         — completed work and remaining tasks
  PROJECT_PROGRESS.md     — chronological, step-by-step project progress
  BQ25756_MCU_CONTROL.txt — reference for controlling BQ25756 from the MCU
hardware/
  KVM.eprj2               — current EasyEDA project source
  backups/                — portable EasyEDA milestone backups
  netlists/               — exported netlists
  schematics/             — schematic sheet images for quick viewing
README.md
```

## Opening the project

1. Install EasyEDA Professional Edition.
2. Open `hardware/KVM.eprj2` as a local project.
3. If the main file cannot be opened, import the newest file from `hardware/backups/`.
4. After making changes, export a new netlist and synchronize the PCB with the schematic.

## Current verification status

As of 2026-07-16:

- the current netlist contains 372 components, 223 named nets, and 281 unconnected pins;
- the MCU has been changed to STM32G474RBT6, with its basic support circuitry and control-signal assignment in place;
- the USB-C0 input power path is built around TPS26750, TPD4S480, and TPS26630, with joint enable control from the MCU and PD controller;
- the always-on converter has been changed to LMR51610Y;
- migration of the output converters to BQ25756 is still in progress: the netlist contains three BQ25756 devices and three legacy TPS55288 devices;
- the latest schematic DRC reported during netlist export is 1 error and 19 warnings; these must be resolved before PCB synchronization;
- the PCB is not synchronized with the actively changing schematic;
- the project must not be submitted to JLCPCB until the schematic, PCB update, routing, and all required reviews are complete.

See [FUNCTIONAL_ARCHITECTURE.md](docs/FUNCTIONAL_ARCHITECTURE.md) for the current architecture and accepted decisions, [ELECTRICAL_DESIGN_NOTES.md](docs/ELECTRICAL_DESIGN_NOTES.md) for implementation details, [PROJECT_PLAN.md](docs/PROJECT_PLAN.md) for the work plan, and [PROJECT_PROGRESS.md](docs/PROJECT_PROGRESS.md) for the project history.

## JLCPCB manufacturing

Before the first order, the following outputs and checks must be complete and error-free:

- schematic ERC/DRC;
- schematic-to-PCB synchronization;
- PCB DRC;
- Gerber and drill files;
- BOM with current LCSC part numbers;
- CPL/Pick-and-Place file;
- assembly drawings;
- stackup and controlled-impedance requirements;
- separate high-speed, power-integrity, thermal, and mechanical reviews.

LCSC availability must be checked immediately before ordering. Some critical parts already have low stock or cannot be found under the saved LCSC number.

## Warning

This is an unfinished engineering project. The USB-C PD power paths can operate at elevated voltages and currents. An incorrect schematic or PCB layout may damage connected laptops, the monitor, the power source, or the board itself. Do not use the current production files without an independent engineering review.

## License

No project license has been selected yet. Until a `LICENSE` file is added, standard copyright remains with the author.
