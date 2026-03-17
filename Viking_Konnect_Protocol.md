# Viking Konnect — RS485-over-TCP Gate Controller Protocol

*Reverse-engineered from `Viking_Konnect_1_3_APKPure.apk` · March 2026*

---

## 1. Overview

Viking Konnect is an Android application that controls Viking gate operators via Bluetooth. The phone connects to a Viking Konnect Bluetooth-to-RS485 bridge module mounted on the gate controller. The bridge accepts commands and returns status packets formatted as an RS485 serial stream tunnelled over the Bluetooth SPP (Serial Port Profile) socket.

The protocol is **text-based and line-delimited ASCII**. Every message — both commands sent by the app and status packets received from the device — is a printable ASCII string terminated by a carriage-return (`\r`, 0x0D). No binary framing, CRC, or length prefix is used at the application layer.

> **Note:** Although this document refers to "RS485 over TCP", analysis of the APK confirms the Android app communicates exclusively via Bluetooth (SPP). The underlying Viking controller hardware uses an RS485 bus; the Bluetooth module bridges that bus. Any TCP-to-RS485 adapter connecting to the same underlying serial stream would use the identical line protocol described here.

---

## 2. Transport Layer

### 2.1 Bluetooth SPP

The app creates an insecure RFCOMM socket using `createInsecureRfcommSocket(uuid)` with the proprietary service UUID:

```
02210893-8f12-4278-83f9-f29e71f30fc0
```

Once the socket is open, a `ConnectedThread` is started. It wraps the socket `InputStream` in a `BufferedReader` and reads lines indefinitely. The `OutputStream` is used for writes.

### 2.2 Character Encoding

All strings are encoded as **UTF-8**. In practice every message uses 7-bit-clean ASCII only.

### 2.3 Line Termination

Commands sent by the app are terminated with `\r` (CR, 0x0D). Device response lines are read with `BufferedReader.readLine()`, which strips the line terminator automatically.

### 2.4 TCP Mode (External Adapter)

The application stores `HOST_NAME` and `PORT_NUMBER` settings in `SharedPreferences`. This is consistent with an optional TCP-to-RS485 Ethernet adapter that exposes the same serial stream on a TCP socket. When used this way the port number is user-configurable; no default is hard-coded in the app. The line protocol is identical.

---

## 3. Packet Format

### 3.1 Device-to-App Status Packets

The device sends unsolicited status packets at regular intervals. Each packet is a single line. Before parsing, the `ConnectedThread` replaces three characters to normalise the byte stream:

| Raw char | Normalised to | Purpose |
|----------|---------------|---------|
| `Y` (0x59) | stripped | Indicator pre-processing |
| `Z` (0x5A) | stripped | Indicator pre-processing |
| `[` (0x5B) | stripped | Indicator pre-processing |

After normalisation, a two-character prefix is checked. If the result begins with the two-byte sequence `0x02 'G'` (STX + `G`), the packet is a gate-status telegram. If the total length exceeds 78 characters, the first 79 bytes are used as the payload (the `0x02` marker is stripped and any `0x02` delimiter within the data is replaced with an empty string before the payload is passed to the parser).

### 3.2 Status Packet Validation Regex

Each field token within the status packet is validated against:

```
^[G-V][0-9A-F]{2}$
```

Every field is exactly **3 characters**: a letter in the range `G`–`V` (0x47–0x56) followed by exactly two uppercase hex digits (`0`–`9`, `A`–`F`).

The leading letter selects the **field type**; the two hex digits carry the **field value** as a two-nibble unsigned integer (0x00–0xFF).

### 3.3 Status Packet Structure

A full status packet is a comma-separated list of 3-character field tokens. Parsing splits on `,` and processes each token sequentially. The token count varies with controller model and firmware.

```
<STX>G<status_hex>,H<leds_byte1_hex>,I<leds_byte2_hex>,
     J<leds_byte3_hex>,K<meter_type_hex>,L<meter_value_hex>,
     M<error_code_hex>,N<options_byte1_hex>,O<options_byte2_hex>,
     P<options_byte3_hex>,Q<options_byte4_hex>,...<CR>
```

### 3.4 Field Type Codes

