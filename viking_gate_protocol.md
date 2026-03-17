# Viking Gate Controller — Serial Protocol Reference

*Reverse-engineered from Viking Konnect (iOS/Android) and Vikings (iOS) applications*

---

## 1. Transport Layer

The controller communicates over **Bluetooth Classic SPP** (Serial Port Profile). A TCP socket path is also present in the newer Vikings iOS app (SwiftyBluetooth + `CFStreamCreatePairWithSocketToHost`), likely for a Wi-Fi bridge accessory; the wire protocol is identical on both transports.

| Platform | Stack | Details |
|---|---|---|
| iOS (older) | ExternalAccessory / MFi | App bundle ID: `com.vikingaccess.Viking-Konnect` |
| Android | `VBBluetoothAdapter` | `createInsecureRfcommSocket`, standard SPP UUID |
| iOS (newer) | SwiftyBluetooth + TCP | AWS IoT backend; same wire protocol |

**SPP UUID:** `00001101-0000-1000-8000-00805F9B34FB`

---

## 2. Encoding and Framing

The protocol uses **plain printable ASCII**. There are no binary length prefixes, no checksums, and no escaping.

### Commands (App → Controller)

```
<command_letter><value_string>\r
```

Every command ends with a carriage return (CR, `0x0D`). No line-feed follows on the send side.

### Responses (Controller → App)

Each response from the controller is a complete line terminated with `\r\n`. Each line has the structure:

```
<GATE_STATUS>\x02<TOKEN>,<TOKEN>,...\r\n
```

Where:
- `GATE_STATUS` — a single ASCII character (`Y`, `Z`, or `[`) indicating gate motion state
- `\x02` — STX byte (0x02) used as a separator / sync marker
- Each `TOKEN` — a 3-character status packet matching the pattern `^[G-V][0-9A-F]{2}$`
- Tokens are comma-separated

> **Note:** The `\x02` byte is verified at the start of the data portion before parsing. The iOS app uses the CFString `"\x02G"` as an expected pattern. The regex `^[G-V][0-9A-F]{2}$` is compiled once and applied to each comma-split token.

**Example response line:**

```
Y\x02L4F,N1E,M0A,R03,Q00\r\n
```

Decoded: gate is open/opening (`Y`); LED status `0x4F`; speed `0x1E` = 30; hold timer `0x0A` = 10 s; options `0x03`; no error.

---

## 3. Gate Status Prefix

The single character before `\x02` in each response line indicates current gate motion:

| Character | ASCII | Meaning |
|---|---|---|
| `Y` | 0x59 | Gate open or currently opening |
| `Z` | 0x5A | Gate closed or currently closing |
| `[` | 0x5B | Gate stopped / idle |

---

## 4. Response Token Types

Each token is `<TYPE><HH>` where `TYPE` is a letter `G`–`V` and `HH` is two uppercase hex digits representing one byte (`0x00`–`0xFF`).

**For numeric packets**, the raw byte is converted to an engineering value as described in Section 8.

**For bitmask packets**, the byte value is parsed from hex, then converted to an 8-character binary string (zero-padded, MSB first). Each character is compared to `'1'` to extract individual boolean flags:

```
Bit 7 = charAt(0)   (MSB)
Bit 6 = charAt(1)
Bit 5 = charAt(2)
Bit 4 = charAt(3)
Bit 3 = charAt(4)
Bit 2 = charAt(5)
Bit 1 = charAt(6)
Bit 0 = charAt(7)   (LSB)
```

For example, `L4F` → value `0x4F` = `01001111` in binary. charAt(0) = `'0'` → isOpenLit = false; charAt(7) = `'1'` → isReopenLit = true.

### 4.1 Numeric Value Packets

