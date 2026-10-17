# Sentinel Hardware

| Path | Contents |
|---|---|
| `PCB/` | LibrePCB project (schematic + board). Source of truth. Current revision exports in `3D/` (STEP), `Board/` and `Schematic/` (PDF). |
| `Fabrication/` | Outputs per board revision, `vX.Y/`: Gerbers zip, BOM, CPL, schematic and PCB print PDFs (JLCPCB). |
| `Enclosure/` | Mechanical: Onshape exports (STEP for editing, STL/3MF for printing). |
| `Datasheets/` | Key component datasheets, or `links.md` pointing to vendor URLs. |
| `pcb_changelog.txt` | Board revision history. |

## Naming

`Sentinel_vX.Y_BOM.xlsx`, `Sentinel_vX.Y_Schematic_Print.pdf`, `Sentinel_vX.Y_PCB_Print.pdf`, `Sentinel_vX.Y_Gerber.zip`, `Sentinel_vX.Y_CPL.csv`, `Sentinel_vX.Y.step`.

Files exported before the rename keep their `BTTM_vX.Y_*` names; the next export of each uses the new name.

Tag each released board in git as `hw-vX.Y`.

## Core components (main unit, v1.4)

| Function | Part | Interface | Notes |
|---|---|---|---|
| MCU | ESP32-S3-WROOM-1-N16R8 | - | 16 MB quad flash, 8 MB octal PSRAM |
| Display | 320x240 TFT (ILI9341) | SPI | |
| GNSS | u-blox MAX-F10S-00B | UART | |
| CAN | TI TCAN3413DR | TWAI | ESD2CAN24 protection; termination is external, on J12 |
| TPMS receiver | TI CC1101RGPR | SPI | 433 MHz, receive-only |
| Storage | microSD | SPI | |
| PSU | Maxim MAX25223 | - | 3-36 V DC input |

## License

Hardware design files are licensed under CC BY-SA 3.0 (https://creativecommons.org/licenses/by-sa/3.0/).
