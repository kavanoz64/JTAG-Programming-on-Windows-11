# Tigard JTAG Setup on Windows 11

Programming and verifying Xilinx XC9500XL CPLDs (and Lattice ECP5) with a
Tigard FT2232H adapter, from a clean Windows install.

---

## Why this configuration

Two facts drive everything below. Both cost hours to discover.

**Tigard's JTAG header is on FT2232H channel B (Interface 1), not channel A.**
Generic "ft2232" cable definitions in JTAG tools assume channel A. They will
open the adapter successfully, report a correct clock frequency, and find
nothing on the chain — indistinguishable from a wiring fault. Use the
Tigard-specific cable entries below.

**Driver binding is per-interface, and per-backend.** Zadig binds a driver to
one USB interface. Which driver you need depends on which USB backend the tool
was compiled against:

| Backend | Driver needed |
|---|---|
| libusb-1.0 (libftdi1) | WinUSB |
| libusb-0.1 (old libftdi) | libusb-win32 |
| FTDI D2XX | FTDI's own driver (VCP) |

The prebuilt Windows xc3sprog binaries use old libftdi and/or D2XX, which
conflicts with openFPGALoader's libusb-1.0. Building xc3sprog from source
against libftdi1 puts both tools on the same stack, so one WinUSB binding
serves both. That build is Part 3.

---

## Part 1 — Hardware setup (Tigard)

For the cheaper CJMCU-2232HL and FT232H boards, see **Adapter variants** below.
Parts 2 and 3 are the same for all three.

1. Set Tigard's protocol switch to **JTAG**.
2. Set the voltage switch:
   - **VTGT** — takes reference from the target's VREF pin. Preferred when the
     target supplies a reference.
   - **3.3V** — Tigard drives its own rail. Use when the target has no VREF pin.
3. Wire TCK, TDI, TDO, TMS and GND per the silkscreen labels on both boards.
   Don't infer pin order from ADBUS numbering.
4. Verify VTGT reads the expected voltage with a meter before trusting a scan.

Keep leads short. A missing or high-impedance ground gives the same empty-scan
symptom as a wrong cable definition.

---

## Part 2 — Install tools

### MSYS2

Install from https://www.msys2.org, then open the **UCRT64** shell (not MSYS,
not MINGW64) and install packages:

```bash
pacman -Syu
pacman -S --needed mingw-w64-ucrt-x86_64-toolchain \
                   mingw-w64-ucrt-x86_64-cmake \
                   mingw-w64-ucrt-x86_64-libftdi \
                   make git
```

### openFPGALoader

Either install via pacman if available in UCRT64, or download a Windows build
and put it on PATH.

### Zadig

Download from https://zadig.akeo.ie — no installation needed.

---

## Part 3 — Build xc3sprog

The upstream code is from 2011 and needs two workarounds with a modern
toolchain.

```bash
cd ~
git clone https://github.com/sifive/xc3sprog.git
cd xc3sprog
mkdir build && cd build

cmake -G "MSYS Makefiles" \
      -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
      -DUSE_FTD2XX=OFF \
      -DCMAKE_CXX_STANDARD=14 \
      ..

make -j4
```

**Why each flag:**

- `CMAKE_POLICY_VERSION_MINIMUM=3.5` — the source declares
  `cmake_minimum_required(VERSION 2.6)`; CMake 4.x hard-errors below 3.5.
- `USE_FTD2XX=OFF` — forces the libftdi1 path, which is what we want.
- `CMAKE_CXX_STANDARD=14` — the code has `typedef unsigned char byte`, which
  became ambiguous with `std::byte` in C++17. Errors read
  "reference to 'byte' is ambiguous".

Configure output must show `Found libftdi1, version 1.5` (or later).

### Verify the linkage

```bash
ldd xc3sprog.exe | grep -iE 'ftdi|usb'
```

Both must resolve to `/ucrt64/bin/`:

```
libftdi1.dll => /ucrt64/bin/libftdi1.dll
libusb-1.0.dll => /ucrt64/bin/libusb-1.0.dll
```

If `libusb-1.0.dll` resolves to `C:/WINDOWS/SYSTEM32/`, a stray copy is
shadowing MSYS2's. Delete or rename it — Windows searches System32 before
PATH, and a version mismatch causes confusing open failures.

---

## Part 4 — Driver binding (Tigard)

1. Plug in Tigard. Two COM ports appear.
2. Run Zadig **as administrator**.
3. **Options → List All Devices.**
4. Select **Tigard V1.1 (Interface 1)** — Interface 1, not 0.
5. Choose **WinUSB** and click Replace Driver.