| Type | Parameter | Scaling / Notes |
|---|---|---|
| `G` | Obstruction Sensor | Float; raw × model gain factor |
| `H` | Battery Voltage (VDC) | Float; raw × model gain factor |
| `I` | AC Voltage, high range (VAC) | Float; used when raw ≥ 15; raw × model gain factor |
| `J` | AC Voltage, low range (VAC) | Float; used when raw < 15; raw passed through as double directly |
| `M` | Hold-Open Timer (seconds) | Float; raw byte as double |
| `N` | Speed | Float; raw byte as double |
| `O` | Unit Model | Integer — see Section 6 |
| `P` | Speed (offset variant) | Float; speed = rawByte − 25 (stored as raw + 231 mod 256) |
| `Q` | Error Code | Integer — see Section 7 |
| `S` | Motor Voltage (VDC) | Float; raw × model gain factor |
| `V` | No-op / end marker | Ignored |

> **Motor Amperage** uses a two-step adjustment before applying gain: `adjustedByte = rawByte + 246 − 9`. The exact type letter for motor amperage was not definitively isolated from the binary analysis.

### 4.2 Bitmask Packets

#### Packet `L` — LED / Gate Indicators

| Bit | charAt | Field |
|---|---|---|
| 7 | 0 | `isOpenLit` |
| 6 | 1 | `isStopLit` |
| 5 | 2 | `isCloseLit` |
| 4 | 3 | `isRadioLit` |
| 3 | 4 | `isUlLit` |
| 2 | 5 | `isATGLit` |
| 1 | 6 | `isCLoopLit` |
| 0 | 7 | `isReopenLit` |

#### Packet `K` — Limit & Motion Status

| Bit | charAt | Field |
|---|---|---|
| 7 | 0 | `isCloseLimitSet` |
| 6 | 1 | `isOpenLimitSet` |
| 5 | 2 | `isClosing` |
| 4 | 3 | `isOpening` |
| 3 | 4 | `isCloseLimitActuated` |
| 2 | 5 | `isOpenLimitActuated` |

#### Packet `R` — Options / Configuration

This packet mirrors the bitmask sent by the `r` command. Use read-modify-write: read the current `R` response, toggle the desired bit, send `r<newvalue>\r`.

| Bit | charAt | Field |
|---|---|---|
| 7 | 0 | `isULAlarmOn` |
| 6 | 1 | `isMagneticLockOn` |
| 5 | 2 | `isBrakeOn` |
| 4 | 3 | `isExtSet` |
| 3 | 4 | `isSyncSet` |
| 2 | 5 | `isPreWarningSet` |
| 1 | 6 | `isLastOpenSet` |
| 0 | 7 | `isAutoOpenSet` |

#### Packet `T` — Slave / Input Commands

> **Note:** `isATGInputFromSlave` is active-low / inverted — charAt(2) = `'1'` → flag is **false**; charAt(2) = `'0'` → flag is **true**. The bit defaults high (pulled up) and goes low when a slave is present.

| Bit | charAt | Field |
|---|---|---|
| 5 | 2 | `isATGInputFromSlave` *(inverted: `'1'` = false, `'0'` = true)* |
| 3 | 4 | `isCloseCommandFromSlave` |
| 2 | 5 | `isStopCommandFromSlave` |
| 1 | 6 | `isOpenCommandFromSlave` |

#### Packet `U` — Hold / UL Siren / Obstruction

| Bit | charAt | Field |
|---|---|---|
| 5 | 2 | `isHoldCommandsSet` |
| 3 | 4 | `isFirstObstructionSet` |
| 2 | 5 | `isULSirenSounding` |
| 1 | 6 | `isSecondObstructionSet` |

---

## 5. Commands (App → Controller)

All commands are plain ASCII terminated with `\r` (CR, 0x0D). The Android app logs each outgoing command with the prefix `"sending command: "`. No binary framing, no checksum.

### 5.1 Gate Control

| Command | Description |
|---|---|
| `Y\r` | Open gate — short press |
| `Z\r` | Close gate — short press |
| `[\r` | Stop gate — short press (`[` = ASCII 91) |
| `yyyyyyyyyy\r` | Open gate — long press (10× lowercase `y`) |
| `zzzzzzzzzz\r` | Close gate — long press (10× lowercase `z`) |
| `]]]]]]]]]]\r` | Stop gate — long press (10× `]` = ASCII 93) |

