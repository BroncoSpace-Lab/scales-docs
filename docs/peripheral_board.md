# SCALES Peripheral Board
By John Pollak

## Usage Guide

The `periph_ethernet_switch` SCALES Peripheral Board is a low-footprint unmanaged Gigabit Ethernet switch. It is intended to provide a local Ethernet network between Ethernet-capable SCALES boards, payloads, and development hardware.

The switch design is based on the Microchip `KSZ9896CTXI`. In the SCALES implementation, five external 1 Gb Ethernet copper ports are routed to integrated-magnetics RJ45 connectors. The sixth switch port is the chip's digital MAC-side interface and is not exposed as an RJ45 port in this design.

Key repositories:

- [BroncoSpace-Lab/scales-hardware](https://github.com/BroncoSpace-Lab/scales-hardware)  
  Hardware source repository for the SCALES hardware.
- [SCALES Peripheral Ethernet Switch KiCad Project](https://github.com/BroncoSpace-Lab/scales-hardware/tree/main/peripheral_board/peripheral_board_v1/periph_ethernet_switch)  
  KiCad schematic, PCB, and project-local libraries for the current peripheral Ethernet switch.

### 1. Set the Strap Switches

The switch must be strapped before power is applied. The board includes a 7-position DIP switch labeled `SW1` for the LED strap-in signals.

Set the strap switches as follows:

| Strap Signal | Required Position |
| --- | --- |
| `LED5_1` | Off |
| `LED3_1` | Off |
| `LED2_0` | Off |
| `LED2_1` | On |
| `LED4_0` | Off |
| `LED4_1` | Off |
| `LED1_1` | Off |

`LED2_1` must be set to **On** to select normal link-up / auto-negotiation. All other strap switches should be set to **Off** for the current unmanaged-switch configuration.

!!! warning

    Set the strap switches before powering the board. The KSZ9896 samples strap pins during reset/power-up, so changing switch positions after boot may not change the active configuration until the board is reset or power-cycled.

### 2. Enable Peripheral Board Power

When the Peripheral Board is wired into the SCALES Compute Module, its input power is controlled by the Compute Module power-sequencing circuitry. The [fprime-scales-ref](https://github.com/BroncoSpace-Lab/fprime-scales-ref) deployment must be installed and running on the SCALES Compute Module so the Peripheral subsystem power rail can be enabled.

1. Before booting the SCALES Compute Module, ensure that a DF11 cable harness has been created and connected between the SCALES Compute Module `CN1` connector and the Peripheral Board. Use the schematics from [scales-hardware](https://github.com/BroncoSpace-Lab/scales-hardware) to build and verify the harness.
2. Boot the SCALES Compute Module.
3. Confirm the `fprime-scales-ref` deployment is running.
4. Allow the deployment to enable the Peripheral subsystem power rail.

If the board is powered outside the full SCALES stack, connect the external harness through `CN1` and provide the board input supply on the `VESP` pins. `VESP` is the 5 V input to the Peripheral Board.

### 3. Connect Ethernet Devices

1. Use CAT5E or better Ethernet cables.
2. Connect Ethernet-capable boards, payloads, or development devices to any of the five RJ45 ports.
3. Apply power or reset the board after the strap switches are set.
4. Wait for link LEDs to indicate link status.
5. Use the IP configuration defined by the connected devices. The switch itself is unmanaged and does not assign IP addresses.

### 4. Use the Board as an Unmanaged Switch

No firmware load is required for the standard SCALES use case. After the strap configuration is sampled, the KSZ9896 switch forwards Ethernet frames in hardware.

Typical use:

1. Power the Peripheral Board through `CN1`.
2. Connect each Ethernet endpoint to one of the five RJ45 ports.
3. Configure IP addresses on the connected endpoints as needed.

### 5. Optional Management Header

The schematic includes a 1x4 management header, `J6`, with the following signals:

| Pin | Signal |
| --- | --- |
| 1 | `SDO` |
| 2 | `SDI/SDA/MDIO` |
| 3 | `SCL/MDC` |
| 4 | TODO: verify connection. `SCS_N` is present in the schematic, but the parsed header net did not confirm it on `J6` pin 4. |

This header is not required for the normal unmanaged-switch use case.

---

## Hardware Overview

The Peripheral Board contains the following major blocks:

- Microchip `KSZ9896CTXI` Gigabit Ethernet switch
- Five 1 Gb Ethernet RJ45 ports with integrated magnetics
- Strap-in configuration network for unmanaged operation
- Local power conversion for switch core, analog, and I/O rails
- DF11 board-to-board / harness connector for power and status signals
- Optional management header for switch-side control/debug signals

### Board Images

Peripheral Board Front
![Peripheral Board Front](<Images/perif-front.png>){ style="display:block; margin:0 auto; max-width:600px; width:60%; height:60%;" }

Peripheral Board Back
![Peripheral Board Back](<Images/perif-back.png>){ style="display:block; margin:0 auto; max-width:600px; width:60%; height:60%;" }


---

## Component Selection

The current board is built around components already represented in the KiCad schematic and project-local libraries.

- [Microchip KSZ9896CTXI 6-Port Gigabit Ethernet Switch](https://www.microchip.com/en-us/product/KSZ9896)
- [Bel Fuse 0826-1G1T-23-F Integrated-Magnetics RJ45 Connector](https://jlcpcb.com/partdetail/BelFuse-0826_1G1T_23F/C5467573)
- [Hirose DF11-16DP-2DSA(08) Connector](https://lcsc.com/product-detail/Wire-To-Board-Wire-To-Wire-Connector_HRS-Hirose-HRS-DF11-16DP-2DSA-08_C530981.html)
- [Texas Instruments TLV62569PDDCR Buck Regulator](https://lcsc.com/product-detail/Pre-ordered-Chips_Texas-Instruments-Texas-Instruments-TLV62569PDDCR_C398365.html)
- [Coilcraft XAL4020-222MEC 2.2 uH Power Inductor](https://www.lcsc.com/product-detail/C3151182.html?s_z=n_XAL4020-222)
- [74LVC1G14 Schmitt Inverter](https://www.ti.com/lit/sg/scyt129e/scyt129e.pdf)
- [Microchip VXM7-9032-25M0000000 25 MHz Clock Source](https://www.lcsc.com/product-detail/C1517995.html?s_z=n_VXM7-9032-25M0000000)
- [7-Position DIP Switch](https://www.lcsc.com/product-detail/C559142.html)
- [Panasonic EVPAA602W Reset Button](https://lcsc.com/product-detail/Tactile-Switches_PANASONIC_EVPAA602W_Switch-3-5-2-9-1-7Plastic-head-3-5N-0-15mm-SMD_C79148.html)
- [Ferrite Bead](https://www.lcsc.com/product-detail/C85840.html?s_z=n_BLM21PG221SN1D)

### KSZ9896 Ethernet Switch

The `KSZ9896CTXI` is a six-port Gigabit Ethernet switch. In this design:

- Ports 1 through 5 are routed to RJ45 connectors.
- Each RJ45 connector includes the Ethernet magnetics inside the connector body.
- Port 6 is the switch's digital MAC-side interface and is not used as an external RJ45 port.
- The switch is configured through strap pins at reset for unmanaged operation.
- The switch performs Layer 2 Ethernet forwarding in hardware after link negotiation.

The switch learns MAC addresses from incoming frames and forwards traffic to the appropriate destination port. If the destination is unknown, broadcast, or multicast, the frame is flooded according to normal Ethernet-switch behavior. IP assignment is still handled by the connected endpoints; the switch does not run DHCP or assign addresses.

### Integrated-Magnetics RJ45 Ports

Each external Ethernet port uses a Bel Fuse `0826-1G1T-23-F` RJ45 connector with integrated magnetics. This reduces the number of discrete magnetics on the PCB and keeps the high-speed port layout compact.

Each port sheet includes:

- Four differential copper pairs: `A`, `B`, `C`, and `D`
- Connector-side center-tap and termination/filtering support
- Two LED signals from the switch:
  - `LEDx_1` indicates link status
  - `LEDx_0` indicates activity status

### Impedance and Length Matching

The PCB is implemented as a 6-layer board:

| Layer | Purpose |
| --- | --- |
| `F.Cu` | Top signal |
| `In1.Cu` | Ground |
| `In2.Cu` | Power |
| `In3.Cu` | Signal |
| `In4.Cu` | Ground |
| `B.Cu` | Bottom mixed / ground |

The Ethernet `TXRX` routes are treated as differential pairs. The KiCad PCB contains diff-pair tuning entries on the `TXRX` nets with:

- `0.13 mm` trace width
- Approximately `0.20 mm` differential-pair gap for the length-tuned sections
- `50 mm` target length on many tuned differential pairs
- `49.9 mm` to `50.1 mm` target-length window
- Additional skew-tuning entries on selected pairs
---

## DF11 Connector Breakout

`CN1` is a Hirose DF11 16-pin connector used for board input power, ground return, and status/control breakout.

| Pin | Signal | Notes |
| --- | --- | --- |
| 1 | `VPG2.5` | 2.5 V power-good/status signal, TODO verify active level |
| 2 | `GND` | Ground return |
| 3 | `VPG1.2` | 1.2 V power-good/status signal, TODO verify active level |
| 4 | `GND` | Ground return |
| 5 | `VPG3.3` | 3.3 V power-good/status signal, TODO verify active level |
| 6 | `GND` | Ground return |
| 7 | `GND` | Ground return |
| 8 | `GND` | Ground return |
| 9 | `VESP` | 5 V Peripheral Board input supply |
| 10 | `VESP` | 5 V Peripheral Board input supply |
| 11 | `VESP` | 5 V Peripheral Board input supply |
| 12 | `VESP` | 5 V Peripheral Board input supply |
| 13 | `VESP` | 5 V Peripheral Board input supply |
| 14 | `VESP` | 5 V Peripheral Board input supply |
| 15 | `VESP` | 5 V Peripheral Board input supply |
| 16 | `VESP` | 5 V Peripheral Board input supply |

!!! warning

    TODO: Verify DF11 mating-cable orientation and pin-1 convention before using this table as a harness build document.

---

## Power Architecture

The Peripheral Board receives 5 V `VESP` from the DF11 connector and locally generates the rails required by the KSZ9896 switch. In the integrated SCALES system, `VESP` is not always present immediately at boot; it is enabled by the SCALES Compute Module power-sequencing logic after the `fprime-scales-ref` deployment starts.

Local rails:

- `+3.3V`
- `+2.5V`
- `+1.2V`
- `+2.5VA`
- `+1.2VA`

These local rails are internal to the Peripheral Board design and should only be used for the circuitry shown in the `periph_ethernet_switch` schematics. They should not be used as general-purpose power outputs or to power external hardware.

The schematic uses three `TLV62569PDDCR` buck converters with 2.2 uH `XAL4020-222MEC` inductors. Ferrite beads separate the analog supply rails from the main digital rails where required by the switch design.

Last updated on 7/7/2026
