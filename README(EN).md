# Final Electronics 2 and Electronic Design 1 Project - PCIe to NVMe M.2 SSD Switch

<p align="center">
  <em>Universidad del Istmo de Guatemala</em><br>
  <em>Faculty of Engineering</em><br>
  <em>Final Project</em><br>
  <em>Electronics 2 and Electronic Design 1</em>
</p>

<div align="center">
  <img src="FOTOS/Logo_UNIS.png" alt="Logo UNIS" width="45%"/>
</div>

<p align="center">
  <em>Maximiliano González</em><br>
  <em>May 2026</em>
</p>

## Overview

<div align="center">
  <img src="FOTOS/PCIe_NVMe_LAYOUT.png" alt="PCB LAYOUT" width="85%"/>
</div>

This repository documents the design of a custom **PCIe Gen 3 switch board for NVMe M.2 SSD expansion**. The goal of the project is to take a PCIe host connection and fan it out through a PCIe switch so multiple NVMe M.2 SSDs can be connected on one board.

The board is intended as a final electronics 2 project and focuses on the practical design work required for a high-speed PCIe system: schematic capture, power-tree design, PCIe lane planning, reference-clock distribution, reset/wake/presence handling, SMBus/I2C management, M.2 connector implementation, layout constraints, stack-up planning, and mechanical integration. It is important to note that the final design **does not** support **hot plug**.

## Table of Contents