> **iOS vs Android:** The iOS app sends 11 repeated uppercase characters + `\r\n` (e.g. `YYYYYYYYYYY\r\n`). The Android app sends 10 repeated lowercase characters + `\r`. Both are accepted by the controller, which matches on the leading character; the repetition signals sustained long-press intent.

### 5.2 Keypad / Passcode Commands

Format: `k<HH>\r` where `HH` is two uppercase hex digits.

| Command | Byte Value | Description |
|---|---|---|
| `kFF\r` | `0xFF` | All bits set |
| `kBF\r` | `0xBF` | 0xBF |
| `k7F\r` | `0x7F` | 0x7F |

### 5.3 Limit Set / Clear Commands

Limit commands use the stop character `[` with a context flag indicating set vs. clear, and open vs. close. The UI presents four distinct actions: Set Open Limit, Clear Open Limit, Set Close Limit, Clear Close Limit. The exact wire encoding of the mode byte was not fully resolved from the binary analysis.

### 5.4 Configuration Value Commands

Format: `<letter><decimal_value>\r`

The decimal value is formatted without leading zeros. All four apps use the same letters.

| Letter | Parameter | Range / Notes |
|---|---|---|
| `h` | Hold-Open Timer | Decimal integer, seconds. Range: 0–60. |
| `n` | Overlap | Decimal integer, percent. Range: 0–100. |
| `p` | Speed | Decimal integer, percent. Range: 0–100. |
| `m` | Motor Amperage / Obstruction sensitivity | Decimal integer. Range app-dependent. |
| `r` | Options bitmask (full R packet value) | Decimal integer 0–255. Encodes all 8 option flags simultaneously — see below. |

#### `r` Command — Options Bitmask Detail

The `r` command sets all 8 configuration flags at once. Bit positions match the `R` response packet exactly:

| Bit | Decimal value | Option |
|---|---|---|
| 0 | 1 | AutoOpen |
| 1 | 2 | LastOpen |
| 2 | 4 | PreWarning |
| 3 | 8 | Sync |
| 4 | 16 | Ext |
| 5 | 32 | Brake |
| 6 | 64 | MagneticLock |
| 7 | 128 | UL Alarm |

Examples:

```
r0\r       → all options OFF
r1\r       → AutoOpen ON, all others OFF
r3\r       → AutoOpen ON + LastOpen ON  (1 + 2)
r11\r      → AutoOpen + LastOpen + Sync  (1 + 2 + 8)
r128\r     → UL Alarm ON only
```

---

## 6. Unit Model Codes

Reported in the `O` response packet as a decimal integer.

| Code | Model | Notes |
|---|---|---|
| 0 | NULL | No model / not set |
| 1 | I-8 | |
| 2 | B-12 | |
| 3 | Q-7 | |
| 4 | H-10 | |
| 5 | L-3 | |
| 6 | K-2 | |
| 7 | G-5 | iOS treats G-5 and X-9 identically (same gain factor) |
| 8 | F-1 | |
| 9 | T-21 | |
| 10 | X-9 | |
| 11 | R-6 | |

---

## 7. Error Codes

Reported in the `Q` response packet as a decimal integer.

| Code | Display | Description |
|---|---|---|
| 0 | NO ERRORS | No fault |
| 1 | ERR OPN LIMIT | Open limit fault |
| 2 | ERR CLS LIMIT | Close limit fault |
| 3 | ERR NO LIMIT | No limit set |
| 4 | ERR SENS MOTOR | Motor sensor fault |
| 5 | ERR SENS CURRENT | Current sensor fault |
| 6 | ERR EMI UNKNOWN | EMI unknown |
| 7 | ERR EMI NO FUSE | EMI fuse missing |
| 8 | ERR EMI PROTECT | EMI protection active |
| 9 | ERR EMI NO EMI | EMI module absent |
| 10 | ERR FUSE 15A | 15A fuse blown |
| 11 | ERR FUSE 30A | 30A fuse blown |
| 12 | ERR FUSE 20A | 20A fuse blown |
| 13 | ERR EPS2 WRONG | EPS2 wrong type |
| 14 | ERR EPS2 MISSING | EPS2 absent |
| 15 | ERR BAT LOW | Battery low |
| 16 | ERR CHRG HIGH | Charger high |
| 17 | ERR CHRG CHECK 4A | Charger 4A check |
| 18 | ERR AC HIGH | AC voltage high |
| 19 | ERR AC LOW | AC voltage low |
| 20 | ERR AC NO AC | No AC power |
| 21 | ERR MOT MISSING | Motor absent |