| Field Letter | Hex Code | Data Field | Notes |
|---|---|---|---|
| `G` | 0x47 | Gate Status | Bit-decoded: OPENING, OPENED, CLOSED, STOP, REOPEN |
| `H` | 0x48 | LED Byte 1 | Bits: Open, Close, Stop, ATG, Radio, Safety, SafetyATG, UL |
| `I` | 0x49 | LED Byte 2 | Bits: CLoop, Brake; additional indicator flags |
| `J` | 0x4A | LED Byte 3 | Slave/secondary input indicators |
| `K` | 0x4B | Meter Display Selector | Selects which meter is currently shown (§3.6) |
| `L` | 0x4C | Meter Value (raw) | Raw integer; scaled by `gainForThisModel()` |
| `M` | 0x4D | Error Code | Encoded error index; decoded by `getErrorCodeFromConstant()` |
| `N` | 0x4E | Options Byte 1 | 8-bit option flags, MSB first (Auto-Open, Last-Open, etc.) |
| `O` | 0x4F | Options Byte 2 | 8-bit option flags (Hold-Open Timer state, etc.) |
| `P` | 0x50 | Options Byte 3 | 8-bit option flags |
| `Q` | 0x51 | Options Byte 4 | 8-bit option flags (Sync, Pre-Warning, etc.) |
| `R` | 0x52 | Hold Open Timer | Current hold-open countdown value |
| `S` | 0x53 | Speed | Motor speed reading |
| `T` | 0x54 | Motor Amperage | Motor current; scaled by gain |
| `U` | 0x55 | Motor Voltage | Motor voltage; scaled by gain |
| `V` | 0x56 | Additional sensor data | Model-specific |

> Fields `K` through `V` are all read and scaled/decoded, but not every model populates every field. The parser advances through the token array positionally, so the order is fixed.

### 3.5 LED Bit Assignments

LED bytes (fields `H`, `I`, `J`) are decoded bit by bit:

| LED Byte | Bit Index | Constant Name | Indicator |
|---|---|---|---|
| `H` | 0 | `kVBLEDBitIndexOpen` | Open limit LED |
| `H` | 1 | `kVBLEDBitIndexClose` | Close limit LED |
| `H` | 2 | `kVBLEDBitIndexStop` | Stop LED |
| `H` | 3 | `kVBLEDBitIndexSafetyATG` | Safety / ATG LED |
| `H` | 4 | `kVBLEDBitIndexRadio` | Radio input LED |
| `H` | 5 | `kVBLEDBitIndexSafety` | Safety sensor LED |
| `H` | 6 | `kVBLEDBitIndexUL` | UL alarm LED |
| `I` | 0 | `kVBLEDBitIndexCloop` | C-Loop LED |
| `I` | 1 | *(brake)* | Brake LED |

### 3.6 Meter Types (Field K)

| K Value | `kVBLEDDisplayIndex` Constant | Display Label |
|---|---|---|
| 0x00 | `kVBLEDDisplayIndexNone` | No meter selected |
| 0x01 | `kVBLEDDisplayIndexGateStatus` | Gate status |
| 0x02 | `kVBLEDDisplayIndexError` | Error code display |
| 0x03 | `kVBLEDDisplayIndexModel` | Controller model identifier |
| 0x04 | `kVBLEDDisplayIndexSpeed` | Gate speed |
| 0x05 | `kVBLEDDisplayIndexMotorAmp` | Motor amperage |
| 0x06 | `kVBLEDDisplayIndexMotorVolt` | Motor voltage |
| 0x07 | `kVBLEDDisplayIndexACVolt` | AC line voltage |
| 0x08 | `kVBLEDDisplayIndexBatVolt` | Battery voltage |
| 0x09 | `kVBLEDDisplayIndexTimer` | Hold-open timer |
| 0x0A | `kVBLEDDisplayIndexOverlap` | Overlap reading |
| 0x0B | `kVBLEDDisplayIndexODSSensor` | ODS obstruction sensor |

### 3.7 Model Gain Factor (`gainForThisModel`)

The raw meter value in field `L` is multiplied by a model-specific gain factor. The gain is selected based on the controller model code returned in the `kVBLEDDisplayIndexModel` meter slot:

