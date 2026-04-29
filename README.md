# p2qr — QR Code Generator for the Parallax Propeller 2

**p2qr** is a pure-Spin2 QR code generator for the Parallax Propeller 2 microcontroller. It encodes any text string into a standards-compliant QR code entirely on-chip — no external libraries, no co-processor, no network connection required. The resulting code can be displayed in the Propeller Tool debug window, printed to a terminal over serial, or rendered on an SSD1331 OLED display.

Original C implementation by B.J. Guillot (bguillot@acm.org) / joyteq LLC (MIT License).  
Spin2 port and enhancements by Terry E. Trapp, KE4PJW (ttrapp@ieee.org).

---

## Features

- **QR Versions 1–40** — automatically selects the smallest version that fits your message
- **Error Correction Level Q** — approximately 25% of the code can be damaged and still scan correctly
- **Full 8-mask evaluation** — tries all 8 QR masks and picks the one with the lowest penalty score, just as the QR specification requires
- **Five output modes** — debug window, ASCII art, ANSI color, Unicode half-blocks, custom characters, or raw image buffer
- **SSD1331 OLED support** — companion object renders the QR code on a 96×64 color display
- **Configurable speed/quality trade-off** — `MASK_LEVELS` constant limits how many masks are scored
- **Self-contained** — all Reed-Solomon math and lookup tables are embedded; no floating-point

---

## Hardware Requirements

- Parallax Propeller 2 (P2X8C4M64P) evaluation board or custom P2 board
- Propeller Tool IDE (for compiling and loading)
- For serial output: any 3.3 V serial terminal (Parallax Serial Terminal, PuTTY, etc.)
- For OLED output: SSD1331-based 96×64 RGB OLED display

---

## File Reference

| File | Description |
|---|---|
| `p2qr.spin2` | **Main object.** Include this in your project. |
| `p2qr_Example.spin2` | Basic usage example — generates a QR code and outputs it over serial or to the debug window. |
| `p2qr_MaskTest.spin2` | Integration test — runs 40 messages and verifies all 8 masks are selected at least once. |
| `qrssd.spin2` | Adapter object that renders a raw p2qr image on an SSD1331 OLED display. |
| `SSD1331_Example.spin2` | Full example combining `p2qr` and `qrssd` to display a QR code on an SSD1331. |
| `SmartSerial.spin2` | Lightweight serial driver (dependency for serial output examples). |
| `oled_AsmDrv.spin2` | Assembly-language SSD1331 SPI driver (dependency for OLED examples). |
| `5x7Font.dat` | 5×7 pixel font data used by the OLED driver. |
| `p1_font.dat` | Additional font data used by the OLED driver. |

---

## Quick Start

### 1. Add p2qr to your project

```spin2
OBJ
  qr     : "p2qr"
  serial : "SmartSerial"
```

### 2. Start your serial port and point SEND at it

```spin2
PUB main()
  serial.start(230400)
  SEND := @serial.tx
```

> **Note:** Speeds above 230,400 baud can cause issues with some terminals.

### 3. Generate and display a QR code

```spin2
  ' Display in the Propeller Tool debug window (dot size = 8 pixels)
  qr.parseMessage(String("https://www.example.com"), qr.TYPE_DEBUG, 8)

  ' Print ASCII art to a serial terminal
  qr.parseMessage(String("https://www.example.com"), qr.TYPE_ASCII, 0)
```

That's it. The object handles version selection, Reed-Solomon error correction, mask evaluation, and output automatically.

---

## Output Types

Pass one of these constants as the `out_type` argument to `parseMessage`:

| Constant | Value | Description |
|---|---|---|
| `TYPE_DEBUG` | 3 | Renders a bitmap in the Propeller Tool debug window. The third argument is the dot size in pixels (e.g. `8`). Requires debug mode enabled. |
| `TYPE_ASCII` | 5 | Prints ASCII art using `@@` for light and spaces for dark modules. Disable auto-newline in your terminal. |
| `TYPE_ANSI` | 1 | Prints ANSI escape-colored blocks. Works well with PuTTY; does **not** work in Parallax Serial Terminal. |
| `TYPE_UNICODE` | 2 | Prints Unicode half-block characters (`▀`, `▄`, `█`, ` `) — two QR rows per terminal line. |
| `TYPE_CUSTCHR` | 6 | Uses strings you supply via `white()`, `black()`, and `newln()` for full control over the characters used. |
| `TYPE_RAW` | 4 | Returns a pointer to the raw image byte array. Each byte is `0` (dark module) or `255` (light module). Use this to drive custom displays. |

