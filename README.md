# USB Logic Analyzer

A project to capture digital signals, inspect waveforms, and decode UART, SPI, and I²C communication on a PC.

**Current focus: V0 on the STM32F3DISCOVERY already available.**
The first goal is a short, repeatable capture transferred to the PC.

## Development path

| Version | Platform | Goal |
|---|---|---|
| **V0** | STM32F3DISCOVERY | Validate acquisition, USB transfer, and initial PC software |
| **V1** | NUCLEO-H723ZG | 8 channels, PSSI/DMA acquisition, triggers, decoders, and GUI |
| **V2** | FPGA | Faster acquisition, deeper memory, and advanced hardware triggers |
| **V3** | Custom PCB | Integrate protected inputs, acquisition, memory, and USB |

V0 lets development begin without buying another board. It does not reduce the goals of later versions.
The V1 target of 50 million samples per second (50 MS/s) requires experimental qualification; it is not a V0 specification.

## Current status and next step

The repository currently contains documentation only. Firmware, host tools, and performance results are not yet available.

**Next step:** follow the [getting started guide](docs/getting-started.md) and complete V0 board bring-up.
The V0 acquisition method, input pin mapping, capture depth, and supported sampling rates remain to be verified.

## Where to read

| Document | What it answers |
|---|---|
| [Getting started](docs/getting-started.md) | What do I need, and where do I begin? |
| [Roadmap](docs/roadmap.md) | What do we build next, and when is each step complete? |
| [Development strategy](docs/strategy.md) | How do V0–V3 lead to the complete instrument? |
| [Architecture](docs/architecture.md) | How do acquisition, memory, USB, and PC software fit together? |
| [STM32F3DISCOVERY hardware](docs/hardware/stm32f3discovery.md) | What can this board do, and what needs checking? |

Start with the guide; use the other documents as references. Source-code directories will be added when implementation begins.

## License

[MIT](LICENSE).