| Model Code (decimal) | Known Models |
|---|---|
| 54 | G5 |
| 55 | R6, Q7 |
| 56 | K2, L3, X9, T21 |
| (default) | B12, F1, H10, I8 |

> Exact gain values are encoded as floating-point literals in the `gainForThisModel()` bytecode; the three distinct code paths correspond to integer constants 54, 55, and 56 loaded before a wide (double) return.

---

## 4. Error Code Enumeration

Field `M` carries a numeric error index. `getErrorCodeFromConstant()` maps it to a display string:

| Index | Internal Constant | Display String |
|---|---|---|
| 0 | `kVBErrorCodeNone` | `NO ERRORS` |
| 1 | `kVBErrorCodeOpnLimit` | `ERR OPN / LIMIT` |
| 2 | `kVBErrorCodeClsLimit` | `ERR CLS / LIMIT` |
| 3 | `kVBErrorCodeNoLimit` | `ERR NO / LIMIT` |
| 4 | `kVBErrorCodeSensMotor` | `ERR SENS / MOTOR` |
| 5 | `kVBErrorCodeSensCurrent` | `ERR SENS / CURRENT` |
| 6 | `kVBErrorCodeEmiUnknown` | `ERR EMI / UNKNOWN` |
| 7 | `kVBErrorCodeEmiNoFuse` | `ERR EMI / NO FUSE` |
| 8 | `kVBErrorCodeEmiProtect` | `ERR EMI / PROTECT` |
| 9 | `kVBErrorCodeEmiNoEmi` | `ERR EMI / NO EMI` |
| 10 | `kVBErrorCodeFuse15A` | `ERR FUSE / 15A` |
| 11 | `kVBErrorCodeFuse30A` | `ERR FUSE / 30A` |
| 12 | `kVBErrorCodeFuse20A` | `ERR FUSE / 20A` |
| 13 | `kVBErrorCodeEPS2Wrong` | `ERR EPS2 / WRONG` |
| 14 | `kVBErrorCodeEPS2Missing` | `ERR EPS2 / MISSING` |
| 15 | `kVBErrorCodeBatLow` | `ERR BAT / LOW` |
| 16 | `kVBErrorCodeChargerHigh` | `ERR CHRG / HIGH` |
| 17 | `kVBErrorCodeChargerCheck4A` | `ERR CHRG / CHECK 4A` |
| 18 | `kVBErrorCodeACHigh` | `ERR AC / HIGH` |
| 19 | `kVBErrorCodeACLow` | `ERR AC / LOW` |
| 20 | `kVBErrorCodeACNoAC` | `ERR AC / NO AC` |
| 21 | `kVBErrorCodeMotorMissing` | `ERR MOT / MISSING` |
| 22+ | *(unmapped)* | `ERR UNKNOWN` |

> The string constant `ERR NO MOT` also appears in the string table, suggesting a motor-not-detected sub-code for some firmware variants.

---

## 5. Command Set (App → Device)

All commands are sent by `sendStringToDevice(String cmd)`. The method appends `\r`, encodes to UTF-8, and writes to the Bluetooth `OutputStream`.

Internal log line: `sending command: <cmd>`

### 5.1 Gate Motion Commands

| Command String | Repeated? | UI Source | Action |
|---|---|---|---|
| `yyyyyyyyyy` | 10 × `y` | Open button (touch-hold) | Open gate |
| `zzzzzzzzzz` | 10 × `z` | Close button (touch-hold) | Close gate |
| `]]]]]]]]]]` | 10 × `]` | Stop button (touch-hold) | Stop gate motion |

> The 10-character repetition is deliberate — the same character is sent ten times in a single string. This acts as a de-facto framing marker and provides robustness against single-byte corruption on the RS485 bus.

### 5.2 Limit Set / Clear Commands

| Command | Action |
|---|---|
| `kFF` | Clear open limit **and** close limit |
| `kBF` | Clear open limit only |
| `k7F` | Clear close limit only |
| `<options-byte-hex>` | Set new open or close limit at current gate position (see §5.3) |

### 5.3 Options / Configuration Commands

Option switches (Auto-Open, Last-Open, Pre-Warning, Sync) send a configuration update packet. Construction sequence from inner classes `$30$2`–`$33$2`:

