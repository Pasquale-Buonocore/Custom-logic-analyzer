# Roadmap

This is the source of truth for milestone status and completion criteria. See the [strategy](strategy.md) for the complete V0–V3 goals.

**Current state:** documentation is available; every implementation milestone below is **planned**. No sampling rate, USB throughput, or hardware behavior has been qualified.

## V0 — STM32F3DISCOVERY

### V0.1 — Board bring-up

Prepare STM32CubeIDE, identify the board revision and its ST-LINK connection, and create the first firmware project. Use a simple LED or equivalent observable test, then verify a breakpoint and reset through the debugger.

**Complete when:** a clean build can be programmed repeatedly, the test runs after reset, and the board revision, tool versions, connection steps, and result are recorded. Add real firmware files and build instructions at this milestone.

### V0.2 — Acquisition feasibility

Check the [hardware constraints](hardware/stm32f3discovery.md) against the board schematic, MCU datasheet, and reference manual. Investigate timer-requested DMA reads of GPIO input registers without assuming the route or transfer width is supported. Prefer channels on one GPIO port for a common sample instant; verify competing board connections.

Select the timer, DMA request/channel, source register, memory region, transfer widths, initial test rate, and candidate channel mapping only after this review. If the preferred path is unavailable, record the evidence and assess a measured alternative before making timing claims.

**Complete when:** a documented, technically supported acquisition approach and test wiring are selected, including memory budget and expected timing limits. The 8-channel goal is confirmed or any limitation is explicitly recorded. PSSI and 50 MS/s are not V0 requirements.

### V0.3 — Finite capture in RAM

Implement a configurable finite capture with explicit start and completion. Begin with static levels and a known periodic pattern. A verified timer output looped back to an input can provide an initial signal; use an independent generator or reference instrument when qualifying timing and continuity.

**Complete when:** sample count, channel mapping, and measured sample timing match the test setup; repeated captures have no unexplained discontinuities. Record buffer placement, actual storage width, rates tested, timing uncertainty, and errors. Reject or identify incomplete captures.

### V0.4 — First capture on the PC

Bring up USB device communication through USB USER. Select and document the USB class, host backend, framing, and Windows driver requirements. Transfer only after acquisition completes. Implement the minimum protocol and Python CLI needed to identify the device, request a capture, and inspect raw samples with metadata.

**Complete when:** the PC receives the expected sample count and channel pattern repeatedly; truncated transfers and disconnects produce clear errors. Build, programming, connection, and CLI instructions work from a fresh setup. Measure transfer throughput separately from acquisition speed.

### V0.5 — Consolidation

Add capture configuration, save/load, a UART decoder, device capabilities, and diagnostics. Validate UART with known payloads and invalid frames. Record qualified operating conditions and remaining limitations.

**Complete when:** a repeatable capture can be configured, acquired, transferred, saved, reloaded, and decoded on the PC. Reloading preserves samples and interpretation metadata. Unsupported configurations and failed acquisitions are reported clearly; decoder tests run without attached hardware.

V0 completion requires these results and a documented outcome for the 8-channel target. Any reduction must be explicit. Purchasing the Nucleo is not required to complete V0.

## V1 — NUCLEO-H723ZG

All stages are planned. Reuse V0 host components and add a separate H723 acquisition implementation.

| Stage | Work | Completion evidence |
|---|---|---|
| V1.1 | Define requirements, bring up H723, resolve input/USB/debug pins and clock arrangement | Working debug build and reviewed wiring/memory design |
| V1.2 | Implement PSSI → DMA → RAM and increase rates progressively toward 50 MS/s | Lossless capture benchmark identifying the maximum qualified rate |
| V1.3 | Circular buffer, rising/falling trigger, configurable pre/post-trigger | Known edges appear at the expected capture position with measured uncertainty |
| V1.4 | USB Full-Speed, binary transfer, capability discovery, and CLI integration | Reliable downloads, error recovery, and measured USB throughput |
| V1.5 | UART completion, SPI modes 0–3, and I²C | Automated known-payload and error-condition decoder vectors |
| V1.6 | Waveform GUI with zoom, pan, cursors, trigger marker, and decode overlays | Saved and live-device captures display consistently |
| V1.7 | Integration and qualification | Repeatable tests, performance report, documented electrical/timing limits |

V1 is complete when all stages pass at the maximum qualified rate. The 50 MS/s target must be reported as achieved or unmet, with evidence; USB Full-Speed streaming is measured separately and does not define buffered capture speed.

## V2 — FPGA acquisition

All stages are **planned**: their scope is defined below, but implementation has not started. The original targets remain 8–16 or more channels, 100–200+ MS/s, deeper captures, and hardware triggers. Rates, depth, and streaming throughput require qualification on the selected hardware.

