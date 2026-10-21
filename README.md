# FSAE Backup CAN Logger

**Status:** Hardware architecture selected; firmware and breadboard validation not yet complete.  
**Updated:** October 21, 2026  
**Target:** A practical demonstration for the FSAE electrical-team freshmen on October 22, 2026, at 8 PM.

> **Purpose:** A low-cost, independent backup system that passively captures the car's Haltech CAN traffic onto microSD and presents recorded runs as ordinary files on Windows, macOS, and Linux. This is a data-collection accessory, **not** a replacement for the ECU/PDM, safety systems, or the main DAQ.

## 1. The design in one minute

- **Microcontroller:** Existing Raspberry Pi **Pico / RP2040**; no replacement development board.
- **CAN interface:** **Waveshare Pico-CAN-B** (SKU 23775) with MCP2515, SIT65HVD230DR, 16 MHz crystal, 3.3 V logic. This replaces the user's existing **8 MHz MCP2515/TJA1050** breakout, which cannot be set to compliant 1 Mbit/s CAN timing.
- **Storage:** Existing SPI microSD breakout, on a separate RP2040 SPI peripheral; breakout model/electrical requirements **still need verification**.
- **Computer connection:** Pico's built-in USB device controller running **USB Mass Storage Class (MSC)**, backed by the microSD card. A computer uses its normal file manager: no serial commands, Python, Wi-Fi, or custom host driver for **retrieving** recordings.
- **USB socket:** Use the Pico's existing **Micro-USB socket on the bench**. For the finished carrier PCB, investigate wiring the already-owned **Adafruit #4090 USB-C breakout** to the Pico's underside USB test pads with short matched traces. Do not connect two simultaneous host cables.
- **Controls:** Button; green power-good indicator; dim/bright red recording indicator; yellow USB-presence indicator.
- **Vehicle power:** A **regulated 5 V supply** fed from a suitable, configured and protected PD16 **battery-voltage** output, **not** the PD16 5 V sensor reference. The final converter and source-selection/backfeed protection circuit are not yet fixed.
- **System software:** C++ using Arduino-Pico, a proven microSD library, TinyUSB MSC, and an MCP2515 driver. **No FreeRTOS required for the first milestone.** PC-side Python handles *optional* decoding/graphing after users have already copied data from the USB drive.
- **Mechanical:** Eventual 2-layer KiCad carrier PCB for the existing modules and a 3D-printed enclosure. An editable **Fritzing wiring sketch is a separate upcoming artifact**. This README does not claim that the wiring is physically verified yet.

### What we already have versus what is new

| Part | Source | Notes |
|---|---|---|
| Raspberry Pi Pico/RP2040 board | Already owned | Assumes a **standard 40-pin Pico pinout**; check variant |
| SPI microSD breakout and microSD card | Already owned | Confirm VCC requirements and module labeling |
| Adafruit 4090 USB-C connector breakout | Already owned | Future enclosure connector, not an SD reader |
| Pushbutton and green/red/yellow LEDs | Already owned | Resistors required |
| **Waveshare Pico-CAN-B** | One replacement module | Needed for 1 Mbit/s and direct 3.3 V interfacing |
| 12 V/battery to 5 V buck converter | Planned order | Only necessary for powered operation in the vehicle |
| Typical resistors/capacitors and connectors | Bench stock | See electrical notes |

**Do not purchase another RP2040, a USB-storage shield, an ESP32, a logic-level translator, or a separate CAN transceiver for this design.** The Waveshare module includes both controller and transceiver and is specified for 3.3 V operation.

## 2. User-facing behavior

1. When vehicle power is applied, the device initializes its CAN receiver and SD card.
2. **Red dim solid** means the logger is ready. Press the button to create a new run.
3. **Red bright, blinking once per second** means recording is active. Every received frame is timestamped and stored, including frames whose meaning is not yet known.
4. Press the button again to stop. The red LED goes **off** while buffers are drained and the FAT filesystem is flushed/closed; once complete the red LED returns to dim solid.
5. Connect a computer over USB. Recording stops/is disabled, the microSD is exclusively handed to the USB mass-storage device, and the computer sees a disk (recommended FAT32 volume label `FSAE_LOGGER`). Users can drag files into a folder without special software.
6. Safely eject the disk before unplugging. After USB disconnect, the logger may return to ready mode once the microSD is reinitialized.
7. Optional: a Python analysis script on an engineering laptop picks the most recent run, decodes the known Haltech channels, and generates graphs. **Python must never be required merely to retrieve log files.**

