# STM32F3DISCOVERY hardware guide

The available board is the STM32F3DISCOVERY with an STM32F303VCT6 MCU. It is the V0 development platform. Board identity is based on its markings; operation, board revision, and acquisition performance still require verification.

## Resources and limits

| Resource | Relevance to V0 |
|---|---|
| Cortex-M4 up to 72 MHz | Controls acquisition and communication; CPU clock is not sample rate |
| 256 KB Flash | Firmware storage |
| 40 KB main SRAM plus 8 KB CCM RAM | Budget capture, stack, application, and USB memory separately |
| Timers, GPIO, and DMA | Candidate acquisition components; exact route needs review |
| USB Full-Speed device | Transfer completed captures through USB USER |
| Embedded ST-LINK | Programming and debugging |

CCM means Core-Coupled Memory. Do not treat the advertised 48 KB total as one DMA capture buffer; verify DMA reachability in the reference manual before allocating buffers. The MCU has no PSSI, so V1's PSSI acquisition design does not apply to V0.

These resources are documented in the [ST board description](https://www.st.com/en/evaluation-tools/stm32f3discovery.html) and [STM32F303xB/xC datasheet, DS9118](https://www.st.com/resource/en/datasheet/stm32f303vc.pdf).

## USB and debug connections

The board has two Mini-B USB connections with different purposes:

- **USB ST-LINK:** programming, debugging, and a supported power source.
- **USB USER:** connected to the target MCU for application USB device communication and supported standalone power.

Use ST-LINK for the first bring-up. Application data will later use USB USER. ST-LINK virtual serial port availability depends on board revision and is not the planned analyzer data connection. Check the exact revision and jumper/power arrangement in [UM1570, the board user manual](https://www.st.com/resource/en/user_manual/um1570-discovery-kit-with-stm32f303vc-mcu-stmicroelectronics.pdf). Obtain the matching board schematic from the CAD resources on the [ST board page](https://www.st.com/en/evaluation-tools/stm32f3discovery.html).

## Acquisition review before wiring

The channel mapping is **not assigned yet**. V0.2 must record:

1. Board revision, MCU package, and header accessibility; fit suitable headers if required.
2. Candidate CH0–CH7 inputs, preferably on one GPIO port, and the bit-to-channel mapping.
3. Conflicts with ST-LINK, USB, user LEDs, sensors, buttons, oscillator pins, or other onboard circuitry. A pin being exposed on a header does not make it electrically unused.
4. Timer clock source, prescalers, DMA request mapping, source-register accessibility, transfer widths, and bus contention constraints.
5. DMA-accessible buffer placement, allocation size, alignment, stack margin, and USB memory budget.
6. Measured sample timing, signal-source connection, and the conditions needed to establish continuity.

Use [RM0316, the MCU reference manual](https://www.st.com/resource/en/reference_manual/dm00043574.pdf) for the DMA request tables, GPIO registers, timer behavior, and memory map. Confirm applicability to STM32F303xC and review the relevant errata linked from the [MCU product documentation](https://www.st.com/en/microcontrollers-microprocessors/stm32f303vc.html).

## Input electrical limits

The initial project baseline is 3.3 V logic with a common ground. Verify each selected pin's input thresholds, absolute limits, and tolerance conditions in DS9118. Some pins are 5 V tolerant under specified conditions; this is not a board-wide guarantee. Onboard circuits can impose additional constraints.

Development-board pins do not provide the protected, high-impedance front-end planned for V3. Choose any series resistance or protection based on the input circuitry and required edge timing. Document allowed signal levels before connecting a device under test (DUT), the circuit being observed. Keep signal and ground wiring short for initial tests.

## What remains unverified

No pin mapping, timer/DMA acquisition route, buffer depth, maximum rate, USB class, or electrical front-end has been selected or qualified. Record those decisions and measured results here as the [V0 milestones](../roadmap.md) progress. The H723 target of 50 MS/s is not a specification for this board.
