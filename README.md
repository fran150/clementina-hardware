# Clementina Hardware

KiCad 9 schematic for Clementina, a 3.3 V homebrew computer built around a
W65C02S CPU. A Raspberry Pi Pico 2 W, called MIA, runs the machine: it
generates the CPU clock, boots the kernel, and provides video, audio, storage
and input.

Status: the rev 1 schematic is complete and ERC-clean. The board layout is not
started. The first build is planned on protoboard.

## Opening the project

Open `clementina-hardware.kicad_pro` in KiCad 9.

- `clementina.kicad_sym` (listed in `sym-lib-table`) holds the parts KiCad's
  own libraries lack, currently the AS6C62256 SRAM.
- The W65C02S and W65C22S symbols come from KiCad's Plugin and Content Manager
  (library `PCM_65xx-library`). The schematic carries copies, so it opens
  without that library. Install it to keep ERC's library check clean.
- One ERC item is excluded on purpose: the Pico's AGND tied to GND. The ADC is
  unused, and the Pico datasheet allows the join.

Sheets:

| Sheet | Contents |
| --- | --- |
| `clementina-hardware.kicad_sch` (A3) | CPU, VIA, RAM, MIA, IRQ and reset circuits, expansion headers, user port, audio output, microSD, power and decoupling |
| `CS Logic.kicad_sch` | Address decoding (74HC138 x2, 74HC08, 74HC04) |
| `OE_RW_PHI2_Sync.kicad_sch` | Read and write strobes: `~OE` from R/W, `~WE` qualified by PHI2 |

## Architecture

| Part | Role |
| --- | --- |
| W65C02S (U1) | CPU. MIA drives PHI2: 1.2 MHz by default, up to 8 MHz. |
| AS6C62256 (U4) | 32 KB base RAM |
| AS6C4008 (U2) | 512 KB extended RAM, seen through a 16 KB window |
| W65C22S (U3) | VIA. Port A selects the extended RAM bank; port B and CA/CB go to the user port. |
| Raspberry Pi Pico 2 W (A1) | MIA: clock, boot loader, video over Wi-Fi, audio, microSD, input, reset |
| 74HC138 x2, 74HC08, 74HC04 | Glue logic |

Memory map:

| Range | Device |
| --- | --- |
| `$0000-$7FFF` | Base RAM |
| `$8000-$BFFF` | Extended RAM window (32 banks, selected by VIA PA0-PA4) |
| `$C000-$DFFF` | I/O: eight 1 KB slots. Slot 0 (`$C000`) is the VIA; slots 1-7 go to the expansion headers. |
| `$E000-$FFFF` | MIA (registers at `$FFE0-$FFFF`) |

MIA's pins:

| GPIO | Signal |
| --- | --- |
| GP0-GP3 | microSD (SPI0: MISO, CS, SCK, MOSI) |
| GP4, GP5 | Audio left and right (PWM) |
| GP6 | MIA chip select |
| GP7 | R/W |
| GP8-GP15 | D0-D7 |
| GP16-GP20 | A0-A4 |
| GP21 | PHI2 |
| GP22 | IRQ |
| GP26 | RESB (MIA holds the CPU and VIA in reset) |
| GP27 | Reset request from the RESET button |
| GP28 | Unused |

Interrupts are wired-AND: MIA and the VIA each pull `~IRQB` low through a BAT85
Schottky diode against a 4.7 kΩ pull-up. `~NMI` (4.7 kΩ) and RDY (1 kΩ) are
also pulled up, so expansion cards can use them.

## Power (rev 1)

Everything runs from the Pico's USB. Its 3V3 output supplies the whole board,
and VBUS supplies the `+5V` pins on the headers.

- Keep the external 3.3 V load under about 300 mA. The board uses roughly
  50 mA idle and 150 mA with the microSD card writing, which leaves about
  150 mA for expansion cards.
- A phone charger works when no PC is attached.
- VSYS is left free for a rev 2 power input.

## Connectors

**J1, J2: expansion bus** (2x20 keyed box headers, identical)

| Pin | Signal | Pin | Signal |
| ---: | --- | ---: | --- |
| 1 | GND | 2 | +3.3V |
| 3 | A0 | 4 | A1 |
| 5 | A2 | 6 | A3 |
| 7 | A4 | 8 | A5 |
| 9 | A6 | 10 | A7 |
| 11 | A8 | 12 | A9 |
| 13 | GND | 14 | PHI2 |
| 15 | GND | 16 | RWB |
| 17 | ~OE | 18 | ~WE |
| 19 | ~RESB | 20 | ~IRQB |
| 21 | ~NMI | 22 | RDY |
| 23 | D0 | 24 | D1 |
| 25 | D2 | 26 | D3 |
| 27 | D4 | 28 | D5 |
| 29 | D6 | 30 | D7 |
| 31 | ~IOCS1 | 32 | ~IOCS2 |
| 33 | ~IOCS3 | 34 | ~IOCS4 |
| 35 | ~IOCS5 | 36 | ~IOCS6 |
| 37 | ~IOCS7 | 38 | +5V |
| 39 | GND | 40 | +3.3V |

**J3: user port** (2x10 keyed box header, VIA port B)

| Pin | Signal | Pin | Signal |
| ---: | --- | ---: | --- |
| 1 | GND | 2 | +3.3V |
| 3 | PB0 | 4 | PB1 |
| 5 | PB2 | 6 | PB3 |
| 7 | PB4 | 8 | PB5 |
| 9 | PB6 | 10 | PB7 |
| 11 | CB1 | 12 | CB2 |
| 13 | CA1 | 14 | CA2 |
| 15 | ~RESB | 16 | GND |
| 17 | GND | 18 | +5V |
| 19 | GND | 20 | +3.3V |

Rules for cards on J1-J3:

- 3.3 V logic only. Never put a 5 V signal on any pin.
- A card claims one I/O slot by jumpering one of `~IOCS1`-`~IOCS7`.
- `~IRQB`, `~NMI` and RDY: pull them low through a Schottky diode or an
  open-drain output, and never drive them high.
- `~RESB` is driven by MIA: read it, never drive it.
- W65C22S port pins have no current limiting. Drive LEDs and similar loads
  through a resistor.

**J4: audio out.** A stereo 3.5 mm jack: tip left, ring right. Each channel
uses the same output stage as the Picocomputer RP6502: a 220 Ω / 100 Ω
divider, a 100 nF filter capacitor, a 47 µF DC block and a 1.8 kΩ bleed
resistor. It gives about 1 Vpp line level and drives headphones directly. MIA
mixes at 48 kHz.

**J5: microSD.** SPI mode on MIA's SPI0: 400 kHz to initialise, then 12 MHz.
RN1 (4 x 10 kΩ) pulls up CS, MISO and the unused DAT1 and DAT2 lines. On
protoboard, a 3 V-only breakout board works in place of the bare socket.

## Not in rev 1

- Board layout
- A separate power input (USB-C into VSYS) with a jumper to power a USB
  keyboard on the Pico's port
- Local video output, likely a second Pico as a display client

## Related repositories

- [clementina-6502](https://github.com/fran150/clementina-6502): emulator
- [clementina-mia](https://github.com/fran150/clementina-mia): MIA firmware
- [clementina-rom](https://github.com/fran150/clementina-rom): kernel,
  BASIC and WozMon
- [clementina-sdk](https://github.com/fran150/clementina-sdk): development SDK
- [clementina-studio](https://github.com/fran150/clementina-studio): asset
  editor
- [clementina-video-client](https://github.com/fran150/clementina-video-client):
  video client

## License

GPLv3. See [LICENSE](LICENSE).
