# Architecture

The instrument samples digital inputs, stores a capture, and sends it to a PC. The PC reconstructs the timeline, decodes protocols, and eventually displays waveforms. See the [roadmap](roadmap.md) for implementation status and the [strategy](strategy.md) for version goals.

## Acquisition and buffering

The planned data flow is:

```text
Digital inputs → acquisition engine → RAM capture → USB → PC
                         ↑
                    sample clock
```

A sample is the set of digital levels observed at one instant. The sample clock sets the observation interval. A sampling rate of 1 MS/s means one million samples per second; it is different from the clock or baud rate of a protocol being observed. Decoder reliability depends on enough samples per signal interval and the actual timing quality.

For 8 channels, one logical sample fits in one byte, with bit 0 representing CH0 through bit 7 representing CH7. The acquisition driver may initially use a wider RAM transfer; normalize that representation before exposing it as an 8-channel capture. Actual DMA storage width must be included in the memory budget.

For uniform sampling, sample time is `t[n] = n / sample_rate`. Capture duration is `sample_count / sample_rate`. Buffer capacity depends on the memory remaining after firmware, stack, and transport allocations, not the total RAM printed on the board specification.

**V0:** verify a timer/GPIO/DMA approach. General-Purpose Input/Output (GPIO) input registers expose pin levels; a timer can provide periodic requests, and Direct Memory Access (DMA) can move data without CPU work for each sample. Whether this exact path is usable on the F303 must be established in V0.2.

**V1:** use PSSI, the Parallel Synchronous Slave Interface, to receive parallel samples and feed DMA. Its clock input must be driven by a verified clock arrangement, potentially a timer or clock output routed to the input pin. Review setup/hold timing, pin multiplexing, FIFO behavior, and lossless DMA bandwidth. Place buffers in DMA-accessible memory and handle Cortex-M7 cache coherency explicitly.

**V2:** move sampling, triggering, and memory control into FPGA logic while retaining the host-facing capture model.

## Capture control and triggers

A trigger is the event that selects the interesting portion of a capture. Pre-trigger samples precede it; post-trigger samples follow it. A circular buffer repeatedly overwrites the oldest samples while waiting for the event.

V0 begins with an explicit start and a finite window. V1 adds rising/falling edge selection, a channel selection, and a configurable trigger position. Trigger detection and the sample clock are separate functions. Correlate any interrupt-based detection with the captured edge and measure its uncertainty; an interrupt timestamp alone does not establish the exact sample index.

The control layer will expose configuration, acquisition progress, completion, download, stop, and errors. Future triggered modes add an armed/waiting phase. An incomplete or failed capture must not be presented as successful. Concrete states and APIs will be specified with firmware implementation.

## USB and host interface

Use buffered capture first: acquisition completes before download. USB Full-Speed has a nominal line rate of 12 Mbit/s and lower application throughput. At one byte per sample, 50 MS/s produces 50 MB/s before overhead, so the V1 target relies on RAM buffering. Measure transfer throughput independently; continuous streaming is a later, bandwidth-limited mode.

Use USB USER on F3DISCOVERY. Select its USB class and host backend during V0.4. A binary device protocol is the intended data path; bulk transfers are preferred for V1 capture downloads. Define framing, endpoint layout, integrity checks, and driver setup when implementing transport.

The device interface must eventually support identification, capabilities, configuration, acquisition start/stop, status, and capture download. Capabilities include channel count, supported rates, buffer capacity, and available trigger/streaming modes. Unsupported settings must produce explicit errors. Host software must use advertised capabilities instead of assuming H723 features on V0.

Captures need raw samples plus actual sample rate, channel mapping/mask, sample count, and device/firmware identity. Add trigger index and settings when supported. Version capture files independently from firmware and the device protocol; define serialization and compatibility rules during implementation. Save/load must preserve both samples and their interpretation.

Handle malformed requests, disconnects, incomplete transfers, DMA errors, overruns, and buffer overflow explicitly. Diagnostics will track captures, failures, transfer counts, and acquisition errors as applicable to each device. Avoid per-sample logging and benchmark with reduced logging.

## Host processing and visualization

Develop the host initially in Python with separate device transport, capture storage, decoder, CLI, and GUI responsibilities. The decoder consumes a capture and configuration; it must also work offline without a connected board. Decoded events need start/end times, channels, values, and error information. Concrete types are defined with implementation.

| Decoder | Intended behavior |
|---|---|
| UART | Configurable baud, data bits, parity, stop bits, polarity, and framing errors |
| SPI | CLK/MOSI/MISO/CS, modes 0–3, word size, bit order |
| I²C | START/STOP, repeated START, address, read/write, ACK/NACK, and data |

Build transport and CLI before the GUI. V0 introduces UART; V1 completes the three decoders and waveform viewer, including zoom, pan, cursors, trigger marker, and decode overlays. Keep decoder outputs separate from rendering.

## Verification and reuse

Test static levels, channel mapping, periodic signals, known sequences, trigger edges, and USB integrity. A counter or deterministic pattern is useful for detecting missing, duplicated, or reordered samples. Include valid and invalid protocol vectors and exercise SPI modes and I²C repeated starts. Decoder tests should run offline; hardware qualification requires a documented signal source and timing reference.

Keep board-specific clocks, pin setup, acquisition, and memory handling behind a platform boundary. Reuse the capture model, versioned protocol concepts, device capabilities, decoder interface, CLI, and GUI across V0–V3. Later compression must preserve sample timing, and external memory/USB upgrades must be qualified under worst-case traffic.
