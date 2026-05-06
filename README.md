# p2qr — QR Code Generator for the Parallax Propeller 2

**p2qr** is a pure-Spin2 QR code generator for the Parallax Propeller 2 microcontroller. It encodes any text string into a standards-compliant QR code entirely on-chip — no external libraries, no co-processor, no network connection required. The resulting code can be displayed in the Propeller Tool debug window, printed to a terminal over serial, or rendered on an SSD1331 OLED display.

Original C implementation by B.J. Guillot (bguillot@acm.org) / joyteq LLC (MIT License).  
Spin2 port and enhancements by Terry E. Trapp, KE4PJW (ttrapp@ieee.org).

\---

## Features

* **QR Versions 1–40** — automatically selects the smallest version that fits your message
* **All four Error Correction levels** — choose L (\~7%), M (\~15%), Q (\~25%), or H (\~30%) recovery via the `EC\_LEVEL` constant; one-line change switches the entire build
* **Full 8-mask evaluation** — tries all 8 QR masks and picks the one with the lowest penalty score, just as the QR specification requires
* **Configurable mask speed/quality trade-off** — `MASK\_LEVELS` constant limits how many masks are scored (1, 2, 3, 4, or 8)
* **Six output modes** — debug window, ASCII art, ANSI color, Unicode half-blocks, custom characters, or raw image buffer
* **SSD1331 OLED support** — companion object renders the QR code on a 96×64 color display
* **Format-info introspection** — `getformatinfo()` returns the expected 15-bit format word for any mask, useful for self-test and verification
* **Self-contained** — all Reed-Solomon math, generator polynomials, and lookup tables for all four EC levels are embedded; no floating-point

\---

## Hardware Requirements

* Parallax Propeller 2 (P2X8C4M64P) evaluation board or custom P2 board
* Propeller Tool IDE (for compiling and loading)
* For serial output: any 3.3 V serial terminal (Parallax Serial Terminal, PuTTY, etc.)
* For OLED output: SSD1331-based 96×64 RGB OLED display

\---

## File Reference

|File|Description|
|-|-|
|`p2qr.spin2`|**Main object.** Include this in your project.|
|`p2qr\_Example.spin2`|Basic usage example — generates a QR code and outputs it over serial or to the debug window.|
|`p2qr\_MaskTest.spin2`|Integration test — runs 40 messages and verifies all 8 masks are selected at least once.|
|`p2qr\_ECTest.spin2`|Integration test for `EC\_LEVEL` — verifies format-info bits, version-capacity selection, and mask coverage for the currently compiled EC level.|
|`qrssd.spin2`|Adapter object that renders a raw p2qr image on an SSD1331 OLED display.|
|`SSD1331\_Example.spin2`|Full example combining `p2qr` and `qrssd` to display a QR code on an SSD1331.|
|`SmartSerial.spin2`|Lightweight serial driver (dependency for serial output examples).|
|`oled\_AsmDrv.spin2`|Assembly-language SSD1331 SPI driver (dependency for OLED examples).|
|`5x7Font.dat`|5×7 pixel font data used by the OLED driver.|
|`p1\_font.dat`|Additional font data used by the OLED driver.|

\---

## Quick Start

### 1\. Add p2qr to your project

```spin2
OBJ
  qr     : "p2qr"
  serial : "SmartSerial"
```

### 2\. Start your serial port and point SEND at it

```spin2
PUB main()
  serial.start(230400)
  SEND := @serial.tx
```

> \*\*Note:\*\* Speeds above 230,400 baud can cause issues with some terminals.

### 3\. Generate and display a QR code

```spin2
  ' Display in the Propeller Tool debug window (dot size = 8 pixels)
  qr.parseMessage(String("https://www.example.com"), qr.TYPE\_DEBUG, 8)

  ' Print ASCII art to a serial terminal
  qr.parseMessage(String("https://www.example.com"), qr.TYPE\_ASCII, 0)
```

That's it. The object handles version selection, Reed-Solomon error correction, mask evaluation, and output automatically.

\---

## Output Types

Pass one of these constants as the `out\_type` argument to `parseMessage`:

|Constant|Value|Description|
|-|-|-|
|`TYPE\_DEBUG`|3|Renders a bitmap in the Propeller Tool debug window. The third argument is the dot size in pixels (e.g. `8`). Requires debug mode enabled.|
|`TYPE\_ASCII`|5|Prints ASCII art using `@@` for light and spaces for dark modules. Disable auto-newline in your terminal.|
|`TYPE\_ANSI`|1|Prints ANSI escape-colored blocks. Works well with PuTTY; does **not** work in Parallax Serial Terminal.|
|`TYPE\_UNICODE`|2|Prints Unicode half-block characters (`▀`, `▄`, `█`, ` `) — two QR rows per terminal line.|
|`TYPE\_CUSTCHR`|6|Uses strings you supply via `white()`, `black()`, and `newln()` for full control over the characters used.|
|`TYPE\_RAW`|4|Returns a pointer to the raw image byte array. Each byte is `0` (dark module) or `255` (light module). Use this to drive custom displays.|

\---

## API Reference

### `parseMessage(freetext, out\_type, printptr)`

Encodes `freetext` into a QR code and outputs it.

|Parameter|Description|
|-|-|
|`freetext`|Pointer to a zero-terminated string to encode (up to 1,663 characters).|
|`out\_type`|One of the `TYPE\_\*` constants above.|
|`printptr`|For `TYPE\_DEBUG`: dot size in pixels. For all other types: pass `0` (output goes through `SEND`). For `TYPE\_RAW`: return value is the image pointer; this argument is ignored.|

### `getqrversion() : version`

Returns the QR version number that was selected for the last call to `parseMessage`, as an integer 0–39. Add 1 for the human-readable version (e.g., `0` = Version 1, `39` = Version 40).

