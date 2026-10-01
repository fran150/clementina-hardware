# Clementina Hardware

KiCad 9 schematic and board for Clementina, a 3.3 V homebrew computer built
around a W65C02S CPU. A Raspberry Pi Pico 2 W, called MIA, runs the machine: it
generates the CPU clock, boots the kernel, and provides video, audio, storage
and input.

Status: the rev 1 schematic is complete and ERC-clean, and the rev 1 board is
laid out, fully routed and DRC-clean. The first build is planned on
protoboard, using the same parts as the board.

## Opening the project

Open `clementina-hardware.kicad_pro` in KiCad 9.

- `clementina.kicad_sym` (listed in `sym-lib-table`) holds the parts KiCad's
  own libraries lack, currently the AS6C62256 SRAM.
- The W65C02S and W65C22S symbols come from KiCad's Plugin and Content Manager
  (library `PCM_65xx-library`). The schematic carries copies, so it opens
  without that library. Install it to keep ERC's library check clean.
- One ERC item is excluded on purpose: the Pico's AGND tied to GND. The ADC is
  unused, and the Pico datasheet allows the join.
- `clementina-hardware.kicad_dru` holds one custom DRC rule: the audio jack's
  barrel hangs over the board edge on purpose, so its silkscreen outline may
  cross the edge.
- DRC shows one warning, and it is expected: the Pico footprint "does not
  match the copy in the library". The library's antenna keep-out lists 32
  copper layers, and a 2-layer board keeps only two of them.

Sheets:

| Sheet | Contents |
| --- | --- |
| `clementina-hardware.kicad_sch` (A3) | CPU, VIA, RAM, MIA, IRQ and reset circuits, expansion headers, user port, audio output, microSD, power and decoupling |
| `CS Logic.kicad_sch` | Address decoding (74AC138 x2, 74AC08, two gates of the 74AC00) |
| `OE_RW_PHI2_Sync.kicad_sch` | Read and write strobes: `~OE` from R/W, `~WE` qualified by PHI2 (two gates of the 74AC00) |

## Architecture

