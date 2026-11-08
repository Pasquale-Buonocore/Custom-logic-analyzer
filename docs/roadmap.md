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

## V2 and V3

**V2 (planned):** select FPGA, memory, and transport; implement sampling and hardware triggers; qualify continuity and memory behavior; integrate the existing host interface. Evaluate deeper buffers, optional RLE, and USB 3.x. Completion requires measured capabilities, compatibility tests, and worst-case throughput results.

**V3 (planned):** specify electrical limits and front-end protection; design and review the PCB; bring up power, clocks, memory, acquisition, and USB in stages; integrate probes and enclosure. Completion requires electrical, functional, and system qualification of the integrated instrument. Detailed milestones follow validation of the preceding versions.

## Evidence to record

For each experiment, record hardware revision, firmware/host versions, build configuration, wiring, signal source, configuration, expected and actual results, and limitations. Use Debug builds for diagnosis and optimized builds with reduced logging for timing tests. Store test vectors and reports alongside implementation when they exist.