- [English Version](#pcie-to-nvme-m2-ssd-switch)
  - [Overview](#overview)
  - [Table of Contents](#table-of-contents)
  - [Project Images](#project-images)
    - [Top Layer](#top-layer)
    - [GND02 Layer](#gnd02-layer)
    - [ART03 Layer](#art03-layer)
    - [ART04 Layer](#art04-layer)
    - [PWR05 Layer](#pwr05-layer)
    - [Bottom Layer](#bottom-layer)
  - [Main PCIe Switch IC](#main-pcie-switch-ic)
  - [Project Goals](#project-goals)
  - [High-Level Architecture](#high-level-architecture)
  - [Major Design Blocks](#major-design-blocks)
    - [1. PCIe Upstream Interface](#1-pcie-upstream-interface)
    - [2. PCIe Switch Core](#2-pcie-switch-core)
    - [3. Downstream M.2 NVMe Ports](#3-downstream-m2-nvme-ports)
    - [4. SMBus / I2C Management](#4-smbus--i2c-management)
    - [5. Configuration Straps](#5-configuration-straps)
    - [6. Power Architecture](#6-power-architecture)
    - [7. Reset, Wake, and Presence Signals](#7-reset-wake-and-presence-signals)
    - [8. Reference Clocking](#8-reference-clocking)
    - [9. PCB Layout and Constraints](#9-pcb-layout-and-constraints)
    - [10. Mechanical Design](#10-mechanical-design)
  - [Design Philosophy](#design-philosophy)
  - [Tools Used](#tools-used)
  - [Repository Contents](#repository-contents)
  - [Current Status](#current-status)
  - [Notes](#notes)

## Project Images

### Top Layer

<div align="center">
  <img src="FOTOS/TOPL.png" alt="TOP LAYER" width="85%"/>
</div>

### GND02 Layer

<div align="center">
  <img src="FOTOS/GND02L.png" alt="GND02 LAYER" width="85%"/>
</div>

### ART03 Layer

<div align="center">
  <img src="FOTOS/ART03L.png" alt="ART03 LAYER" width="85%"/>
</div>

### ART04 Layer

<div align="center">
  <img src="FOTOS/ART04L.png" alt="ART04 LAYER" width="85%"/>
</div>

### PWR05 Layer

<div align="center">
  <img src="FOTOS/PWR05L.png" alt="PWR05 LAYER" width="85%"/>
</div>

### BOTTOM Layer

<div align="center">
  <img src="FOTOS/BOTTOML.png" alt="BOTTOM LAYER" width="85%"/>
</div>

## Main PCIe Switch IC

<div align="center">
  <img src="FOTOS/PI7C9X3G816GP.webp" alt="PI7C9X3G816GP Chip" width="50%"/>
</div>

The design is based around the **Diodes Incorporated PI7C9X3G816GP**, a PCI Express Gen 3 packet switch. In this project, the switch is used to connect one upstream PCIe interface to multiple downstream NVMe M.2 SSD ports.

Relevant switch features used by this project include:

- PCIe Gen 3 switching for high-speed NVMe storage expansion.
- Multi-lane port configuration through `PORTCFG_x[2:0]` strap pins.
- Switch partitioning through `SWP_MODE[1:0]`.
- Normal operating mode selected through `CHIPMODE[1:0] = 00`.
- Single-reference-clock style operation selected with `CKMODE = 0` / BASE mode.
- SMBus or I2C management selection through `SMBUS_EN_L`.
- Optional JTAG/boundary-scan configuration through `JTAG_SEL_L`.
- Hot-plug and low-power-management related signals reviewed but not used as the primary operating mode for the M.2 SSD board.

## Project Goals

- Design a PCIe Gen 3 switch board that can host multiple NVMe M.2 SSDs.
- Use a full-height PCIe add-in-card style mechanical format.
- Follow the Diodes evaluation board/reference schematic where useful, while simplifying parts that are not needed for the final design.
- Keep configuration straps deterministic instead of relying on floating pins.
- Use default-enabled operating modes with optional DNP resistor positions where useful for debugging or future changes.
- Build a manufacturable PCB in Cadence OrCAD/Allegro.

## High-Level Architecture

```text
Host PCIe Edge Connector
        |
        | Upstream PCIe Gen 3 link
        |
+-----------------------------+
| Diodes PI7C9X3G816GP        |
| PCIe Gen 3 Packet Switch    |
+-----------------------------+
        |
        | Downstream PCIe Gen 3 links
        |
+-------+-------+-------+----------------+
| M.2 1 | M.2 2 | M.2 3 | ... NVMe SSDs  |
+-------+-------+-------+----------------+
```

The switch receives the upstream PCIe link from the host and distributes downstream PCIe links to the M-key M.2 NVMe SSD connectors. Each SSD connector uses PCIe lanes, reference clock, reset, presence, wake, and SMBus/I2C-related signals as required by the M.2 and PCIe design.

## Major Design Blocks

<div align="center">
  <img src="FOTOS/PCIe_NVMe_DOWNTREAM.png" alt="Schematic Downstream Ports 1 - 4" width="75%"/>
</div>

### 1. PCIe Upstream Interface

The upstream interface connects the board to the host system through a PCIe edge connector. This side includes the upstream PCIe transmit/receive differential pairs, reference clock input, reset, wake/presence-related signals, and the required power rails from the host connector.

### 2. PCIe Switch Core

The PI7C9X3G816GP is the central component of the board. Important design work around this IC includes:

- Selecting the correct port/lane configuration for the number of NVMe SSDs.
- Defining strap resistor values for mode pins.
- Routing high-speed PCIe differential pairs with controlled impedance.
- Managing reference-clock inputs and outputs.
- Adding local decoupling close to the switch power pins.
- Separating and filtering rails as required by the reference design.

### 3. Downstream M.2 NVMe Ports

The downstream ports use M-key M.2 connectors for NVMe SSDs. Each port includes:

- PCIe TX/RX differential pairs.
- REFCLK pair.
- PERST# reset signal.
- PEWAKE#/WAKE# handling.
- PRSNT#/presence-related handling where applicable.
- Local bulk and high-frequency decoupling.

### 4. SMBus / I2C Management

The project includes SMBus/I2C support instead of leaving the management bus unused. 

The switch supports selecting between SMBus and I2C using `SMBUS_EN_L`:

- High: I2C mode.
- Low: SMBus mode.

The design keeps this selection configurable with resistor options so the board can follow the desired default while still allowing modification during bring-up.

### 5. Configuration Straps

Several switch configuration pins are strapped with pull-up or pull-down resistors. The project decisions include:

| Signal Group | Project Intent |
|---|---|
| `CHIPMODE[1:0]` | Normal operating mode, `00`. |
| `PORTCFG_x[2:0]` | Set according to the desired downstream lane distribution. |
| `CKMODE` | Single reference clock / BASE-style operation. |
| `I2C_ADDR[2:0]` | Default address option set to `000`. |
| `JTAG_SEL_L` | Configurable option for boundary-scan/JTAG behavior. |
| `SMBUS_EN_L` | Configurable option for SMBus vs I2C selection. |

The reference EVB uses DIP-switch style configuration, but this project replaces that approach with fixed resistor straps and DNP options where practical.

### 6. Power Architecture

The board uses the PCIe/ATX input rails and local point-of-load regulators to generate the rails needed by the PCIe switch and M.2 connectors.

Power-design work included:

- 12 V input distribution.
- 3.3 V rail for M.2 SSDs and supporting logic.
- 1.8 V and lower-voltage switch-core rails as required by the PCIe switch.
- POL module selection with integrated inductors where possible.
- Bulk capacitors on 12 V and 3.3 V rails.
- High-frequency ceramic decoupling near IC power pins.
- LED indicators using NMOS switching for rail/status indication.

### 7. Reset, Wake, and Presence Signals

The project reviewed the PCIe sideband signals needed for correct behavior:

- `PERST#`: reset distribution to the switch and downstream devices.
- `PEWAKE#` / `WAKE#`: wake signaling from downstream devices.
- `PRSNT#`: presence detection or connector-related presence behavior.
- `INTA_L / PM_L11_EN_L`: reviewed as part of the reference design; not treated as a primary hot-plug signal for the final simplified M.2 switch use case.

### 8. Reference Clocking

The design uses PCIe reference-clock distribution consistent with the selected `CKMODE` configuration. For the default single-reference-clock approach, the switch tile lanes are driven from a shared clock source. Differential REFCLK routing is treated as high-speed routing and should follow the impedance and length-matching constraints used for PCIe clock pairs.

### 9. PCB Layout and Constraints

The board is implemented in Cadence Allegro/OrCAD. Important layout work includes:

- PCIe Gen 3 differential-pair routing.
- Controlled-impedance routing for TX/RX and REFCLK pairs.
- Differential-pair intra-pair matching.
- Lane-to-lane matching where required by the constraint manager.
- Necking rules near BGA breakout and connector regions.
- Via selection for BGA fanout and high-speed transitions.
- Route keep-in and package keep-in creation.
- Full-height PCIe bracket/mechanical outline planning.
- Use of Allegro board-geometry export/import where helpful.

### 10. Mechanical Design

Mechanical work included:

- PCIe add-in-card outline planning.
- Full-height bracket/faceplate consideration.
- Mounting-hole sizing and placement.
- Package keep-in and route keep-in geometry.
- M.2 connector placement and SSD clearance.
- Board-outline and DXF conversion/import workflow in Allegro.

## Design Philosophy

This board follows the evaluation board where it makes sense, but removes unnecessary complexity for the final project. DIP switches are replaced with resistor straps, debug options are kept only where useful, and the design is optimized around the intended use case: a PCIe host connected to multiple NVMe M.2 SSDs through a PCIe Gen 3 switch.

The design emphasizes deterministic hardware defaults, clean high-speed routing, adequate decoupling, and practical manufacturability.

## Tools Used

- Cadence OrCAD Capture for schematic design.

<div align="left">
  <img src="FOTOS/OrCAD-X-Capture.webp" alt="OrCad Capture Logo" width="25%"/>
</div>

- Cadence Allegro PCB Editor for layout.

<div align="left">
  <img src="FOTOS/OrCAD-X-PCB_Editor-1.webp" alt="OrCad Allegro Logo" width="25%"/>
</div>

- Diodes PI7C9X3G816GP datasheet and EVB documentation as references.

<div align="left">
  <img src="FOTOS/DIODES-logo.webp" alt="DIODES Inc. Logo" width="25%"/>
</div>

- Manufacturer datasheets for regulators, connectors, capacitors, resistors, MOSFETs, and logic devices.

## Repository Contents

```text
.
├── README(EN).md
├── README(ES).md
├── Schematic.pdf
├── BOARD TEMPLATE/
│   ├── CNV & DXF FILES/
│   └── PCIe_BOARD_TEMPLATE_FL.brd
├── BOM/
│   ├── BOM_PCIe_NVMe.BOM
│   └── BOM_PCIe_NVMe.BOM.xlsx
├── DRL/
│   └── pcie-nvme_rework2-1-6.drl
├── DSN & BRD/
│   ├── PCIE-NVME.DSN
│   └── pcie-nvme.brd
├── FOTOS/
│   ├── ART03L.png
│   ├── ART04L.png
│   ├── BOTTOML.png
│   ├── DIODES-logo.webp
│   ├── GND02L.png
│   ├── Logo_UNIS.png
│   ├── OrCAD-X-Capture.webp
│   ├── OrCAD-X-PCB_Editor-1.webp
│   ├── PCIe_NVMe_DOWNTREAM.png
│   ├── PCIe_NVMe_LAYOUT.png
│   ├── PI7C9X3G816GP.webp
│   ├── PWR05L.png
│   └── TOPL.png
├── GERBER/
│   ├── ART03L.art
│   ├── ART04L.art
│   ├── BOARD.art
│   ├── BOTTOML.art
│   ├── GND02L.art
│   ├── PASTEB.art
│   ├── PASTET.art
│   ├── PWR05L.art
│   ├── SILKB.art
│   ├── SILKT.art
│   ├── SOLDERB.art
│   ├── SOLDERT.art
│   └── TOPL.art
└── Referencias/
    ├── PI7C9X3G816GP-Design-kit-2.3.zip
    └── PI7C9X3G816GP-Support-kit-1.0.zip
```

## Current Status

The project work covered schematic decisions, signal planning, component selection, footprint creation, Allegro constraint setup, mechanical planning, and PCB layout guidance. Final validation should include DRC cleanup, impedance verification with the selected stack-up, review of all strap defaults, power-rail sequencing checks, and pre-fabrication schematic/layout review.

## Notes

This project is an educational hardware design project. Before manufacturing or using the board with expensive NVMe SSDs, all high-speed routing, power rails, reset timing, reference-clock topology, connector pinouts, and configuration straps should be independently reviewed.