Replace the acquisition engine while preserving the host protocol, capture format, decoder interface, GUI, and user workflow where practical. See the [FPGA architecture](architecture.md#fpga-acquisition-v2) for the planned modules and data flow.

| Stage | Work | Completion evidence |
|---|---|---|
| V2.1 | Select FPGA development board, clocking, memory, and USB transport; define resource and bandwidth budgets | Reviewed architecture, channel/rate/depth targets, interfaces, and hardware selection rationale |
| V2.2 | Implement the input register bank, sample clock, configuration registers, and basic capture | Simulation and hardware patterns verify channel mapping, sample order, and timing constraints |
| V2.3 | Add FIFO, circular buffering, on-chip block RAM, and external SDRAM/DDR where required | Memory tests and repeated captures verify capture depth, wraparound, continuity, and explicit overflow reporting |
| V2.4 | Implement hardware edge, level, masked-pattern, and sequence triggers, with pre/post-trigger capture | Deterministic vectors verify trigger selection and sample position, including events at buffer boundaries |
| V2.5 | Extend triggers to pulse width, glitches, and protocol-aware conditions | Defined detection limits and valid/invalid test sequences verify each advertised trigger mode |
| V2.6 | Evaluate optional RLE compression and host reconstruction | Reconstructed samples and timing match raw input for constant, changing, and incompressible patterns; resource/throughput costs are measured |
| V2.7 | Integrate USB transport and existing host capabilities, file handling, CLI, decoders, and GUI | Captures work through the existing workflow; transfer integrity, disconnect recovery, and compatibility tests pass |
| V2.8 | Qualify acquisition, memory, triggers, and transport together | Published maximum lossless rate, depth, trigger accuracy, streaming limits, and worst-case traffic results |

For V2.1 and V2.7, evaluate USB 3.x and a suitable dedicated controller; FX3/FX3S and FPGA interface options from the original plan remain candidates, not selected parts. PCIe remains a later experiment. At 100 MS/s, raw acquisition generates 100 MB/s for 8 channels or 200 MB/s for 16 channels, before transport overhead. Budget buffered download and continuous streaming separately.

RLE is optional: record whether it is implemented or deferred and why. Compression gains must not be required to sustain the advertised worst-case capture mode. Hardware triggers and deeper buffering remain core V2 goals.

V2 completion requires the core stages, a documented RLE decision, host compatibility evidence, and measured results against the original targets. Report targets as achieved or unmet rather than presenting them as guaranteed specifications.

## V3 — Integrated custom board

All stages are **planned**. Integrate the validated acquisition architecture into an instrument with protected inputs, external memory, USB connectivity, probes, and an enclosure. Board design follows validation of the preceding prototypes.

| Stage | Work | Completion evidence |
|---|---|---|
| V3.1 | Define channel count, bandwidth, electrical limits, power budget, connectors, and mechanical requirements | Reviewed system specification tied to the V2 results and intended use |
| V3.2 | Design input buffering, thresholds, and protection; evaluate 1.8/2.5/3.3/5 V logic support | Front-end review and prototype measurements establish loading, thresholds, propagation delay, channel skew, and allowed input conditions |
| V3.3 | Integrate FPGA, control MCU where required, external memory, clocks, power rails, USB-C, and the qualified USB 3.x transport | Reviewed schematics, component selection, power sequencing, and programming/debug connections |
| V3.4 | Lay out and review the PCB, including input matching, memory and USB routing, grounding, decoupling, and protection placement | Layout review covers controlled impedance, timing, return paths, manufacturability, and mechanical fit before fabrication |
| V3.5 | Bring up the assembled board in stages: power, clocks, debug, memory, acquisition, and USB | Measurements and subsystem tests pass before complete-system operation |
| V3.6 | Integrate probes, ground connections, connectors, and enclosure | Fit and usability checks pass; acquisition remains within limits using the intended probes and cabling |
| V3.7 | Qualify the integrated instrument and host workflow | Electrical, protection, functional, timing, and sustained-operation results are documented with supported configurations and limitations |

The front-end study includes high input impedance, programmable/selectable thresholds, comparators where appropriate, series resistance, overvoltage and electrostatic discharge (ESD) protection, termination options, and matched channel delays. Supported voltage levels become specifications only after validation.

PCB review must cover differential USB routing, any chosen ULPI interface, memory timing, FPGA power rails, clock integrity, ground return paths, ESD placement, and connector mechanics. USB-C describes the connector; transport performance comes from the selected and qualified USB implementation.

V3 is complete when the integrated hardware meets its reviewed specification and reproduces qualified captures through the existing host software, with probe, electrical, and operating limits documented.

## Evidence to record

For each experiment, record hardware revision, firmware/host versions, build configuration, wiring, signal source, configuration, expected and actual results, and limitations. Use Debug builds for diagnosis and optimized builds with reduced logging for timing tests. Store test vectors and reports alongside implementation when they exist.
