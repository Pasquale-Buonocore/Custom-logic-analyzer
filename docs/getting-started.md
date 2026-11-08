# Getting started

Start with the STM32F3DISCOVERY already available. The first development milestone is programming and debugging a simple application; the first analyzer milestone is a finite capture transferred to the PC.

**There is no firmware or host application to build yet.** The steps below prepare implementation. Build and CLI commands will be added with the corresponding code.

## Reading order

1. Read the [README](../README.md) for the project overview.
2. Follow the V0 milestones in the [roadmap](roadmap.md).
3. Review the [F3DISCOVERY hardware guide](hardware/stm32f3discovery.md) before connecting signals.
4. Consult the [architecture](architecture.md) for terminology and data flow, and the [strategy](strategy.md) for later versions.

## Equipment

For board bring-up, use the STM32F3DISCOVERY, a PC, and a Mini-B USB data cable connected to ST-LINK. A charging-only cable will not provide a debug connection.

For acquisition experiments, prepare suitable header connections or soldered headers, short signal wires, and a ground connection. Inspect the actual board before choosing connectors. A second USB data cable allows debugging and USB USER communication at the same time.

A known signal source will be needed for acquisition tests. Start with a verified board-generated pattern where feasible; use an independent source or reference instrument for qualification. The source need not be purchased before board bring-up.

## Prepare the development environment

1. Obtain [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html) and the STM32CubeF3 firmware package through ST's tooling. Record versions when creating the project.
2. Identify the board revision, MCU marking, and connectors using the [hardware guide](hardware/stm32f3discovery.md).
3. Connect ST-LINK and verify that the debugger recognizes the target. Follow ST's driver and debugger setup instructions for the installed tools.
4. During V0.1, create and commit the minimal firmware project and verify build, flash, breakpoint, and reset behavior.

Host development will initially use Python. Select and document the Python version, USB library, and driver setup when the host implementation begins; there are no Python dependencies to install from this repository yet.

## Before connecting test signals

Complete the V0.2 pin and acquisition review first. It must establish which pins are inputs, their allowed voltage levels, their existing board connections, and the sampling path. Use a common ground and the documented 3.3 V baseline; check individual pin limits before applying any other logic level.

Use ST-LINK for programming/debugging and USB USER for analyzer data. An ST-LINK connection alone does not establish a connection to the future analyzer firmware.

## First capture workflow

The planned workflow is: configure a finite capture, acquire into RAM, wait for completion, transfer samples through USB USER, and inspect them on the PC. Save/load and UART decoding follow in V0.5. Exact commands and test wiring belong to the implementation milestones and are not yet available.