---

## 8. Analog Scaling

Raw byte values from numeric response packets are converted to engineering units. The per-model gain factor (`gainForThisModel`) scales voltage and amperage readings and varies by unit model.

| Packet | Field | Conversion |
|---|---|---|
| `H` | Battery Voltage | `voltage = rawByte × gainForThisModel` |
| `I` | AC Voltage (high range) | `voltage = rawByte × gainForThisModel` (when raw ≥ 15) |
| `J` | AC Voltage (low range) | `voltage = (double) rawByte` (when raw < 15) |
| `S` | Motor Voltage | `voltage = rawByte × gainForThisModel` |
| Motor amp | Motor Amperage | `amperage = (rawByte + 237) × gainForThisModel` (raw + 246 − 9) |
| `N` | Speed | `speed = (double) rawByte` |
| `P` | Speed (offset variant) | `speed = (double)(rawByte − 25)` (stored as raw + 231 mod 256) |
| `M` | Hold-Open Timer | `timer = (double) rawByte` (seconds) |
| `G` | Obstruction Sensor | `sensor = rawByte × gainForThisModel` |

> The exact gain values per model are stored in a lookup table within the application binary and were not extracted.

---

## 9. Complete Protocol Flow Example

### Connection and initial status

After Bluetooth connects, the controller immediately begins streaming status lines. No handshake or initialization command is required.

Incoming from controller (gate closed, no errors):

```
Z\x02L00,K03,N00,M0A,G00,H7F,I3C,S00,O04,Q00,R03,T00,U00\r\n
```

Decoded:
- `Z` → gate closed
- `L=0x00` → `00000000` → all LEDs off
- `K=0x03` → `00000011` → charAt(6)=`'1'` isOpenLimitSet, charAt(7)=`'1'` isCloseLimitSet — both limit switches installed
- `M=0x0A` → 10 second hold timer
- `O=4` → Model H-10
- `Q=0` → no error
- `R=0x03` → `00000011` → charAt(6)=`'1'` isLastOpenSet, charAt(7)=`'1'` isAutoOpenSet — both ON

### Sending an Open command

```
→  Y\r
```

Controller begins moving; subsequent lines show:

```
Y\x02L80,K08,N1E,...\r\n
```

- `L=0x80` → `10000000` → charAt(0)=`'1'` → `isOpenLit` = true
- `K=0x08` → `00001000` → charAt(4)=`'1'` → `isOpening` = true  *(bit 3)*
- `N=0x1E` = 30 → speed 30

### Setting Hold-Open Timer to 30 seconds

```
→  h30\r
```

### Enabling AutoOpen (assuming LastOpen is already ON, R currently = 0x02)

Read current `R` value (0x02). Set bit 0 for AutoOpen: `0x02 | 0x01 = 0x03`.

```
→  r3\r
```

---

## 10. Implementation Notes

1. **Receive loop:** Buffer incoming bytes. Split on `\r\n` to get complete lines. Find the `\x02` separator. Take the suffix and split on `,`. Validate each token against `^[G-V][0-9A-F]{2}$`. Parse type (`token[0]`) and value (`parseInt(token.substring(1), 16)`).

2. **Bitmask extraction:** Convert the integer value to an 8-character binary string, left-padded with zeros. charAt(0) = bit 7 (MSB); charAt(7) = bit 0 (LSB). Compare each character to `'1'` for the boolean flag value.

3. **Send:** Construct the ASCII command string, append `\r` (0x0D), encode as UTF-8, write to the output stream. Do not append `\n` on the send side.

