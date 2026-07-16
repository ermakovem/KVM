# KVM Electrical Design Notes

Updated: 2026-07-16

This document is the implementation-level electrical record for the project. It contains accepted schematic connections, pin handling, component values, protection choices, datasheet corrections, and PCB-sensitive notes.

For port roles, system behavior, functional power and data paths, and control ownership, see [FUNCTIONAL_ARCHITECTURE.md](FUNCTIONAL_ARCHITECTURE.md).

If this document conflicts with an older progress note or control memo, the newest dated accepted decision in this document takes precedence for electrical implementation. A newer exported netlist may show that the schematic has not yet caught up with an accepted decision; report the mismatch without silently changing the decision.

## Working rule

- Consult this document before answering a pin-connectivity, component-value, protection, power-stage, schematic-review, or PCB-implementation question.
- Record accepted wiring and component changes here in the same work cycle.
- Keep system roles, behavior, sequencing, and ownership decisions in `FUNCTIONAL_ARCHITECTURE.md`.
- Mark unresolved implementation choices as **OPEN**.
- Use the newest user-provided netlist for checks. Do not use the EasyEDA API unless the user explicitly re-enables it.

## Current electrical snapshot

- Netlist: `Netlist_Schematic1_2026-07-16.enet`
- Export timestamp observed: 2026-07-16 23:08:21
- PD2 controller: `U4`, TPS65987DDHRSHR
- PD2 converter: `U1`, BQ25756RRVR
- USB-C3 protector: `U21`, TPD4S480RUKR

## BQ25756 common implementation

Status: **DECIDED / IN PROGRESS**

- The four BQ25756 channels use the same basic power-stage BOM where practical.
- USB-C output channels operate in reverse mode, with the common protected input bus on the SRN/SRP side and the individual USB-C output on the VAC/ACP/ACN side.
- Planned output range is 5 V to 20 V and up to 5 A per port, subject to the system power budget and thermal limits.
- `DRV_SUP` is connected to `REGN`.
- `ILIM_HIZ` remains populated as an independent hardware current limit.
- BQ25756 `INT#` is not used. Faults are handled by status polling and PD-controller policy.
- The fixed 7-bit BQ25756 I2C address is `0x6B`; two devices must not share the same physical I2C bus.

The older [BQ25756_MCU_CONTROL.txt](BQ25756_MCU_CONTROL.txt) remains the detailed register and MCU-control reference for the fixed system channel. USB-C1, USB-C2, and USB-C3 converters are owned by their PD controllers as defined in the functional architecture.

## PD1 / TPS65988 electrical implementation

Status: **DECIDED / IN PROGRESS**

### I2C pin assignment

| U24 pins | Electrical function | Connection |
|---|---|---|
| 27/28 | I2C1 SCL/SDA, master | U11 BQ25756 pins 1/2 for USB-C1 |
| 21/22 | I2C3 SCL/SDA, master | U12 BQ25756 pins 1/2 for USB-C2 |
| 32/33 | I2C2 SCL/SDA, target | MCU PD1 host bus |
| 34 | I2C2 IRQ | MCU PD1 IRQ |
| 23/29 | unused I2C IRQ pins | NC |

Each physical bus has its own pull-ups to `+3V3_PD1`.

### HPD implementation

- TPS65988 HPD1 and HPD2 each use a 100 kOhm pull-down to GND.
- TMUXHS4612 routes the DisplayPort connector HPD signal to the selected TPS65988 port.
- `SIDEBAND_SEL = 0` selects USB-C1/port A; `SIDEBAND_SEL = 1` selects USB-C2/port B.
- Final MCU pins for TMUXHS4612 `EN`, `D_AB_SEL`, and `SIDEBAND_SEL` are **OPEN**.

### VBUS surge protection

Place one Schottky clamp from each connector-side VBUS net to GND:

```text
GND -- anode | Schottky | cathode -- VBUS_Cx
```

