# KVM Functional Architecture — Revision 2

Updated: 2026-07-18

This is the second revision of the functional architecture. It describes the
device as a product and then defines its six functional blocks. The original
[`FUNCTIONAL_ARCHITECTURE.md`](FUNCTIONAL_ARCHITECTURE.md) is retained as the
first revision and is not replaced by this document.

Electrical pin connections, component values, protection circuits, and PCB
implementation details remain outside the scope of this document.

## 1. Overall Device Description

### 1.1 Purpose and operating concept

The device is a desktop KVM and USB-C docking station for two computers or
laptops. It allows the user to:

- connect two computers through USB-C;
- select either computer as the active KVM input;
- send DisplayPort video from the active computer to one shared DisplayPort
  output;
- connect shared USB 2.0 peripherals to the active computer;
- charge both connected computers and one additional USB-C device;
- manually select which USB-C output receives charging priority;
- power the complete system from a dedicated USB-C PD 3.1 EPR input;
- distribute the available input power between the system and output ports.

The device has eight external ports, one round display, and one rotary encoder
with an integrated push button.

The active KVM selection controls the DisplayPort and USB 2.0 paths together.
Video and shared USB peripherals must always be connected to the same selected
computer.

### 1.2 External ports and user interfaces

| Port | Primary purpose | Data capability | Power capability |
|---|---|---|---|
| USB-C0 | Dedicated system power input | — (D+/D− are used for MCU programming) | USB PD 3.1 EPR, up to 48 V / 5 A / 240 W |
| USB-C1 | First KVM computer | DisplayPort Alt Mode and USB 2.0 | Source, up to 20 V / 5 A / 100 W |
| USB-C2 | Second KVM computer | DisplayPort Alt Mode and USB 2.0 | Source, up to 20 V / 5 A / 100 W |
| USB-C3 | Fast charging for a phone, tablet, or other USB-C device | USB 2.0; no DisplayPort | Fixed PDOs and PPS, up to 20 V / 5 A / 100 W |
| USB-A1 | Shared USB peripheral | USB 2.0 | 5 V / 1.5 A |
| USB-A2 | Shared USB peripheral | USB 2.0 | 5 V / 1.5 A |
| USB-A3 | Shared USB peripheral | USB 2.0 | 5 V / 1.5 A |
| DisplayPort | Shared monitor output | DisplayPort 2.1 target | No external load power |

USB-A1 uses one single-port connector. USB-A2 and USB-A3 use one stacked
dual-port connector but remain independent USB ports.

The round display and encoder are external user-interface elements, not data or
power ports.

### 1.3 USB-C0 input power

USB-C0 is the only normal power input for the KVM. The target input capability
is USB PD 3.1 EPR up to 48 V, 5 A, and 240 W.

PD0 always requests the highest valid power level offered by the connected
source, limited by the 240 W project maximum. It does not reduce the requested
input contract merely because the present output load is small.

If the source later reduces or loses the active input contract, the MCU must
detect the change and safely reduce the system load. The exact shedding,
renegotiation, and shutdown policy is **OPEN**.

### 1.4 USB-C1 and USB-C2 KVM ports

USB-C1 and USB-C2 connect the two computers. Each port supports:

- DisplayPort Alt Mode;
- USB 2.0 data;
- charging up to 20 V, 5 A, and 100 W;
- an independent USB-PD contract and independent VBUS converter.

Only one port is the active KVM input at a time. Selecting a port switches both
the DisplayPort path and the USB hub upstream path to that computer.

The inactive computer can continue charging. Under the default power-allocation
policy it has lower priority than the active KVM computer, so its charging rate
can be reduced when the available input power is limited.

#### DisplayPort lane modes

The preferred USB-C DisplayPort configuration is **VESA Pin Assignment C**:

- four high-speed lanes carry DisplayPort;
- USB 3.x is not carried on the high-speed lanes;
- USB 2.0 remains available on its dedicated D+/D− pins;
- the video-path target is DisplayPort 2.1 up to UHBR20.

The device must also accept **VESA Pin Assignment D**:

- two high-speed lanes carry DisplayPort;
- the other high-speed lanes are assigned to USB 3.x by the computer;
- the KVM uses only the two DisplayPort lanes;
- the USB 3.x lane pair is intentionally not used;
- USB 2.0 remains available through the HUB block.

If the computer supports both assignments, the KVM prefers Pin Assignment C.
The KVM does not contain a USB 3.x hub and does not switch or expose USB 3.x
data.

### 1.5 USB-C3 fast-charging port

USB-C3 is primarily a fast-charging port for a phone, tablet, laptop, or other
USB-C device. It supports fixed USB-PD output profiles and PPS from 5 V to
20 V, up to 5 A and 100 W.

Its secondary function is a USB 2.0 downstream port from the shared HUB block.
USB-C3 does not support DisplayPort Alt Mode or USB 3.x data.

### 1.6 USB-A ports

The three USB-A ports are independent USB 2.0 downstream ports. Each port is
allocated 5 V at up to 1.5 A.

The USB-A ports switch between USB-C1 and USB-C2 together with the DisplayPort
video path. When the active KVM input changes, the MCU changes both the MUX
selection and the HUB upstream selection as one coordinated operation.

For power budgeting, the USB-A ports are treated as a fixed worst-case load,
not as participants in the USB-C charging-priority algorithm:

```text
3 × 5 V × 1.5 A = 22.5 W reserved for USB-A ports
```

### 1.7 DisplayPort output

The full-size DisplayPort connector outputs the video path selected from
USB-C1 or USB-C2. The MUX block carries the DisplayPort Main Link, AUX, and HPD
signals. The video-path target is DisplayPort 2.1 up to UHBR20.

### 1.8 Display and encoder

The final information and menu layout shown on the round display is **OPEN**.

The encoder must provide two independent user selections:

1. active KVM input: USB-C1 or USB-C2;
2. charging-priority port: USB-C1, USB-C2, or USB-C3.

By default, the active KVM port is also the highest-priority charging port. The
user can change charging priority without changing the active KVM input.

### 1.9 Power allocation

The MCU calculates the USB-C output budget from the maximum active USB-C0
contract. Before distributing power to USB-C1, USB-C2, and USB-C3, it reserves:

- the internal consumption of the KVM;
- estimated conversion losses;
- a system power margin;
- 22.5 W for the three USB-A ports.

```text
USB-C0 contract power
    − internal KVM consumption
    − conversion losses
    − system power margin
    − 22.5 W USB-A reservation
    = USB-C1/C2/C3 charging budget
```

The exact system power margin remains **OPEN**.

The remaining USB-C charging budget is assigned in the following order:

| Selection state | First priority | Second priority | Third priority |
|---|---|---|---|
| Default | Active KVM port | Inactive KVM port | USB-C3 |
| USB-C1 selected as charging priority | USB-C1 | USB-C2 | USB-C3 |
| USB-C2 selected as charging priority | USB-C2 | USB-C1 | USB-C3 |
| USB-C3 selected as charging priority | USB-C3 | Active KVM port | Inactive KVM port |

The 100 W figures are per-port maximum capabilities. They do not guarantee that
all three USB-C outputs can provide 100 W simultaneously.

## 2. Functional Device Description

The internal architecture is divided into exactly six logical blocks:

1. PD0;
2. PD1;
3. PD2;
4. MUX;
5. MCU;
6. HUB.

Power converters and protection devices belong to the logical block that owns
their function. In particular, the fixed `+5V` converter belongs to PD0,
although it is configured and controlled by the MCU.

### Functional block and connector ownership diagram