### LED definitions

| LED | Meaning | Recommended implementation |
|---|---|---|
| Green | Regulated supply in acceptable operating range | GPIO output based on a divider/ADC measurement of the 5 V system rail; do not equate an LED connected to power with precision voltage supervision |
| Red dim solid | SD + logging subsystem initialized, ready | RP2040 PWM low duty cycle |
| Red bright blink at 1 Hz | Recording | RP2040 PWM high duty cycle, 1-second blink period |
| Red off | Booting, finalizing, USB MSC or fault | State machine |
| Yellow on | USB host power (VBUS) is present | Pico's onboard USB VBUS-sense path (GPIO24 on a **standard Pico**) drives separate GPIO LED; host enumeration is a different status |

**Fail-safe behavior:** Never indicate 'recording' when the SD card is missing, full, unmounted, or has failed writes. Log faults/counters in a sidecar when possible.

## 3. Electrical interfaces and proposed GPIO map

**These are proposed electrical assignments, not a finished PCB netlist.** Cross-check the actual Pico variant, SD module pin labels, Waveshare board revision, and the physical board stacking clearance before wiring.

### CAN: Waveshare Pico-CAN-B (manufacturer schematic)

The board contains an MCP2515 with a **16 MHz oscillator** and an **SN65HVD230-family 3.3 V CAN transceiver** (Waveshare product description names it SIT65HVD230DR). The schematic connects:

| CAN signal | RP2040 GPIO | Pico physical pin | Function |
|---|---:|---:|---|
| SPI0 MISO (MCP2515 SO) | GP4 | 6 | Controller -> Pico |
| SPI0 CS0 | GP5 | 7 | Chip select |
| SPI0 SCK | GP6 | 9 | Clock |
| SPI0 MOSI (MCP2515 SI) | GP7 | 10 | Pico -> controller |
| MCP2515 INT | GP21 | 27 | Receive/event interrupt (default 0-ohm option) |
| CAN H | Module terminal | — | CAN high |
| CAN L | Module terminal | — | CAN low |
| CAN ground | Module terminal | — | Reference / harness ground, verify terminal orientation |

**Conflict resolved:** The previous draft used **GP5 for a green/red indicator**; that GPIO now belongs to the CAN module. **Do not use GP5 as an LED.** No additional SPI-level converter is needed with this specified Waveshare module.

**Bus configuration:** Classic CAN, 1,000,000 bit/s, standard 11-bit identifiers. Configure the MCP2515 for its **actual 16 MHz oscillator** and **listen-only receive**; the logger must not transmit commands, acknowledgments or error frames onto the running vehicle's bus. Other active nodes supply any needed ACKs. Verify behavior on an isolated test bus before connecting to the car.

**Termination:** The Waveshare board has a selectable **120 Ω** termination. The existing bus should have **two 120 Ω terminations at its physical endpoints**. If the logger is a short additional stub on an already terminated bus, leave the module's termination **off**. Do not casually add a third termination resistor.

**Important receive limitation:** MCP2515 provides only two hardware receive buffers; SD write stalls can therefore cause real CAN losses even with a RAM queue. Count MCP2515 RX overflow and firmware queue overflow separately, make draining CAN a priority, and load-test under representative bus traffic.

### microSD: separate RP2040 SPI1 peripheral

| SD signal (confirm module silkscreen) | RP2040 GPIO | Pico physical pin |
|---|---:|---:|
| DO / MISO | GP12 | 16 |
| CS / SS | GP13 | 17 |
| CLK / SCK | GP14 | 19 |
| DI / MOSI | GP15 | 20 |
| VCC | 3V3(OUT) **only if the module is 3.3 V compatible** | 36 |
| GND | GND | Any suitable GND pin |