1. Call `binaryStringForOptions()` to get the current 8-bit options state as a binary string (`"0"` / `"1"` per bit, MSB first).
2. Toggle the relevant bit (`'0'` → `'1'` or `'1'` → `'0'`).
3. Call `convertBinaryStringToInt()` to obtain the integer value of the new byte.
4. Call `Integer.toHexString()` to format as two hex digits.
5. Prepend the letter `r` to form the command string: `r<HH>` where `HH` is the two-hex-digit options value.
6. Call `sendStringToDevice("r" + hexValue)`.

**Example — toggle Auto-Open from off to on:**

```
current binaryStringForOptions() = "00000000"
flip bit 0 (Auto-Open)           = "10000000"
convertBinaryStringToInt(...)    = 128 = 0x80
toHexString(128)                 = "80"
command sent                     = "r80\r"
```

> The 8 option bits are MSB-first: Auto-Open, Last-Open, Pre-Warning/Horn, Sync (bits 0–3); bits 4–7 are reserved. Inner class `$33` includes the string `"Would you like to change the SYNC to"`, confirming `$33` = Sync, and inferring `$30` = Auto-Open, `$31` = Last-Open, `$32` = Pre-Warning by order.

### 5.4 Full Command Summary

| Command | Direction | Meaning |
|---|---|---|
| `yyyyyyyyyy` | App → Device | Open gate |
| `zzzzzzzzzz` | App → Device | Close gate |
| `]]]]]]]]]]` | App → Device | Stop gate |
| `kFF` | App → Device | Clear both limits |
| `kBF` | App → Device | Clear open limit |
| `k7F` | App → Device | Clear close limit |
| `r<HH>` | App → Device | Write options byte `HH` (hex) to controller |

---

## 6. Status Packet Parsing (`updateWithData`)

### 6.1 Receive Loop

`ConnectedThread.run()`:

1. `readLine()` on the `BufferedReader` (blocking; strips `\r`/`\n`).
2. Discard line if it does not begin with STX (0x02) + `'G'`.
3. If length > 78, trim payload to bytes `[2..79]` (skip the 2-byte prefix).
4. Broadcast a `"dataReadyNotification"` `LocalBroadcast` `Intent` so the UI thread can update.

### 6.2 Field Parsing Algorithm

```
String[] tokens = payload.split(",");

// Positional parsing; each token encodes one field
for each token:
  if token matches /^[G-V][0-9A-F]{2}$/:
    letter   = token.charAt(0)         // field type
    valueHex = token.substring(1, 3)   // two hex digits
    intVal   = Integer.parseInt(valueHex, 16)
    binaryStr = convertIntegerToBinary(intVal, 8)  // zero-padded to 8 bits
    // extract individual bits as boolean flags,
    // or use intVal directly for meter fields
```

### 6.3 Gate Status Bits (Field G)

| Bit (0 = LSB) | Flag Constant | Meaning |
|---|---|---|
| 0 | `FLAG_IS_OPENING` | Gate is currently opening |
| 1 | `FLAG_IS_OPENED` | Gate is fully open |
| 2 | *(CLOSED)* | Gate is fully closed |
| 3 | *(STOP)* | Gate is stopped |
| 4 | *(REOPEN)* | Gate is in re-open cycle |
| 5–7 | *(reserved)* | Not used by current app logic |

### 6.4 Gate State Display Strings

The string constant `"GATE IS\n"` is used as a prefix for the state label in the LED output text view. The state word appended is one of: `OPENING`, `OPENED`, `CLOSED`, `STOP`, `RE-OPEN`, or empty.

---

## 7. Options Byte Encoding

### 7.1 `binaryStringForOptions()`

Serialises 8 stored option boolean fields into an 8-character `'0'`/`'1'` string, MSB (first field) first:

| Bit Position (MSB = 0) | Field Name | UI Label |
|---|---|---|
| 0 | `isAutoOpenSet` | Auto-Open switch |
| 1 | `isLastOpenSet` | Last-Open switch |
| 2 | *(Pre-Warning)* | Pre-Warning / Horn |
| 3 | *(Sync)* | Sync switch |
| 4–7 | *(reserved)* | — |