### `getimage() : ptr`

Returns a pointer to the internal 31,329-byte image buffer. Each byte represents one module: `0` = dark, `255` = light. Valid after any `parseMessage` call.

### `getmask() : mask`

Returns the mask number (0–7) that was selected as the best mask for the last QR code generated.

### `getformatinfo(mask\_num) : fi`

Returns the expected 15-bit format-information word for the supplied mask number (0–7) at the currently compiled `EC\_LEVEL`. This is the BCH-encoded value that `parseMessage` writes into the QR image's format strips. Useful for verification, self-test code, or external tools that need to compare against the spec value without re-deriving the BCH math.

### `img\_get(img\_ptr, sz, row, col) : value`

Reads one module from a raw image buffer. `img\_ptr` is the pointer returned by `getimage()`, `sz` is the QR's width in modules (`(getqrversion() + 1) \* 4 + 17`), and `row`/`col` index into the QR grid. Returns `0` for a dark module, `255` for a light module.

### `white(string\_ptr)` / `black(string\_ptr)` / `newln(string\_ptr)`

Set the strings used for light modules, dark modules, and line endings when using `TYPE\_CUSTCHR`. Call these before `parseMessage`.

```spin2
qr.white(String("  "))   ' two spaces = light module
qr.black(String("##"))   ' hash marks = dark module
qr.newln(String(13, 10)) ' CR+LF
qr.parseMessage(String("Hello"), qr.TYPE\_CUSTCHR, 0)
```

\---

## Configuration

### `MASK\_LEVELS` (in `p2qr.spin2` CON section)

Controls how many of the 8 QR masks are evaluated for penalty scoring. Masks are tried in order from least to most computationally expensive.

```spin2
MASK\_LEVELS = 8   ' default: full spec-compliant, best QR quality
MASK\_LEVELS = 4   ' faster: evaluates 4 cheapest masks only
MASK\_LEVELS = 1   ' fastest: always uses mask 1, no scoring
```

|Value|Masks tried|Trade-off|
|-|-|-|
|1|Mask 1 only|Fastest; skips penalty scoring entirely|
|2–3|Masks 1, 0, …|Moderate speed; good coverage for most inputs|
|4|Masks 1, 0, 2, 3|Evaluates all non-multiply masks|
|8|All 8 masks|Slowest; fully QR-spec compliant (default)|

> The mask evaluation order (cheapest first) is: \*\*1, 0, 2, 3, 4, 5, 6, 7\*\*.

### `EC\_LEVEL` (in `p2qr.spin2` CON section)

Selects the QR error correction level for all generated codes. Change this one constant to switch levels — no other source changes are required.

```spin2
EC\_LEVEL = EC\_L   ' Level L — \~7% recovery; largest data capacity
EC\_LEVEL = EC\_M   ' Level M — \~15% recovery
EC\_LEVEL = EC\_Q   ' Level Q — \~25% recovery (default)
EC\_LEVEL = EC\_H   ' Level H — \~30% recovery; smallest data capacity
```

|Constant|Recovery|Max message (V40, Byte mode)|Notes|
|-|-|-|-|
|`EC\_L`|\~7%|2,953 bytes|Largest capacity; less resistant to damage|
|`EC\_M`|\~15%|2,331 bytes|Good balance for most uses|
|`EC\_Q`|\~25%|1,663 bytes|Default; suitable for noisy environments|
|`EC\_H`|\~30%|1,273 bytes|Best damage resistance; largest code size|

Higher EC levels devote more of the QR code to error-correction data, so a given message requires a higher (larger) QR version. The object automatically selects the smallest version that fits your message at the chosen level.

\---

## Capacity Reference

Maximum byte-mode message length per version, for each EC level:

|QR Version|EC\_L|EC\_M|EC\_Q|EC\_H|
|-|-|-|-|-|
|1|17|14|11|7|
|2|32|26|20|14|
|4|78|62|46|34|
|6|134|106|74|58|
|10|271|213|151|119|
|13|425|334|241|198|
|20|858|666|482|382|
|40|2,953|2,331|1,663|1,273|

The object selects the smallest version that fits your message automatically, based on the currently compiled `EC\_LEVEL`.

\---

## SSD1331 OLED Display Example

Wire your SSD1331 display and set the pin constants in `SSD1331\_Example.spin2`:

```spin2
CON
  CS   = 50
  RST  = 52
  DC   = 51
  CLK  = 49
  DATA = 48
```

Then load `SSD1331\_Example.spin2`. It encodes a URL, fetches the raw image buffer, and renders it on the display. QR versions up to V4 (33×33) are drawn at 2 pixels per module; larger versions use 1 pixel per module.

\---

## Encoding Limitations

* **Encoding:** 8-bit Byte mode only (supports full ASCII and UTF-8 content).
* **Error correction:** All four levels (L, M, Q, H) supported, but the level is fixed at compile time via the `EC\_LEVEL` constant — it cannot be changed at run time.
* **Message length:** Depends on the chosen `EC\_LEVEL`. Maximum at QR Version 40 is 2,953 bytes (EC\_L), 2,331 (EC\_M), 1,663 (EC\_Q), or 1,273 (EC\_H).

\---

## Credits and License

Spin2 port, enhancements, and mask selection implementation:  
**Terry E. Trapp, KE4PJW** — ttrapp@ieee.org — Ver 1.4, 2026

Original C QR code algorithm:  
**B.J. Guillot** — bguillot@acm.org — Copyright © 2016 joyteq LLC

Both the original work and this Spin2 port are released under the **MIT License**. See the header of `p2qr.spin2` for the full license text.

QR code structure reference: [Thonky QR Code Tutorial](https://www.thonky.com/qr-code-tutorial/)