**Breakout specification is still unknown.** Some SD breakout boards expect a **5 V input** because they carry an onboard regulator/level shifting; some expect only **3.3 V**. Confirm the actual board before applying power. Native microSD cards are 3.3 V devices. Put **100 nF ceramic + around 10 µF bulk capacitance** near the module supply, subject to what the module already provides. All grounds must share a valid common reference.

### Controls and indicators (nonconflicting GPIOs)

| Function | RP2040 GPIO | Pico physical pin | Connection |
|---|---:|---:|---|
| Record button | GP2 | 4 | Normally-open switch to GND; internal pull-up; debounce in firmware |
| Red recording LED | GP3 | 5 | GPIO -> 330-1k Ω resistor -> LED -> GND; PWM brightness |
| Green power LED | GP8 | 11 | GPIO -> 330-1k Ω resistor -> LED -> GND |
| Yellow USB LED | GP9 | 12 | GPIO -> 330-1k Ω resistor -> LED -> GND |
| Regulated 5 V rail monitor | GP26 / ADC0 | 31 | Through a voltage divider only; **never feed 5 V directly to a GPIO** |
| Native USB VBUS sense | GP24 (internal Pico wiring) | Not header-accessible | Reads presence of USB VBUS on standard Pico |

**Example voltage divider:** 100 kΩ from regulated 5 V rail to GP26, 100 kΩ from GP26 to ground. At a 5.5 V maximum rail, ADC input is about 2.75 V. Select and calibrate a threshold in firmware; green LED only means the monitored rail passes the defined threshold, not that the power supply is automotive-certified. Confirm ADC reading stability in USB-only operation. Keep LED forward current within Pico GPIO constraints; 330-1k Ω series resistance is appropriate for preliminary testing.

### USB mass storage and the eventual USB-C interface

- **Bench:** Use the Pico's own **Micro-USB** receptacle. The RP2040 already has native USB device hardware; no USB bridge module is required.
- **Storage firmware:** Use Adafruit TinyUSB MSC on Arduino-Pico. A supported SD-card MSC example exists, but it must be adapted to the specific SPI bus and to exclusive SD ownership.
- **Carrier PCB concept:** The standard Pico exposes USB **D- on TP2** and **D+ on TP3** underneath the board; **TP1 is USB-adjacent ground**. The Adafruit #4090 exposes USB-C D-/D+/VBUS/GND and incorporates CC resistors. A two-layer carrier can route the short D+/D- pair over continuous ground with appropriate differential geometry; exact impedance is layout-dependent. Accessing Pico underside pads will require a sound manufacturing/contact method; standard headers alone do not connect the test pads.
- **USB-C caveat:** Connecting a separate USB-C socket to TP2/TP3 leaves the Pico's own Micro-USB receptacle electrically on the same USB interface. **Only one cable/host may be used at a time.** Review stubs, ESD protection, power routing and mechanical reliability before fabrication.
- **Power caveat:** The USB VBUS conductor must not be hard-shorted to a powered vehicle 5 V buck output. Plan proper power-source selection/backfeed blocking. This is deliberately not finalized in the breadboard sketch.
- **Filesystem ownership:** Never have MCU FAT writes active while USB MSC is exporting raw SD sectors. On USB insertion, end the run, flush, close, synchronize the writer, and only then make MSC ready. In MSC mode, the internal logger is locked out. Start with read-only MSC if reliable host read-only behavior is acceptable; test on all three operating systems.
- **Volume label:** Set FAT32 volume label to `FSAE_LOGGER` when formatting the card; the USB product string does not itself set the filesystem volume label.

### Vehicle power (still to be finalized)

The PD16 has a dedicated **5 V sensor supply limited to 200 mA total** (PD16 wiring diagram, Connector A pin 12). This is **not** recommended for the logger; its headroom is shared with other sensors. Configurable high-current PD16 outputs provide **switched battery voltage**, not an adjustable regulated 5 V output. **PWM averaging is not a safe substitute for a buck regulator.**

