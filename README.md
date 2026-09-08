# JTAG Programming on Windows 11

Programming and verifying Xilinx XC9500XL CPLDs (and Lattice ECP5) with an
FTDI-based JTAG adapter, from a clean Windows install.

Covers three adapters: **Tigard** (recommended), **CJMCU-2232HL**, and a
single-channel **FT232H** board. The software setup is identical for all three;
only wiring, driver binding, and cable names differ.

---

## Choosing an adapter

All three are FTDI MPSSE devices and all three will program an XC9500XL. The
differences matter more on a mixed-vintage bench than on a single project.

### Tigard — worth the extra cost

An open-hardware FT2232H adapter designed for exactly this kind of work.
Documentation, schematics, and pinouts: **https://github.com/tigard-tools/tigard**

- **Level shifting with selectable target voltage.** A switch picks 1.8V, 3.3V,
  5V, or VTGT (referenced from the target's own VREF pin). The generic boards
  are fixed 3.3V with unprotected inputs — a real hazard around 5V logic,
  which includes a lot of retro hardware.
- **Labelled protocol headers.** Separate, silkscreened headers for JTAG, SWD,
  UART, SPI and I2C, with a mode switch routing channel A. No counting pins
  from a `Dn` label and hoping.
- **Programmed EEPROM.** Identifies itself as "Tigard V1.1" in Device Manager
  and Zadig, instead of the generic "Dual RS232-HS" that every unbranded
  FT2232H board reports. Matters as soon as you own more than one adapter.
- **Series protection resistors** on the I/O lines.
- **Second channel stays free** for a UART to the same target while JTAG is
  connected.

**Where to buy:**
- https://1bitsquared.com/products/tigard
- https://1bitsquared.de/en/products/tigard
- https://www.crowdsupply.com/securinghw/tigard#products
- https://hackerwarehouse.com/product/tigard/

### When a generic board is fine

If the targets are all 3.3V, the wiring is fixed, and only one adapter is ever
plugged in, a CJMCU-2232HL does the same job for a fraction of the price. The
FT232H board is cheaper still, at the cost of the second channel.

Both are covered under **Adapter variants** below.

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

**1. Install MSYS2.** From an ordinary PowerShell or Command Prompt:

```powershell
winget install --id=MSYS2.MSYS2
```

Or download and run the installer from **https://www.msys2.org** and accept the
defaults.

**2. Open the right shell.** The install creates several Start menu entries.
Open the one named **"MSYS2 UCRT64"** — the yellow icon. Not "MSYS2 MSYS", not
"MSYS2 MINGW64". Everything in this guide assumes the UCRT64 shell.

**3. Update the package database.** Answer `Y` to any prompts:

```bash
pacman -Syu
```

If it tells you to close the terminal, close it, reopen **MSYS2 UCRT64**, and
run `pacman -Syu` again.

**4. Install everything needed:**

```bash
pacman -S --needed mingw-w64-ucrt-x86_64-toolchain \
                   mingw-w64-ucrt-x86_64-cmake \
                   mingw-w64-ucrt-x86_64-libftdi \
                   mingw-w64-ucrt-x86_64-openFPGALoader \
                   make git
```

Press Enter to accept the default (all packages) when asked which members of
the toolchain group to install.

**5. Check openFPGALoader is installed:**

```bash
openFPGALoader --Version
```

### Zadig

Download from **https://zadig.akeo.ie**. It's a single .exe — no installation.
Save it somewhere you can find again; you'll need it whenever you connect a new
adapter.

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

Windows installs FTDI's serial driver automatically, which gives you two COM
ports. JTAG tools can't use that driver, so one interface has to be switched to
WinUSB. This is what Zadig does.

1. Plug in Tigard. Two COM ports appear in Device Manager under
   **Ports (COM & LPT)**.
2. Run Zadig **as administrator** (right-click → Run as administrator).
3. **Options → List All Devices.** Nothing useful appears until you do this.
4. In the dropdown, select **Tigard V1.1 (Interface 1)**. Check the interface
   number carefully — Interface 0 is the wrong one and will appear to work
   right up until scans come back empty.
5. In the driver box to the right of the green arrow, use the small up/down
   arrows to select **WinUSB**.
6. Click **Replace Driver**. It takes a few seconds.

Interface 1's COM port disappears. Interface 0 keeps its COM port and can still
be used as a normal serial port.

To confirm: in Device Manager, the interface should now appear under a
**Universal Serial Bus devices** or **WinUSB devices** heading rather than
Ports (COM & LPT).

This is a one-time change per adapter, per PC. Windows remembers it across
reboots and replugs.

### Undoing it

If you ever want the COM port back: Device Manager → right-click the device →
**Uninstall device**, tick **"Attempt to remove the driver"**, then unplug and
replug.

---

## Part 5 — Verify it works

Run xc3sprog from the **MSYS2 UCRT64** shell so it finds its DLLs.
openFPGALoader works from any shell once installed, but it's simplest to use
the same window for both.

Power the target board, connect the adapter, then:

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

### Getting to your files from the shell

xc3sprog is a native Windows program, so give it **Windows-style paths** —
exactly what you get from Explorer's "Copy as path":
`O:\Projects\MyBoard\logic\design.jed`. MSYS2's `/o/...` style does **not**
work here — the colon in the filespec (`design.jed:v`) stops MSYS2 translating
the path, and you get `Can't open datafile ...: No such file or directory`.

Quote any path containing spaces (Explorer's "Copy as path" already adds the
quotes).

Easiest approach: `cd` to the folder holding your `.jed` file, then use the bare
filename. For `cd` itself, MSYS2 paths work fine — it's a shell builtin:

```bash
cd "/o/Projects/MyBoard/logic"
~/xc3sprog/build/xc3sprog.exe -c bbv2_2 -v -p 0 "design.jed:v"
```

Or stay put and give the full Windows path:

```bash
~/xc3sprog/build/xc3sprog.exe -c bbv2_2 -v -p 0 "O:\Projects\MyBoard\logic\design.jed:v"
```

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

Use Windows-style paths (`O:\Projects\design.jed`), not MSYS2 `/o/...` paths —
see "Getting to your files from the shell" above.

XC9500XL parts take **JEDEC** files, not bitstreams.

---

## Adapter variants

Parts 2 and 3 (MSYS2, openFPGALoader, building xc3sprog) are identical for all
three adapters. Only the pin wiring, the Zadig interface, and the cable name
change — so for a generic board, follow Part 4 exactly but pick the interface
named in the table below instead of Interface 1.

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
