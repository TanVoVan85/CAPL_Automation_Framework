# CAPL Automation Framework

A lightweight, quickly adaptable CAPL library for Vector CANoe targeting:

- **CAN TP** — ISO 15765-2 transport-protocol layer (single-frame, multi-frame, flow control)
- **UDS Diagnostics** — ISO 14229-1 services (session control, security access, read/write by ID, DTC management, routine control)
- **CAN Flashing** — Full ECU software update sequence driven by UDS (erase, download, transfer, verify, reset)

---

## Repository layout

```
CAPL_Automation_Framework/
├── lib/
│   ├── Common/
│   │   └── utils.can          # Shared logging, error handling, helper functions
│   ├── CANTP/
│   │   └── can_tp.can         # ISO 15765-2 CAN Transport Protocol
│   ├── UDS/
│   │   └── uds_diag.can       # ISO 14229-1 UDS Diagnostics
│   └── Flash/
│       └── can_flash.can      # ECU flashing state machine
└── examples/
    ├── example_can_tp.can     # CAN TP send / receive example
    ├── example_uds.can        # UDS diagnostic session example
    └── example_flash.can      # Full flash sequence example
```

---

## Quick start

### 1. Include the desired library in your CAPL node

```capl
// CAN TP only
includes { #include "lib/CANTP/can_tp.can" }

// UDS (automatically includes CAN TP)
includes { #include "lib/UDS/uds_diag.can" }

// Flashing (automatically includes UDS and CAN TP)
includes { #include "lib/Flash/can_flash.can" }
```

### 2. Set addressing variables before use

```capl
gUDS_TxId    = 0x7E0;  // Tester → ECU
gUDS_RxId    = 0x7E8;  // ECU → Tester
gUDS_Channel = 1;      // CAN channel number
```

### 3. Route received frames into the library

```capl
on message 0x7E8
{
  byte  frame[8];
  int   i, state;
  for (i = 0; i < 8; i++) frame[i] = this.byte(i);
  state = canTP_processRxFrame(frame);
  // handle CANTP_STATE_COMPLETE …
}
```

---

## CAN TP library (`lib/CANTP/can_tp.can`)

Implements the full ISO 15765-2 PDU exchange for classical CAN (8-byte frames).

| Function | Description |
|---|---|
| `canTP_sendRequest(txId, rxId, data, length, channel)` | Send a payload of 1–4095 bytes; handles SF/FF/CF/FC automatically |
| `canTP_processRxFrame(frameData)` | Feed an 8-byte frame into the RX state machine; returns current state |
| `canTP_getResponse(outData, maxLen)` | Copy assembled SDU after `CANTP_STATE_COMPLETE`; returns byte count or negative error code |
| `canTP_sendFlowControl(canId, flag, bs, stMin, ch)` | Send a raw FC frame (CTS / WAIT / OVFLW) |
| `canTP_reset()` | Reset TX/RX state |

### State constants

| Constant | Value | Meaning |
|---|---|---|
| `CANTP_STATE_IDLE` | `0x00` | No active transfer |
| `CANTP_STATE_WAIT_FC` | `0x03` | Waiting for Flow Control |
| `CANTP_STATE_RX_CF` | `0x11` | Receiving consecutive frames |
| `CANTP_STATE_COMPLETE` | `0x20` | Transfer finished successfully |
| `CANTP_STATE_ERROR` | `0xFF` | Protocol error |

---

## UDS library (`lib/UDS/uds_diag.can`)

Built on top of CAN TP. Provides per-service helper functions and a
response-processing pipeline.

### Service helpers

| Function | SID | Description |
|---|---|---|
| `uds_sessionControl(subFunc)` | `0x10` | Switch diagnostic session |
| `uds_ecuReset(resetType)` | `0x11` | Trigger ECU reset |
| `uds_clearDTC(groupOfDTC)` | `0x14` | Clear stored DTCs |
| `uds_readDTCByStatusMask(mask)` | `0x19` | Read DTCs by status |
| `uds_readDataByIdentifier(did)` | `0x22` | Read a data record |
| `uds_securityAccessRequestSeed(level)` | `0x27` | Request seed |
| `uds_securityAccessSendKey(level, key, len)` | `0x27` | Send computed key |
| `uds_writeDataByIdentifier(did, data, len)` | `0x2E` | Write a data record |
| `uds_routineControl(sub, id, params, len)` | `0x31` | Start/stop/result routine |
| `uds_testerPresent(suppress)` | `0x3E` | Keep session alive |

### Session management

```capl
uds_startTesterPresent(2000);   // Auto-send every 2 s
uds_startTesterPresent(0);      // Stop periodic Tester Present
```

### Response processing

After `canTP_getResponse()` returns `RESULT_OK`:

```capl
uds_processResponse(respData, respLen);
// Checks for 0x7F negative response, decodes NRC, updates gUDS_ResponseReceived
```

### Key global variables

| Variable | Default | Purpose |
|---|---|---|
| `gUDS_TxId` | `0x7E0` | Request CAN ID |
| `gUDS_RxId` | `0x7E8` | Response CAN ID |
| `gUDS_Channel` | `1` | CAN channel |
| `gUDS_ResponseReceived` | `0` | Set to `1` when response is ready |
| `gUDS_LastResult` | `RESULT_OK` | Last operation result code |

---

## Flash library (`lib/Flash/can_flash.can`)

Drives a complete ECU flash sequence through a state machine.

### Configuration

```capl
gFlash_MemoryAddress = 0x08000000;  // Target start address
gFlash_MemorySize    = 0x00040000;  // Image size in bytes
gFlash_BlockSize     = 0x200;       // Bytes per Transfer Data block
gFlash_SecurityLevel = 0x03;        // Security access level
```

### Sequence

```capl
flash_startSequence();              // Kick off the sequence
```

In `on message` for the ECU response CAN ID, call the appropriate
`flash_on*Response()` function for each completed step. See
`examples/example_flash.can` for a fully wired-up example.

### Custom seed-to-key algorithm

Override `flash_computeKey()` in your own node to supply the OEM algorithm:

```capl
int flash_computeKey(byte seed[], int seedLen, byte key[])
{
  // Your OEM algorithm here
}
```

### Flash state constants

| Constant | Value |
|---|---|
| `FLASH_STATE_IDLE` | `0x00` |
| `FLASH_STATE_SESSION` | `0x01` |
| `FLASH_STATE_SECURITY_SEED` | `0x02` |
| `FLASH_STATE_SECURITY_KEY` | `0x03` |
| `FLASH_STATE_ERASE` | `0x04` |
| `FLASH_STATE_REQUEST_DOWNLOAD` | `0x05` |
| `FLASH_STATE_TRANSFER_DATA` | `0x06` |
| `FLASH_STATE_TRANSFER_EXIT` | `0x07` |
| `FLASH_STATE_CHECK_MEMORY` | `0x08` |
| `FLASH_STATE_ECU_RESET` | `0x09` |
| `FLASH_STATE_COMPLETE` | `0x0A` |
| `FLASH_STATE_ERROR` | `0xFF` |

---

## Common utilities (`lib/Common/utils.can`)

### Logging

```capl
logMessage(LOG_INFO, "MY_NODE", "Descriptive message");
logMessageWithCode(LOG_ERROR, "MY_NODE", "Failed", errorCode);
gLogLevel = LOG_DEBUG;   // Show all messages (default: LOG_INFO)
```

| Level constant | Value | Prefix |
|---|---|---|
| `LOG_DEBUG` | `0` | `[DBG]` |
| `LOG_INFO` | `1` | `[INF]` |
| `LOG_WARN` | `2` | `[WRN]` |
| `LOG_ERROR` | `3` | `[ERR]` |

### Result codes

| Constant | Value | Meaning |
|---|---|---|
| `RESULT_OK` | `0x00` | Success |
| `RESULT_TIMEOUT` | `0x01` | No response in time |
| `RESULT_NEGATIVE_RESP` | `0x02` | UDS negative response |
| `RESULT_INVALID_PARAM` | `0x03` | Bad argument |
| `RESULT_BUFFER_OVERFLOW` | `0x04` | Buffer too small |
| `RESULT_SEQUENCE_ERROR` | `0x05` | Wrong call order |
| `RESULT_CHECKSUM_ERROR` | `0x06` | Data integrity failure |

### Helper functions

| Function | Description |
|---|---|
| `bytesToHexString(data, len, outStr, outLen)` | Format bytes as `"AB CD EF …"` |
| `calculateChecksum(data, length)` | XOR checksum of a byte array |
| `validateBufferLength(available, required, ctx)` | Guard against buffer overflows |
| `isPositiveResponse(respSID, reqSID)` | True if `respSID == reqSID \| 0x40` |
| `isNegativeResponse(byte)` | True if byte is `0x7F` |
| `nrcToString(nrc, outStr, outLen)` | Decode a UDS NRC to text |

---

## Error handling

All library functions return `RESULT_*` codes. Errors are automatically
logged to the CANoe Write window with context tags (`[CANTP]`, `[UDS]`,
`[FLASH]`). The NRC decoder (`nrcToString`) provides human-readable UDS
negative-response descriptions.

---

## Requirements

- Vector CANoe 10.0 or later (CAPL compiler with `snprintf`, `memcpy`,
  `memcpy_off`, `waitEx`, `timeNow` support)
- Classical CAN channel (CAN FD not required; frame DLC assumed 8)
