# Unified Diagnostic Services (UDS)

DiveCAN uses [UDS (ISO-14229)](https://en.wikipedia.org/wiki/Unified_Diagnostic_Services) for settings/firmware/log dumps.

UDS runs on top of **ISO-TP (ISO-15765-2)** and supports multi-frame transfers, structured data identifiers (DIDs), and memory upload/download operations.

---

## Transport Layer (ISO-TP)

UDS messages are transmitted over ISO-TP using Single Frame (SF), First Frame (FF), Consecutive Frame (CF), and Flow Control (FC) formats.

## Interoperability Quirks

- Shearwater handsets respond with flow control (FC) to a fixed address (`0xFF`).
- Shearwater handsets do not wait for flow control (FC) before sending the first consecutive frame.
- Shearwater uses UDS extended addressing (N_AE) set to `0x00`, resulting in the first byte of the UDS payload always being `0x00`.


## Negative Response Codes

Negative responses found:

```text
7F <ServiceID> <ResponseCode>
```

| Code | Name                                        | Description                            |
|------|---------------------------------------------|----------------------------------------|
| `0x10` | `GeneralReject`                             | Generic failure                         |
| `0x13` | `IncorrectMessageLengthOrInvalidFormat`     | Payload length / format is not valid   |
| `0x21` | `BusyRepeatRequest`                         | ECU is busy, caller should retry       |
| `0x22` | `ConditionsNotCorrect`                      | Preconditions not met                  |
| `0x24` | `RequestSequenceError`                      | Request is out of expected sequence    |
| `0x31` | `RequestOutOfRange`                         | Address / DID / value not supported    |
| `0x34` | `AuthenticationRequired`                    | Authentication step missing/failed     |
| `0x72` | `GeneralProgrammingFailure`                 | Programming (flash/NVM) failed         |
| `0x73` | `WrongBlockSequenceCounter`                 | TransferData block counter mismatch    |

---

## UDS Services Used on DiveCAN

The following UDS services are known to be used by SOLO:

| Service                   | Request SID | Response SID | Notes                      |
|---------------------------|-------------|--------------|----------------------------|
| ReadDataByIdentifier      | `0x22`      | `0x62`       | RDBI                       |
| WriteDataByIdentifier     | `0x2E`      | `0x6E`       | WDBI                       |
| RequestDownload           | `0x34`      | `0x74`       | Firmware / data download   |
| RequestUpload             | `0x35`      | `0x75`       | Memory upload / dumping    |
| TransferData              | `0x36`      | `0x76`       | Chunked transfer blocks    |
| TransferExit *(inferred)* | `0x37`      | `0x77`       | End of transfer sequence   |
| Negative Response         | `0x7F`      | —            | Error reporting            |

---

## ReadDataByIdentifier / WriteDataByIdentifier

Both services operate on 16‑bit Data Identifiers (DIDs).
DiveCAN only uses one RDBI per request.


## Known Data Identifiers (DIDs)

The following DIDs are defined in firmware and confirmed via implementation:

| DID       | Name                                   | Access | Comment                                                                 |
|-----------|----------------------------------------|--------|-------------------------------------------------------------------------|
| `0x8010`  | Serial number (ASCII)                  | R      | 8 bytes                                                                 |
| `0x8011`  | Firmware version (ASCII)               | R      | 3 bytes                                                                 |
| `0x8020`  | Firmware download capability           | R      | Base address and maximum writable size                                  |
| `0x8021`  | Log upload capability                  | R      | Base address and readable size                                          |
| `0x8200`  | Serial number (binary)                 | R/W    | 4 bytes                                                                 |
| `0x8201`  | Device identifier                      | R      | 12 bytes                                                                |
| `0x8202`  | Encrypted configuration blob           | R/W    | 16 bytes                                                                |
| `0x8203`  | O₂ cell calibration state (SOLO)       | R      | 3 calibration values + validity flags                                   |
| `0x8204`  | O₂ cell calibration request (SOLO)     | W      | Triggers calibration                                                    |
| `0x8205`  | O₂ cell zero‑offsets (SOLO)            | R      | Per‑cell ADC offsets                                                    |
| `0x8206`  | O₂ zero‑offset calibration trigger (SOLO) | W   | Triggers zero‑offset calibration                                        |
| `0x8209`  | Firmware CRC                           | R      | CRC32                                                                   |
| `0x820A`  | Voltage reference calibration (SOLO)   | R/W    | ADC reference value                                                     |
| `0x820B`  | Control configuration (SOLO)           | R      | Packed control and limit flags                                          |
| `0x9100`  | User setting count                     | R      | Number of user settings                                                 |
| `0x9110`  | User setting metadata (base)           | R      | Label, type, editable                                                   |
| `0x9130`  | User setting value (base)              | R      | Value/max for setting                                                   |
| `0x9150`  | User setting label (base)              | R      | Label for selection                                                     |
| `0x9350`  | User setting commit                    | W      | Commit settings                                                         |

---

## RequestUpload (Memory Dump)

`RequestUpload (0x35)` is used to **read spi flash/MCU data** on SOLO

### SOLO

| Region     | Virtual Address Range        | Real Address | Notes |
|------------|------------------------------|--------------|-------|
| Unknown | `0xC2000080` – `0xC2000FFF` | `0x00000080` (FLASH) | Size align **8** |
| Log (encrypted) | `0xC3001000` – `0xC3FFFFFF` | `0x00010000` (FLASH)| Size align **12** (12 is size of log entry)|
| Transfer security context | `0xC5000000` – `0xC500007F` | `0x1FFFF7F0` (MCU) | Size align **0**|
> **Note:** The address range is treated as *virtual* by higher-level code; only ranges matching these constraints are accepted by the device. Misaligned or out-of-range requests will typically return `RequestOutOfRange (0x31)`.

#### Transfer security context (SOLO)

The transfer security context region exposes a fixed‑format structure containing
per‑transfer integrity data and encryption context. It is used when validating and
decoding downloaded log data.

| Offset | Size | Field | Description |
|------:|-----:|-------|-------------|
| `0x00` | 4 | CRC32 | CRC32 of the transferred **encrypted** log data |
| `0x04` | 1 | Length | Length field (always `0x10`, decimal 16) |
| `0x05` | 4 | RTC timestamp | Timestamp used as part of the log encryption context |
| `0x09` | 12 | Device ID | Device identifier derived from the MCU unique ID |

**Notes:**
- The CRC32 represents the checksum of the **encrypted data as transferred** and is validated **after** the upload completes.
- The RTC timestamp and device ID bind logs to a specific device and time context.


## RequestDownload (Firmware Upload)

`RequestDownload (0x34)` is used for **writing** flashing firmware to device.
Before issuing a RequestDownload, the valid base address and maximum writable length can obtained by reading DID 0x8020 (FirmwareDownload)


## User Settings (Menu System)

User settings are exposed via the `0x91xx` DID range and encode setting indices directly in the DID.

### DID Encoding

| Type              | Base DID   | Encoding                        |
|-------------------|-----------:|---------------------------------|
| SettingCount      | `0x9100`   | —                               |
| SettingInfo(i)    | `0x9110`   | `0x9110 + i`                    |
| SettingValue(i)   | `0x9130`   | `0x9130 + i`                    |
| SettingLabel(i,j) | `0x9150`   | `0x9150 + i + (j << 4)`         |
| SettingSave       | `0x9350`   | —                               |

Where:

- `i` = setting index (0–15 encoded in low nibble)
- `j` = option index (selection variants; encoded in high nibble)

### Payload Formats

#### SettingCount (`0x9100`)

```text
<count: u8>
```

Number of available settings.

#### SettingInfo(i) (`0x9110 + i`)

```text
<label bytes…> <kind: u8> <editable: u8>
```

- `label`    : raw bytes, usually ASCII or UTF‑8
- `kind`     :
  - `0` = Integer (hex if divisor = 0, scaled decimal otherwise)
  - `1` = Selection
  - `2` = Scaled (same encoding as Integer; semantic distinction)
- `editable` : `0` = read‑only, `1` = editable

#### SettingValue(i) (`0x9130 + i`)

```text
<max: u64 BE> <current: u64 BE>
```

- `max`     : maximum allowed numeric value
- `current` : current setting value

For non‑numeric settings (e.g. selections), `current` is used as an index into the available options.

#### SettingLabel(i,j) (`0x9150 + i + (j << 4)`)

```text
<label bytes…>
```

Human‑readable label for the selection option `j` of setting `i`.

Setting labels are only used for `Selection` type settings, and are not required for `Text` / `Number`.

#### SettingSave (`0x9350 + index`)

Writes a new value for a user setting.

- The DID is **per‑setting**, encoded as `0x9350 + index`.
- The payload contains **raw input bytes** (1–8 bytes).

---