- USB-C1: one clamp on `VBUS_C1`.
- USB-C2: one clamp on `VBUS_C2`.
- Selected part: DSK34, LCSC/JLCPCB `C727076`, SOD-123FL, 40 V, 3 A, 80 A non-repetitive surge rating.

## PD2 / TPS65987D electrical implementation

Status: **DECIDED / IN PROGRESS**

### Power-path pins

| U4 pins | Function | Connection |
|---|---|---|
| 3/4 and 13/14 | VBUS2 and VBUS1 | all tied together to `VBUS_C3` |
| 1/2 | active PP_HV2 path | `PPHV_VBUS3` |
| 11/12 | unused PP_HV1 path | GND |
| 7/52/56/57 | DRAIN2 | tied together as isolated `DRAIN2_PD2` copper only |
| 8/15/19/58 | DRAIN1 | tied together as isolated `DRAIN1_PD2` copper only |
| 59 | exposed ground pad | GND plane |

`DRAIN1_PD2` and `DRAIN2_PD2` must not connect to GND or to each other. Each group becomes a separate thermal copper island on the PCB.

### Supply and bypass implementation

| U4 pin/net | Local implementation |
|---|---|
| pin 5 `VIN_3V3` / `+3V3` | C101, 10 uF to GND |
| pin 9 `LDO_3V3` / `+3V3_PD2` | C108, 10 uF to GND |
| pin 35 `LDO_1V8` / `+1V8_PD2` | C109, 4.7 uF to GND |
| pin 25 `PP_CABLE` / `+5V` | C114, 4.7 uF to GND |
| `VBUS_C3` | C119, 4.7 uF to GND, plus connector-side 10 nF capacitors |
| `PPHV_VBUS3` | C120, 4.7 uF to GND, plus the BQ25756 output network |
| pins 24/26 `CC1_C3`/`CC2_C3` | C116/C117, 220 pF to GND |

### Boot straps and address

| Function | Implementation | Result |
|---|---|---|
| ADCIN1 pin 6 | R80 100 kOhm to `+3V3_PD2`, R82 10 kOhm to GND | divider ratio 0.091 |
| SPI_POCI pin 36 | R83 10 kOhm to `+3V3_PD2` | logic high boot strap |
| boot behavior | ADCIN1 ratio 0.091 with SPI_POCI high | `BP_NoResponse`, Safe Configuration |
| ADCIN2 pin 10 | R79 100 kOhm to `+3V3_PD2`, R81 10 kOhm to GND | address decode `000b`, default I2C1 address `0x20` |

The MCU supplies the TPS65987D patch and application configuration after boot; no external SPI flash is used.

### I2C and reset

| U4 pins | Function | Connection |
|---|---|---|
| 27/28 | I2C1 SCL/SDA, target | `MCU_PD2_SCL/SDA`; each has a 10 kOhm pull-up to `+3V3_PD2` |
| 29 | I2C1 IRQ | `MCU_PD2_IRQ`, 10 kOhm pull-up to `+3V3_PD2`; currently reaches MCU PA2 |
| 21/22 | I2C3 SCL/SDA, master | `PD2_BQ3_SCL/SDA` to U1 pins 1/2; 10 kOhm pull-ups to `+3V3_PD2` |
| 23 | I2C3 IRQ/GPIO7 | NC; BQ25756 interrupt is not used |
| 32/33 | unused I2C2 SCL/SDA | separate 10 kOhm pull-ups to `+3V3_PD2` |
| 34 | unused I2C2 IRQ | NC |
| 44 | HRESET | `HRESET_PD2`, 100 kOhm pull-down to GND; currently reaches MCU PC3 |

**OPEN:** `MCU_PD2_SCL` and `MCU_PD2_SDA` currently terminate at U4 and their pull-ups; they do not yet reach MCU pins. Final MCU I2C peripheral and pin assignment is unresolved.

### GPIO and unused-pin handling