The proposed final route is: appropriately programmed/fused PD16 high-side battery output -> suitable 12 V-to-5 V buck converter -> protected, backfeed-safe 5 V logger supply -> Pico VSYS and any compatible peripherals. Keep USB power availability for file retrieval with the vehicle switched off. Test supply behavior during startup/shutdown and SD writes; an ordinary low-cost buck should not automatically be treated as a fully transient-qualified automotive design.

**For today, use the Pico's normal USB power. No buck converter is required for simulated data/SD/MSC development at the desk.**

## 4. Firmware architecture (proposed; code not started)

```text
Waveshare CAN H/L -> 16 MHz MCP2515/SIT65HVD230 -> SPI0 + GP21 interrupt
                                                    |
                                         Receive + timestamp raw frame
                                                    |
                                               RAM ring buffer
                                                    |
                                       batched, sequential SD writer
                                                    |
                           SPI1 -> microSD FAT32 -> RUN00001.BIN, etc.
                                                    |
                      exclusive ownership handoff (recording stopped)
                                                    |
                     Native RP2040 USB device -> USB MSC -> host file manager
```

Design principles:

1. **Log raw CAN frames first; decode later.** This preserves Elite ECU, PD16, wideband and other traffic even when interpretation is incomplete.
2. Start with a simple nonblocking single-core implementation for generated traffic and SD/USB bring-up. Once hardware CAN is connected, test whether SD stalls cause MCP2515 hardware overrun. If necessary, use RP2040's two cores with a synchronized queue and separate SPI peripherals: one core continuously services CAN; the other writes SD. Two SPI interfaces **alone** do not make CPU operations concurrent.
3. Avoid FreeRTOS for v1. It is an option only if measurable timing/ownership requirements justify it.
4. Allocate fixed-size buffers; prohibit indefinite blocking in the receive path. Record dropped-frame/overflow/write-error counters rather than silently misrepresenting missing data.
5. Have exactly one owner of the SD controller/filesystem at a time: recorder **or** USB host mass storage.
6. USB storage and recording are **mutually exclusive**. USB attachment during recording should stop and finalize first, then enumerate or report unit-not-ready until the card is released.
7. Storage is a **best-effort backup** without a guaranteed power-fail hold-up. Periodically flush and validate how logs behave under unexpected removal of power.

### Logical states

```text
BOOT -> READY -> RECORDING -> FINALIZING -> READY
            \                         /
             \--- USB insertion -----/ -> USB_MSC -> (USB disconnect) -> READY
Any state --(unrecoverable fault)--> ERROR
```

USB MSC must never advertise the card as ready while the MCU still has open log files. A USB cable connected to a USB-powered logger is not the same as a connected host successfully mounting a filesystem: separate those diagnostics when testing.

## 5. Log file format (design proposal; freeze before code generation)

Use FAT32 and incrementing session filenames such as `RUN00001.BIN` plus a human-readable `RUN00001.JSON` summary on a **clean stop**. No real-time clock is assumed; filename/session index, not creation date, defines the latest run.

The format below is a proposal from the first planning draft, retained for continuity. It is **not** a Haltech proprietary format and not yet implemented.

**32-byte header** (all container integers little-endian):

| Offset | Bytes | Field |
|---:|---:|---|
| 0 | 8 | ASCII `FSAECAN1` |
| 8 | 2 | Format version = 1 (`uint16`) |
| 10 | 2 | Header size = 32 (`uint16`) |
| 12 | 2 | Record size = 24 (`uint16`) |
| 14 | 2 | Flags = 0 (`uint16`) |
| 16 | 4 | Timebase ticks per second = 1,000,000 (`uint32`) |
| 20 | 4 | Session number (`uint32`) |
| 24 | 8 | Reserved zeros |

**24-byte frame record** (`struct` format `<QHBB8sI`):