### 7.2 `convertBinaryStringToInt()`

Accepts an 8-character `'0'`/`'1'` string (MSB first). Calls `recursiveToBinary()` then zero-pads to 8 characters. Equivalent to `Integer.parseInt(str, 2)`.

---

## 8. Meter Display and Scaling

The main screen shows a scrollable list of meters. Each entry is defined by `decimalNumberForMeterType()` and `readingDisplayForMeterType()`:

| `kVBMeterType` Constant | Display Label | Scaling |
|---|---|---|
| `kVBMeterTypeHoldOpenTimer` | Hold Open Timer | Raw integer (seconds) |
| `kVBMeterTypeBattery` | Battery Voltage | `roundOneDecimal(raw × gain)` |
| `kVBMeterTypeACVoltage` | AC Voltage | `roundOneDecimal(raw × gain)` |
| `kVBMeterTypeMotorAmperage` | Motor Amperage | `roundOneDecimal(raw × gain)` |
| `kVBMeterTypeMotorVoltage` | Motor Voltage | `roundOneDecimal(raw × gain)` |
| `kVBMeterTypeSpeed` | Speed | `roundOneDecimal(raw × gain)` |
| `kVBMeterTypeOverlap` | Overlap | Raw integer |
| `kVBMeterTypeObstuctionSensor` | Obstruction Sensor | Raw integer |
| `kVBMeterTypeTempVoltage` | Temperature Voltage | `roundOneDecimal(raw × gain)` |

> `roundOneDecimal()` formats to one decimal place using the format string `"%1.1f"`.

---

## 9. Full Status Display String

`VBVikingAccessDevice.description()` builds a human-readable summary by concatenating all decoded fields:

```
GATE IS <state>
, Id=<unit-model>
, Open: <true|false>
, Close: <true|false>
, Stop: <true|false>
, Re-Open: <true|false>
, ATG: <true|false>
, C. Loop: <true|false>
, Radio: <true|false>
, Hold Open Timer: <value>
, Speed: <value>
, Motor Amperage: <value>
, Motor Voltage: <value>
, AC Voltage: <value>
, Battery Voltage: <value>
, Obstruction Sensor: <value>
, Overlap: <value>
, Temperature: <value>
```

---

## 10. Supported Controller Models

| Internal Constant | Model Name |
|---|---|
| `kVBUnitModel_B12` | B12 |
| `kVBUnitModel_F1` | F1 |
| `kVBUnitModel_G5` | G5 |
| `kVBUnitModel_H10` | H10 |
| `kVBUnitModel_I8` | I8 |
| `kVBUnitModel_K2` | K2 |
| `kVBUnitModel_L3` | L3 |
| `kVBUnitModel_NULL` | NULL (not detected) |
| `kVBUnitModel_Q7` | Q7 |
| `kVBUnitModel_R6` | R6 |
| `kVBUnitModel_T21` | T21 |
| `kVBUnitModel_X9` | X9 |

> The string `"G-5"` (with hyphen) appears as a display-name variant distinct from `"G5"`; `unitModelDisplayName()` maps internal constants to user-facing strings.

---

## 11. Example Session

```
-- App connects to Bluetooth device --
-- ConnectedThread starts --

DEVICE -> APP:
  \x02G00,H3A,I00,J00,K04,L5E,M00,N00,O00,P00,Q00\r

  G=0x00  gate status = CLOSED (all bits 0)
  H=0x3A  LED byte 1 = 0011 1010
            bit1=Close lit, bit3=SafetyATG lit,
            bit4=Radio lit, bit5=Safety lit
  I=0x00  LED byte 2 all off
  J=0x00  slave LEDs all off
  K=0x04  speed meter selected
  L=0x5E  raw meter value = 94
  M=0x00  no error
  N..Q    all options bytes = 0x00


APP -> DEVICE  (user presses Open):
  yyyyyyyyyy\r


DEVICE -> APP  (gate now opening):
  \x02G01,H3A,I00,...\r
  G=0x01  bit0 set = OPENING


DEVICE -> APP  (gate fully open):
  \x02G03,H3E,...\r
  G=0x03  bit0+bit1 = OPENING|OPENED -> OPENED
  H=0x3E  Open LED now lit (bit0 set)


APP -> DEVICE  (user presses Stop):
  ]]]]]]]]]]\r


APP -> DEVICE  (user enables Auto-Open option):
  r80\r
  0x80 = 1000 0000 binary → bit 0 (Auto-Open) = 1
```