```text
                  [Display] [Encoder]
                       \       /
                        \     /
                     +-----------+
                     |    MCU    |
                     | system    |
                     | control   |
                     +--+--+--+--+
                        |  |  |
        control/status  |  |  |  control/status
              +---------+  |  +--------------------+
              |            |                       |
              v            v                       v
[USB-C0] -> +------+    +--------+             +--------+ <- [USB-C3]
            | PD0  |    |  PD1   |             |  PD2   |
            |      |    |        |             |        |
            |input |    |C1 + C2 |             |C3 PD + |
            |PD +  |    |dual PD |             |PPS +   |
            |eFuse |    |+ BQ1/2 |             |BQ3     |
            |+ 5V  |    +---+----+             +---+----+
            +--+---+        |                      |
               |            | DP lanes/AUX         | USB 2.0
               |            v                      |
               |        +--------+                 |
               |        |  MUX   |                 |
               |        |DP mux +|                 |
               |        |redriver|                 |
               |        +---+----+                 |
               |            |                      |
               |            v                      |
               |      [DisplayPort]                |
               |                                   |
               |          USB 2.0 from C1/C2       |
               |                    \               |
               |                     v              v
               |                  +-------------------+
               |                  |        HUB        |
               |                  | upstream selector |
               |                  | + four-port hub   |
               |                  +--+------+------+
               |                     |      |      |
               |                     v      v      v
               |                 [USB-A1][USB-A2][USB-A3]
               |
               +--> protected power bus and +5V rail
                    power the downstream functional blocks
```

External interface ownership is defined as follows:

| Functional block | Owned external connectors and interfaces |
|---|---|
| PD0 | USB-C0 |
| PD1 | USB-C1 and USB-C2 |
| PD2 | USB-C3 |
| MUX | Full-size DisplayPort output |
| MCU | Round display, encoder, and MCU programming through USB-C0 D+/D− |
| HUB | USB-A1, USB-A2, and USB-A3 |

USB-C3 physically belongs to PD2, while its D+/D− data pair is handled by the
HUB. USB-C1 and USB-C2 physically belong to PD1, while their DisplayPort lanes
go to the MUX and their USB 2.0 pairs go to the HUB.

### 2.1 PD0

#### Owned interface

- USB-C0.

#### Purpose and internal scope

PD0 is the complete input-power block. It includes:

- USB-C0;
- connector protection;
- the TPS26750 input PD controller;
- the always-on +3.3 V converter;
- the input eFuse and protected power path;
- the fixed BQ25756 system converter that generates +5 V.

The fixed +5 V BQ25756 is controlled by the MCU but functionally belongs to
PD0 because it is part of the input and system-power path.

#### Data and control interfaces

- USB-C0 CC1 and CC2;
- host I²C connection with the MCU;
- IRQ to the MCU;
- reset from the MCU;
- active input PDO/RDO and contract status to the MCU;
- eFuse enable, power-good, current-monitor, and fault information;
- MCU control bus to the fixed +5 V BQ25756.

USB-C0 D+/D− bypass the PD controller and connect to the MCU programming
interface.

#### Power interfaces

PD0 receives USB-C0 VBUS and produces:

- always-on `+3V3_MCU` for the MCU and PD0 startup domain;
- the protected common high-voltage power bus;
- the fixed `+5V` rail.

#### Responsibilities

- negotiate the maximum valid input contract;
- boot the MCU and PD0 from the initial USB-C0 supply;
- keep the main load isolated until the input contract is accepted;
- protect and enable the common power bus;
- generate the fixed `+5V` rail under MCU control;
- report contract, power-good, current-monitor, and fault state to the MCU;
- safely remove the main power path after a critical input fault.

### 2.2 PD1

#### Owned interfaces

- USB-C1;
- USB-C2.

#### Purpose and internal scope

PD1 is the dual laptop-facing USB-C block. It includes the TPS65988 controller,
the two laptop-port protection and power paths, and two independent BQ25756
converter channels.

#### Data and control interfaces

- CC1 and CC2 for USB-C1 and USB-C2;
- host I²C, IRQ, and reset connections with the MCU;
- one independent BQ control bus for USB-C1;
- one independent BQ control bus for USB-C2;
- DisplayPort Main Link and AUX from both connectors to the MUX;
- HPD from the MUX;
- USB 2.0 D+/D− from both connectors to the HUB.

#### Power interfaces

PD1 receives:

- system logic power;
- the protected common high-voltage power bus.

PD1 produces and controls:

- `VBUS_C1` up to 20 V / 5 A;
- `VBUS_C2` up to 20 V / 5 A;
- the independent output power path for each laptop port.

#### Responsibilities

- negotiate independent contracts on USB-C1 and USB-C2;
- advertise only profiles allowed by the MCU power budget;
- configure each port's BQ25756;
- enable VBUS only after the corresponding converter is ready;
- enter and maintain DisplayPort Alt Mode;
- support Pin Assignment C and Pin Assignment D;
- report attachment, power request, contract, role, and fault status to the MCU.

### 2.3 PD2

#### Owned interface

- USB-C3.

#### Purpose and internal scope

PD2 is the fast-charging and USB 2.0 block for USB-C3. It includes the TPS26750
controller, connector protection, the USB-C3 BQ25756 converter, and the
reverse-current-blocking output path.

#### Data and control interfaces

- USB-C3 CC1 and CC2;
- host I²C, IRQ, and reset connections with the MCU;
- BQ control bus to the USB-C3 BQ25756;
- control and status signals for the USB-C3 output path;
- USB-C3 D+/D− connection to the HUB.

#### Power interfaces

PD2 receives:

- system logic power;
- the protected common high-voltage power bus.

PD2 produces and controls `VBUS_C3` from 5 V to 20 V, up to 5 A and 100 W.

#### Responsibilities

- negotiate USB-C3 fixed and PPS output contracts;
- advertise only profiles allowed by the MCU power budget;
- configure the USB-C3 BQ25756;
- enable the output path only after the requested converter output is valid;
- prevent reverse current from USB-C3 into the system;
- coordinate the USB 2.0 data role;
- report attachment, request, contract, and fault status to the MCU.

### 2.4 MUX

#### Owned interface

- full-size DisplayPort output.

#### Purpose and internal scope

MUX selects the DisplayPort path from USB-C1 or USB-C2 and carries it to the
shared monitor output. It includes the high-speed selector and DisplayPort
redriver.

#### Data and control interfaces

- two DisplayPort Main Link inputs from USB-C1 and USB-C2;
- two corresponding AUX paths;
- one Main Link and AUX output to the DisplayPort connector;
- HPD from the monitor back to the selected PD1 port;
- selection, enable, reset, and configuration from the MCU;
- status or fault information to the MCU where supported.

#### Power interfaces

- `+3V3`;
- a dedicated 1.8 V redriver supply, with its final net name still open;
- any required local auxiliary rails.

#### Responsibilities

- switch the DisplayPort Main Link, AUX, and HPD together;
- follow the same active-computer selection as the HUB;
- carry both four-lane and two-lane DisplayPort configurations;
- provide the required high-speed signal conditioning.

### 2.5 MCU

#### Owned interfaces

- round display;
- rotary encoder and push button;
- MCU programming through USB-C0 D+/D−;
- hardware recovery and test interfaces.

#### Purpose and internal scope

The MCU is the system-level policy and sequencing authority. It loads the PD
controller configurations, validates the input contract, controls system power,
selects the active KVM input, and allocates the available output power.

#### Data and control interfaces

- separate host interfaces to PD0, PD1, and PD2;
- control interface to the fixed +5 V BQ25756 in PD0;
- input eFuse and system-rail control and monitoring;
- MUX selection, enable, reset, and configuration;
- HUB upstream selection, reset, and configuration;
- power-good, current-monitor, fault, and IRQ inputs;
- display and encoder interfaces;
- USB DFU/programming interface.

#### Power interfaces

The MCU is powered from the always-on `+3V3_MCU` rail. It must remain operational
while `+5V`, `+3V3`, and the other downstream rails are disabled.

#### Responsibilities

- perform the complete startup and shutdown sequences;
- configure and monitor PD0, PD1, and PD2;
- read the maximum active USB-C0 contract;
- reserve internal, loss, margin, and USB-A power;
- allocate the remaining power according to charging priority;
- choose the active KVM input;
- switch the MUX and HUB together;
- control the fixed `+5V` converter;
- respond to faults and input-contract changes;
- implement the display and encoder user interface.