| Offset | Bytes | Field |
|---:|---:|---|
| 0 | 8 | Microseconds since start of run (`uint64`) |
| 8 | 2 | Standard CAN ID (`uint16`, 0–0x7FF) |
| 10 | 1 | DLC (`uint8`, 0–8) |
| 11 | 1 | Flags; zero for ordinary standard CAN data frame |
| 12 | 8 | Raw CAN payload bytes, zero-padded only beyond DLC |
| 20 | 4 | Monotonic sequence number (`uint32`) |

**Important:** Haltech's **payload bytes remain unchanged**. The binary container's little-endian integers and Haltech's big-endian *signal interpretation* are separate concepts. Include a clean-stop JSON with frame counts, queue drops, MCP2515 overflow counts, SD write errors, bitrate/oscillator, session number, and whether finalization succeeded. Be tolerant of truncation or missing JSON after sudden loss of power.

Example bandwidth: 5,000 recorded frames/s × 24 bytes = **120,000 bytes/s** not counting filesystem overhead. Size the RAM queue according to measurement, not that one example alone.

## 6. CAN protocol notes from the attached Haltech PDFs

**ECU Broadcast CAN Protocol v2.0:** 1 Mbit/s; standard 11-bit IDs; big-endian multibyte fields; listed ECU Elite firmware **3.11+** for full support. Available broadcast channels depend on the ECU firmware and actual NSP setup. A CAN logger cannot access arbitrary internal ECU logger channels that are not transmitted on the bus.

| ECU ID | Documented rate | Sample decoding targets |
|---|---:|---|
| `0x360` | 50 Hz | Bytes 0–1 RPM (`u16be`); 2–3 manifold pressure kPa absolute (`u16be / 10`); 4–5 throttle % (`u16be / 10`) |
| `0x361` | 50 Hz | 0–1 fuel pressure kPa gauge (`u16be/10 - 101.3`); 2–3 oil pressure gauge (same) |
| `0x362` | 50 Hz | 0–1 and 2–3 injector duty cycles; 4–5 ignition angle (`s16be/10`) |
| `0x368` | 20 Hz | Four wideband lambda values: consecutive 16-bit unsigned words divided by 1,000 |
| `0x36C` | 20 Hz | Four wheel speeds in km/h: four unsigned 16-bit words divided by 10 |
| `0x372` | 10 Hz | 0–1 battery voltage V (`u16be/10`) |
| `0x3E0` | 5 Hz | Four temperatures: each `u16be/10` Kelvin, then subtract 273.15 for Celsius |

**PD16 Multiplexed CAN Protocol v2.0 (newly supplied PDF):** Also uses 1 Mbit/s, 11-bit IDs, and big-endian values. PD16 base CAN identifiers: **A `0x6D0`, B `0x6D8`, C `0x6E0`, D `0x6E8`**. The first payload byte of a multiplexed message selects IO type/index; **do not** interpret every message sharing a CAN ID as the same channel without using that multiplex field. The document separately defines **PD16-to-ECU** broadcast IDs for monitored input states (PD16 A includes `0x320`, `0x324`, `0x328` at listed 100 Hz rates). Device commands and reported state data are different directions. Read the protocol tables before building a full decoder.

See PDF source files at project root or team repository `docs/` folder, as available:

- `Broadcast_CAN_Protocol_Document_Version_2.pdf` — ECU protocol, pp. 2–7 and subsequent channel tables.
- `Multiplexed_CAN_Protocol_Document_Version_2.pdf` — PD16 support/IDs, pp. 3, 5, 13–17 and later tables.
- `haltech-can-protocol-specification.pdf` — supplementary bit addressing, units and conventions.

The ECU's `0x368` values are **ECU-broadcast wideband channels**; an independent wideband module may use another protocol. Record every ID and determine what really appears on the car before promising channel coverage.

## 7. Desktop analysis expectations

The **USB drive** is the nontechnical user's default interface. Its raw run files must be accessible without Python. A separate, engineering-oriented Python program may:

1. Select the numerically highest valid `RUNxxxxx.BIN`, or accept a specific file path.
2. Validate header, DLCs, record length, monotonic time/sequence, and truncated tail data.
3. Decode known Elite/PD16 channels with proper byte order, signedness, rates and scaling; retain unknown frames.
4. Plot selected channels against **run-relative time** (RPM, throttle, MAP, fuel/oil pressure, coolant temperature, voltage, lambda, PD16 power-channel data where documented/observed).
5. Export selected channels as CSV for Excel or other tools; a double-clickable desktop front end can come later, but is not needed merely to copy logs.