| U4 pin | Function | Required handling |
|---|---|---|
| 16 | GPIO0 | NC |
| 17 | GPIO1 | R31, 1 MOhm to GND |
| 18 | GPIO2 | NC |
| 30 | GPIO3/HPD | NC; USB-C3 has no DisplayPort function |
| 31 | GPIO4 | NC |
| 36 | GPIO8/SPI_POCI | boot-strap pull-up described above |
| 37/38/39 | GPIO9/SPI_PICO, GPIO10/SPI_CLK, GPIO11/SPI_CS | GND |
| 40/41/42/43 | GPIO12/GPIO13/GPIO14/GPIO15 | NC |
| 48/49 | GPIO16/PP_EXT1 and GPIO17/PP_EXT2 | NC; external power-path GPIO control is not used |
| 50/53 | GPIO18/C_USB_P and GPIO19/C_USB_N | NC; TPS65987D BC1.2 function is not used |
| 54/55 | GPIO20/GPIO21 | NC |

### USB-C3 VBUS surge protection

- D21 is DSK34, LCSC/JLCPCB `C727076`, SOD-123FL.
- D21 cathode connects to `VBUS_C3`; its anode connects to GND.
- The part is rated for 40 V reverse voltage, 3 A average current, and 80 A non-repetitive surge current.
- Place it close to the U4 VBUS1/VBUS2 connection with a short ground return.

### TPS65987D Figure 9-1 correction

TPS65987D Figure 9-1 incorrectly shows the unused `VBUS2` pins grounded. TI has confirmed that the reference-schematic connection is an error. `VBUS1` and `VBUS2` must be tied together even when only one `PP_HV` path is used. Ground the unused system-side `PP_HV` pins, not the corresponding `VBUS` pins.

## TPD4S480 implementation boundary

- USB-C1, USB-C2, and USB-C3 use TPD4S480 protection.
- USB-C3 is an SPR-only port; EPR is disabled for that port.
- Detailed TPD4S480 pin and capacitor review remains a separate electrical task.
- USB-C0 protection is reviewed separately because it uses TPS26750 plus TPS26630 and is not electrically equivalent to the TPS65987D/TPS65988 source ports.

## Open electrical items

1. Connect `MCU_PD2_SCL` and `MCU_PD2_SDA` to final MCU I2C-capable pins.
2. Add and verify the selected Schottky clamps on `VBUS_C1` and `VBUS_C2`.
3. Complete the final MCU GPIO allocation for reset, IRQ, faults, mux selection, port enables, UI, display, and encoder.
4. Verify that no disabled rail can be back-powered through I2C, IRQ, GPIO, CC, or USB signals.
5. Complete detailed TPD4S480, USB2514B, FSUSB42, TMUXHS4612, and PI2DPX2020 electrical reviews.
6. Define final fault wiring and responses for TPD4S480, BQ25756, TPS26630, and the PD controllers.

## Electrical decision log

### 2026-07-16 - Separate electrical implementation from functional architecture

Pin mappings, values, exact nets, protection choices, datasheet corrections, and PCB notes are maintained in this document. `FUNCTIONAL_ARCHITECTURE.md` contains only system behavior and functional relationships.

### 2026-07-16 - TPS65987D VBUS inputs are tied together

For U4, both VBUS1 pins and both VBUS2 pins connect to `VBUS_C3`. PP_HV2 is the active system-side path to `PPHV_VBUS3`; unused PP_HV1 is grounded.

### 2026-07-16 - DSK34 selected for TPS65987D/TPS65988 VBUS surge clamps

DSK34 (`C727076`) is the selected compact Schottky clamp. One part is required on each connector-side VBUS net for USB-C1, USB-C2, and USB-C3.

### 2026-07-16 - PD2 unused-pin handling recorded

Unused TPS65987D GPIOs are left NC except GPIO1, which uses a 1 MOhm pull-down. Unused SPI data, clock, and chip-select pins are grounded while SPI_POCI remains a boot strap.

## Updating this document

When an electrical decision is accepted:

1. update the relevant implementation section;
2. add a dated electrical decision entry when the change affects future reviews;
3. mark replaced decisions explicitly;
4. verify the change against the next exported netlist;
5. commit it with the corresponding schematic snapshot only after user approval.
