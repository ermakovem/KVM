# DPUSB2TypeC KVM Project Progress

This file records the project history step by step. Detailed future tasks are maintained in `PROJECT_PLAN.md`.

## Summary

Current stage: **schematic development / before PCB synchronization**.

Status by area as of 2026-07-15:

| Area | Status |
|---|---|
| System architecture | Defined |
| Power architecture | Partially implemented |
| MCU/AON | Partially implemented |
| Two laptop USB-C PD ports | Partially implemented |
| TPS26750 DC input | Nearly assembled; mandatory fixes remain |
| Downstream TPS65987D | Component added; connections not completed |
| DP mux/redriver | Components added; connections not completed |
| USB 2.0 hub/mux | Components added; connections not completed |
| PCB placement | Partially completed |
| PCB routing | Initial/incomplete stage |
| JLCPCB production package | Not started |
| Firmware | Architecture documented; implementation not started |

## Step 1. Define the device purpose

Status: **completed**.

- Defined the device as a KVM/USB-C docking station.
- Selected two laptop USB-C inputs, one DisplayPort output, and a shared USB 2.0 hub.
- Added a dedicated USB-C PD input.
- Identified the need for power-budget management and a local user interface.

## Step 2. Select the base architecture

Status: **completed at the concept level**.

- MCU: STM32G474RBT6.
- Input PD: TPS26750.
- Dual laptop PD: TPS65988.
- Future downstream Type-C PD: TPS65987D.
- Programmable VBUS/system rails: BQ25756; migration from the legacy TPS55288 is not finished.
- Always-on rail: LMR51610Y.
- Video path: TMUXHS4612 + PI2DPX2020.
- USB2 KVM: USB2514B + FSUSB42.
- PD-controller configurations will be loaded by the MCU without external SPI flash devices.

## Step 3. Design power and I2C architecture

Status: **partially completed**.

- Defined the always-on `+3V3_MCU` rail.
- Defined the switchable `+3V3_SYS` rail through AP2553.
- Built the base BQ25756 channel for system +5 V and started copying the architecture to the USB-C outputs.
- The current netlist contains three BQ25756 devices; three legacy TPS55288 devices still need to be replaced or removed.
- Separated the I2C buses for the PD controllers and converters.
- Partially built converter stages, current shunts, AON6278 MOSFET stages, inductors, and decoupling.

Open items:

- complete power budget;
- compensation and worst-case converter calculations;
- hardware OVP/eFuse verification;
- dedicated 1.8 V regulator for PI2DPX2020;
- startup and fault sequencing.

## Step 4. Build the EasyEDA schematic

Status: **active development**.

Created sheets:

1. `MCU`;
2. `2xUSB_PD`;
3. `USB_DCIN`;
4. `4xUSB2_HUB`;
5. `DP2.1_MUX_REDRIVE`.

As of 2026-07-11, the schematic contained 238 components. Every physical part had a footprint; the exceptions in MPN/LCSC fields were the logical `BOOT0_MCU` and `NRST_MCU` entries.

## Step 5. Netlist evolution

Status: **recorded**.

| Date | Components | Unique nets | Unconnected pins | Main change |
|---|---:|---:|---:|---|
| 2026-07-03 | 19 | 0 | 569 | Initial set of key ICs |
| 2026-07-04 | 50 | 20 | 605 | Added DC input, power, and USB-related parts |
| 2026-07-07 | 78 | 43 | Expanded power paths and passives |
| 2026-07-08 | 180 | 155 | Major expansion of PD, USB, and MCU blocks |
| 2026-07-09 | 196 | 143 | Added additional power paths |
| 2026-07-10 | 238 | 174 | Added protection, hub/DP blocks, and MCU support parts |
| 2026-07-11 | 238 | 174 | Corrected I2C pin mappings |
| 2026-07-15 | 372 | 223 | 281 | MCU, USB-C0 power path, and BQ25756 migration checkpoint |

Changes on 2026-07-11:

- TPS65988 `I2C1_SCL` pin 27 connected to `I2C_PD1_SCL`;
- TPS65988 `I2C1_SDA` pin 28 connected to `I2C_PD1_SDA`;
- STM32 `I2C_SYS_SDA/SCL` moved to PA6/PA7, pins 23/24.