## 8. Recommended development stages

| Milestone | What to demonstrate | Requires new CAN module? |
|---|---|---|
| 1. SD smoke test | Initialize card; write/read a test file | No |
| 2. Simulated run capture | Generate plausible test CAN frames, record on button press and stop reliably | No |
| 3. Native USB mass storage | Mount card in Windows/Linux/macOS file manager via Pico Micro-USB; safely copy files | No |
| 4. Indicators and ownership | LED states; no simultaneous host/MCU card access; repeated start/stop/eject cycles | No |
| 5. PC decoder/plots | Convert simulated log to sensible graphs / CSV | No |
| 6. Receive CAN traffic | Initialize Waveshare MCP2515 at 16 MHz/1 Mbps, listen-only, count overflows | **Yes** |
| 7. Stress test | CAN bursts while SD writes, verify no silent losses | **Yes** |
| 8. Car integration | Check vehicle CAN termination/bus configuration and safely compare data with NSP | **Yes** |
| 9. Carrier and enclosure | Fritzing wiring sketch, KiCad PCB, USB-C routing, final power circuit | Physical measurements and review |

**Meeting goal:** Demonstrate steps 1–5 if time permits, with generated data; clearly label simulated traffic so it is not confused with live car data. A functioning, honestly limited prototype is better than an unverified complete system.

### Draft repository layout

```text
fsae-can-logger/
├── README.md
├── firmware/
│   ├── include/           # config/pin map; logging record definition
│   └── src/               # app, state machine, CAN, SD, USB MSC, LEDs
├── tools/
│   └── plot_latest.py     # engineering-side decoder and plots (planned)
├── tests/
│   └── ...                # fake frames, binary parser tests (planned)
├── hardware/
│   ├── fritzing/          # future editable .fzz sketch
│   └── kicad/             # future carrier board
└── docs/
    └── ...                # optional copies of Haltech PDFs
```

## 9. Outstanding questions (before fabrication, not blockers for milestone 1)

- Exact model/photo of microSD breakout; verify whether its **VCC** is 3.3 V or 5 V, presence of level shifting and on-board capacitors.
- Exact variant of the Pico and access to its pins while Pico-CAN-B is stacked.
- Whether Waveshare module arrives as ordered; verify its crystal and physical jumper position. Confirm **MCP2515 INT on GP21** and controller CS on GP5 on the delivered board.
- Existing CAN physical-end termination, tap length, and Elite CAN broadcast configuration.
- Selected PD16 high-side power pin, programmed fuse/enable behavior, regulated 5 V converter, and USB/vehicle power source-selection protection.
- Mechanical arrangement and tolerances of the Pico, CAN module, SD breakout and USB-C breakout, including Pico underside USB test-pad contact plan.
- Whether the initial USB drive will be **read-only** or writable by the host; in either case enforce exclusive SD ownership.
- Whether USB should stop an ongoing run automatically or require the operator to press Stop first. Default recommendation: automatic safe finalization when feasible, never immediate raw mounting during active recording.

## 10. Updated and spelling-corrected project brief

