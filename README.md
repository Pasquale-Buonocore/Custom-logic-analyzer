# USB Logic Analyzer

A custom logic analyzer designed from the ground up to capture, visualize, and decode digital protocols such as **UART**, **SPI**, and **I²C**.

The project is developed in three hardware generations:

- **V1 — STM32H723**: MCU-based acquisition, trigger, USB transport, host software, and protocol decoders.
- **V2 — FPGA**: higher sample rates, deeper buffering, hardware triggers, and USB 3.x-class throughput.
- **V3 — Custom Board**: integrated front-end, FPGA/MCU, memory, protected inputs, high-speed USB, and enclosure.

> **Current focus:** V1 based on the **STM32H723** using the **NUCLEO-H723ZG** development board.

---

## Project Goals

The objective is not to build a simple GPIO sampling demo, but a reusable embedded debugging instrument with a clean separation between:

- digital acquisition;
- triggering;
- capture buffering;
- USB transport;
- host-side protocol decoding;
- waveform visualization.

The first supported protocol decoders will be:

- UART
- SPI
- I²C

The host architecture is intentionally hardware-independent so that the PC software can be reused when the acquisition engine moves from STM32 to FPGA.

---

## High-Level Architecture

```text
                         DUT
                          │
                          ▼
                 ┌─────────────────┐
                 │ Digital Inputs  │
                 │     CH0..CH7    │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │   Acquisition   │
                 │   STM32 PSSI    │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │       DMA       │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Capture Buffer  │
                 │ Pre/Post Trigger│
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ USB Full-Speed  │
                 └────────┬────────┘
                          │
══════════════════════════╪══════════════════════════
                          │ PC
                          ▼
                 ┌─────────────────┐
                 │ Host Software   │
                 └────────┬────────┘
                          │
               ┌──────────┼──────────┐
               ▼          ▼          ▼
             UART        SPI        I²C
            Decoder     Decoder    Decoder
               │          │          │
               └──────────┼──────────┘
                          ▼
                 ┌─────────────────┐
                 │ Waveform Viewer │
                 └─────────────────┘
```

---

# Version 1 — STM32H723

## Target Hardware

**MCU:** STM32H723  
**Development board:** NUCLEO-H723ZG

Relevant MCU resources:

- Arm Cortex-M7;
- PSSI — Parallel Synchronous Slave Interface;
- DMA;
- internal SRAM;
- timers / clock outputs;
- USB 2.0 Full-Speed device interface.

---

## V1 Target Specifications

| Feature | Target |
|---|---|
| Digital channels | 8 |
| Sample width | 8 bit |
| Input logic | 3.3 V baseline |
| Sampling | 1–50 MS/s target |
| Maximum target rate | 50 MS/s buffered |
| Acquisition | PSSI + DMA |
| Primary capture mode | Buffered / triggered |
| Pre-trigger | Yes |
| Post-trigger | Yes |
| Trigger position | Configurable |
| Edge trigger | Rising / Falling |
| USB | USB 2.0 Full-Speed |
| USB PHY | Internal STM32 PHY |
| UART decoding | Host-side |
| SPI decoding | Host-side |
| I²C decoding | Host-side |
| CLI | Planned |
| GUI | Planned |

> **50 MS/s is a qualification target, not yet a guaranteed measured performance.**

The maximum lossless rate will be established experimentally.

---

## Buffered Capture First

USB Full-Speed is deliberately **not part of the real-time sampling path**.

```text
PSSI
  │
  ▼
DMA
  │
  ▼
RAM
  │
  │ capture complete
  ▼
USB Full-Speed
  │
  ▼
PC
```

For 8 channels represented by one byte per sample:

```text
50 MS/s = 50 MB/s acquisition bandwidth
```

USB Full-Speed cannot continuously transport that data rate, but it does not need to. The V1 architecture first stores a triggered acquisition in RAM and then downloads it to the PC.

Continuous streaming is therefore a secondary V1 feature and will be limited to the throughput experimentally supported by USB Full-Speed.

---

# Trigger Model

The trigger defines the event of interest and establishes:

```text
t = 0
```

Example:

```text
Trigger = CH3 rising edge
```

The analyzer continuously samples into a circular buffer before the trigger occurs.

```text
                           TRIGGER
                              │
                              ▼
██████████████████████████████│████████████████████████
          PRE-TRIGGER         │         POST-TRIGGER
                              │
                             t=0
```

The trigger position will be configurable.

Examples:

```text
0%    -> all post-trigger
50%   -> half pre-trigger / half post-trigger
90%   -> mostly pre-trigger
```

This is particularly useful for investigating the events that caused a fault.

---