Interface 1's COM port disappears. Interface 0 keeps its COM port for UART use.

Device Manager should show the interface under WinUSB devices rather than
Ports (COM & LPT).

---

## Part 5 — Verify it works

Run xc3sprog from the UCRT64 shell so it finds its DLLs.

```bash
openFPGALoader --detect --cable tigard
```

Expected:

```
index 0:
        idcode 0x9604093
        manufacturer xilinx
        family xc9500xl
        model  xc9572xl
        irlength 8
```

```bash
cd ~/xc3sprog/build
./xc3sprog.exe -c bbv2_2 -j -v
```

Expected: `JTAG chainpos: 0 Device IDCODE = 0x59604093  Desc: XC9572XL`

The IDCODE differs in the top nibble between tools — that's the die revision,
which openFPGALoader masks off. Same part.

---

## Command reference

### openFPGALoader

```bash
openFPGALoader --detect --cable tigard
openFPGALoader --cable tigard design.jed
openFPGALoader --cable tigard --freq 1000000 design.jed
```

### xc3sprog

The `-v` flag is required in all cases on this build — omitting it fails.

```bash
# Detect chain
./xc3sprog.exe -c bbv2_2 -j -v

# Verify device against a JEDEC file
./xc3sprog.exe -c bbv2_2 -v -p 0 "design.jed:v"

# Read device back to a file (r = won't overwrite, R = overwrite)
./xc3sprog.exe -c bbv2_2 -v -p 0 "readback.jed:r"

# Program (default action is write+verify)
./xc3sprog.exe -c bbv2_2 -v -p 0 "design.jed"

# Erase
./xc3sprog.exe -c bbv2_2 -v -p 0 -e

# Lower JTAG clock
./xc3sprog.exe -c bbv2_2 -v -J 1000000 -p 0 "design.jed:v"
```

Paths in the UCRT64 shell use `/o/path` or `o:/path`, not `O:\path`.

XC9500XL parts take **JEDEC** files, not bitstreams.

---

## Adapter variants

Parts 2 and 3 (MSYS2, openFPGALoader, building xc3sprog) are identical for all
three adapters. Only the pin wiring, the Zadig interface, and the cable name
change.

| | Tigard | CJMCU-2232HL | FT232H board |
|---|---|---|---|
| Chip | FT2232H | FT2232H | FT232H |
| USB ID | 0403:6010 | 0403:6010 | 0403:6014 |
| JTAG channel | B | A | (single) |
| Zadig interface | **Interface 1** | **Interface 0** | **Interface 0** |
| openFPGALoader cable | `tigard` | `ft2232` | `ft232` |
| xc3sprog cable | `bbv2_2` | `ftdi` | `ft232h` |
| Level shifting | Yes, with VTGT | No — 3.3V only | See note below |
| Spare UART | Yes | Yes | No |

### CJMCU-2232HL

Cheap generic FT2232HL breakout. No jumpers, no switches, no level shifting —
the FT2232H's I/O is fixed at 3.3V CMOS and the inputs are **not 5V tolerant**.
Fine for XC9500XL and ECP5 targets; do not connect it to 5V logic.

Silkscreen uses `ADn`/`BDn` (channel A / channel B data bus) and `ACn`/`BCn`
for the control pins. JTAG goes on the channel A pins:

| Silkscreen | FT2232H | JTAG |
|---|---|---|
| **AD0** | ADBUS0 | TCK |
| **AD1** | ADBUS1 | TDI |
| **AD2** | ADBUS2 | TDO |
| **AD3** | ADBUS3 | TMS |

Plus any **GND** pin to the target's ground.

`RST` sits immediately next to `AD0` on the header — count carefully. TCK
landed on RST gives an empty scan with no other symptom.

The `3V3` pins are outputs, not a VCCIO selector. Don't wire the `VCC` pins to
a target; the board is USB bus-powered and back-feeding it can damage things.

These boards ship with a blank EEPROM, so they enumerate as **"Dual RS232-HS
(Interface 0)"** and report no serial number. openFPGALoader prints
`Can't read iSerialNumber field from FTDI: considered as empty string` —
harmless.

Because it is also `0403:6010` and now unnamed, Zadig cannot distinguish it
from a Tigard when both are plugged in. Bind and test one board at a time.
xc3sprog's `-s <serial>` won't help — there is no serial to match.

**Where to buy:**
- https://www.aliexpress.us/item/3256809348878557.html
- https://www.amazon.com/MusRock-FT2232HL-Serial-Adapter-Development/dp/B0FXWND8RM

