# GRU 04 Hardware Details

Detailed technical specifications for the Ericsson GRU 04 GNSS timing modules (NCD 901 65/1 and NCD 901 78/1).

## Table of Contents

- [Overview](#overview)
- [Main Chip](#main-chip)
- [External Components](#external-components)
- [Power Supply](#power-supply)
- [Serial Interface](#serial-interface)
- [Antenna Interface](#antenna-interface)
- [LED Indicator](#led-indicator)
- [Manufacturing Information](#manufacturing-information)
- [References](#references)

---

## Overview

The GRU 04 represents the fourth generation of Ericsson GNSS timing modules for Radio Base Station (RBS) applications. Unlike earlier generations that used separate MCU and GNSS receiver chips, the GRU 04 employs a single-chip architecture based on the ST Teseo III.

**Key features:**
- Single-chip design (no separate MCU)
- GPS + GLONASS dual-constellation support
- 1PPS output for precision timing
- RS-422 differential serial interface
- Isolated 5V antenna phantom power
- Custom Trimble-derived firmware

**Model variants:**
- **GRU 04 01 (NCD 901 65/1):** SMA antenna connector
- **GRU 04 02 (NCD 901 78/1):** QMA antenna connector

The modules are functionally identical with the same firmware and PCB. The only difference is the antenna connector type.

---

## Main Chip

### ST Teseo III (STA8089FGB)

The GRU 04 uses the STMicroelectronics STA8089FGB, a single-chip GNSS receiver with integrated processing.

**Part number:** STA8089FGB (sometimes marked as STA8089FG)

**Architecture:**
- **CPU:** ARM Cortex-R4F (real-time processor)
- **Clock speed:** Up to 200 MHz
- **RAM:** Internal SRAM for positioning engine
- **GNSS constellations:** GPS, GLONASS, Galileo, BeiDou, QZSS (firmware-dependent)
- **RF frontend:** Integrated multi-GNSS RF receiver
- **Package:** QFN package with exposed thermal pad

**Key capabilities:**
- Execute-in-place (XIP) from external flash memory
- Hardware-accelerated positioning algorithms
- Integrated 1PPS output generation
- Multiple serial interfaces (UART, SPI, I2C)
- Low power modes

**GNSS specifications:**
- **Channels:** 32+ tracking channels (multi-constellation)
- **Sensitivity:** -160 dBm tracking, -147 dBm acquisition
- **Update rate:** Up to 10 Hz position updates (firmware-dependent)
- **Time-to-first-fix (TTFF):**
  - Cold start: 29 seconds (typical)
  - Warm start: 26 seconds (typical)
  - Hot start: 1 second (typical)

**Note:** The GRU 04 firmware enables GPS + GLONASS. Other constellations are supported by the chip but not enabled in the Ericsson firmware.

**Datasheet:** [ST STA8089FG Datasheet](https://www.st.com/resource/en/datasheet/sta8089fg.pdf)

---

## External Components

### Flash Memory

**Part:** Macronix MX25L3233F
**Type:** SPI NOR Flash
**Capacity:** 4 MB (32 Mbit)
**Interface:** Quad-SPI (supports 1/2/4-bit modes)
**Purpose:** Firmware storage and configuration database (CDB)

The Teseo III executes firmware directly from flash (XIP mode). The flash contains:
- Trimble-derived firmware binary
- Configuration database (CDB) at 0x000000
- NMEA/PERC sentence format tables
- Variant-specific settings


### RS-422 Transceivers

**Receiver:** Texas Instruments DS26LV32ATM (quad RS-422 line receiver)
**Transmitter:** Texas Instruments AM26LV31 (quad RS-422 line driver)

**Interface specifications:**
- Differential voltage: ±200mV (typ), ±6V (max)
- Common-mode range: -7V to +12V
- Data rate: Up to 10 Mbps (module operates at 9600 baud)
- Protection: Built-in failsafe for open/short conditions

**Pin connections:**
- TxD (pins 4-5): NMEA output from Teseo III via AM26LV31
- RxD (pins 3-6): Command input to Teseo III via DS26LV32ATM
- 1PPS (pins 1-2): Direct from Teseo III via AM26LV31

---

## Power Supply

**Input voltage:** +12V DC ±10%
**Current consumption:**
- Typical: ~100 mA @ 12V (~1.2W)
- Maximum: ~200 mA @ 12V (~2.4W) during acquisition

**Input protection:** Reverse polarity protection via series diode

**Internal regulators:**
- 3.3V main supply for Teseo III and logic
- 5V isolated supply for antenna phantom power

**Note:** The module does NOT support Power over Ethernet (PoE). It requires a dedicated +12V DC supply.

---

## Serial Interface

**Protocol:** RS-422 differential
**Baud rate:** 9600 bps (fixed)
**Format:** 8 data bits, no parity, 1 stop bit (8N1)
**Flow control:** None

**TX (Output):**
- Default sentences: $PERC,GPsts / GPppr / GPppf (1Hz)
- Configurable via PTNLSNM command

**RX (Input):**
- Command interface for configuration
- NMEA-format commands ($PERC, $PTNL)
- Some RS-422/RS-485 adapters may require DTR/RTS control lines

**Connector:** RJ45 pins 3-6 (differential pairs)

---

## Antenna Interface

**Connector:**
- GRU 04 01: SMA female (standard GPS connector)
- GRU 04 02: QMA female (quick-connect variant)

**Frequency bands:**
- GPS L1: 1575.42 MHz
- GLONASS L1: 1602 MHz (center)

**Antenna requirements:**
- Active antenna (requires phantom power)
- Multi-GNSS capable (GPS + GLONASS L1)
- 50Ω impedance

**Phantom power:**
- Voltage: 5V DC (isolated)
- Current: Up to 200 mA (1W max via isolated DC-DC converter)
- Isolation: Provided by MuRata NXE1S0305MC DC-DC converter

**Note:** The isolated power supply prevents ground loops and improves signal integrity.

---

## LED Indicator

**Location:** Front panel, green LED

**Behavior:**
- **Blinking (~1Hz):** Acquiring satellites (Modes 0-1)
- **Steady green:** GNSS lock achieved, in position hold (Mode 2)

**Implementation:** LED is directly controlled by Teseo III firmware based on fix status.

---

## Manufacturing Information

**Production dates:**
- GRU 04 01: 2020-2024 (China, Thailand)
- GRU 04 02: 2019 (China)

**Serial number format:**
- Hexadecimal format (e.g., E640226882)
- Encoded in firmware version query response

**Build numbers:**
- Typical range: 2035-2520
- Included in $PERC,GPver response

**Observed Hardware revisions:**
- GRU 04 01: R1A through R1E
- GRU 04 02: R1C

**PCB marking:** ROA designation (exact number varies by revision)

---

## References

**ST Documentation:**
- [STA8089FG Datasheet](https://www.st.com/resource/en/datasheet/sta8089fg.pdf)
- [UM2399 - Teseo III Binary Image User Manual](https://www.st.com/resource/en/user_manual/um2399-st-teseo-iii-binary-image--user-manual-stmicroelectronics.pdf)

**Related Documentation:**
- [README.md](README.md) - Getting started guide
- [PROTOCOL_REFERENCE.md](PROTOCOL_REFERENCE.md) - Command documentation

---

*Last updated: 2026-02-16*
*Hardware verified via PCB inspection and component datasheets*