### 2.6 HUB

#### Owned interfaces

- USB-A1;
- USB-A2;
- USB-A3.

The HUB also handles the USB 2.0 data pair of USB-C3, although the USB-C3
connector itself belongs to PD2.

#### Purpose and internal scope

HUB selects USB-C1 or USB-C2 as the upstream computer and distributes USB 2.0
to three USB-A ports and USB-C3. It includes the upstream USB selector, the
four-port USB hub, and downstream port-power switches.

#### Data and control interfaces

- USB-C1 D+/D− upstream input;
- USB-C2 D+/D− upstream input;
- selected upstream pair to the USB2514B;
- three downstream D+/D− pairs to USB-A1/A2/A3;
- one downstream D+/D− pair to USB-C3;
- selection, reset, and configuration from the MCU;
- overcurrent and fault status to the MCU.

#### Power interfaces

- `+3V3` for hub and selector logic;
- `+5V` for downstream USB power;
- up to 5 V / 1.5 A reserved for each USB-A port.

PD2, not the HUB, controls the USB-C3 VBUS power path.

#### Responsibilities

- select the active USB-C1 or USB-C2 upstream host;
- match the active DisplayPort selection;
- distribute USB 2.0 data to all four downstream ports;
- control and protect USB-A power;
- report downstream overcurrent and fault conditions to the MCU.

### Data rails

```text
USB-C0:
    CC1/CC2 --------------------------> PD0
    D+/D− ----------------------------> MCU programming

USB-C1:
    CC1/CC2 --------------------------> PD1
    DP Main Link + AUX ---------------> MUX
    D+/D− ----------------------------> HUB upstream selector

USB-C2:
    CC1/CC2 --------------------------> PD1
    DP Main Link + AUX ---------------> MUX
    D+/D− ----------------------------> HUB upstream selector

USB-C3:
    CC1/CC2 --------------------------> PD2
    D+/D− <---------------------------> HUB downstream port

PD0 <--------- host I²C / IRQ / reset ---------> MCU
PD1 <--------- host I²C / IRQ / reset ---------> MCU
PD2 <--------- host I²C / IRQ / reset ---------> MCU

MCU ---------- select / enable / config --------> MUX
MCU ---------- select / reset / config ---------> HUB

PD1 ---------- converter control ---------------> BQ C1 + BQ C2
PD2 ---------- converter control ---------------> BQ C3
MCU ---------- converter control ---------------> fixed +5 V BQ in PD0

MUX ---------- selected DP Main Link + AUX ------> DisplayPort
DisplayPort -- HPD ------------------------------> MUX -> selected PD1 port

HUB ---------- USB 2.0 --------------------------> USB-A1
HUB ---------- USB 2.0 --------------------------> USB-A2
HUB ---------- USB 2.0 --------------------------> USB-A3
HUB ---------- USB 2.0 --------------------------> USB-C3

Encoder / display <------------------------------> MCU
```

### Power rails

```text
USB-C0 VBUS
    |
    +--> PD0 always-on regulator
    |        |
    |        +--> +3V3_MCU
    |                  +--> MCU
    |                  +--> PD0 startup domain
    |
    +--> PD0 input PD negotiation and eFuse
             |
             +--> protected common high-voltage bus
                       |
                       +--> PD0 fixed +5 V BQ (controlled by MCU)
                       |         |
                       |         +--> +5V
                       |                   +--> USB-A power: 22.5 W reserved
                       |                   +--> +3V3
                       |                   +--> dedicated 1.8 V redriver rail
                       |
                       +--> PD1 BQ channel 1
                       |         +--> VBUS_C1
                       |
                       +--> PD1 BQ channel 2
                       |         +--> VBUS_C2
                       |
                       +--> PD2 BQ channel
                                 +--> reverse-blocking output path
                                           +--> VBUS_C3

+3V3:
    +--> PD1 logic
    +--> PD2 logic
    +--> MUX logic
    +--> HUB logic

Dedicated 1.8 V rail (final net name OPEN):
    +--> DisplayPort redriver
```