## Step 6. Project handoff

Status: **completed**.

- Created a handoff document dated 2026-07-11.
- Recorded the architecture, accepted decisions, constraints, and open questions.
- Saved local TI, ST, and Diodes datasheets plus related application notes.

## Step 7. EasyEDA API audit

Status: **completed on 2026-07-11**.

Checked:

- active project `KVM`;
- one board, `Board1`;
- one schematic with five sheets;
- `PCB1` document;
- MPN, LCSC, and footprint fields;
- schematic/PCB correspondence;
- schematic and PCB DRC;
- layers and high-speed rule structures.

Result:

- schematic parts: 238;
- PCB components: 207;
- missing on PCB: 33;
- extra on PCB: 2 (`U20`, `U21`);
- schematic DRC: 1 error, 20 warnings;
- PCB DRC: 662 connection errors, 1 netlist error;
- active copper layers: 2;
- differential pairs: 0;
- net classes: 0;
- equal-length groups: 0.

## Step 8. Identify blocking schematic issues

Status: **audit completed; findings must be rechecked against the latest netlist**.

The 2026-07-11 audit found:

1. TPS26750 `POWER_PATH_EN` incorrectly connected to GND when the function was unused.
2. TPS26750 `NC` connected to GND.
3. TPS65988 `SPI_POCI` pulled up to 3.3 V instead of GND when no flash was used.
4. `BOOT0_MCU` and `NRST_MCU` required a decision on real footprints and BOM treatment.
5. U2, U4, U9, U10, and U14 were completely unconnected.

## Step 9. Preliminary JLCPCB/LCSC check

Status: **preliminary snapshot completed on 2026-07-11**.

- Most components are Extended Parts.
- Low stock was observed for TPS55288, PI2DPX2020, LMR51606, and TPS65988.
- The saved TMUXHS4612 and GT-USB-7013C items were not found in the catalog search used at the time.
- Final stock and substitute checks are deferred until the schematic is stable and must be repeated immediately before ordering.

## Step 10. Publish the project

Status: **completed**.

- Created a project README.
- Created a detailed project plan.
- Created a progress log.
- Published the EasyEDA source and netlist in the public repository.

## Step 11. Rework the MCU and USB-C0

Status: **checkpoint saved on 2026-07-15**.

- Replaced the MCU with STM32G474RBT6, providing four hardware I2C peripherals.
- Added the basic MCU support circuitry: power, VDDA, reset, BOOT0/DFU, and an optional DNP external crystal.
- Changed the always-on `+3V3_MCU` supply to LMR51610Y.
- Built USB-C0 around TPS26750 and TPD4S480.
- Added TPS26630 to the power path. EF0 enable is jointly controlled by MCU and PD0 through MOSFET logic.
- Documented the startup sequence: MCU/PD0 boot from raw VBUS, PD0 configuration and negotiation, MCU contract validation, EF0 enable, PGOOD wait, and system-rail startup.

## Step 12. Migrate to BQ25756

Status: **in progress**.

- Selected BQ25756RRVR as the common programmable four-switch buck-boost converter for system +5 V and USB-C outputs.
- Built a base power stage with external AON6278 MOSFETs, inductor, shunts, bootstrap/REGN components, and control pins.
- Created `BQ25756_MCU_CONTROL.txt` as a reference for the proposed MCU control flow.
- The current netlist contains three BQ25756 devices and three remaining TPS55288 devices; migration is not complete.
- Four identical channels, separate I2C routing, and another PD1/PD2 review are still required.

## Step 13. EasyEDA checkpoint

Status: **saved on 2026-07-15**.

- Saved the current `hardware/KVM.eprj2`.
- Created the portable backup `hardware/backups/KVM_2026-07-15-23-10.epro2`.
- Exported `hardware/netlists/Netlist_Schematic1_2026-07-15.enet`.
- The netlist contains 372 components, 223 named nets, and 281 unconnected pins.
- EasyEDA reported 1 schematic error and 19 warnings during export; they remain open.

## Next step

Complete the migration from the remaining TPS55288 devices to a fourth identical BQ25756 channel, then rework PD1/PD2 and their I2C/GPIO connections to the MCU. Export a new netlist and perform a full power-system review before synchronizing the PCB.