| Part | Role |
| --- | --- |
| W65C02S (U1) | CPU. MIA drives PHI2: 1.2 MHz by default, up to 8 MHz. |
| AS6C62256 (U4) | 32 KB base RAM |
| AS6C4008 (U2) | 512 KB extended RAM, seen through a 16 KB window |
| W65C22S (U3) | VIA. Port A selects the extended RAM bank; port B and CA/CB go to the user port. |
| Raspberry Pi Pico 2 W (A1) | MIA: clock, boot loader, video over Wi-Fi, audio, microSD, input, reset |
| 74AC138 x2, 74AC08, 74AC00 | Glue logic (see [Glue logic timing](#glue-logic-timing)) |

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

### Glue logic timing

The glue logic is 74AC, not 74HC, because 74HC is slow at 3.3 V (roughly
13 ns per gate typical, up to about 30 ns). Two places depend on it:

- **Write strobe.** `~WE = NAND(PHI2, ~OE)` in one 74AC00 gate, so `~WE`
  rises within about 10 ns of PHI2 falling. The W65C02S only guarantees its
  address and data for 10 ns after PHI2 falls, and the SRAMs need both held
  until `~WE` rises. This timing doesn't depend on clock speed, so running
  slower doesn't help. The Picocomputer RP6502 makes WE# the same way.
- **Chip selects.** The VIA and the expansion cards need their chip select
  10 ns before PHI2 rises. U6 is enabled straight from A13-A15 (A15
  through one inverting gate), not through U8, which removes a decoder level
  from those selects.

With 74AC parts, the chip-select paths meet the datasheets up to about
6.5 MHz with worst-case parts, and typical parts reach 8 MHz. 74HC parts give
the same logic but lose the write-strobe guarantee, and the VIA's chip select
limits them to about 3.5 MHz worst case.

Interrupts are wired-AND: MIA and the VIA each pull `~IRQB` low through a BAT85
Schottky diode against a 4.7 kΩ pull-up. `~NMI` (4.7 kΩ) and RDY (2.2 kΩ) are
also pulled up, so expansion cards can use them. RDY is also an output: the
CPU pulls it low while it executes `WAI`. The W65C02S only guarantees 1.6 mA
at 0.4 V there, so the RDY pull-up can't be stronger than about 2 kΩ.

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
- RDY also goes low while the CPU executes `WAI`. Don't read RDY low as a
  wait request from another card.
- `~RESB` is driven by MIA: read it, never drive it.
- W65C22S port pins have no current limiting. Drive LEDs and similar loads
  through a resistor.

**J4: audio out.** A stereo 3.5 mm jack (CUI SJ1-3533NG, through-hole, round
pins): tip left, ring right. Each channel
uses the same output stage as the Picocomputer RP6502: a 220 Ω / 100 Ω
divider, a 100 nF filter capacitor, a 47 µF DC block and a 1.8 kΩ bleed
resistor. It gives about 1 Vpp line level and drives headphones directly. MIA
mixes at 48 kHz.

**J5: microSD.** A 1x9 female header for the Adafruit 4682 microSD breakout
(3 V only), used in SPI mode on MIA's SPI0: 400 kHz to initialise, then
12 MHz. The breakout has its own pull-ups (CS, MISO, DAT1, DAT2) and bypass
capacitors. DAT1, DAT2 and card detect stay unconnected.

| Pin | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Breakout | 3V | GND | CLK | SO | SI | CS | DAT1 | DAT2 | DET |
| Signal | +3.3V | GND | SD_SCK | SD_MISO | SD_MOSI | SD_CS | - | - | - |

## Board (rev 1)

`clementina-hardware.kicad_pcb`: 172.7 x 108.6 mm, 2 layers, every part
through-hole. The chips sit in sockets and the Pico plugs into female headers,
so the same parts move from the protoboard build to the board.

- Layout: the MIA Pico is at the top left, with its USB port on the top edge
  and its Wi-Fi antenna over a copper keep-out. The microSD breakout sits to
  its left, with the card slot at the left board edge. The audio stage and jack
  run down the left edge, and the reset button sits well below the Pico. The CPU,
  RAM, extended RAM and VIA form one row, with the user port at the right edge.
  The glue logic is in a row below them, and the two expansion headers are
  stacked along the bottom edge.
- Every chip has a 100 nF capacitor next to its supply pin; C14 (22 µF) is by
  the Pico's 3V3 pin.
- Wi-Fi antenna: Raspberry Pi recommends putting the Pico W's antenna at a
  board edge with nothing close to it. Here it sits inside the board, as on the
  Picocomputer RP6502. The Pico stands about 10 mm up on its headers, no copper
  runs under the antenna, and a rule area ("Wi-Fi antenna clearance") keeps
  tracks, vias and ground fill about 6 mm past the Pico's end. Leave that area
  clear of parts and cables. The expected cost is a few dB of range, which the
  video protocol tolerates. To check a build, compare the Wi-Fi signal
  strength (RSSI) of the Pico on its own and on the board at the same spot; if
  the board costs more than about 6 dB, move the antenna to an edge in rev 2.
- Routing: 0.25 mm signal tracks, 0.4 mm for GND, +3.3V and +5V (net class
  `Power`), 0.2 mm clearance, and 0.6 / 0.3 mm vias. The top layer runs mostly
  horizontal and the bottom mostly vertical. Both layers carry a GND pour,
  stitched with vias. Any hobby PCB fab can make it (1.6 mm FR-4, 1 oz copper).
- The routing was produced by a script and then checked with KiCad's DRC and
  its schematic parity check. Edit it freely in KiCad.
- Silkscreen: part values are printed on the resistors and capacitors, and the
  chip names inside the socket outlines. The J5 pin names and the breakout's
  outline are printed too.
- Mounting: four M3 holes in the corners. Two M2.5 holes under the microSD
  breakout match its own holes. They're optional, to support it with
  standoffs (about 11 mm).

### Bill of materials

Every part has two schematic fields that keep the bill of materials generic,
so you can source parts yourself (for the protoboard build) or hand them to an
assembly service:

- **Factory**: what gets soldered to the board at that designator. Chips get a
  DIP socket, and the Pico (A1) gets two 1x20 female headers.
- **Plug-in**: what you plug in afterwards, if anything.

[`bom/factory-bom.csv`](bom/factory-bom.csv) lists everything that's
soldered: passives, diodes, sockets, headers, the jack and the button. With
those soldered (by you or the factory), the board needs no more soldering;
plug in these parts:

| Ref | Part |
| --- | --- |
| U1 | WDC W65C02S CPU, DIP-40 (W65C02S6TPG-14) |
| U3 | WDC W65C22S VIA, DIP-40 (W65C22S6TPG-14) |
| U4 | Alliance AS6C62256-55PCN, 32K x 8 SRAM, DIP-28 |
| U2 | Alliance AS6C4008-55PCN, 512K x 8 SRAM, DIP-32 |
| U6, U8 | 74AC138, DIP-16 |
| U7 | 74AC08, DIP-14 |
| U9 | 74AC00, DIP-14 |
| A1 | Raspberry Pi Pico 2 WH (the version with pins already soldered) |
| J5 | Adafruit 4682 microSD breakout, 3 V |

The Adafruit 4682 ships with its header loose: solder the 9 pins to the
module once, or use a breakout sold with the header fitted.

The "A1" row of the factory BOM is two headers, one per side of the Pico.

To regenerate the factory BOM after changing parts:

```
kicad-cli sch export bom --fields 'Factory,${QUANTITY},Reference' --labels 'Part,Qty,References' --group-by Factory --sort-field Reference --sort-asc -o bom/factory-bom.csv clementina-hardware.kicad_sch
```

### Fabrication files

The Gerbers are generated from the board, so they aren't committed: git
ignores `fabrication/`. Rebuild them after any board change:

```
mkdir -p fabrication/gerbers
kicad-cli pcb export gerbers --no-x2 --no-netlist --subtract-soldermask -l F.Cu,B.Cu,F.Mask,B.Mask,F.Silkscreen,Edge.Cuts -o fabrication/gerbers/ clementina-hardware.kicad_pcb
kicad-cli pcb export drill --excellon-separate-th -o fabrication/gerbers/ clementina-hardware.kicad_pcb
rm fabrication/gerbers/*.gbrjob
zip -j -FS fabrication/clementina-rev1-gerbers.zip fabrication/gerbers/*
```

Upload `fabrication/clementina-rev1-gerbers.zip` to the fab. It holds eight
files: both copper layers, both masks, the front silkscreen, the board outline,
and separate drill files for plated and unplated holes.

- `--no-x2 --no-netlist`: PCBWay's KiCad guide asks for Gerbers without X2
  attributes. With them, its upload preview showed only the holes.
- No back silkscreen: it's empty.
- `--subtract-soldermask` keeps silkscreen ink off the pads.
- The zip leaves out KiCad's `.gbrjob` job file.
- `zip -FS` also removes files that are no longer in `fabrication/gerbers/`.

When you order boards, tag the commit you ordered from and attach the zip you
uploaded to a GitHub release on that tag. That keeps a record of exactly what
was made.

## Not in rev 1

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