> I have an opportunity to develop a backup data logger for our FSAE team's car. Our primary data acquisition system may not be fully reliable or ready for testing, so I want an independent system that collects data already transmitted on the car's CAN network. I want a concrete proposal and ideally a small demonstration for our freshmen team members tomorrow at 8 PM.
>
> The car uses a Haltech Elite 1500 ECU, a Haltech Nexus PD16 power distribution module, and a Haltech CAN-connected oxygen sensor. Much of the information we want is already on the CAN bus. The backup device should capture complete raw CAN frames to a microSD card and allow us to decode and graph selected measurements after a run.
>
> The most important usability requirement is that any team member, including mechanical engineers who do not know Python or command-line tools, can plug the device into a Windows, macOS, or Linux laptop and see the recorded files in the operating system's file manager. Wi-Fi is not suitable as the primary retrieval method because the laptop may not be near the car after a run, and connecting the car to the university network is inconvenient.
>
> Hardware requirements: a CAN reader compatible with Haltech's 1 Mbit/s, 11-bit classic CAN network; an SPI microSD card interface; native USB mass storage, eventually through our Adafruit #4090 USB-C connector breakout; a green adequate-power LED; a red status LED that is faint and steady in standby, bright and blinking once per second during recording, and off while processing or not ready; a yellow USB-connected LED; and a pushbutton to start/stop recording. The unit should eventually be powered by a protected, regulated 5 V source from the PD16 system.
>
> We chose our existing RP2040/Pico because it is inexpensive, has native USB device support and enough capacity for a small CAN-to-SD logger. Our current MCP2515/TJA1050 module has an 8 MHz crystal and is not a good match for 1 Mbit/s operation, so we are replacing **only that CAN module** with a Waveshare Pico-CAN-B, which includes a 16 MHz MCP2515 and 3.3 V CAN transceiver. We will reuse our existing microSD breakout, LEDs, pushbutton and USB-C breakout. We do not need FreeRTOS unless later measurements demonstrate a need.
>
> Eventually the components will mount to a two-layer KiCad carrier PCB manufactured at school, inside a 3D-printed enclosure. We may use Fritzing for a clear editable breadboard wiring diagram. The design does not have to be waterproof but must be electrically safe for a passive, receive-only connection to the car and able to produce useful, retrievable logs.
>
> Please help validate the hardware, architecture and pinout; identify specification mismatches; document the system and protocols; plan realistic development phases; and prepare reviewed, modular prompts for Claude to implement the firmware and Python analysis tools. Do not begin generating code until we approve the design and set up the development environment and breadboard.

## 11. Verified reference links

- [Waveshare Pico-CAN-B product page (SKU 23775)](https://www.waveshare.com/product/pico-can-b.htm)
- [Waveshare Pico-CAN-B schematic (Pico pin map, controller/transceiver, 16 MHz clock)](https://files.waveshare.com/upload/1/13/PICO-CAN-B-sch.pdf)
- [Waveshare Pico-CAN-B documentation](https://www.waveshare.com/wiki/Pico-CAN-B)
- [Raspberry Pi Pico datasheet (pinout, USB TP1/TP2/TP3, supply constraints)](https://datasheets.raspberrypi.com/pico/pico-datasheet.pdf)
- [Adafruit USB-C breakout #4090](https://www.adafruit.com/product/4090)
- [Arduino-Pico USB / Adafruit TinyUSB documentation](https://arduino-pico.readthedocs.io/en/latest/usb.html)
- [Adafruit TinyUSB SD-card USB MSC example](https://github.com/adafruit/Adafruit_TinyUSB_Arduino/blob/master/examples/MassStorage/msc_sd/msc_sd.ino)
- [Haltech PD16 wiring diagram (5 V sensor supply: 200 mA maximum)](https://www.jbtechracing.com/wp-content/uploads/2023/10/PD16_WiringDiagram-1.pdf)
- [Haltech Elite/ECU broadcast CAN reference](https://support.haltech.com/portal/en/kb/articles/haltech-can-ecu-broadcast-protocol)
- [Haltech PD16 multiplexed CAN reference](https://support.haltech.com/portal/en/kb/articles/haltech-can-pd-16-protocol)
- [Fritzing application](https://github.com/fritzing/fritzing-app)
- [Fritzing parts library](https://github.com/fritzing/fritzing-parts)
- [Fritzing editable `.fzz` sketch format](https://github.com/fritzing/fritzing-app/wiki/2.2-Sketch-file-format)

---

**Validation status:** This README is a reviewed *design specification*, not a proof of successful bench or vehicle operation. The SPI pin assignment for the Waveshare CAN module is based on its published schematic. The SD breakout, dual-power circuit and USB-C carrier implementation must be confirmed on the actual hardware before making a production PCB or tapping the live car network.
