# DPUSB2TypeC KVM Project Plan

Updated: 2026-07-15

## Legend

- `[x]` — completed and verified at the current project stage.
- `[~]` — started or partially completed.
- `[ ]` — not completed.
- `BLOCKER` — prevents manufacturing or the next project stage.

## Completed work

### Architecture

- [x] Defined the overall KVM/docking-station architecture.
- [x] Defined the roles of the two laptop USB-C ports: Power Source + USB Data UFP + DP Sink/UFP_D.
- [x] Selected a dedicated USB-C PD power input.
- [x] Selected STM32G474RBT6 as the central controller with four hardware I2C peripherals.
- [x] Chosen to load PD-controller patches and configurations from MCU flash without separate SPI flash devices.
- [x] Separated the I2C buses for PD controllers and programmable converters.
- [x] Selected the updated power architecture based on LMR51610Y, TPS26630, BQ25756, and AP2553.
- [x] Selected the main video and USB components.

### Schematic

- [x] Created the EasyEDA project `KVM`.
- [x] Created five sheets: `MCU`, `2xUSB_PD`, `USB_DCIN`, `4xUSB2_HUB`, and `DP2.1_MUX_REDRIVE`.
- [~] Added STM32G474, TPS26750, TPS65988, TPS65987D, TPS26630, three BQ25756 devices, the remaining legacy TPS55288 devices, and the main power paths.
- [x] Added USB-C, USB-A, and DisplayPort connectors plus TPD4S480/TPD1E0B04 protection.
- [x] Corrected the TPS65988 `I2C1_SCL/SDA` connections in the 2026-07-11 netlist.
- [x] Corrected the STM32 `I2C_SYS` pin selection to PA6/PA7, pins 23/24.
- [~] Built the USB-C0 block around TPS26750, TPD4S480, and TPS26630. The startup and power-path sequence is documented but still needs final verification.
- [~] Partially built the dual-port TPS65988 block and two laptop power-converter channels.
- [~] Added MCU power, reset, BOOT0, USB DFU, an optional DNP external crystal, and four I2C buses. The GPIO map is not finished.

### PCB and verification

- [x] Created the PCB1 document.
- [~] Transferred 207 components to the PCB.
- [x] Performed an EasyEDA API audit and recorded the project state.
- [x] Compared the schematic, PCB, and latest netlist.
- [x] Checked the local datasheets and preliminary LCSC availability of critical components.
- [x] Created the public project documentation structure.

## Remaining work

> The 2026-07-15 snapshot is current. Findings carried over from the 2026-07-11 audit must be rechecked against the latest netlist before they are changed.

### Stage 1. Resolve blocking schematic issues

- [ ] `BLOCKER` TPS26750 pin 20 `POWER_PATH_EN`: remove the GND connection; leave it floating if unused, or implement the recommended buffer/power-path circuit.
- [ ] `BLOCKER` TPS26750 pin 21 `NC`: remove the GND connection.
- [ ] `BLOCKER` TPS65988 pin 36 `SPI_POCI`: when no SPI flash is used, replace the pull-up with the required GND connection.
- [ ] Obtain the complete schematic DRC list and resolve every error and warning. The latest export reported 1 error and 19 warnings.
- [ ] Finalize `BOOT0_MCU` and `NRST_MCU`: use real buttons/test pads or exclude placeholder symbols from the BOM and PCB.
- [ ] Check every NC, reserved, thermal-pad, and unused pin against the datasheets.
- [ ] Verify that I2C/GPIO connections cannot back-power disabled domains from the always-on domain.

Completion criterion: schematic ERC/DRC has no unexplained errors, and every intentional warning is documented.

### Stage 2. Complete unfinished functional blocks

- [ ] `BLOCKER` Fully connect TPS65987D (`U4`): power, CC, VBUS/PP_HV, cable power, I2C, reset, protection, and policy.
- [ ] `BLOCKER` Fully connect TMUXHS4612 (`U2`): lanes, AUX/DDC, HPD, power, and control inputs.
- [ ] `BLOCKER` Fully connect PI2DPX2020 (`U10`): 1.8 V supply, decoupling, I2C/address, enable, AUX/SBU, and high-speed lanes.
- [ ] Select and calculate a dedicated 1.8 V regulator from +5 V for PI2DPX2020.
- [ ] `BLOCKER` Fully connect USB2514B (`U9`): power, crystal/clock, reset, RBIAS, upstream/downstream USB2, straps/SMBus, and port-power control.
- [ ] `BLOCKER` Fully connect FSUSB42 (`U14`) between the laptop D+/D− lines and the USB2514B upstream port.
- [ ] Complete the USB-A and downstream USB-C power paths, OCS/FAULT handling, and MCU override.
- [ ] Complete the HPD path and define switching behavior.
- [ ] Complete the MCU GPIO map: enable, reset, fault, IRQ, HPD, mux selection, display, and encoder.

Completion criterion: every functional block has power, decoupling, control, and a complete signal path; unused pins are handled according to the datasheet.

### Stage 3. Verify power and USB-C PD