---

## API Reference

### `parseMessage(freetext, out_type, printptr)`

Encodes `freetext` into a QR code and outputs it.

| Parameter | Description |
|---|---|
| `freetext` | Pointer to a zero-terminated string to encode (up to 1,663 characters). |
| `out_type` | One of the `TYPE_*` constants above. |
| `printptr` | For `TYPE_DEBUG`: dot size in pixels. For all other types: pass `0` (output goes through `SEND`). For `TYPE_RAW`: return value is the image pointer; this argument is ignored. |

### `getqrversion() : version`

Returns the QR version number that was selected for the last call to `parseMessage`, as an integer 0–39. Add 1 for the human-readable version (e.g., `0` = Version 1, `39` = Version 40).

### `getimage() : ptr`

Returns a pointer to the internal 31,329-byte image buffer. Each byte represents one module: `0` = dark, `255` = light. Valid after any `parseMessage` call.

### `getmask() : mask`

Returns the mask number (0–7) that was selected as the best mask for the last QR code generated.

### `white(string_ptr)` / `black(string_ptr)` / `newln(string_ptr)`

Set the strings used for light modules, dark modules, and line endings when using `TYPE_CUSTCHR`. Call these before `parseMessage`.

```spin2
qr.white(String("  "))   ' two spaces = light module
qr.black(String("##"))   ' hash marks = dark module
qr.newln(String(13, 10)) ' CR+LF
qr.parseMessage(String("Hello"), qr.TYPE_CUSTCHR, 0)
```

---

## Configuration

### `MASK_LEVELS` (in `p2qr.spin2` CON section)

Controls how many of the 8 QR masks are evaluated for penalty scoring. Masks are tried in order from least to most computationally expensive.

```spin2
MASK_LEVELS = 8   ' default: full spec-compliant, best QR quality
MASK_LEVELS = 4   ' faster: evaluates 4 cheapest masks only
MASK_LEVELS = 1   ' fastest: always uses mask 1, no scoring
```

| Value | Masks tried | Trade-off |
|---|---|---|
| 1 | Mask 1 only | Fastest; skips penalty scoring entirely |
| 2–3 | Masks 1, 0, … | Moderate speed; good coverage for most inputs |
| 4 | Masks 1, 0, 2, 3 | Evaluates all non-multiply masks |
| 8 | All 8 masks | Slowest; fully QR-spec compliant (default) |

> The mask evaluation order (cheapest first) is: **1, 0, 2, 3, 4, 5, 6, 7**.

---

## Capacity Reference

| QR Version | Max message length (Byte / Q) |
|---|---|
| 1 | 11 characters |
| 2 | 20 characters |
| 4 | 46 characters |
| 6 | 74 characters |
| 10 | 151 characters |
| 13 | 241 characters |
| 20 | 482 characters |
| 40 | 1,663 characters |

The object selects the smallest version that fits your message automatically.

---

## SSD1331 OLED Display Example

Wire your SSD1331 display and set the pin constants in `SSD1331_Example.spin2`:

```spin2
CON
  CS   = 50
  RST  = 52
  DC   = 51
  CLK  = 49
  DATA = 48
```

Then load `SSD1331_Example.spin2`. It encodes a URL, fetches the raw image buffer, and renders it on the display. QR versions up to V4 (33×33) are drawn at 2 pixels per module; larger versions use 1 pixel per module.

---

## Encoding Limitations

- **Encoding:** 8-bit Byte mode only (supports full ASCII and UTF-8 content).
- **Error correction:** Level Q only (~25% recovery capability).
- **Message length:** Maximum 1,663 bytes (QR Version 40-Q).

---

## Credits and License

Spin2 port, enhancements, and mask selection implementation:  
**Terry E. Trapp, KE4PJW** — ttrapp@ieee.org — Ver 1.3, 2026

Original C QR code algorithm:  
**B.J. Guillot** — bguillot@acm.org — Copyright © 2016 joyteq LLC

Both the original work and this Spin2 port are released under the **MIT License**. See the header of `p2qr.spin2` for the full license text.

QR code structure reference: [Thonky QR Code Tutorial](https://www.thonky.com/qr-code-tutorial/)
