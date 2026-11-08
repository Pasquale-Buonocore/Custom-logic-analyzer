# Development strategy

Build a reusable debugging instrument in four versions, starting with hardware already available. The complete project includes digital acquisition, triggering, capture storage, USB transfer, protocol decoding, and waveform visualization.

The [roadmap](roadmap.md) owns milestone status and acceptance criteria. This document owns the long-term goals; the [architecture](architecture.md) explains the shared design.

## V0 — Learn and validate on STM32F3DISCOVERY

Use the existing STM32F303VCT6 board to establish a complete path from input signals to a capture on the PC. Target 8 channels, subject to pin and acquisition feasibility checks. Use finite captures stored in RAM, followed by USB download.

Determine the sampling method and supported rates experimentally. The board has no PSSI peripheral, so the V1 acquisition driver cannot be assumed to work here. Start with board bring-up, verify acquisition feasibility, then add RAM capture, USB transfer, a minimal command-line interface (CLI), save/load, a UART decoder, and diagnostics.

V0 is a useful working prototype with measured limits. Full triggering, SPI/I²C decoding, and the graphical user interface (GUI) remain part of V1 completion.

## V1 — Complete the MCU instrument on NUCLEO-H723ZG

Move acquisition to the STM32H723 while reusing the host software and capture concepts established in V0.

| Capability | V1 goal |
|---|---|
| Channels and representation | 8 simultaneous digital channels, one byte per sample |
| Input logic | 3.3 V baseline with documented electrical limits |
| Acquisition | Parallel Synchronous Slave Interface (PSSI) and Direct Memory Access (DMA) |
| Sampling | Configurable rates; up to 50 MS/s buffered as a qualification target |
| Trigger | Selectable channel, rising/falling edge, configurable pre/post-trigger position |
| Buffering | Circular buffer before trigger, finite capture window afterward |
| USB | USB 2.0 Full-Speed through the internal PHY, the physical USB transceiver |
| Host | Device discovery, configuration, capture, save/load, CLI, and GUI |
| Decoders | UART, SPI modes 0–3, and I²C, operating on the PC |
| Verification | Automated test vectors, continuity checks, diagnostics, performance report |

Capture into RAM first and download afterward. Streaming is secondary and limited by measured USB throughput. USB High-Speed through an external ULPI PHY and external capture RAM are optional extensions, not prerequisites for V1.

Qualify the sample-clock arrangement, PSSI pin mapping, DMA bandwidth, accessible memory, cache handling, and trigger accuracy. Nominal peripheral speed alone does not establish lossless operation at 50 MS/s.

Use the [STM32H723 datasheet](https://www.st.com/resource/en/datasheet/stm32h723zg.pdf), [RM0468 reference manual](https://www.st.com/resource/en/reference_manual/dm00603761.pdf), and [Nucleo board manual UM2407](https://www.st.com/resource/en/user_manual/um2407-stm32h7-nucleo144-board-stmicroelectronics.pdf) for V1 design. Review the relevant errata before qualification.

## V2 — FPGA acquisition

Replace the acquisition engine with a field-programmable gate array (FPGA), which implements sampling and trigger logic directly in hardware. Preserve the device workflow, host protocol, capture format, decoder interface, and viewer where practical.

Explore 8–16 or more channels, 100–200+ MS/s, hardware edge/level/pattern/sequence triggers, pulse-width and glitch detection, and protocol-aware triggers. These are development targets, not measured specifications.

Use on-chip block RAM plus external SDRAM/DDR where needed for deeper captures. Evaluate USB 3.x transport and a suitable dedicated USB controller. Optional Run-Length Encoding (RLE) stores repeated samples as a value and count; qualification must also cover incompressible signals. PCIe remains a possible later transport experiment.

## V3 — Integrated custom board

Integrate the validated acquisition engine, control MCU where required, external memory, power supplies, USB connectivity, probes, and enclosure on a dedicated printed circuit board (PCB).

Design a high-impedance front-end with input buffers, electrostatic discharge (ESD) and overvoltage protection. Evaluate 1.8 V, 2.5 V, 3.3 V, and 5 V logic support, selectable thresholds, termination options, channel delay matching, and probe grounding. Integrate USB-C and the qualified USB 3.x transport.

PCB work includes signal integrity, controlled impedance, USB and memory routing, clock distribution, FPGA power rails, decoupling, ground returns, protection placement, and connector mechanics. Start integration after the preceding architectures have been validated.

## Principles carried across versions

- Verify acquisition before investing in the GUI. Prefer peripheral-driven sampling and DMA over CPU work for every sample; any V0 experimental alternative must have its timing measured and limitations recorded.
- Keep raw samples separate from protocol decoding. Develop the host initially in Python and keep hardware-specific behavior behind a device interface.
- Report capabilities and versions so software can adapt to each device. Version hardware, firmware, protocol, and capture files independently.
- Distinguish targets, implemented features, and measured results. Publish limitations and reproducible test conditions.
- Use deterministic signals to test channel mapping, continuity, triggers, and decoders. Avoid per-sample logging in the acquisition path.
- Introduce complexity progressively. Future options include compression, capture export/import, scripting, sigrok/PulseView interoperability, and Linux/macOS host support.

This strategy replaces the previous H723-only master plan. Its original feature goals remain assigned to V1–V3; V0 is the added starting point. Previous wording is available in Git history.