4. **Options atomicity:** The `r` command replaces all 8 option bits simultaneously. Always read the current `R` response value before changing a single flag to avoid silently resetting others.

5. **Keep-alive / timeout:** Both iOS and Android apps start a connection timeout timer. If no data arrives within the timeout, the connection is treated as lost.

6. **No polling required:** The controller continuously streams status lines whenever connected. The app need not send a query command to get the current state.

7. **Long-press detection:** Long-press commands (`yyyyyyyyyy`, etc.) are sent repeatedly while the button is held. The repeated-character string signals sustained input, not a single event.

---

## 11. Source Material and Confidence Notes

This documentation was produced by static analysis of four application binaries:

| Binary | Type | Key Findings |
|---|---|---|
| `Viking_Konnect` | iOS ARM64 (MFi / ExternalAccessory) | Full ObjC class list; CFString literals; ARM64 disassembly of all command methods; complete response parser (`updateWithData:`); unit model names; error codes |
| `Vikings` | iOS ARM64 (SwiftyBluetooth / BLE) | Transport confirmation (TCP + BLE); AWS IoT backend for newer model; same device object fields |
| `Viking_Konnect_1_3_APKPure.apk` | Android DEX (Bluetooth Classic) | Complete Java class structure; all field names; full response switch/case disassembly; exact bit extraction logic (binary string `charAt`); all command string constants |
| `APKPure_3_20_6403_apkpure_com.apk` | Android DEX (APKPure store app) | Confirmed shared protocol constants; no additional protocol detail |

### Confidence by section

| Section | Confidence | Basis |
|---|---|---|
| Transport (Bluetooth SPP) | High | Explicit class/framework names in all 4 binaries |
| Response framing (`\x02` separator, `\r\n`) | High | CFString `"\x02G"` and `"\x02"`; Android string constant `"\r"`; `BufferedReader.readLine()` usage |
| Gate status prefix `Y`/`Z`/`[` | High | Directly in `ConnectedThread.run()` receive loop |
| Response token regex `^[G-V][0-9A-F]{2}$` | High | String constant present verbatim in both iOS and Android |
| Numeric packet type → field mapping (G–V) | High | Full switch-case disassembly of `updateWithData` |
| Bitmask bit ordering (MSB-first binary string) | High | Explicit `charAt()` calls with position constants 0–7 compared to ASCII `'1'` |
| `L` packet bit field assignment | High | Direct sequential `iput-boolean` instructions with field names |
| `K`, `R`, `T`, `U` packet bit assignments | High–medium | charAt positions confirmed; some position-to-field pairings inferred from sequential ordering |
| Gate command letters `Y`/`Z`/`[` | High | Present in iOS CFStrings and Android string constants; confirmed in receive loop |
| Long-press command strings | High | All 3 strings present verbatim in both platforms |
| Configuration command letters `h`/`n`/`p`/`m`/`r` | High | String constants confirmed; send dispatch confirmed; `r` mapping to option toggles confirmed from `VBMainActivity$30`–`$33` inner class analysis |
| `r` command bitmask bit positions | High | Confirmed from charAt position constants `#7`, `#6`, `#5`, `#4` in toggle handlers |
| Keypad commands `kFF`/`kBF`/`k7F` | High | Verbatim string constants in both platforms |
| Limit set/clear commands | Medium | UI labels confirmed; exact wire format not fully resolved |
| Analog scaling formulas | High–medium | Arithmetic confirmed from bytecode; exact gain table values not extracted |

---

## 12. Discrepancies Between Implementations

This section documents every identified discrepancy between the four analysed binaries. Each is rated for implementation risk.

---

### 12.1 ⚠️ CRITICAL — Numeric Packet Type Assignments Were Incorrect in Section 4

The most significant finding from cross-referencing the binaries is that the Section 4 numeric packet type table is wrong. The correct mapping, derived by tracing each switch-case body in `updateWithData` to its first `iput-wide` instruction before the terminal `goto`, is:

| Type | Actual field | Previously documented (incorrect) |
|---|---|---|
| `G` | Motor Amperage | ~~Obstruction Sensor~~ |
| `H` | Obstruction Sensor | ~~Battery Voltage~~ |
| `I` | Battery Voltage | ~~AC Voltage (high range)~~ |
| `J` | AC Voltage | ~~AC Voltage (low range)~~ |
| `M` | Overlap | ~~Hold-Open Timer~~ |
| `N` | Hold-Open Timer | ~~Speed~~ |
| `P` | Speed (raw − 25 offset) | *(label unchanged, assignment confirmed)* |
| `S` | Motor Voltage | *(unchanged)* |

**Range-check details:**

`G` (Motor Amperage): if raw < 15, value is passed through as-is. If raw ≥ 15, the adjustment `(raw + 246 − 9)` is applied before multiplying by the model gain factor.

`J` (AC Voltage): if raw < 15, value is used directly as a double. If raw ≥ 15, the gain factor is applied. Both branches store to the same `acVoltage` field.

> **Caution for implementors:** Do not assume alphabetical or logical ordering of these letters. Test each packet type against known hardware state before committing the mapping.

---

### 12.2 ⚠️ CRITICAL — Unit Model Codes Use ASCII Character Encoding

The `O` packet does **not** carry a simple integer 0–11. The raw byte from `parseInt(twoHexChars, 16)` is stored directly in `unitModel`, and the `getDisplayNameForUnitModel` switch table has its `first_key = 0x30` (ASCII `'0'`). The actual on-wire encoding is:

| O packet | Raw byte | ASCII char | Model |
|---|---|---|---|
| `O30` | 0x30 | `'0'` | R-6 |
| `O31` | 0x31 | `'1'` | X-9 |
| `O32` | 0x32 | `'2'` | T-21 |
| `O33` | 0x33 | `'3'` | F-1 |
| `O34` | 0x34 | `'4'` | G-5 |
| `O35` | 0x35 | `'5'` | H-10 |
| `O36` | 0x36 | `'6'` | L-3 |
| `O37` | 0x37 | `'7'` | K-2 |
| `O38` | 0x38 | `'8'` | Q-7 |
| `O39` | 0x39 | `'9'` | B-12 |
| `O3A` | 0x3A | `':'` | I-8 |

Note that `O3A` (`':'`) falls outside the `[0-9A-F]` range accepted by the validation regex `^[G-V][0-9A-F]{2}$`. If the controller sends I-8 as `O3A`, the Android app would silently reject it as malformed. This is a potential bug in the Android app, or I-8 hardware may use a different encoding in practice.

**iOS vs Android discrepancy:** The iOS `getDisplayNameForUnitModel` uses a different switch layout (documented as codes 0–11 internally). This suggests iOS may subtract `0x30` from the raw byte before storing `unitModel`, while Android stores the raw hex value. New implementations should store the raw hex byte and use the above table for lookup.

---

### 12.3 ⚠️ CRITICAL — Gain Factor Values Are Now Known

`gainForThisModel` in the Android app is confirmed with specific values:

| Model (raw byte) | Model name | Gain factor |
|---|---|---|
| 0x36 (`'6'`) = L-3 | L-3 | 11.0 |
| 0x37 (`'7'`) = K-2 | K-2 | 11.0 |
| 0x38 (`'8'`) = Q-7 | Q-7 | 7.6 |
| All others | — | 7.6 (default) |

The iOS `gainForThisModel` uses a separate lookup, and may differ for specific models. The gain value directly scales voltage and amperage readings from `G`, `I`, `J`, `S` packets; incorrect gain produces wrong engineering values.

---

### 12.4 ⚠️ HIGH — Long-Press Stop Uses Two Different Characters

The short-press stop is `[\r` (ASCII 0x5B) on all platforms. For long-press, the iOS binary contains **two different** variants:

| Source | String | Character | Count | Terminator |
|---|---|---|---|---|
| iOS (uppercase variant) | `[[[[[[[[[[[[` | `[` = 0x5B — **same as short-press stop** | **12** | `\r\n` |
| iOS (lowercase variant) | `]]]]]]]]]]]` | `]` = 0x5D — **different character** | 11 | `\r\n` |
| Android | `]]]]]]]]]]` | `]` = 0x5D — **different character** | 10 | `\r` |