# Acquisition Data Format

For 8 digital channels:

```text
bit7 bit6 bit5 bit4 bit3 bit2 bit1 bit0
 CH7  CH6  CH5  CH4  CH3  CH2  CH1  CH0
```

Each sample occupies exactly one byte.

The timestamp of sample `n` is reconstructed as:

```text
t[n] = n / sample_rate
```

No per-sample timestamp is required in the raw capture stream.

---

# Firmware Architecture

Planned structure:

```text
firmware/
│
├── app/
│   ├── main.c
│   ├── analyzer_app.c
│   └── analyzer_state_machine.c
│
├── acquisition/
│   ├── acquisition.c
│   ├── pssi_sampler.c
│   ├── sample_clock.c
│   ├── capture_buffer.c
│   └── trigger.c
│
├── transport/
│   ├── usb_device.c
│   ├── usb_bulk.c
│   ├── transport_protocol.c
│   └── packetizer.c
│
├── platform/
│   ├── clocks.c
│   ├── mpu.c
│   ├── cache.c
│   ├── gpio.c
│   └── error.c
│
├── diagnostics/
│   ├── counters.c
│   ├── profiler.c
│   └── telemetry.c
│
└── tests/
```

The high-speed sample path must not depend on per-sample CPU polling.

```text
GPIO → PSSI → DMA → RAM
```

---

# Host Software

The host-side software will initially be developed in Python to keep iteration fast.

```text
host/
│
├── device/
│   ├── discovery.py
│   ├── transport.py
│   ├── protocol.py
│   └── device.py
│
├── capture/
│   ├── capture.py
│   ├── file_format.py
│   └── timeline.py
│
├── decoders/
│   ├── base.py
│   ├── uart.py
│   ├── spi.py
│   └── i2c.py
│
├── cli/
│   └── main.py
│
├── gui/
│   ├── main_window.py
│   ├── waveform.py
│   └── decode_overlay.py
│
└── tests/
```

Development order:

1. USB transport
2. CLI
3. capture file format
4. UART decoder
5. SPI decoder
6. I²C decoder
7. waveform viewer
8. integrated GUI

The GUI intentionally comes after the acquisition and decoding layers.

---

# Protocol Decoders

## UART

Planned support:

- configurable baud rate;
- data bits;
- parity;
- stop bits;
- polarity;
- framing error detection.

Example:

```text
125.340 ms    0x48    'H'
125.427 ms    0x65    'e'
125.514 ms    0x6C    'l'
```

## SPI

Planned support:

- CLK;
- MOSI;
- MISO;
- CS;
- Mode 0–3;
- MSB/LSB first;
- configurable word size.

Example:

```text
MOSI: 9F 00 00 00
MISO: 00 EF 40 18
```

## I²C

Planned decoding:

- START;
- STOP;
- repeated START;
- address;
- R/W;
- ACK;
- NACK;
- data bytes.

Example:

```text
START
ADDRESS 0x68 WRITE
ACK
DATA 0x10
ACK
RESTART
ADDRESS 0x68 READ
ACK
DATA 0x42
NACK
STOP
```

---

# V1 Roadmap

```text
M0   Requirements
 │
M1   STM32H723 bring-up
 │
M2   PSSI sampling
 │
M3   High-speed acquisition qualification
 │
M4   Circular buffer + trigger
 │
M5   USB Full-Speed
 │
M6   USB throughput benchmark
 │
M7   Binary protocol + CLI
 │
M8   UART decoder
 │
M9   SPI decoder
 │
M10  I²C decoder
 │
M11  Waveform viewer
 │
M12  Integration and qualification
 │
 ▼
V1 COMPLETE
```

---

# Version 2 — FPGA

V2 replaces the STM32 acquisition engine with FPGA logic while preserving as much of the host software and device protocol as possible.

Preliminary targets:

| Feature | V2 Target |
|---|---|
| Channels | 8–16+ |
| Sampling | 100–200+ MS/s |
| Trigger | Hardware |
| Edge trigger | Yes |
| Pattern trigger | Yes |
| Protocol trigger | Planned |
| Buffer | BRAM + external RAM |
| Compression | Optional RLE |
| USB | USB 3.x target |

Conceptual architecture:

```text
Digital Inputs
      │
      ▼
┌───────────────┐
│ FPGA Sampler  │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ Trigger Engine│
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ FIFO / BRAM   │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ DDR / SDRAM   │
└───────┬───────┘
        │
        ▼
      USB 3.x
```

---

# Version 3 — Custom Board

The final hardware generation will integrate the complete instrument.

Potential features:

- 8–16 digital channels;
- protected inputs;
- high-impedance input buffers;
- 1.8 V / 2.5 V / 3.3 V / 5 V logic support;
- programmable input thresholds;
- FPGA;
- external RAM;
- MCU/control processor;
- USB-C;
- USB 3.x;
- ESD protection;
- dedicated probes;
- enclosure.

```text
PROBES
  │
  ▼
Input Protection
  │
  ▼
Input Buffer / Level Stage
  │
  ▼
FPGA
  │
  ├──── External RAM
  │
  └──── MCU / USB Controller
               │
               ▼
             USB-C
               │
               ▼
               PC
```

---

# Testing Strategy

A second MCU or FPGA will be used as a deterministic signal generator.

```text
Signal Generator
      │
      │ UART / SPI / I²C / patterns
      ▼
Logic Analyzer
      │
      ▼
     PC
```

Example automated UART test:

```text
Generate:
115200 baud
payload = 55 AA 12 34

Capture
Decode

Assert:
decoded == [0x55, 0xAA, 0x12, 0x34]
```

Equivalent test vectors will be created for:

- UART;
- SPI Mode 0–3;
- I²C;
- edge triggers;
- channel mapping;
- sample continuity;
- maximum acquisition rate.

---

# Repository Structure

```text
logic-analyzer/
│
├── README.md
│
├── docs/
│   ├── master/
│   ├── requirements/
│   ├── architecture/
│   ├── firmware/
│   ├── hardware/
│   ├── usb/
│   ├── decoders/
│   ├── gui/
│   └── verification/
│
├── firmware/
├── host/
├── fpga/
│
├── hardware/
│   ├── prototype/
│   └── custom_board/
│
├── test/
│   ├── generators/
│   ├── captures/
│   ├── vectors/
│   └── automation/
│
└── tools/
```

---

# Documentation

The project is documentation-driven.

Master technical plan:

```text
docs/master/Logic_Analyzer_Master_Project_Plan.md
```

Planned documents:

```text
LA-V1-SRS-001       System Requirements Specification
LA-V1-ARCH-001      STM32H723 Logic Analyzer Architecture
LA-V1-HW-001        Prototype Hardware and Pin Mapping
LA-V1-CLOCK-001     Sampling Clock and PSSI Design
LA-V1-DMA-001       PSSI DMA and Memory Architecture
LA-V1-TRG-001       Trigger and Pre/Post-Trigger Design
LA-V1-USB-001       USB 2.0 Full-Speed Architecture
LA-HOST-PROTO-001   Device-to-PC Binary Protocol
LA-HOST-CLI-001     Host CLI Architecture
LA-DEC-UART-001     UART Decoder Design
LA-DEC-SPI-001      SPI Decoder Design
LA-DEC-I2C-001      I²C Decoder Design
LA-GUI-001          Waveform Viewer Architecture
LA-VER-001          Verification and Qualification Plan
```

Each major implementation phase should be supported by a dedicated technical document.

---

# Current Status

**Project stage:** architecture and planning.

```text
V1 Platform       STM32H723
Dev Board         NUCLEO-H723ZG
Channels          8
Sampling Target   up to 50 MS/s buffered
Acquisition       PSSI + DMA
USB               USB 2.0 Full-Speed
Trigger           Rising / Falling edge
Pre-trigger       Planned
Post-trigger      Planned
UART Decoder      Planned
SPI Decoder       Planned
I²C Decoder       Planned
GUI               Planned
```

No performance value should be considered qualified until it has been validated by dedicated tests.

---

# Design Principles

1. **Correct acquisition before GUI development.**
2. **DMA/peripheral-driven sampling instead of per-sample CPU polling.**
3. **Raw capture data remains independent from protocol decoders.**
4. **Every performance target must eventually be measured.**
5. **Host software should remain reusable across STM32 and FPGA versions.**
6. **Hardware complexity is introduced progressively.**
7. **Protocol decoding remains host-side unless hardware triggering requires otherwise.**
8. **The final architecture must be testable using deterministic signal generators.**

---

# Future Ideas

Possible later features include:

- 16+ channels;
- USB 3.x streaming;
- FPGA hardware triggering;
- protocol-aware triggers;
- pulse-width trigger;
- glitch trigger;
- Run-Length Encoding;
- deep external memory;
- sigrok/PulseView compatibility;
- custom probes;
- programmable voltage thresholds;
- capture export/import;
- scripting API;
- Linux/macOS support.

---

# License

License has not yet been selected.

---

# Project Status

🚧 **Work in progress**

The project is currently in the architecture and requirements phase.

The next major activity is the formal definition of the **V1 System Requirements Specification**, followed by STM32H723 hardware bring-up.
