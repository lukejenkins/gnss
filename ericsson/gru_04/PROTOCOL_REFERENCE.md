# GRU 04 Protocol Reference

Complete technical reference for PERC and PTNL commands supported by the Ericsson GRU 04 GNSS modules (GRU 04 01 and GRU 04 02), based on observed serial traffic from stock unmodified hardware.

## Table of Contents

- [Introduction](#introduction)
- [Command Format](#command-format)
- [PERC Sentences](#perc-sentences)
  - [Periodic Output](#periodic-output)
  - [Configuration Commands](#configuration-commands)
  - [Debug and Diagnostics](#debug-and-diagnostics)
- [PTNL Sentences](#ptnl-sentences)
  - [PTNL Query/Response Convention](#ptnl-queryresponse-convention)
  - [Port Configuration](#port-configuration)
  - [PPS Configuration](#pps-configuration)
  - [Sentence Mask](#sentence-mask)
  - [Other PTNL Commands](#other-ptnl-commands)
- [Standard NMEA Output](#standard-nmea-output)
- [Appendix: Checksum Calculation](#appendix-checksum-calculation)

---

## Introduction

The GRU 04 modules support two command protocols:

- **PERC (Proprietary ERicsson Commands):** Ericsson-specific extensions for timing and status
- **PTNL (Proprietary Trimble NMEA-Like):** Trimble protocol commands inherited from the ST Teseo III firmware

All sentences follow NMEA 0183 format with checksums.

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

## PERC Sentences

### Periodic Output

These sentences are emitted continuously by the module without any query.

---

#### GPppr — Pseudorange/PPS Report (1 Hz)

**Format:**
```
$PERC,GPppr,<tow_sec>,<gps_week>,<constant>,<sv_count>,<pps_flag>,0*XX
```

**Fields:**
- `tow_sec`: GPS Time of Week in seconds (0–604799), wraps at end of week
- `gps_week`: GPS week number
- `constant`: Fixed value `00050` (always)
- `sv_count`: Number of satellites used (2 digits, e.g. `08`)
- `pps_flag`: PPS quality — `3` = not locked/acquiring, `0` = PPS locked
- `0`: Fixed trailing zero

**Examples:**
```
$PERC,GPppr,588250,02403,00050,00,3,0*4C   (pre-fix, no satellites)
$PERC,GPppr,426428,02404,00050,13,0,0*46   (PPS locked)
$PERC,GPppr,493578,02404,00050,16,0,0*49   (mode 2, steady state)
```

**Notes:**
- `pps_flag` transitions 3→0 approximately 15 seconds after first fix
- Before GPS time is acquired, `tow_sec` increments from near-zero (seconds since boot)

---

#### GPsts — GPS Status (1 Hz)

**Format:**
```
$PERC,GPsts,<mode>,<survey_flag>,<constellation>,<capabilities>*XX
```

**Fields:**
- `mode`: Operating mode
  - `0` = Acquisition (searching for satellites)
  - `1` = Survey (averaging position)
  - `2` = Position hold (timing mode)
- `survey_flag`: `0` = position valid, `1` = no position
- `constellation`: Active constellation — `0` = GPS only, `2` = GPS+GLONASS
- `capabilities`: Fixed 18-character bitmask `010011111111011111` for GRU 04

**Example:**
```
$PERC,GPsts,2,0,2,010011111111011111*7B
```

**Notes:**
- Mode 2 (position hold) is the target state for timing applications
- The 18-character capabilities field identifies GRU 04 (vs 4 chars for GPS 02 01/GRU, 26 chars for GRU 05 01)

---

#### GPppf — PPS Phase and Frequency (1 Hz)

**Format:**
```
$PERC,GPppf,<phase_err>,<freq_offset>,<holdover>,<leap_sec>,<quality>*XX
```

**Fields:**
- `phase_err`: PPS phase error in nanoseconds (signed, e.g. `+04.1` or `-07.6`)
- `freq_offset`: Frequency offset in ppb (signed, e.g. `+001.2` or `-025.1`)
- `holdover`: `1` = holdover (no GPS steering), `0` = GPS-disciplined
- `leap_sec`: UTC-GPS leap second offset (18 as of 2026); empty while acquiring
- `quality`: PPS quality — `0` = locked, `1` = good, `2` = degraded

**Examples:**
```
$PERC,GPppf,-07.6,+000.0,1,,*73         (holdover, no GPS)
$PERC,GPppf,-04.1,-025.1,0,,*76         (tracking, leap sec not yet known)
$PERC,GPppf,+04.1,+001.2,0,18,0*4A     (steady state, 18 leap seconds)
```

**Notes:**
- Phase error oscillates ±7.6 ns (sawtooth) when in holdover (TCXO free-running)
- Frequency offset is zero during holdover; shows initial transients up to ±74.6 ppb after fix, settles to single digits
- `holdover` clears simultaneously with mode 0→1 transition (~37–40 s after cold start)
- Leap second field appears ~2.5 minutes after first fix

---

#### GPavp — Averaged Position (Mode 2 only)

**Format:**
```
$PERC,GPavp,<lat>,<N|S>,<lon>,<E|W>,<alt>,M*XX
```

**Fields:**
- `lat`: Averaged latitude (DDMM.MMMMMM — 6 decimal places)
- `N/S`: Hemisphere
- `lon`: Averaged longitude (DDDMM.MMMMMM — 6 decimal places)
- `E/W`: Hemisphere
- `alt`: Averaged altitude in metres
- `M`: Metres unit indicator

**Example:**
```
$PERC,GPavp,4112.908401,N,11156.174847,W,1329.5,M*XX
```

**Notes:**
- Only output in Mode 2 (position hold)
- Higher precision than standard NMEA (6 vs 4 decimal places in minutes)

---

#### GPreh — Receiver Health (~19 s)

**Format:**
```
$PERC,GPreh,<HH>:<MM>:<SS> <DD>/<MM>/<YYYY>,<status>*XX
```

**Fields:**
- `HH:MM:SS DD/MM/YYYY`: GPS time and date; `00:00:00 00/00/0000` before fix
- `status`: Health status code (hex); `8` = healthy, no time yet

**Example:**
```
$PERC,GPreh,00:00:00 00/00/0000,8*58
$PERC,GPreh,14:23:11 17/02/2026,0*XX
```

---

#### FWsts — Firmware Status (~19 s)

**Format:**
```
$PERC,FWsts,<grp1><grp2><grp3>,<state>,<substatus>,<type>*XX
```

**Fields:**
- `grp1`–`grp3`: Three 6-character status groups (each = 3 values formatted as 2-digit decimals); empty if inactive
- `state`: Firmware state (0–3)
- `substatus`: Sub-status code
- `type`: Type indicator

**Example:**
```
$PERC,FWsts,010100,,,0,0,0*7D
```

**Notes:**
- Emitted together with `GPreh` approximately every 19 seconds

---

#### GPctr — Control Heartbeat (~15 s)

**Format:**
```
$PERC,GPctr,R,,,<value>,,*XX
```

**Example:**
```
$PERC,GPctr,R,,,1,,*39
```

**Notes:**
- Periodic configuration heartbeat; `R` = read/report
- Always present in default output; must be filtered when scanning for command responses

---

#### GPver — Version (boot + on-demand)

Emitted once at boot. Also returned in response to `$PERC,GPver,Q`.

**Response format:**
```
$PERC,GPver,<model>,<part_number>,<hw_rev>,<serial>.<build>*XX
```

**Fields:**
- `model`: Model name (`GRU 04 01` or `GRU 04 02`)
- `part_number`: NCD part number (`NCD 901 65/1` or `NCD 901 78/1`)
- `hw_rev`: Hardware revision (`R1E` for GRU 04 01, `R1C` for GRU 04 02)
- `serial`: Decimal serial number
- `build`: 2-digit firmware build number

**Examples:**
```
$PERC,GPver,GRU 04 01,NCD 901 65/1,R1E,E640226882.2050*0B
$PERC,GPver,GRU 04 02,NCD 901 78/1,R1C,E640027336.1951*XX
```

---

### Configuration Commands

These commands follow a query/set/response pattern:
- `Q` = Query current value → module responds with `V,<value>`
- `S,<value>` = Set new value → module responds with `R,<value>` (accepted) or `V` (rejected)

---

#### GPssl — Survey Window (Satellite Selection Limit)

**Commands:**
```
$PERC,GPssl,Q*2E                 (query)
$PERC,GPssl,S,21600*35           (set to 6 hours — factory default)
```

**Responses:**
```
$PERC,GPssl,R,<value>*XX         (accepted)
$PERC,GPssl,V*29                 (rejected — out of range)
```

**Fields:**
- `value`: Survey window in seconds (1–86400; default 21600 = 6 hours)

**Examples:**
```
Query:    $PERC,GPssl,Q*2E
Response: $PERC,GPssl,R,2000*03

Set:      $PERC,GPssl,S,21600*35
Response: $PERC,GPssl,R,21600*34

Over-max: $PERC,GPssl,S,86401*3B
Response: $PERC,GPssl,V*29
```

---

#### GPdly — Cable Delay Compensation

**Commands:**
```
$PERC,GPdly,Q*33                 (query)
$PERC,GPdly,S,0*2D               (set to 0 ns)
```

**Responses:**
```
$PERC,GPdly,R,<delay_ns>*XX      (accepted)
$PERC,GPdly,V*XX                 (rejected)
```

**Fields:**
- `delay_ns`: Cable delay in nanoseconds (range ±100,000,000 ns = ±100 ms)

**Examples:**
```
Query:    $PERC,GPdly,Q*33
Response: $PERC,GPdly,R,0*2C
```

---

#### GPcst — Constellation Configuration

**Commands:**
```
$PERC,GPcst,Q*26                 (query)
```

**Responses:**
```
$PERC,GPcst,R,<mask>,,,<idx>,*XX
```

**Fields:**
- `mask`: Constellation bitmask (`3` = GPS+GLONASS observed on GRU 04 01/02)

**Example:**
```
Query:    $PERC,GPcst,Q*26
Response: $PERC,GPcst,R,3,,,0,*0A
```

---

#### GPbrt — Baud Rate

**Commands:**
```
$PERC,GPbrt,Q*26                 (query)
```

**Responses:**
```
$PERC,GPbrt,R,<value>*XX
```

**Fields:**
- `value`: Baud rate index (`1` = 9600 baud)

**Example:**
```
Query:    $PERC,GPbrt,Q*26
Response: $PERC,GPbrt,R,1*38
```

**Notes:**
- The baud rate table in flash is unprogrammed (all 0xFF); this command is non-functional for changing baud rate on stock hardware

---

#### GPcnf — Receiver Configuration

**Commands:**
```
$PERC,GPcnf,Q*29                 (query)
```

**Responses:**
```
$PERC,GPcnf,R,<elev_mask>,<cn0_threshold>,<field3>,<field4>*XX
```

**Example:**
```
Query:    $PERC,GPcnf,Q*29
Response: $PERC,GPcnf,R,10,10,6,1*2D
```

---

#### GPrst — Reset

**Commands:**
```
$PERC,GPrst,S,0*29               (no-op)
$PERC,GPrst,S,1*28               (warm reset — device restarts)
$PERC,GPrst,S,2*2B               (no-op variant)
```

**Responses:**
```
$PERC,GPrst,R,1*29               (accepted — warm reset will follow)
$PERC,GPrst,V*30                 (no-op acknowledged)
```

**Notes:**
- `S,1` triggers a warm restart; the module will stop transmitting briefly and re-emit boot sentences
- `S,0` and `S,2` return `V` (no action taken)

---

### Debug and Diagnostics

#### GPdbg — Satellite Tracking Debug

**Enable/disable:**
```
$PERC,GPdbg,0*42                 (disable)
$PERC,GPdbg,1*43                 (enable satellite tracking — level 1)
$PERC,GPdbg,2*40                 (enable status counter — level 2)
$PERC,GPdbg,3*41                 (enable both)
```

**Level 1 output format (satellite tracking):**
```
$PERC,GPdbg,1,<total_pages>,<page_num>,<sv1>,<sv2>,...*XX
```

**Fields (pages 1–N, satellite tracking):**
- `total_pages`: Number of pages in this epoch
- `page_num`: Current page (1-based)
- `svN`: 6-digit satellite code — first 3 digits = PRN (e.g. `006` = GPS PRN 6, `109` = GLONASS slot 9), last 3 = signal metric

**Examples:**
```
$PERC,GPdbg,1,2,1,006441,011481,012120,021481,025461,028361,109468,111438*42
$PERC,GPdbg,1,2,2,117468,119408,122468,123458,,,,*49
```

**Level 2 output format (status counter):**
```
$PERC,GPdbg,2,<counter>*XX
```

**Notes:**
- Level 1 produces ~4 messages per second (split across multiple pages as needed)
- Useful for antenna placement and real-time signal troubleshooting
- Send `$PERC,GPdbg,0*42` to return to normal output

---

## PTNL Sentences

### PTNL Query/Response Convention

PTNL commands use a consistent naming pattern:

| Prefix | Meaning | Example |
|--------|---------|---------|
| `$PTNLQxx` | Query current value | `$PTNLQPT*53` |
| `$PTNLSxx` | Set new value | `$PTNLSPT,...` |
| `$PTNLRxx` | Response from module | `$PTNLRPT,...` |

The two-character suffix identifies the subsystem: `PT` = port, `PS` = PPS, `BA` = baud/antenna, `CR` = crystal, `TF` = timing fix, `NM` = sentence mask, `VR` = version, `RT` = real-time data.

Response `R` values with data = accepted/returning value. Response `R` with just `A` = command accepted. Response `V` = rejected or no value.

---

### Port Configuration

#### PTNLQPT / PTNLSPT / PTNLRPT — Serial Port Config

**Query:**
```
$PTNLQPT*53
```

**Set:**
```
$PTNLSPT,009600,8,N,1,4,4*19
```

**Response:**
```
$PTNLRPT,009600,8,N,1,4,4*18   (current config, in response to query)
$PTNLRPT,A*3D                   (accepted, in response to set)
```

**Fields:**
- `009600`: Baud rate (zero-padded to 6 chars)
- `8`: Data bits
- `N`: Parity (N = none)
- `1`: Stop bits
- `4,4`: Port mode fields

---

### PPS Configuration

#### PTNLQPS / PTNLSPS / PTNLRPS — PPS Config

**Query:**
```
$PTNLQPS*54
```

**Set:**
```
$PTNLSPS,1,2000000,1,0,1*49
```

**Response:**
```
$PTNLRPS,1,2000000,1,0,1*48    (current config, in response to query)
$PTNLRPS,A*3A                   (accepted, in response to set)
```

**Fields:**
- `enabled`: `1` = enabled, `0` = disabled
- `width`: Pulse width in nanoseconds (2000000 = 2 ms)
- `polarity`: `1` = positive pulse on second
- `offset`: PPS offset in nanoseconds
- `cable_delay`: Cable delay compensation

**Notes:**
- PPS is hardware-generated and always active; this command configures parameters only
- The 1PPS output is on a separate RS-422 pair (pins 1–2 on the RJ45)

---

### Sentence Mask

#### PTNLSNM / PTNLRNM — Sentence Output Mask

**Set sentence mask:**
```
$PTNLSNM,<mask>,<rate>,255*XX
```

**Response (ACK):**
```
$PTNLRNM,A*3A
```

**Fields:**
- `mask`: Hexadecimal bitmask of sentences to enable (see table below)
- `rate`: Output interval in seconds (`01` = 1 Hz)
- `255`: Must always be `255` or the command is silently rejected

**Sentence bitmask:**

| Bit | Sentence | Notes |
|-----|----------|-------|
| 0 | `$GPGGA` | GPS fix data |
| 1 | `$GNGLL` | Geographic position |
| 2 | `$GPGSA` / `$GNGSA` | DOP and active satellites |
| 3 | `$GPGSV` / `$GLGSV` | Satellites in view |
| 4 | `$GNRMC` | Recommended minimum |
| 5 | `$GNVTG` | Track made good |
| 8 | `$GNZDA` | UTC date and time |
| 11 | `$GNGNS` | Multi-GNSS fix data |
| 13 | `$PTNLRXO` | Oscillator data |
| 14 | `$PTNLRBA` | Antenna/status data (periodic) |
| 16 | `$PTNLRTP` | Real-time position |

**Common configurations:**

Enable all supported sentences at 1 Hz (13 sentence types):
```
$PTNLSNM,7E83F,01,255*76
```

Factory default (5 sentence types):
```
$PTNLSNM,60019,01,255*77
```

**Notes:**
- Bit 10 (`$GPGST`) does not function on GRU 04 firmware despite appearing in documentation
- Multi-constellation sentences use `GN` talker prefix (e.g. `$GNRMC`, `$GNGSA`)
- When GLONASS is active, `$GLGSV` is emitted alongside `$GPGSV`

---

### Other PTNL Commands

#### PTNLQBA / PTNLSBA / PTNLRBA — Antenna/Baud Status

**Query:**
```
$PTNLQBA*54
```

**Set:**
```
$PTNLSBA,1,0*57
```

**Responses:**
```
$PTNLRBA,1,1*57     (status response to query)
$PTNLRBA,V*2D       (rejected, in response to set)
```

**Notes:**
- `PTNLRBA` is also emitted periodically when enabled via PTNLSNM bit 14
- Response `1,1` = enabled and active

---

#### PTNLQCR / PTNLRCR — Crystal Rate Configuration

**Query:**
```
$PTNLQCR*46
```

**Response:**
```
$PTNLRCR,10.00,10.00,06.00,0.0,0.0,7,0,0,0,03,1*74
```

**Fields:**
- Fields 1–2: Elevation mask settings (degrees)
- Field 3: CN0 threshold
- Fields 4–5: Rate parameters
- Remaining fields: Additional crystal/timing configuration

---

#### PTNLQTF / PTNLRTF — Timing Fix

**Query:**
```
$PTNLQTF*45
```

**Response:**
```
$PTNLRTF,V,<valid>,<tow>,<week>,<mode>,<lat>,<N|S>,<lon>,<E|W>,<alt>,<clk_bias>,<clk_drift>,<time_drift>,<gps_week>,<flags>*XX
```

**Fields:**
- `valid`: `A` = valid fix, `V` = invalid
- `tow`: GPS Time of Week (seconds)
- `week`: GPS week (modulo 1024 in the 3rd field)
- `mode`: Fix mode
- `lat/lon/alt`: Position
- `clk_bias`: Clock bias offset
- `clk_drift`: Clock drift
- `time_drift`: Time drift

**Example:**
```
Query:    $PTNLQTF*45
Response: $PTNLRTF,V,A,257257,13,1,4112.90908,N,11156.18030,W,01430,+000.000,+000.020,-000.024,2406,00*50
```

---

#### PTNLQRT / PTNLRRT — Real-Time Data

**Query:**
```
$PTNLQRT*51
```

**Response:**
```
$PTNLRRT,V*28
```

**Notes:**
- `V` response indicates no real-time data available or feature not configured

---

#### PTNLQVR / PTNLRVR — Version

**Query:**
```
$PTNLQVR*53
```

**Response:**
```
$PTNLRVR,V*2A
```

**Notes:**
- Use `$PERC,GPver,Q` for the full Ericsson-formatted version string (model, part number, revision, serial)

---

#### PTNLRXO — Oscillator Data (periodic, when enabled)

Enabled via `PTNLSNM` bit 13.

**Format:**
```
$PTNLRXO,<fields>*XX
```

**Notes:**
- GRU 04-specific sentence (Teseo III firmware)
- Enable with `$PTNLSNM,7E83F,01,255*76`

---

#### PTNLRTP — Real-Time Position (periodic, when enabled)

Enabled via `PTNLSNM` bit 16.

**Format:**
```
$PTNLRTP,T,<fields>*XX
```

**Notes:**
- GRU 04-specific sentence (Teseo III firmware)
- Enable with `$PTNLSNM,7E83F,01,255*76`

---

## Standard NMEA Output

Standard NMEA sentences are enabled via `PTNLSNM`. The module uses multi-constellation talker prefixes (`GN`) when GLONASS is active.

| Sentence | PTNLSNM Bit | Description |
|----------|-------------|-------------|
| `$GPGGA` | 0 | GPS fix data (time, position, quality) |
| `$GNGLL` | 1 | Geographic position (lat/lon + time) |
| `$GPGSA` / `$GNGSA` | 2 | DOP and active satellites |
| `$GPGSV` | 3 | GPS satellites in view |
| `$GLGSV` | 3 | GLONASS satellites in view (when GLONASS active) |
| `$GNRMC` | 4 | Recommended minimum navigation data |
| `$GNVTG` | 5 | Track made good and ground speed |
| `$GNZDA` | 8 | UTC date and time |
| `$GNGNS` | 11 | Multi-GNSS fix data |

**Notes:**
- The factory default mask (`60019`) enables a subset of these
- `$GPGSA` and `$GNGSA` may both be emitted in multi-constellation mode
- Standard sentences conform to NMEA 0183 v2.3 / v3.0 field definitions

---

## Appendix: Checksum Calculation

NMEA sentences use an XOR checksum for error detection.

**Algorithm:**
1. Start with the character after `$` (usually `P`)
2. XOR each character up to (but not including) `*`
3. Convert result to two-digit uppercase hexadecimal

**Python:**
```python
def nmea_checksum(sentence):
    """Calculate NMEA checksum. Pass sentence without $ or *XX."""
    checksum = 0
    for char in sentence:
        checksum ^= ord(char)
    return f"{checksum:02X}"

# Example
print(nmea_checksum("PERC,GPver,Q"))          # 3C
print(nmea_checksum("PTNLSNM,7E83F,01,255"))  # 76
```

**bash:**
```bash
nmea_checksum() {
    echo -n "$1" | awk '{
        s=0; for(i=1;i<=length($0);i++) s=xor(s,ord(substr($0,i,1)));
        printf "%02X\n", s
    }' 'function ord(c) { return sprintf("%d", c+0) }'
}
```

**Common checksums for reference:**

| Sentence | Checksum |
|----------|----------|
| `$PERC,GPver,Q` | `3C` |
| `$PERC,GPdbg,0` | `42` |
| `$PERC,GPdbg,1` | `43` |
| `$PERC,GPssl,Q` | `2E` |
| `$PERC,GPdly,Q` | `33` |
| `$PERC,GPcst,Q` | `26` |
| `$PERC,GPbrt,Q` | `26` |
| `$PERC,GPcnf,Q` | `29` |
| `$PERC,GPrst,S,1` | `28` |
| `$PTNLQPT` | `53` |
| `$PTNLQPS` | `54` |
| `$PTNLQBA` | `54` |
| `$PTNLQCR` | `46` |
| `$PTNLQTF` | `45` |
| `$PTNLQRT` | `51` |
| `$PTNLQVR` | `53` |
| `$PTNLSNM,7E83F,01,255` | `76` |
| `$PTNLSNM,60019,01,255` | `77` |

**Verification:** Checksums must be correct or the module silently ignores the command.

---

*Last updated: 2026-02-21*
*Based on observed serial traffic from GRU 04 01 (NCD 901 65/1 R1E) and GRU 04 02 (NCD 901 78/1 R1C)*
*For hardware details, see [HARDWARE_DETAILS.md](HARDWARE_DETAILS.md)*
*For getting started guide, see [README.md](README.md)*
