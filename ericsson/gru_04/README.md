# Ericsson GRU 04 GNSS Timing Module

The Ericsson GRU 04 is a professional-grade GNSS timing module originally designed for Ericsson Radio Base Station (RBS) equipment. It provides GPS + GLONASS dual-constellation timing with 1PPS output, making it suitable for NTP servers, frequency standards, and precision timing applications.

Built around the ST Teseo III (STA8089FGB) single-chip GNSS receiver, these modules offer professional timing capabilities at surplus equipment prices.

## Disclaimer

This information is provided without any guarantees. Anyone following this documentation does so at their own risk.

## Quick Links

- [PROTOCOL_REFERENCE.md](PROTOCOL_REFERENCE.md) - Complete PERC/PTNL command documentation
- [HARDWARE_DETAILS.md](HARDWARE_DETAILS.md) - Detailed chip specifications
- [GitHub Repository](https://github.com/lukejenkins/gnss)

---

## Quick Start

Continue reading for a complete getting started guide.

### Models Covered

This documentation applies to both GRU 04 variants:

- **GRU 04 01 (NCD 901 65/1):** SMA antenna connector
- **GRU 04 02 (NCD 901 78/1):** QMA antenna connector

The modules are functionally identical - the only hardware difference is the antenna connector type. Both use the same ST Teseo III chip and firmware.

### Hardware Overview

#### Chip & Architecture

- **GNSS Receiver:** ST Teseo III (STA8089FGB)
- **Processor:** ARM Cortex-R4F (integrated in GNSS chip)
- **Architecture:** Single-chip design - no separate MCU
- **Flash Memory:** 4MB external SPI NOR flash for firmware and configuration
- **Constellations:** GPS + GLONASS

For complete chip specifications, see [HARDWARE_DETAILS.md](HARDWARE_DETAILS.md).

#### Physical Specifications

- **Power Input:** +12V DC via RJ45 pin 7 (+12V) and pin 8 (GND)
  - **NOT 5V, NOT PoE** - requires +12V DC
  - Current draw: ~100mA typical
- **Antenna:**
  - Active GNSS antenna via SMA (GRU 04 01) or QMA (GRU 04 02)
  - 5V phantom power provided (isolated, 1W max)
- **LED Indicator:** Green LED
  - Blinking (~1Hz): Acquiring satellites
  - Steady: GNSS lock acquired
- **Serial Interface:** RS-422 differential (9600 baud, 8N1)
- **1PPS Output:** Differential output on pins 1-2

#### RJ45 Pinout

Both RJ45 connectors are functionally identical.

| Pin | Signal  | Direction | Purpose                          |
|-----|---------|-----------|----------------------------------|
| 1   | 1PPS+   | Out       | Pulse-per-second                 |
| 2   | 1PPS-   | Out       | Pulse-per-second (inverted)      |
| 3   | RxD+    | In        | Command input                    |
| 4   | TxD+    | Out       | NMEA output                      |
| 5   | TxD-    | Out       | NMEA output (inverted)           |
| 6   | RxD-    | In        | Command input (inverted)         |
| 7   | +12V    | In        | Power input                      |
| 8   | GND     | In        | Ground                           |

### What You Need

To use this module, you'll need:

- **GRU 04 module** (01 or 02 variant)
- **+12V DC power supply** (few hundred mA capacity)
- **Active GNSS antenna** (SMA for GRU 04 01, QMA for GRU 04 02)
  - Must support GPS L1 + GLONASS L1 frequencies
  - Requires 5V phantom power (provided by module)
- **Raspberry Pi** (or similar Linux single-board computer)
- **USB to RS-422 adapter**
- **RJ45 cable** or breakout adapter for connections
- **Optional:** PPS input hardware for Raspberry Pi GPIO

**Recommended antenna placement:** Outdoors or near window with clear sky view for optimal satellite reception.

### Physical Setup

**Step 1: Power connections**

Connect +12V DC power to the RJ45 connector:
- Pin 7: +12V (positive)
- Pin 8: GND (ground)

**WARNING:** Verify voltage before connecting. This module requires +12V DC, not 5V. Incorrect voltage may damage the unit.

**Step 2: Antenna connection**

Connect an active GNSS antenna to the SMA (GRU 04 01) or QMA (GRU 04 02) connector. The module provides 5V isolated phantom power to the antenna automatically.

**Step 3: Serial data connection**

Connect the RS-422 data output to your USB-RS-422 adapter:
- Pin 4 (TxD+) to RS-422 RX+
- Pin 5 (TxD-) to RS-422 RX-

For command input (optional):
- Pin 3 (RxD+) to RS-422 TX+
- Pin 6 (RxD-) to RS-422 TX-

**Step 4: PPS connection (optional)**

For precision timing applications, connect the 1PPS output:
- Pin 1 (1PPS+) to PPS input positive
- Pin 2 (1PPS-) to PPS input negative

**Verification:** After applying power, the green LED should start blinking, indicating the module is searching for satellites.

### Software Setup

The GRU 04 works with **gpsd** (GPS daemon), which provides time and position data to applications like NTP, chrony, and PTP.

**NOTE:** GRU 04 support is currently only available in the gpsd development branch. If you don't want to wait for the next major release of gpsd, build from source using the instructions below.

Repository: https://gitlab.com/gpsd/gpsd.git

#### Building and Installing gpsd

**Install build dependencies:**

```bash
sudo apt-get update
sudo apt-get install build-essential scons libncurses-dev python3-dev pps-tools
```

**Clone and build gpsd:**

```bash
# Clone the repository
git clone https://gitlab.com/gpsd/gpsd.git
cd gpsd

# ────────────────────────────────────────────────────────────
# Option 1: Use latest development version
# ────────────────────────────────────────────────────────────
git checkout master

# ────────────────────────────────────────────────────────────
# Option 2: Use a specific commit
# ────────────────────────────────────────────────────────────
git checkout 5476fa25

# ────────────────────────────────────────────────────────────
# Build and install (same for both options)
# ────────────────────────────────────────────────────────────
scons
sudo scons install
sudo scons udev-install
```

**Verify installation:**

```bash
gpsd -V    # Should show gpsd version
cgps       # Curses GPS client should be available
gpsctl     # GPS control tool should be available
```

#### Setting Up gpsd as a System Service

**Create systemd service file:**

```bash
sudo nano /etc/systemd/system/gpsd.service
```

**Service file contents:**

```ini
[Unit]
Description=GPS (Global Positioning System) Daemon
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/sbin/gpsd -N /dev/ttyUSB0
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**Note:** Adjust `/dev/ttyUSB0` to match your USB-RS-422 adapter device.

**Enable and start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable gpsd
sudo systemctl start gpsd
```

**Verify gpsd is running:**

```bash
sudo systemctl status gpsd
```

Expected output should show "active (running)".

#### Configuring the Module

Now that you've got gpsd running as a system service (see above), you can use `gpsctl` to configure the module.

**Enable additional NMEA sentences:**

```bash
gpsctl -x '$PTNLSNM,7E83F,01,255*76\r\n' /dev/ttyUSB0
```

This enables 13 additional sentence types at 1Hz, including standard NMEA sentences like GPGGA, GPGSV, and GPGSA.

**Enable satellite debug output:**

```bash
gpsctl -x '$PERC,GPdbg,1*43\r\n' /dev/ttyUSB0
```

Shows real-time satellite tracking status at 4 messages per second. Useful for antenna placement and troubleshooting.

**Query module version:**

```bash
gpsctl -x '$PERC,GPver,Q*XX\r\n' /dev/ttyUSB0
```

Returns module identification including model number (GRU 04 01 or GRU 04 02) and firmware build.

For complete command documentation, see [PROTOCOL_REFERENCE.md](PROTOCOL_REFERENCE.md).

### First Boot and Verification

**LED behavior:**

- **Blinking (~1Hz):** Module is acquiring satellites (initial state)
- **Steady green:** GNSS lock achieved, 1PPS active

**Cold start timing:**

From power-on with no previous satellite data:
- First fix: ~2-5 minutes (Mode 1)
- Position hold: ~30-40 minutes (Mode 2)

**Viewing GPS data:**

Use the `cgps` curses client to view real-time GPS data:

```bash
cgps
```

You should see position, satellite count, and time information updating.

**Default NMEA sentences:**

The module outputs these sentences by default every second:
- `$PERC,GPsts` - GPS status (mode, satellites, capabilities)
- `$PERC,GPppr` - Position/time data
- `$PERC,GPppf` - Frequency and timing information

**Verifying 1PPS signal:**

If you've connected the 1PPS output, verify the signal:
- LED should be steady (not blinking)
- Check with oscilloscope: 1Hz pulse train
- Typical pulse width: 100ms high, 900ms low

Once you see steady lock and consistent NMEA output, your module is ready for timing applications.

### Troubleshooting

**No power / LED not lighting:**
- Check 12V supply voltage with multimeter
- Verify pin 7 (+12V) and pin 8 (GND) connections
- Ensure proper polarity (pin 7 positive, pin 8 ground)

**LED blinking but never achieves lock:**
- Check antenna connection (SMA or QMA)
- Move antenna outdoors or near window with clear sky view
- Cold start can take 30-40 minutes for position hold
- Verify antenna has unobstructed view of sky

**No serial data / gpsd not seeing device:**
- Check RS-422 adapter connections (pins 4-5 for TxD)
- Verify baud rate: 9600 8N1
- Check USB device enumeration: `ls -l /dev/ttyUSB*`
- Test raw output: `cat /dev/ttyUSB0` (should see NMEA sentences)
- Check dmesg for USB adapter recognition

**gpsd running but no time sync:**
- Verify 1PPS signal present on pins 1-2
- Check gpsd is configured for PPS device
- Verify LED is steady green (not blinking)
- Check system logs: `journalctl -u gpsd`
- Confirm PPS device: `ls -l /dev/pps*`

**Module responds but shows wrong data:**
- Power cycle the module (remove 12V, wait 10 seconds, reconnect)
- Query firmware version: `gpsctl -x '$PERC,GPver,Q*XX\r\n' /dev/ttyUSB0`
- Verify you have a GRU 04 module (not a different Ericsson GPS model)

**Can't send commands / no response to transmitted commands:**
- Certain RS-485/RS-422 adapters require DTR/RTS control lines to be high for transmitting
- If you're having trouble sending commands, check your serial port configuration

### Contributing

This documentation is a work in progress and contributions are very welcome. Send comments or improvements to https://github.com/lukejenkins/gnss, or if they're gpsd related, send them directly to the gpsd project.

### References

**Links:**
- [gpsd project](https://gpsd.io)
- [gpsd GitLab repository](https://gitlab.com/gpsd/gpsd)

**Unofficial documentation:**
- [Synchronization of RBS in a network using a configurable GPS Receiver Emulator](https://web.archive.org/web/1/http://uu.diva-portal.org/smash/get/diva2:573991/FULLTEXT01.pdf) by Ashar Waseem (November 2012)
- [Osmocom RBS 6xxx Wiki](https://osmocom.org/projects/ericsson-rbs-6xxx/wiki)
- [Miller's Wiki](https://wiki.millerjs.org/)
- [This project repository](https://github.com/lukejenkins/gnss)

**Official documentation:**
- [ST STA8089FG Datasheet](https://www.st.com/resource/en/datasheet/sta8089fg.pdf)
- [UM2399 - ST Teseo III Binary Image User Manual](https://www.st.com/resource/en/user_manual/um2399-st-teseo-iii-binary-image--user-manual-stmicroelectronics.pdf)

**Additional resources:**
- [PROTOCOL_REFERENCE.md](PROTOCOL_REFERENCE.md) - Complete command documentation
- [HARDWARE_DETAILS.md](HARDWARE_DETAILS.md) - Detailed hardware specifications