---

## 12. Implementation Notes

### 12.1 Connection Timeout

A configurable connection timeout timer (`_connectionTimoutTimer`) is set after pairing. If no status packet is received within the timeout window, the app transitions to a connection-failed state.

### 12.2 Previous Device Cache

The most recently paired device address and name are stored in `SharedPreferences` under `VB_BLUETOOTH_DEVICE_ADDRESS_KEY` and `VB_BLUETOOTH_DEVICE_NAME_KEY`. On launch the app attempts to reconnect automatically.

### 12.3 Limit Button Behaviour

The Set Open Limit and Set Close Limit buttons use `OnTouchListener` (not `OnClickListener`) to detect the precise moment the user lifts their finger, sending the "set limit at current position" command at that instant. The open-limit command sends an options-byte string using the `kBF` prefix; the close-limit uses `k7F`.

### 12.4 Slave / Dual-Gate Support

The UI includes a slave section (`slaveFrame`, `slaveFrame1`) with mirrored LEDs (`openSlaveLED`, `closeSlaveLED`, `stopSlaveLED`, `atgSlaveLED`). The booleans `isOpenCommandFromSlave`, `isCloseCommandFromSlave`, and `isStopCommandFromSlave` indicate whether commands originated from a secondary gate controller on the same RS485 bus.

### 12.5 `dataReadyHandler` Timing

The `dataReadyHandler()` processes the broadcast Intent on the UI thread. The integer constants `50`, `4`, and `11` found in its bytecode are likely timer intervals (ms or animation-frame counts) used to throttle UI refresh.

---

## Appendix A: Key String Literals

| String Value | Usage Context |
|---|---|
| `02210893-8f12-4278-83f9-f29e71f30fc0` | Bluetooth SPP service UUID |
| `yyyyyyyyyy` | Open command body (10 × `y`) |
| `zzzzzzzzzz` | Close command body (10 × `z`) |
| `]]]]]]]]]]` | Stop command body (10 × `]`) |
| `kFF` | Clear both limits |
| `kBF` | Clear open limit |
| `k7F` | Clear close limit |
| `r` | Options write command prefix |
| `^[G-V][0-9A-F]{2}$` | Per-field validation regex |
| `\x02G` | Packet header (STX + `G`) |
| `,` | Field delimiter in status packet |
| `GATE IS\n` | Gate-state display prefix |
| `sending command: ` | Internal log tag |
| `Cannot read input, socket has been disconnected` | Read-loop termination log |
| `UTF-8` | Byte encoding for Bluetooth stream |

---

## Appendix B: Key Methods Cross-Reference

| Method | Class | DEX Offset | Purpose |
|---|---|---|---|
| `updateWithData(String)` | `VBVikingAccessDevice` | 0xC945C | Parse raw status packet into device fields |
| `binaryStringForOptions()` | `VBVikingAccessDevice` | 0xC8D64 | Serialise option booleans to binary string |
| `convertIntegerToBinary(int)` | `VBVikingAccessDevice` | 0xC8EE0 | Integer to zero-padded 8-bit binary string |
| `gainForThisModel()` | `VBVikingAccessDevice` | 0xC8CE4 | Return analogue scaling gain for model |
| `getErrorCodeFromConstant(int)` | `VBVikingAccessDevice` | 0xC91D0 | Map error index to display string |
| `sendStringToDevice(String)` | `VBBluetoothAdapter` | 0xC3FE8 | Append CR, encode UTF-8, write to socket |
| `run()` (ConnectedThread) | `VBBluetoothAdapter$ConnectedThread` | 0xC3A78 | Receive loop: read, validate, broadcast |
| `dataReadyHandler()` | `VBMainActivity` | 0xC6F9C | UI update triggered by broadcast |

---

*Produced by static analysis of `Viking_Konnect_1_3_APKPure.apk`. All offsets reference the extracted `classes.dex`.*