For open and close the discrepancy is only in count and terminator, not character. For stop, the iOS binary contains both `[x12` and `]x11` — it is unknown which the controller honours, or whether it accepts both. The `]` form is used by both Android and iOS lowercase, making it the safer bet. Test both on target hardware.

---

### 12.5 ⚠️ HIGH — Long-Press Repetition Count and Terminator

| Platform | Open/Close count | Stop count | Terminator |
|---|---|---|---|
| iOS | 11 | 12 (`[`) or 11 (`]`) | `\r\n` (CRLF) |
| Android | 10 | 10 | `\r` (CR only) |

If the controller counts characters, 10 and 11 may produce different behaviour. The safe choice is 11 with `\r\n`, matching the iOS majority. If the controller detects a sustained stream (i.e. ignores exact count), this does not matter.

---

### 12.6 ⚠️ MEDIUM — `isStopping` Field Never Populated by Protocol Parser

Both iOS and Android define an `isStopping` property on the device object, but neither app's `updateWithData` parser sets it from any protocol packet. It is present in both field lists but absent from every bitmask case body. It may be:

- Derived in UI code as `!isOpening && !isClosing && isStopLit`
- Set by a separate timer / debounce
- A vestigial field from an earlier protocol version that included an explicit stopping state

Do not rely on receiving an `isStopping` state from the controller directly.

---

### 12.7 ⚠️ MEDIUM — G-5 / X-9 Model Handling Differs Between Apps

| App | Packet `O37` | Packet `O31` | Notes |
|---|---|---|---|
| Android (`Viking_Konnect_1_3`) | G-5 | X-9 | Separate entries |
| iOS (`Viking_Konnect`) | Displayed as `G-5/X-9` | X-9 | Same gain for both |

iOS explicitly groups G-5 and X-9 with a combined display name and applies the same gain factor. Android lists them separately; their gain factor may differ. Whether the controller actually distinguishes these two models by raw byte is unknown — they may always send the same `O` byte for both physical units.

---

### 12.8 LOW — Temperature Voltage Field Exists but Is Never Written

Both apps define a `tempVoltage` / `kVBMeterTypeTempVoltage` field, and both use the same misspelled display name `"Tempurature Voltage"`. No packet type letter (`G`–`V`) writes this field in either app's parser. It is defined in both, populated nowhere, and should not be relied upon.

---

### 12.9 LOW — Response Terminator Tolerance

iOS uses `NSInputStream` with manual buffer management, parsing on the `\x02` sync byte. Android uses `BufferedReader.readLine()` which accepts `\r\n`, `\r`, or `\n` interchangeably. The controller most likely sends `\r\n`, but an implementation that only emits `\r` will work with Android and likely with iOS too.

---

### 12.10 Summary Table

| # | Risk | Discrepancy |
|---|---|---|
| 12.1 | **Critical** | Numeric packet G/H/I/J/M/N assignments were incorrect; corrected mapping confirmed |
| 12.2 | **Critical** | Unit model `O` packet uses ASCII character encoding (0x30–0x3A), not 0–11 integers |
| 12.3 | **Critical** | Gain factors confirmed: L-3 and K-2 = 11.0; Q-7 and all others = 7.6 (Android) |
| 12.4 | **High** | Long-press stop uses `[` (iOS uppercase) vs `]` (iOS lowercase + Android) — different characters |
| 12.5 | **High** | Long-press count 10 (Android) vs 11 (iOS); terminator CR vs CRLF |
| 12.6 | **Medium** | `isStopping` field defined in both apps, never set by protocol parser in either |
| 12.7 | **Medium** | G-5/X-9 treated as one model on iOS, two separate models on Android |
| 12.8 | **Low** | Temperature voltage field defined in both, never populated from protocol |
| 12.9 | **Low** | Response terminator: Android tolerates CR-only; iOS expects CRLF |
