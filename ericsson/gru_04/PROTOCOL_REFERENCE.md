# GRU 04 Protocol Reference

Complete technical reference for PERC and PTNL commands supported by the Ericsson GRU 04 GNSS modules.

## Table of Contents

- [Introduction](#introduction)
- [Command Format](#command-format)
- [PERC Commands](#perc-commands)
  - [Status and Information](#status-and-information)
  - [Configuration](#configuration)
  - [Debug and Diagnostics](#debug-and-diagnostics)
- [PTNL Commands](#ptnl-commands)
- [Appendix: Checksum Calculation](#appendix-checksum-calculation)

---

## Introduction

The GRU 04 modules support two command protocols:

- **PERC (Proprietary ERicsson Commands):** Ericsson-specific extensions for timing and status
- **PTNL (Proprietary Trimble NMEA-Like):** Trimble protocol commands inherited from the ST Teseo III firmware

All commands follow NMEA 0183 format with checksums.

---

## Command Format

**Syntax:**
```
$PREFIX,COMMAND[,PARAMETERS]*CHECKSUM\r\n
```

**Components:**
- `$` - Start delimiter
- `PREFIX` - Protocol identifier (PERC or PTNL)
- `COMMAND` - Command name
- `PARAMETERS` - Comma-separated parameters (varies by command)
- `*CHECKSUM` - Two-digit hexadecimal XOR checksum
- `\r\n` - CR+LF terminator

**Example:**
```
$PERC,GPver,Q*3C\r\n
```

**Sending commands via gpsctl:**
```bash
gpsctl -x '$PERC,GPver,Q*3C\r\n' /dev/ttyUSB0
```

---

## PERC Commands

### Status and Information

#### GPsts - GPS Status

**Format:**
```
$PERC,GPsts,<mode>,<reserved>,<sats>,<capabilities>*XX
```

**Description:** Primary status sentence, output every second by default.

**Fields:**
- `mode`: Operating mode
  - `0` = Acquisition (searching for satellites)
  - `1` = Survey (averaging position)
  - `2` = Position hold (timing mode)
- `reserved`: Reserved field (usually 0)
- `sats`: Number of satellites in use
- `capabilities`: 18-character capability bitmask (GRU 04)

**Example:**
```
$PERC,GPsts,2,0,8,010011111111011111*XX
```

**Notes:**
- Mode 2 (position hold) is the target state for timing applications
- Capabilities field identifies GRU 04 (18 characters) vs older models (4-6 characters)

---

#### GPver - Version Query

**Query format:**
```
$PERC,GPver,Q*3C
```

**Response format:**
```
$PERC,GPver,<model>,<part_number>,<hw_rev>,<serial>.<build>*XX
```

**Description:** Queries module identification and firmware version.

**Response fields:**
- `model`: Model name (e.g., "GRU 04 01" or "GRU 04 02")
- `part_number`: NCD part number (e.g., "NCD 901 65/1")
- `hw_rev`: Hardware revision (e.g., "R1E")
- `serial`: Hexadecimal serial number
- `build`: Firmware build number

**Example:**
```
Query:    $PERC,GPver,Q*3C
Response: $PERC,GPver,GRU 04 01,NCD 901 65/1,R1E,E640226882.2050*XX
```

---

#### GPppf - Position/Pulse/Frequency

**Format:**
```
$PERC,GPppf,<lat>,<lon>,<alt>,<freq_offset>,<time_offset>*XX
```

**Description:** Provides position, frequency offset, and time offset data. Output every second by default.

**Fields:**
- `lat`: Latitude in degrees (positive = North)
- `lon`: Longitude in degrees (positive = East)
- `alt`: Altitude in meters above mean sea level
- `freq_offset`: Frequency offset in parts per billion (ppb)
- `time_offset`: Time offset in nanoseconds

**Example:**
```
$PERC,GPppf,31.123456,-123.456789,1234.5,0.523,-125*XX
```

**Notes:**
- Frequency offset indicates oscillator drift from nominal
- Time offset shows deviation from GPS time

---

#### GPppr - Position/Pulse Report

**Format:**
```
$PERC,GPppr,<time>,<date>,<lat>,<ns>,<lon>,<ew>,<alt>,<quality>*XX
```

**Description:** Position and time report, similar to NMEA GPGGA but with PERC prefix.

**Fields:**
- `time`: UTC time (HHMMSS.SS)
- `date`: UTC date (DDMMYY)
- `lat`: Latitude (DDMM.MMMMMM)
- `ns`: North/South indicator (N or S)
- `lon`: Longitude (DDDMM.MMMMMM)
- `ew`: East/West indicator (E or W)
- `alt`: Altitude in meters
- `quality`: Fix quality indicator

**Example:**
```
$PERC,GPppr,183045.00,160226,4111.599260,N,11156.188869,W,1329.2,1*XX
```

---

#### GPavp - Averaged Position

**Format:**
```
$PERC,GPavp,<lat>,<ns>,<lon>,<ew>,<alt>*XX
```

**Description:** Averaged position after entering Mode 2 (position hold). Only appears after survey complete.

**Fields:**
- `lat`: Averaged latitude (DDMM.MMMMMM)
- `ns`: North/South indicator
- `lon`: Averaged longitude (DDDMM.MMMMMM)
- `ew`: East/West indicator
- `alt`: Averaged altitude in meters

**Example:**
```
$PERC,GPavp,4112.908401,N,11156.174847,W,1329.5*XX
```

**Notes:**
- Only output in Mode 2
- Represents the surveyed position used for timing hold

---

### Configuration

#### GPdbg - Debug Output Control

**Enable debug (level 1 - satellite tracking):**
```
$PERC,GPdbg,1*43
```

**Disable debug:**
```
$PERC,GPdbg,0*42
```

**Description:** Enables/disables satellite tracking debug output at 4 messages per second.

**Debug output format:**
```
$PERC,GPdbg,<page>,<data>*XX
```

Pages 1, 2, 3: Satellite tracking data (PRN, SNR, status)
Page 10: Tracking bitmask and timing data

**Example debug output:**
```
$PERC,GPdbg,1,032450001215500*XX
```

**Field encoding (pages 1-3):**
- 3 digits: PRN number (e.g., 032 = PRN 32)
- 2 digits: SNR (signal-to-noise ratio)
- 2 digits: Status code (00=solid, 01=weak, 05=acquiring)

**Notes:**
- Level 1 is satellite tracking debug
- 4 pages output per second when enabled
- Useful for antenna placement and signal troubleshooting

---

## PTNL Commands

### PTNLSNM - Sentence Mask

**Set sentence mask:**
```
$PTNLSNM,<mask>,<rate>,<reserved>*XX
```

**Description:** Configures which NMEA and PTNL sentences are output and at what rate.

**Fields:**
- `mask`: Hexadecimal bitmask of sentences to enable
- `rate`: Output interval in seconds (01 = 1Hz, 02 = every 2 seconds, etc.)
- `reserved`: Must be 255

**Common configurations:**

**Enable all standard sentences at 1Hz:**
```
$PTNLSNM,7E83F,01,255*76
```

This enables 13 sentence types:
- Bits 0-5: GPGGA, GPGLL, GPGSA, GPGSV, GPRMC, GPVTG
- Bits 8-9: GPZDA, GPGBS
- Bit 11: GNGNS (multi-GNSS)
- Bit 13-14: PTNLRXO (oscillator), PTNLRTP (timing)

**Factory default:**
```
$PTNLSNM,60019,01,255*XX
```

**Notes:**
- Third parameter MUST be 255 or command is rejected
- Maximum mask: 7E83F for GRU 04
- Bit 10 (GPGST) does not function on Teseo III firmware
- Rate values: 01-99 for seconds, 60 hex = 96 decimal seconds

---

### PTNLQPS - Query PPS Configuration

**Query format:**
```
$PTNLQPS,Q*XX
```

**Response format:**
```
$PTNLQPS,<enabled>,<width>,<polarity>,<offset>,<cable_delay>*XX
```

**Description:** Queries 1PPS output configuration.

**Response fields:**
- `enabled`: 1 = enabled, 0 = disabled
- `width`: Pulse width in microseconds (e.g., 2000000 = 2 seconds... wait, this is likely wrong - typically 100000 = 100ms)
- `polarity`: Pulse polarity (1 = positive edge aligned to second)
- `offset`: PPS offset in nanoseconds
- `cable_delay`: Cable delay compensation setting

**Example:**
```
Query:    $PTNLQPS,Q*XX
Response: $PTNLQPS,1,2000000,1,0,1*XX
```

**Notes:**
- PPS configuration is set via firmware/CDB, not modifiable via serial commands
- Query confirms PPS is enabled but doesn't control the hardware signal
- 1PPS is hardware-generated, independent of this query

---

### PTNLRXO - Oscillator Data

**Format:**
```
$PTNLRXO,<time>,<freq_offset>,<temp>,<voltage>*XX
```

**Description:** Oscillator status and environmental data. Enabled via PTNLSNM bit 14.

**Fields:**
- `time`: GPS time
- `freq_offset`: Frequency offset in ppb
- `temp`: Temperature (if available)
- `voltage`: Supply voltage (if available)

**Enable via PTNLSNM:**
```
$PTNLSNM,7E83F,01,255*76
```
(Bit 14 included in mask 7E83F)

**Notes:**
- GRU 04-specific sentence (Teseo III exclusive)
- Not available in all firmware variants

---

### PTNLRTP - Real-Time Position

**Format:**
```
$PTNLRTP,<mode>,<time>,<date>,<lat>,<ns>,<lon>,<ew>,<alt>*XX
```

**Description:** Real-time position update. Enabled via PTNLSNM bit 16.

**Enable via PTNLSNM:**
```
$PTNLSNM,7E83F,01,255*76
```
(Bit 16 included in mask 7E83F)

**Notes:**
- GRU 04-specific sentence (Teseo III exclusive)
- Similar to GPppr but with PTNL prefix

---

## Appendix: Checksum Calculation

NMEA sentences use an XOR checksum for error detection.

**Algorithm:**

1. Start with the character after `$` (usually 'P')
2. XOR each character up to (but not including) `*`
3. Convert result to two-digit hexadecimal

**Example (Python):**

```python
def nmea_checksum(sentence):
    """Calculate NMEA checksum for sentence without $ or *XX"""
    checksum = 0
    for char in sentence:
        checksum ^= ord(char)
    return f"{checksum:02X}"

# Example
sentence = "PERC,GPver,Q"
checksum = nmea_checksum(sentence)
print(f"${sentence}*{checksum}")  # $PERC,GPver,Q*3C
```

**Example (bash):**

```bash
# Calculate checksum for a sentence
sentence="PERC,GPver,Q"
checksum=$(echo -n "$sentence" | \
    awk '{
        sum=0;
        for(i=1;i<=length($0);i++) {
            sum=xor(sum,substr($0,i,1));
        }
        printf "%02X", sum
    }')
echo "\$$sentence*$checksum"
```

**Verification:**

When sending via gpsctl, the checksum must be correct or the module will ignore the command silently.

**Common checksums:**
- `$PERC,GPver,Q*3C`
- `$PERC,GPdbg,0*42`
- `$PERC,GPdbg,1*43`
- `$PTNLSNM,7E83F,01,255*76`
- `$PTNLQPS,Q*XX` (calculate XX based on your implementation)

---

*Last updated: 2026-02-16*
*For hardware details, see [HARDWARE_DETAILS.md](HARDWARE_DETAILS.md)*
*For getting started guide, see [README.md](README.md)*