- [ ] Create a worst-case power tree and startup/shutdown sequencing table.
- [ ] Define the allowed input profiles and verify the entire TPS26750/TPD4S480/TPS26630 USB-C0 path for the claimed range up to 48 V.
- [ ] Verify hardware OVP/eFuse protection between the input and downstream converters.
- [ ] Replace the remaining TPS55288 devices with BQ25756 and obtain four identical, controllable channels.
- [ ] Calculate each BQ25756 channel: MOSFET and inductor currents, shunts, ILIM, MOSFET losses, output capacitors, and thermal margin.
- [ ] Check DC-bias derating for all MLCCs, especially on the 20 V VBUS rails.
- [ ] Calculate the `+3V3_MCU` and `+3V3_SYS` budgets and confirm LMR51610Y/AP2553 margin.
- [ ] Verify discharge, reverse-current, dead-battery, and fault behavior for every Type-C port.
- [ ] Document PDO/APDO tables and the power-allocation algorithm.

Completion criterion: the power tree operates in all 5/9/15/20/28/36 V modes without exceeding absolute or recommended limits.

### Stage 4. Synchronize the PCB with the schematic

- [ ] `BLOCKER` Import the current schematic changes into the PCB.
- [ ] Add the 33 missing parts or document each intentional exclusion.
- [ ] Remove or replace legacy `U20` and `U21` after verifying that their functions are no longer required.
- [ ] Make the schematic and PCB netlists match exactly.
- [ ] Generate an updated BOM after synchronization.

Completion criterion: EasyEDA DRC no longer reports `PCB and schematic netlist does not match`.

### Stage 5. Stackup and high-speed rules

- [ ] `BLOCKER` Do not use the current two-layer stackup for the final DP2.1 board.
- [ ] Select a manufacturable JLCPCB stackup, provisionally six or more layers.
- [ ] Obtain controlled-impedance trace geometry from JLCPCB for the selected stackup.
- [ ] Create net classes for power, USB2, I2C/GPIO, DP AUX, and high-speed lanes.
- [ ] Create every DP/USB differential pair with correct polarity.
- [ ] Define impedance, width/gap, clearance, via, and length/skew constraints.
- [ ] Document allowed polarity swaps and lane mapping in the schematic and firmware configuration.

Completion criterion: every high-speed net has explicit rules and the manufacturer has confirmed the stackup.

### Stage 6. Placement and routing

- [ ] Finalize board dimensions, connector positions, and enclosure constraints.
- [ ] Verify every connector footprint and pin number against the official drawing and a physical sample.
- [ ] Place ESD/OVP devices as close as possible to the connectors.
- [ ] Place decoupling components according to datasheet recommendations.
- [ ] Place buck-boost power stages with minimum hot-loop area.
- [ ] Route the DP2.1/USB high-speed channel with continuous return paths and a minimum number of transitions.
- [ ] Route USB2, I2C, clocks, and control signals.
- [ ] Route power planes, ground, thermal copper, and stitching vias.
- [ ] Configure and repour copper zones.

Completion criterion: zero unrouted connections and no PCB-rule violations.

### Stage 7. Engineering review

- [ ] Complete PCB DRC.
- [ ] Independent schematic review.
- [ ] Signal-integrity review for DP2.1, AUX/HPD, and USB2.
- [ ] Power-integrity and voltage-drop review.
- [ ] Thermal review of converters, MOSFETs, shunts, connectors, and PD controllers.
- [ ] EMC/ESD review and pre-compliance test plan.
- [ ] Mechanical/3D collision review.
- [ ] Verify test points and the factory programming/recovery interface.

Completion criterion: every blocking finding is resolved or explicitly accepted and documented.

### Stage 8. Prepare JLCPCB PCB+PCBA files

- [ ] Recheck stock, MOQ, and lifecycle for every LCSC part.
- [ ] Find production substitutes for unavailable TMUXHS4612 and GT-USB-7013C parts, or arrange them as consigned parts.
- [ ] Reduce the number of unique Extended Parts where doing so does not degrade the design.
- [ ] Export Gerber/drill files and inspect them in a Gerber viewer.
- [ ] Export BOM and CPL files with matching designators.
- [ ] Verify rotations, layers, and package orientations in the JLCPCB viewer.
- [ ] Prepare an assembly drawing and DNP list.
- [ ] Order a small prototype revision first.

Completion criterion: JLCPCB BOM/CPL validation has no unmatched or conflicting parts.

### Stage 9. Bring-up and firmware

- [ ] Implement safe MCU startup, watchdog handling, and fault logging.
- [ ] Implement TPS26750/TPS65988/TPS65987D patch and configuration loading with timeout, retry, and CRC handling.
- [ ] Implement BQ25756 configuration and safe output sequencing on separate I2C buses.
- [ ] Implement the PD power-budget manager and port policy.
- [ ] Implement KVM switching: USB mux, DP mux/redriver, and HPD.
- [ ] Implement the display/encoder UI.
- [ ] Prepare factory-test firmware and a bring-up checklist.
- [ ] Verify USB DFU and SWD recovery.

## Conditions for the first prototype order

The board may be considered ready for its first prototype only when all of the following are true:

- [ ] The schematic is complete and checked against the datasheets.
- [ ] The schematic and PCB netlists match.
- [ ] The PCB has zero unrouted connections.
- [ ] PCB DRC contains no unexplained errors.
- [ ] A multilayer controlled-impedance stackup has been selected and confirmed.
- [ ] High-speed rules and differential pairs are configured.
- [ ] The BOM and CPL validate successfully at JLCPCB.
- [ ] Power, thermal, SI, EMC/ESD, and mechanical reviews are complete.
- [ ] A bring-up plan, current-limited power-up procedure, and measurement points are available.