### FT232H board (cheapest option — UNTESTED)

Single-channel FT232H module, USB-C, with an I2C-mode slide switch. Cheaper
than the FT2232HL and adequate for JTAG, at the cost of losing the second
channel (no spare UART, no logic-analyser use while JTAG is connected).

**This variant has not been verified — the notes below are from the datasheet
and board silkscreen. Confirm before relying on them.**

Different USB PID (`0403:6014`), so the cable names differ:

```bash
openFPGALoader --detect --cable ft232
./xc3sprog.exe -c ft232h -j -v
```

Only one interface exists, so Zadig binds **Interface 0** — and unlike the
two-channel boards, there is no second COM port left over.

Silkscreen uses `Dn` for the ADBUS pins:

| Silkscreen | FT232H | JTAG | (SPI label on reverse) |
|---|---|---|---|
| **D0** | ADBUS0 | TCK | SCK/SCL |
| **D1** | ADBUS1 | TDI | MOSI |
| **D2** | ADBUS2 | TDO | MISO |
| **D3** | ADBUS3 | TMS | SPICS |

Plus **Gnd** to the target's ground.

Set the **I2C Mode switch to off** for JTAG. In I2C mode the board ties D1 and
D2 together, which will break JTAG.

The reverse silkscreen claims *"3V logic, 5V safe"*. Treat that as meaning the
inputs tolerate 5V, not that it can drive 5V logic — outputs are still 3.3V.
Verify against the actual FT232H datasheet before connecting it to anything 5V;
"5V safe" on a cheap board silkscreen is not a guarantee.

---

## Troubleshooting

**`empty` / no devices found, but clock reports correctly**
Wrong FTDI channel — the tool is driving pins that aren't wired to your header.
Check the cable name matches your adapter in the Adapter variants table.
Tigard is channel B (`tigard` / `bbv2_2`); the generic FT2232H boards are
channel A (`ft2232` / `ftdi`). This was the single biggest time sink.

**`unable to claim usb device. Make sure the default FTDI driver is not in use`**
Interface 1 is still on the FTDI driver. Bind WinUSB with Zadig.

**`unable to open ftdi device: -4 (usb_open() failed)`**
Wrong driver for the backend. openFPGALoader needs WinUSB on Interface 1.

**`Could not open FTDI device (using libftdi): device not found`**
The binary's libftdi can't see through the current driver. If this is a
prebuilt xc3sprog, it wants libusb-win32 or libusbK. Build from source instead.

**High retry counts in xc3sprog output**
Not a signal problem. The old D2XX backend polls an empty read FIFO — counts go
*up* as you lower the clock. The libftdi1 build reports `retries 0`. Only chase
signal integrity if verification actually fails.

**Verification failures through long leads**
Lower the clock with `-J` (xc3sprog) or `--freq` (openFPGALoader). The FT2232H
only produces 60 MHz ÷ (2 × (n+1)) rates, so requested and actual differ
(`-J 500000` gives 461.538 kHz — expected).

**Readback returns zeros or garbage**
XC9500XL parts have a read-protect fuse. If set, readback and verify are
unavailable regardless of tooling.

**Programming a card in a live machine**
CPLD outputs go high-Z during ISP mode, which can cause bus contention. Prefer
programming with the host machine powered but idle, or the card on the bench.

---

## Portability note

The built `xc3sprog.exe` depends on MSYS2 DLLs and only runs from the UCRT64
shell as-is. To run it elsewhere, copy every `/ucrt64/bin/` dependency listed by
`ldd xc3sprog.exe` alongside the executable.

---

## Also works for

The same adapter, binding, and openFPGALoader install drive Lattice ECP5
targets (e.g. Colorlight i5) — `--cable tigard` with the appropriate SRAM or
flash flags. No separate setup needed.

---

## What not to use

The Xilinx Platform Cable USB II (DLC10) and its clones are a dead end on
Windows 11. The cable's firmware lives in volatile FX2 RAM and must be
downloaded on every plug-in by `xusbdfwu.sys`, a Jungo WinDriver-based kernel
driver that AMD has not re-signed since the Windows 7 era. It fails with
**Code 52** (signature verification), and the only workarounds are disabling
driver signature enforcement or test-signing mode — both of which weaken the
machine. Loading the firmware from userspace instead (Linux `fxload` via WSL2,
or a Python/pyusb script on Windows) does not work either: the writes are
acknowledged but the FX2 never re-enumerates to its working `03fd:0008`
identity.

The cable still works fine under ISE 14.7 iMPACT on Windows 7 if you have such
a machine available.
