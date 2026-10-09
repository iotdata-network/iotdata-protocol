# IoT Sensor Telemetry Protocol (iotdata)

## Specification

```text
    Title:     IoT Sensor Telemetry Protocol
    Version:   0.90 (breakingly unstable, until 1.00)
    Status:    Running Code
    Created:   2026-02-07
    Authors:   Matthew Gream
    Licence:   Attribute-ShareAlike 4.0 International
               https://creativecommons.org/licenses/by-sa/4.0
    Location:  https://libiotdata.org
               https://github.com/matthewgream/libiotdata

```

## Status of This Document

This document specifies a bit-packed telemetry protocol for battery- and
transmission- constrained IoT sensor systems, with particular emphasis on
LoRa-based remote environmental monitoring, and seamlessly deployable in
point-to-point and mesh-relay topologies.

The protocol has a reference implementation in C (`libiotdata`) which is the
normative source for any ambiguity in this specification. In the tradition of
RFC 1, the specification is informed by running code.

Discussion of this document and the reference implementation takes place on the
project's GitHub repository.

## Table of Contents

- [1. Introduction](#1-introduction)
- [2. Conventions and Terminology](#2-conventions-and-terminology)
- [3. Design Principles](#3-design-principles)
- [4. Packet Structure Overview](#4-packet-structure-overview)
- [5. Header](#5-header)
- [6. Presence Bytes](#6-presence-bytes)
- [7. Variant Definitions](#7-variant-definitions)
- [8. Field Encodings](#8-field-encodings)
  - [8.1. Battery](#81-battery)
  - [8.2. Link](#82-link)
  - [8.3. Environment](#83-environment)
  - [8.4. Solar](#84-solar)
  - [8.5. Depth](#85-depth)
  - [8.6. Flags](#86-flags)
  - [8.7. Position](#87-position)
  - [8.8. Datetime](#88-datetime)
  - [8.9. Temperature (standalone)](#89-temperature-standalone)
  - [8.10. Pressure (standalone)](#810-pressure-standalone)
  - [8.11. Humidity (standalone)](#811-humidity-standalone)
  - [8.12. Wind (bundle)](#812-wind-bundle)
  - [8.13. Wind Speed (standalone)](#813-wind-speed-standalone)
  - [8.14. Wind Direction (standalone)](#814-wind-direction-standalone)
  - [8.15. Wind Gust (standalone)](#815-wind-gust-standalone)
  - [8.16. Rain (bundle)](#816-rain-bundle)
  - [8.17. Rain Rate (standalone)](#817-rain-rate-standalone)
  - [8.18. Rain Size (standalone)](#818-rain-size-standalone)
  - [8.19. Air Quality (bundle)](#819-air-quality-bundle)
  - [8.20. Air Quality Index (standalone)](#820-air-quality-index-standalone)
  - [8.21. Air Quality PM (standalone)](#821-air-quality-pm-standalone)
  - [8.22. Air Quality Gas (standalone)](#822-air-quality-gas-standalone)
  - [8.23. Radiation (bundle)](#823-radiation-bundle)
  - [8.24. Radiation CPM (standalone)](#824-radiation-cpm-standalone)
  - [8.25. Radiation Dose (standalone)](#825-radiation-dose-standalone)
  - [8.26. Clouds](#826-clouds)
  - [8.27. Image](#827-image)
- [9. TLV Data](#9-tlv-data)
  - [9.1. TLV Header](#91-tlv-header)
  - [9.2. Raw Format](#92-raw-format)
  - [9.3. Packed String Format](#93-packed-string-format)
  - [9.4. TLV Types](#94-tlv-types)
  - [9.5. Global TLV Types](#95-global-tlv-types)
- [10. Canonical JSON Representation](#10-canonical-json-representation)
- [11. Receiver Considerations](#11-receiver-considerations)
  - [11.1. Datetime Year Resolution](#111-datetime-year-resolution)
  - [11.2. Position Source Ambiguity](#112-position-source-ambiguity)
  - [11.3. Sensor Metadata and Interoperability](#113-sensor-metadata-and-interoperability)
  - [11.4. Unknown Variants](#114-unknown-variants)
  - [11.5. Quantisation Error Budgets](#115-quantisation-error-budgets)
  - [11.6. Error Handling and Malformed Packets](#116-error-handling-and-malformed-packets)
- [12. Packet Size Reference](#12-packet-size-reference)
- [13. Implementation Notes](#13-implementation-notes)
  - [13.1. Reference Implementation](#131-reference-implementation)
  - [13.2. Encoder Strategy](#132-encoder-strategy)
  - [13.3. Compile-Time Options](#133-compile-time-options)
  - [13.4. Build Size and Stack Usage](#134-build-size-and-stack-usage)
  - [13.5. Variant Table Extension](#135-variant-table-extension)
- [14. Security Considerations](#14-security-considerations)
- [15. Future Work](#15-future-work)
- [16. Versioning and Forward Compatibility](#16-versioning-and-forward-compatibility)
- [Appendix A. 6-Bit Character Table](#appendix-a-6-bit-character-table)
- [Appendix B. Quantisation Worked Examples](#appendix-b-quantisation-worked-examples)
- [Appendix C. Complete Encoder Example](#appendix-c-complete-encoder-example)
- [Appendix D. Transmission Medium Considerations](#appendix-d-transmission-medium-considerations)
- [Appendix E. System Implementation Considerations](#appendix-e-system-implementation-considerations)
- [Appendix F. Example Weather Station Output](#appendix-f-example-weather-station-output)
- [Appendix G. Mesh Protocol](#appendix-g-mesh-protocol)
- [Appendix H. System Architecture Considerations](#appendix-h-system-architecture-considerations)
- [Appendix I. Comparison with Alternative Encodings and Embedded Libraries](#appendix-i-comparison-with-alternative-encodings-and-embedded-libraries)
- [Appendix J. Known Limitations and Open Issues](#appendix-j-known-limitations-and-open-issues)

---

## 1. Introduction

Remote environmental monitoring systems — weather stations, snow depth sensors,
ice thickness gauges — are frequently deployed in locations without mains power
or wired connectivity. These devices are constrained along three axes
simultaneously:

1. **Power** — battery and/or small solar panel, particularly in locations with
   limited winter daylight.

2. **Communications** — LoRa, 802.11ah, SigFox, cellular SMS, low-frequency RF,
   or similar low-power point-to-point, wide-area, or mesh networks with
   effective payload limits of tens of bytes per transmission. Regulatory limits
   on transmission time (typically 1% duty cycle in EU ISM bands) mean that
   every byte transmitted has a direct cost in time-on-air and energy.

3. **Compute** — small, inexpensive embedded microcontrollers running at tens of
   megahertz with tens or hundreds of kilobytes of RAM and program storage,
   where code size and complexity are real constraints, and where there are no,
   or limited, operating system or protocol support.

Existing serialisation approaches — JSON, Protobuf, CBOR, even raw C structs —
waste bits on byte alignment, field delimiters, schema metadata, or fixed-width
fields for data that could be represented in far fewer bits.

The IoT Sensor Telemetry Protocol (iotdata) addresses this by defining a
bit-packed wire format where each field is quantised to the minimum number of
bits required for its operational range and resolution. A typical weather
station packet — battery, link quality, temperature, pressure, humidity, wind
speed, direction and gust, rain rate and drop size, solar irradiance and UV
index, plus 8 flag bits — fits in 16 bytes. A full-featured packet adding air
quality, cloud cover, radiation CPM and dose, position latitude and longitude,
and timestamp fits in 32 bytes.

The protocol is designed for transmit-only devices. There is no negotiation,
handshake, or acknowledgement at this layer. A sensor wakes, encodes its
readings, transmits, and sleeps. Transmissions are typically infrequent (minutes
to hours), bursty, and rely on lower-layer integrity (checksums or CRC) without
lower-layer reliability (retransmission or acknowledgement).

The protocol can be deployed in point-to-point arrangements, where edge devices
transmit directly to one or more gateways or in a mesh arrangement, where
intermediate relays automatically and periodically (re)configure to determine
primary and backup paths to gateways. Edge devices need no awareness of the mesh
protocol and can operate identically with or without it.

## 2. Conventions and Terminology

[moved]

## 3. Design Principles

The following principles guided the protocol design:

1. **Bit efficiency over simplicity.** Every field is quantised to the minimum
   bits that preserve operationally useful resolution. There is no
   byte-alignment padding between fields.

2. **Presence flags over fixed structure.** Optional fields are indicated by
   presence bits, so a battery-only packet is 6 bytes and a full-telemetry
   packet is 24 bytes — the same protocol serves both.

3. **Variants over negotiation.** Different sensor types (snow, ice, weather)
   can prioritise different fields in the compact first presence byte. The 4-bit
   variant field in the header selects the field mapping. No runtime negotiation
   is needed.

4. **Source-agnostic fields.** Position may come from GNSS, WiFi geolocation,
   cell tower triangulation, or static configuration. Datetime may come from
   GNSS, NTP, or a local RTC. The wire encoding is the same regardless of
   source.

5. **Extensibility via TLV.** Diagnostic data, firmware metadata, and
   user-defined payloads use a trailing TLV (type-length-value) section that
   does not affect the fixed field layout. These are typically designed to be
   system data, rather than sensor data.

6. **Encode-only on the sensor.** The encoder is small enough for
   resource-constrained MCUs. JSON serialisation and other server-side features
   are optional and can be excluded from embedded builds. The reference
   implementation can build to 1 KB and non-reference implementations to less
   than 512 bytes.

7. **Transport-delegated integrity.** The protocol carries no checksum, CRC,
   length field, or encryption. These functions are delegated to the underlying
   medium (LoRa CRC, LoRaWAN MIC, cellular security, etc.). A redundant CRC
   would cost 16-32 bits — significant when the entire payload may be 46 bits.
   Packet loss is tolerated: the sequence number (Section 5) enables detection
   without requiring retransmission.

8. **No global interoperability.** It is expressly not a goal to support
   interoperability between implementations, e.g. between vendors. Rather, the
   design intends to provide an optimal framework and reference for a given
   deployment across a suite of devices. Interoperability may be a goal for
   future versions.

## 4. Packet Structure Overview

[moved]

## 5. Header

[moved]

## 6. Presence Bytes

[moved]

### Presence Byte 0 (always present)

[moved]

### Presence Byte N (N ≥ 1, conditional)

[moved]

### Field Capacity

[moved]

### Extension Byte Optimisation

[moved]

### Field Ordering

[moved]

## 7. Variant Definitions

[moved]

### Variant Table Structure

[moved]

### Default Variant: Weather Station

[moved]

### Custom Variant Maps

[moved]

### Registered Variants

[moved]

## 8. Field Encodings

[moved]

### 8.1. Battery

[moved]

### 8.2. Link

[moved]

### 8.3. Environment

[moved]

### 8.4. Solar

[moved]

### 8.5. Depth

[moved]

### 8.6. Flags

[moved]

### 8.7. Position

[moved]

### 8.8. Datetime

[moved]

### 8.9. Temperature (standalone)

[moved]

### 8.10. Pressure (standalone)

[moved]

### 8.11. Humidity (standalone)

[moved]

### 8.12. Wind (bundle)

[moved]

### 8.13. Wind Speed (standalone)

[moved]

### 8.14. Wind Direction (standalone)

[moved]

### 8.15. Wind Gust (standalone)

[moved]

### 8.16. Rain (bundle)

[moved]

### 8.17. Rain Rate (standalone)

[moved]

### 8.18. Rain Size (standalone)

[moved]

### 8.19. Air Quality (bundle)

[moved]

### 8.20. Air Quality Index (standalone)

[moved]

### 8.21. Air Quality PM (standalone)

[moved]

### 8.22. Air Quality Gas (standalone)

[moved]

### 8.23. Radiation (bundle)

[moved]

### 8.24. Radiation CPM (standalone)

[moved]

### 8.25. Radiation Dose (standalone)

[moved]

### 8.26. Clouds

[moved]

### 8.27. Image

[moved]

#### Design Philosophy

[moved]

#### Variable-Length Decoding

[moved]

#### 8.27.1. RLE Compression

[moved]

#### 8.27.2. Heatshrink Compression

[moved]

#### 8.27.3. Payload Budget

[moved]

#### 8.27.4. Recommended Practices

[moved]

#### 8.27.5. JSON Representation

[moved]

## 9. TLV Data

[moved]

### 9.1. TLV Header

[moved]

### 9.2. Raw Format

[moved]

### 9.3. Packed String Format

[moved]

### 9.4. TLV Types

[moved]

### 9.5. Global TLV Types

[moved]

#### 9.5.1. Version (0x01)

[moved]

#### 9.5.2. Status (0x02)

[moved]

#### 9.5.3. Health (0x03)

[moved]

#### 9.5.4. Config (0x04)

[moved]

#### 9.5.5. Diagnostic (0x05)

[moved]

#### 9.5.6. Userdata (0x06)

[moved]

## 10. Canonical JSON Representation

[moved]

### Example JSON (Variant 0, weather_station)

[moved]

### TLV in JSON

[moved]

#### Globally Defined Types

[moved]

#### Unrecognised and Proprietary Types

[moved]

#### Format Field Summary

[moved]

### Round-Trip Guarantee

[moved]

## 11. Receiver Considerations

This section describes algorithms and considerations that apply to the receiving
side (gateway, server, or any device decoding packets).

### 11.1. Datetime Year Resolution

The datetime field encodes seconds from the start of the current year but does
not transmit the year. The receiver MUST resolve the year using the following
algorithm:

1. Let `T_rx` be the receiver's current UTC time.
2. Let `Y` be the year component of `T_rx`.
3. Decode the datetime field to obtain `S` seconds from year start.
4. Compute `T_decoded = Y-01-01T00:00:00Z + S seconds`.
5. If `T_decoded` is more than 6 months in the future relative to `T_rx`,
   subtract one year: `T_decoded = (Y-1)-01-01T00:00:00Z + S`.

This handles the year boundary: a packet timestamped December 31 and received
January 1 is correctly attributed to the previous year.

The 24-bit field at 5-second resolution supports approximately 971 days, so the
encoding does not wrap within a single year.

The accuracy of the decoded timestamp depends on the accuracy of the
transmitter's time source. For GNSS-synchronised devices this is typically
sub-second; for free-running RTC devices it may drift by seconds per day. See
Section 11.3.

### 11.2. Position Source Ambiguity

The position field (Section 8.7) encodes latitude and longitude without
indicating the source. The practical accuracy varies significantly by source:

- **GNSS (GPS/Galileo/GLONASS):** typically 2-5 metre accuracy, well within the
  ~1.2m quantisation of the 24-bit encoding.

- **WiFi geolocation:** typically 15-50 metre accuracy. The quantisation error
  is negligible relative to the source error.

- **Cell tower:** typically 100-1000 metre accuracy.

- **Static configuration:** the operator programmes the known coordinates at
  deployment time. Accuracy depends on the method used (surveyed, map click,
  etc.).

In a closed system where the operator controls all devices and knows their
position sources, this ambiguity is acceptable. For open or interoperable
systems, the source and accuracy SHOULD be communicated via a sensor metadata
TLV (Section 11.3).

Similarly, for fixed-position sensors, the position field is typically
transmitted once at startup or periodically at a low rate, not on every packet.
The receiver SHOULD cache the last known position for a station and associate it
with subsequent packets that omit position.

### 11.3. Sensor Metadata and Interoperability

The core protocol deliberately omits sensor metadata such as:

- Sensor type (NTC thermistor, BME280, SHT40, etc.)
- Measurement accuracy or precision class
- Position source (GNSS, static config, etc.)
- Time source (GNSS, NTP, free-running RTC, etc.)
- Calibration date or coefficients

In a **closed system** — where one operator controls all devices and the gateway
software — this information is known out-of-band. The operator knows that
station 42 uses a BME280 for environment readings with ±1°C accuracy, has a
static position programmed at deployment, and synchronises time via NTP. No wire
overhead is needed.

For **interoperable systems** — where devices from different manufacturers or
deployments share a common receiver — sensor metadata becomes important. The
protocol reserves TLV types 0x10-0x1F for future standardised metadata TLVs that
could convey:

- Source type per field (e.g. "position source = GNSS" or "temperature sensor =
  BME280")
- Accuracy class or error bounds
- Calibration metadata

The design of these metadata TLVs is deferred to a future revision of this
specification. Implementers requiring interoperability before that revision MAY
use application-defined TLV types (0x20-) for this purpose, with the
understanding that these are not standardised.

This approach follows a deliberate design philosophy: add wire overhead only
when it is needed. A snow depth sensor transmitting to its own gateway every 15
minutes on a coin cell battery should not pay the cost of metadata bytes that
the receiver already knows.

### 11.4. Unknown Variants

[moved]

### 11.5. Quantisation Error Budgets

Receivers should be aware of the quantisation errors inherent in the encoding.
These are systematic and deterministic — not noise — and SHOULD be accounted for
in any downstream processing.

| Field            | Bits | Range           | Resolution   | Max quant error |
| ---------------- | ---- | --------------- | ------------ | --------------- |
| Battery level    | 5    | 0-100%          | ~3.23%       | ±1.6%           |
| Link RSSI        | 4    | -120 to -60 dBm | 4 dBm        | ±2 dBm          |
| Link SNR         | 2    | -20 to +10 dB   | 10 dB        | ±5 dB           |
| Temperature      | 9    | -40 to +80°C    | 0.25°C       | ±0.125°C        |
| Pressure         | 8    | 850-1105 hPa    | 1 hPa        | ±0.5 hPa        |
| Humidity         | 7    | 0-100%          | 1%           | ±0.5%           |
| Wind speed       | 7    | 0-63.5 m/s      | 0.5 m/s      | ±0.25 m/s       |
| Wind direction   | 8    | 0-355°          | ~1.41°       | ±0.7°           |
| Wind gust        | 7    | 0-63.5 m/s      | 0.5 m/s      | ±0.25 m/s       |
| Rain rate        | 8    | 0-255 mm/hr     | 1 mm/hr      | ±0.5 mm/hr      |
| Rain size        | 4    | 0-6.0 mm/d      | 0.25 mm/d    | ±0.5 mm/d       |
| Solar Irradiance | 10   | 0-1023 W/m²     | 1 W/m²       | ±0.5 W/m²       |
| Solar UV Index   | 4    | 0-15            | 1            | ±0.5            |
| Clouds           | 4    | 0-8 okta        | 1 okta       | ±0.5 okta       |
| AQ Index         | 9    | 0-500 AQI       | 1 AQI        | ±0.5 AQI        |
| AQ PM channels   | 8    | 0-1275 µg/m³    | 5 µg/m³      | ±2.5 µg/m³      |
| AQ Gas VOC idx   | 8    | 0-510           | 2 idx pts    | ±1 idx pt       |
| AQ Gas NOx idx   | 8    | 0-510           | 2 idx pts    | ±1 idx pt       |
| AQ Gas CO₂       | 10   | 0-51,150 ppm    | 50 ppm       | ±25 ppm         |
| AQ Gas CO        | 10   | 0-1,023 ppm     | 1 ppm        | ±0.5 ppm        |
| AQ Gas HCHO      | 10   | 0-5,115 ppb     | 5 ppb        | ±2.5 ppb        |
| AQ Gas O₃        | 10   | 0-1,023 ppb     | 1 ppb        | ±0.5 ppb        |
| Radiation CPM    | 16   | 0-65535 CPM     | 1 CPM        | ±0.5 CPM        |
| Radiation dose   | 14   | 0-163.83 µSv/h  | 0.01 µSv/h   | ±0.005 µSv/h    |
| Depth            | 10   | 0-1023 cm       | 1 cm         | ±0.5 cm         |
| Latitude         | 24   | -90° to +90°    | ~0.00001073° | ~0.6 m          |
| Longitude        | 24   | -180° to +180°  | ~0.00002146° | ~1.2 m (eq)     |
| Datetime         | 24   | 0-83.9M seconds | 5 s          | ±2.5 s          |

These quantisation errors are generally smaller than the measurement uncertainty
of the sensors themselves. For example, a typical BME280 temperature sensor has
±1°C accuracy, well above the 0.125°C quantisation error.

#### Boundary Conditions

The encode formulae in Section 8 define behaviour for values within the stated
range of each field. The protocol does not mandate a specific behaviour for
out-of-range inputs (e.g. a temperature of -45°C when the defined range is -40°C
to +80°C, or a wind speed of 70 m/s when the maximum is 63.5 m/s).

Implementations SHOULD adopt one of the following strategies, applied
consistently across all field types:

1. **Clamp to range.** Values below the minimum are encoded as the minimum;
   values above the maximum are encoded as the maximum. This preserves the
   invariant that every input produces a valid encoded value. Clamped values are
   silently distorted — the receiver cannot distinguish a clamped reading from a
   genuine reading at the boundary.

2. **Reject and omit.** The encoder refuses to encode the field and clears its
   presence bit. The receiver sees the field as absent rather than as a
   potentially misleading value. This is appropriate for safety-critical
   deployments where an out-of-range reading may indicate a sensor fault.

3. **Clamp and flag.** As (1), but the encoder also sets a deployment-specific
   flag bit (Section 8.6) or emits a DIAGNOSTIC TLV indicating the condition.

The reference implementation uses strategy (1): all values are clamped to the
representable range with no indication to the receiver. Deployments that require
out-of-range detection SHOULD use the Flags field or a DIAGNOSTIC TLV to
communicate the condition.

For fields with a defined "not available" sentinel (e.g. CPU Temperature = 0x7F
in the Health TLV), the sentinel MUST NOT be produced by clamping. If the
clamped value would collide with the sentinel, the encoder MUST use strategy (2)
instead.

## 11.6. Error Handling and Malformed Packets

[moved]

### General Principle

[moved]

### Specific Conditions

[moved]

### Conditions That SHOULD NOT Cause Discard

[moved]

### Implementation Guidance

[moved]

## 12. Packet Size Reference

[moved]

## 13. Implementation Notes

### 13.1. Reference Implementation

The reference implementation (`libiotdata`) is written in C11 and targets both
embedded systems (ESP32-C3, STM32, nRF52, Raspberry Pi) and Linux
gateways/servers. It consists of:

- `iotdata.h` — Public API, constants, and type definitions.
- `iotdata.c` — Encoder, decoder, JSON, print, and dump.
- `tests/test_default.c` — Test suite for the default variant.
- `tests/test_custom.c` — Test suite for custom variant maps.
- `tests/test_failures.c` — Test suite for failure modes.
- `tests/test_version.c` — Test smoke evaluation for build versions.
- `tests/test_example.c` — Test example for a periodic weather station.
- `Makefile` — Builds `libiotdata.a` static library and tests.

Build:

```bash
make                # Build library and both test suites
make tests          # Build and run default and custom tests
make test-example   # Build and run example test
make test-versions  # Build and run versions tests
make lib            # Build static library only
make minimal        # Measure minimal encoder-only build
```

Dependencies: C11 compiler, `libm`, and `cJSON` (optional, only required for
JSON serialisation).

### 13.2. Encoder Strategy

The encoder uses a "store then pack" strategy:

1. `iotdata_encode_begin()` initialises the context with header values and a
   buffer pointer.
2. `iotdata_encode_battery()`, `iotdata_encode_environment()`, etc. validate
   inputs and store native-typed values in the context. Fields may be added in
   any order.
3. `iotdata_encode_end()` performs all bit-packing in a single pass, consulting
   the variant field table to determine field order and presence byte layout.

This separation means that the encoder validates eagerly (at the `encode_*()`
call site where the developer can handle the error) and packs lazily (in one
pass, knowing the complete field set).

```c
/* Example: encode a weather station packet */
#define IOTDATA_VARIANT_MAPS_DEFAULT
#include "iotdata.h"

iotdata_encoder_t enc;
uint8_t buf[64];
size_t len;

iotdata_encode_begin(&enc, buf, sizeof(buf), 0, 42, seq++);
iotdata_encode_battery(&enc, 84, false);
iotdata_encode_link(&enc, -95, 8.5f);
iotdata_encode_environment(&enc, 21.5f, 1013, 45);
iotdata_encode_wind(&enc, 5.2f, 180.0f, 8.7f);
iotdata_encode_rain(&enc, 3, 15); // x10 units
iotdata_encode_solar(&enc, 850, 7);
iotdata_encode_end(&enc, &len);
/* buf[0..len-1] is now a 15-byte packet */
```

### 13.3. Compile-Time Options

The library supports extensive compile-time configuration to minimise code size
and memory usage on constrained targets.

**Variant selection:**

| Define                           | Effect                                  |
| -------------------------------- | --------------------------------------- |
| `IOTDATA_VARIANT_MAPS_DEFAULT`   | Enable built-in weather station variant |
| `IOTDATA_VARIANT_MAPS=<sym>`     | Use custom variant map array            |
| `IOTDATA_VARIANT_MAPS_COUNT=<n>` | Number of entries in custom map         |

**Field support compilation:**

| Define                          | Effect                                         |
| ------------------------------- | ---------------------------------------------- |
| `IOTDATA_ENABLE_SELECTIVE`      | Only compile elements explicitly enabled below |
| `IOTDATA_ENABLE_BATTERY`        | Compile battery field                          |
| `IOTDATA_ENABLE_LINK`           | Compile link field                             |
| `IOTDATA_ENABLE_ENVIRONMENT`    | Compile environment bundle                     |
| `IOTDATA_ENABLE_TEMPERATURE`    | Compile temperature field                      |
| `IOTDATA_ENABLE_PRESSURE`       | Compile pressure field                         |
| `IOTDATA_ENABLE_HUMIDITY`       | Compile humidity field                         |
| `IOTDATA_ENABLE_WIND`           | Compile wind bundle                            |
| `IOTDATA_ENABLE_WIND_SPEED`     | Compile wind speed field                       |
| `IOTDATA_ENABLE_WIND_DIR`       | Compile wind direction field                   |
| `IOTDATA_ENABLE_WIND_GUST`      | Compile wind gust field                        |
| `IOTDATA_ENABLE_RAIN`           | Compile rain bundle                            |
| `IOTDATA_ENABLE_RAIN_RATE`      | Compile rain rate field                        |
| `IOTDATA_ENABLE_RAIN_SIZE`      | Compile rain size field                        |
| `IOTDATA_ENABLE_SOLAR`          | Compile solar field                            |
| `IOTDATA_ENABLE_CLOUDS`         | Compile clouds field                           |
| `IOTDATA_ENABLE_AIR_QUALITY`    | Compile air quality field                      |
| `IOTDATA_ENABLE_RADIATION`      | Compile radiation bundle                       |
| `IOTDATA_ENABLE_RADIATION_CPM`  | Compile radiation CPM field                    |
| `IOTDATA_ENABLE_RADIATION_DOSE` | Compile radiation dose field                   |
| `IOTDATA_ENABLE_DEPTH`          | Compile depth field                            |
| `IOTDATA_ENABLE_POSITION`       | Compile position field                         |
| `IOTDATA_ENABLE_DATETIME`       | Compile datetime field                         |
| `IOTDATA_ENABLE_FLAGS`          | Compile flags field                            |
| `IOTDATA_ENABLE_TLV`            | Compile TLV fields                             |

When `IOTDATA_ENABLE_SELECTIVE` is defined, only the element types with their
corresponding `IOTDATA_ENABLE_xxx` defined will be compiled. When
`IOTDATA_VARIANT_MAPS_DEFAULT` is defined (without `IOTDATA_ENABLE_SELECTIVE`),
all elements used by the default weather station variant are automatically
enabled. When neither is defined, all elements are compiled.

In particular, avoidance of the TLV element will save considerable footprint.

**Functional subsetting:**

| Define                     | Effect                                                           |
| -------------------------- | ---------------------------------------------------------------- |
| `IOTDATA_NO_DECODE`        | Exclude decoder functions (also excludes print and JSON encoder) |
| `IOTDATA_NO_ENCODE`        | Exclude encoder functions (also excludes JSON decoder)           |
| `IOTDATA_NO_PRINT`         | Exclude print functions                                          |
| `IOTDATA_NO_DUMP`          | Exclude dump functions                                           |
| `IOTDATA_NO_JSON`          | Exclude JSON functions                                           |
| `IOTDATA_NO_TLV_SPECIFIC`  | Exclude TLV specific type handling                               |
| `IOTDATA_NO_CHECKS_STATE`  | Exclude state checking logic                                     |
| `IOTDATA_NO_CHECKS_TYPES`  | Exclude type checking logic                                      |
| `IOTDATA_NO_ERROR_STRINGS` | Exclude error strings (and iotdata_strerror)                     |

These allow building an encoder-only image for a sensor node (smallest possible
footprint) or a decoder-only image for a gateway.

The JSON encoding functions have a dependency on the decoder (decode from wire
format and encode into JSON), and JSON decoding functions equivalently are
dependent on the encoder (decode from JSON and encode into wire format). The
print functions, for brevity, are also dependant upon the decoder. The dump
functions work directly upon the the wire format buffer are are not dependent on
either the encoder or decoder.

Be aware that `IOTDATA_NO_CHECKS_STATE` will cease verification of non null
`iotdata_encoder_t*` and the ordering of the encoding calls (i.e. that `begin`
must be first, followed by individual `encode_` functions before a final `end`.
This is moderately safe, and acceptable to turn on during development and off
for production. It will also turn off null checks for buffers passed into the
`dump`, `print` and `json` function.s

`IOTDATA_NO_CHECKS_TYPES` will cease verification of type boundaries on calls to
`encode_` functions, for example that temperatures passed are between
quantisable minimum and maximum values. This is less safe, but results only in
bad data (and badly quantised data) passed over the wire: this may fail to
interpret bad data obtained from sensors. This option will turn off length
checking in TLV encoded strings (and worst case, truncate them) as well as TLV
encoded string validity (and worst case, transmit these as spaces).

Unless there are considerable space constraints, such as on Class 1
microcontrollers (Appendix E), it is not recommended to engage either of the
`NO_CHECKS` options.

**Floating-point control:**

| Define                        | Effect                                           |
| ----------------------------- | ------------------------------------------------ |
| (default)                     | `double` for position, `float` for other fields  |
| `IOTDATA_NO_FLOATING_DOUBLES` | Use `float` instead of `double` everywhere       |
| `IOTDATA_NO_FLOATING`         | Integer-only mode: all values as scaled integers |

In integer-only mode (`IOTDATA_NO_FLOATING`), temperature is passed as
degrees×100 (e.g. 2250 for 22.50°C), wind speed as m/s×100, radiation dose as
µSv/h×100, position as degrees×10^7, and SNR as dB×10. This eliminates all
floating-point dependencies. Future implementations SHOULD utilise this
multiple-of-ten approach.

#### Test targets

The `test-versions` target will build each of versions across the Functional
subsetting and Floating-point control, including a combined `NO_JSON` and
`NO_FLOATING` version. This is intended as a build smoke test to verify
compilation control paths. Note that the combined version will, on x86
platforms, force the compiler to reject floating-point operations, so as to
ensure they are not latent in the implementation.

### 13.4. Build Size and Stack Usage

The `minimal` and `minimal-esp32` targets yield object files for the purpose of
establishing minimal build sizes (with a comparison to full build sizes) using
the host (`minimal`) or cross-compiler (`minimal-esp32`) tools.

#### Build summary for x86-64, aarch64 and esp32-c3 systems

The following measurements are from GCC on x86-64, aarch64 and ESP32-C3 using
the `minimal` build target. With space optimisation, the minimal implementation
is less than 1KB on the embedded target.

| Configuration                                       | x86-64 -O6 | x86-64 -Os | aarch64 -O6 | aarch64 -Os | esp32-c3 -O6 | esp32-c3 -Os |
| --------------------------------------------------- | ---------- | ---------- | ----------- | ----------- | ------------ | ------------ |
| Full library (all elements, encode + decode + JSON) | ~85 KB     | ~29 KB     | ~87 KB      | ~31 KB      | ~67 KB       | ~19 KB       |
| Encoder-only, battery + environment only            | ~5.5 KB    | ~1.1 KB    | ~5.4 KB     | ~1.1 KB     | ~5.0 KB      | ~0.7 KB      |

#### Build output for x86-64 (-O6 and -Os)

```text
--- Full library ---
gcc -Wall -Wextra -Wpedantic -Werror -Wcast-align -Wcast-qual -Wstrict-prototypes -Wold-style-definition -Wcast-align -Wcast-qual -Wconversion -Wfloat-equal -Wformat=2 -Wformat-security -Winit-self -Wjump-misses-init -Wlogical-op -Wmissing-include-dirs -Wnested-externs -Wpointer-arith -Wredundant-decls -Wshadow -Wstrict-overflow=2 -Wswitch-default -Wundef -Wunreachable-code -Wunused -Wwrite-strings  -O6 -DIOTDATA_VARIANT_MAPS_DEFAULT -c iotdata.c -o iotdata_full.o
   text    data     bss     dec     hex filename
  85866    2112    4096   92074   167aa iotdata_full.o
--- Minimal encoder (battery + environment, integer-only) ---
gcc -Wall -Wextra -Wpedantic -Werror -Wcast-align -Wcast-qual -Wstrict-prototypes -Wold-style-definition -Wcast-align -Wcast-qual -Wconversion -Wfloat-equal -Wformat=2 -Wformat-security -Winit-self -Wjump-misses-init -Wlogical-op -Wmissing-include-dirs -Wnested-externs -Wpointer-arith -Wredundant-decls -Wshadow -Wstrict-overflow=2 -Wswitch-default -Wundef -Wunreachable-code -Wunused -Wwrite-strings  -O6 -mno-sse -mno-mmx -mno-80387 \
        -DIOTDATA_NO_DECODE \
        -DIOTDATA_ENABLE_SELECTIVE -DIOTDATA_ENABLE_BATTERY -DIOTDATA_ENABLE_ENVIRONMENT \
        -DIOTDATA_NO_JSON -DIOTDATA_NO_DUMP -DIOTDATA_NO_PRINT \
        -DIOTDATA_NO_FLOATING -DIOTDATA_NO_ERROR_STRINGS -DIOTDATA_NO_CHECKS_STATE -DIOTDATA_NO_CHECKS_TYPES \
        -c iotdata.c -o iotdata_minimal.o
Minimal object size:
   text    data     bss     dec     hex filename
   5559      32       0    5591    15d7 iotdata_minimal.o
0000000000000000 l     O .data.rel.ro.local     0000000000000010 _iotdata_field_ops
0000000000000018 l     O .data.rel.ro.local     0000000000000008 _iotdata_field_def_battery
0000000000000010 l     O .data.rel.ro.local     0000000000000008 _iotdata_field_def_environment
0000000000000000 l    d  .data.rel.ro.local     0000000000000000 .data.rel.ro.local
```

```text
--- Full library ---
gcc -Wall -Wextra -Wpedantic -Werror -Wcast-align -Wcast-qual -Wstrict-prototypes -Wold-style-definition -Wcast-align -Wcast-qual -Wconversion -Wfloat-equal -Wformat=2 -Wformat-security -Winit-self -Wjump-misses-init -Wlogical-op -Wmissing-include-dirs -Wnested-externs -Wpointer-arith -Wredundant-decls -Wshadow -Wstrict-overflow=2 -Wswitch-default -Wundef -Wunreachable-code -Wunused -Wwrite-strings  -Os -DIOTDATA_VARIANT_MAPS_DEFAULT -c iotdata.c -o iotdata_full.o
   text    data     bss     dec     hex filename
  29603    2112    4096   35811    8be3 iotdata_full.o
--- Minimal encoder (battery + environment, integer-only) ---
gcc -Wall -Wextra -Wpedantic -Werror -Wcast-align -Wcast-qual -Wstrict-prototypes -Wold-style-definition -Wcast-align -Wcast-qual -Wconversion -Wfloat-equal -Wformat=2 -Wformat-security -Winit-self -Wjump-misses-init -Wlogical-op -Wmissing-include-dirs -Wnested-externs -Wpointer-arith -Wredundant-decls -Wshadow -Wstrict-overflow=2 -Wswitch-default -Wundef -Wunreachable-code -Wunused -Wwrite-strings  -Os -mno-sse -mno-mmx -mno-80387 \
        -DIOTDATA_NO_DECODE \
        -DIOTDATA_ENABLE_SELECTIVE -DIOTDATA_ENABLE_BATTERY -DIOTDATA_ENABLE_ENVIRONMENT \
        -DIOTDATA_NO_JSON -DIOTDATA_NO_DUMP -DIOTDATA_NO_PRINT \
        -DIOTDATA_NO_FLOATING -DIOTDATA_NO_ERROR_STRINGS -DIOTDATA_NO_CHECKS_STATE -DIOTDATA_NO_CHECKS_TYPES \
        -c iotdata.c -o iotdata_minimal.o
Minimal object size:
   text    data     bss     dec     hex filename
   1101      32       0    1133     46d iotdata_minimal.o
0000000000000000 l     O .data.rel.ro.local     0000000000000010 _iotdata_field_ops
0000000000000018 l     O .data.rel.ro.local     0000000000000008 _iotdata_field_def_battery
0000000000000010 l     O .data.rel.ro.local     0000000000000008 _iotdata_field_def_environment
0000000000000000 l    d  .data.rel.ro.local     0000000000000000 .data.rel.ro.local
```

#### Build output for esp32-c3 (-Os)

```text
--- ESP32-C3 full library (no JSON) ---
riscv32-esp-elf-gcc -march=rv32imc -mabi=ilp32 -Os -DIOTDATA_NO_JSON -c iotdata.c -o iotdata_esp32c3_full.o
   text    data     bss     dec     hex filename
  19513       0       0   19513    4c39 iotdata_esp32c3_full.o
--- ESP32-C3 minimal encoder (battery + environment, integer-only) ---
riscv32-esp-elf-gcc -march=rv32imc -mabi=ilp32 -Os \
        -DIOTDATA_NO_DECODE \
        -DIOTDATA_ENABLE_SELECTIVE -DIOTDATA_ENABLE_BATTERY -DIOTDATA_ENABLE_ENVIRONMENT \
        -DIOTDATA_NO_JSON -DIOTDATA_NO_DUMP -DIOTDATA_NO_PRINT \
        -DIOTDATA_NO_FLOATING -DIOTDATA_NO_ERROR_STRINGS -DIOTDATA_NO_CHECKS_STATE -DIOTDATA_NO_CHECKS_TYPES \
        -c iotdata.c -o iotdata_esp32c3_minimal.o
Minimal object size:
   text    data     bss     dec     hex filename
    768       0       0     768     300 iotdata_esp32c3_minimal.o
00000000 l    d  .data  00000000 .data
```

#### Stack usage for x86-64 (-Os)

The `test-example` target compiled with `gcc -fstack-usage -Os` on x86-64
illustrates per-function stack frames; nested calls accumulate.

| Function                   | Stack (bytes) | Notes                          |
| -------------------------- | ------------- | ------------------------------ |
| `iotdata_dump_to_string`   | 5872          | `iotdata_dump_t` on stack      |
| `iotdata_dump_to_file`     | 5872          | `iotdata_dump_t` on stack      |
| `iotdata_decode_to_json`   | 2768          | `iotdata_decoder_t` + cJSON    |
| `iotdata_print_to_string`  | 2224          | `iotdata_decoder_t` on stack   |
| `iotdata_print_to_file`    | 2208          | `iotdata_decoder_t` on stack   |
| `iotdata_encode_from_json` | 416           | Encoder context + JSON parsing |
| `iotdata_dump_build`       | 192           | Dynamic, bounded               |
| `iotdata_encode_begin`     | < 64          | —                              |
| `iotdata_encode_end`       | < 64          | —                              |
| `iotdata_decode`           | < 128         | —                              |

The dump and print functions dominate because they allocate `iotdata_dump_t`
(5.8 KB) or `iotdata_decoder_t` (2.2 KB) on the stack. These are
gateway/diagnostic functions not intended for constrained devices. The macro
`IOTDATA_MAX_DUMP_ENTRIES` defaults to 48, and may be reduced to tune down this
size. Note that `iotdata_decoder_t` contains a complete set of all decoded
variables, which in this example is the weather station variant with expensive
TLV entries.

The encode path — `encode_begin`, field calls, `encode_end` — peaks well under
500 bytes total stack depth. On a Class 2 device with 20 KB RAM, this leaves
ample room for the RTOS stack, radio driver, and application logic.

To reduce stack usage on memory-constrained targets, allocate `iotdata_dump_t`
or `iotdata_decoder_t` as a static or global rather than calling the convenience
wrappers which declare them locally.

### 13.5. Variant Table Extension

[flag: leans SPECIFICATION - variant-table extension is a normative mechanism]

The variant table in the reference implementation is a compile-time array.
Adding a new variant requires defining a `variant_def_t` entry with the desired
field mapping and compiling with the `IOTDATA_VARIANT_MAPS` and
`IOTDATA_VARIANT_MAPS_COUNT` defines. No changes to the encoder or decoder logic
are needed — the dispatch mechanism automatically handles any valid variant
table entry.

If a new encoding type is needed (not just a relabelling of an existing type),
the implementer must:

1. Add a new `IOTDATA_FIELD_*` enum value.
2. Implement the six per-field functions (pack, unpack, json_add, json_read,
   dump, print).
3. Add a case to each of the six dispatcher functions.
4. Add the appropriate constants and quantisation helpers.

## 14. Security Considerations

[flag: RFC convention places Security in the spec; here it reads as scope/rationale - confirm side]

This protocol provides no confidentiality, integrity, or authentication
mechanisms at the packet level. It is designed for environments where these
properties are provided at other layers:

- **LoRaWAN** provides AES-128 encryption and message integrity checks at the
  MAC layer.

- **TLS/DTLS** may be used for IP-based transports.

- **Physical security** may be sufficient for isolated deployments on private
  land.

Specific risks to consider:

- **Replay attacks:** An attacker could retransmit captured packets. The
  sequence number provides detection (not prevention) of replayed packets, but
  only if the receiver tracks per-station sequence state.

- **Spoofing:** Station IDs are not authenticated. An attacker within radio
  range could transmit packets with a forged station ID.

- **Eavesdropping:** The wire format is not encrypted. Sensor readings
  (temperature, position, etc.) are transmitted in the clear.

Deployments with security requirements MUST use an appropriate underlying
transport that provides the needed properties.

## 15. Future Work

The following items are identified for future revisions:

1. **Sensor metadata TLVs (types 0x10-0x1F):** Standardised TLV formats for
   conveying sensor type, accuracy class, time source, position source, and
   calibration metadata. This would enable interoperability between devices from
   different manufacturers or deployments without prior out-of-band knowledge.

2. **Quality indicator fields:** Per-field quality/confidence indicators (e.g.
   GNSS fix quality, HDOP, number of satellites). These would likely use the
   reserved TLV type range.

3. **Extended header (variant 15):** A future header format with more variant
   bits, larger station ID space, or additional structural fields.

4. **Implementation singularity limitation:** The wire format supports multiple
   instances of the same field type in different slots (e.g. two independent
   temperature readings). The current reference implementation uses fixed named
   storage in the encoder/decoder structs, limiting each field type to one
   instance. A future implementation could decouple field type from field
   storage, allowing the variant map to bind each slot to an independent storage
   location.

## 16. Versioning and Forward Compatibility

[moved]

### 16.1. Protocol Version

[moved]

### 16.2. Compatibility Model

[moved]

### 16.3. Receiver Requirements

[moved]

### 16.4. Upgrading Deployments

[moved]

### 16.5. Mesh Protocol Versioning

[moved]

### 16.6. Future Version Considerations

If a future protocol revision requires an explicit version field, the following
mechanisms are available without breaking the v1 header layout:

- **Variant-based versioning.** Reserve one or more variant IDs (e.g. 14) for
  "versioned payload" packets where the first N bits after the presence bytes
  carry a version identifier. Existing v1 receivers discard variant 14 packets
  as unknown.

- **TLV-based version advertisement.** A VERSION TLV (type 0x01) already exists
  for firmware identification. A similar mechanism could carry a protocol
  version, though this is available only to receivers that successfully decode
  the packet — a circular dependency for breaking changes.

- **Out-of-band signalling.** LoRaWAN FPort, MQTT topic, HTTP header, or other
  transport-layer metadata can indicate the protocol version without consuming
  payload bits.

---

## Appendix A. 6-Bit Character Table

[moved]

## Appendix B. Quantisation Worked Examples

[moved]

### B.1. Battery Level

[moved]

### B.2. Temperature

[moved]

### B.3. Position (59.334591°N, 18.063240°E)

[moved]

### B.4. Datetime

[moved]

## Appendix C. Complete Encoder Example

[moved]

## Appendix D. Transmission Medium Considerations

### D.1. Design Principle: One Frame, One Transmission

The iotdata protocol is designed so that a useful telemetry packet typically
fits within a single link-layer frame on the target medium. Fragmenting a packet
across multiple frames defeats the core design goals: each additional frame
incurs a separate preamble, MAC header, and — critically on duty-cycle-regulated
media — a separate transmission window. On LoRa at SF12, a single 24-byte frame
takes approximately 1.5 seconds to transmit; two frames would take over 3
seconds, consume twice the energy, and halve the effective reporting rate under
duty cycle constraints.

Implementers SHOULD select fields per packet to remain within the target
medium's payload limit. The presence flag mechanism makes this straightforward:
each packet is self-describing, so the receiver correctly handles any
combination of fields without prior negotiation.

### D.2. LoRa (Raw PHY)

LoRa is the primary target medium. At 125 kHz bandwidth with coding rate 4/5 and
explicit header, the time-on-air for representative iotdata packet sizes is:

| Packet            | Bytes | SF7   | SF8    | SF9    | SF10   | SF11   | SF12   |
| ----------------- | ----- | ----- | ------ | ------ | ------ | ------ | ------ |
| Minimal (battery) | 6     | 36 ms | 62 ms  | 124 ms | 248 ms | 496 ms | 991 ms |
| Typical (b+e+d)   | 10    | 41 ms | 72 ms  | 144 ms | 289 ms | 578 ms | 991 ms |
| With link+flags   | 12    | 41 ms | 82 ms  | 144 ms | 289 ms | 578 ms | 1.16 s |
| Full telemetry    | 24    | 62 ms | 113 ms | 206 ms | 371 ms | 823 ms | 1.48 s |

(Computed using the Semtech AN1200.13 formula with 8-symbol preamble and low
data rate optimisation enabled for SF11/SF12.)

All iotdata packets (6-24 bytes) fit well within the raw LoRa PHY payload limit
of 255 bytes at any spreading factor. The binding constraint is not payload size
but **time-on-air and duty cycle**.

In the EU868 ISM band, the regulatory duty cycle limit is typically 1%. This
means a device must remain silent for 99× the transmission duration after each
packet. The implications are significant:

| Packet         | Bytes | SF7       | SF12          |
| -------------- | ----- | --------- | ------------- |
| Minimal        | 6     | 1 per 4 s | 1 per 1.7 min |
| Typical        | 10    | 1 per 4 s | 1 per 1.7 min |
| Full telemetry | 24    | 1 per 6 s | 1 per 2.5 min |

The difference between a 10-byte typical packet and a 24-byte full packet at
SF12 is the difference between transmitting every 1.7 minutes and every 2.5
minutes — or equivalently, ~35 vs ~24 transmissions per hour. This means that
for bit-packing: the savings are not merely aesthetic, they directly translate
to reporting frequency, battery life, or both.

**Spreading factor selection** is an implementation decision that balances range
against airtime. SF7 provides the shortest airtime but the least range; SF12
provides maximum range (approximately 10-15 km line of sight) at the cost of 32×
the airtime. The iotdata protocol is agnostic to spreading factor — the same
packet is valid regardless of the underlying modulation parameters.

### D.3. LoRaWAN

LoRaWAN adds a MAC layer on top of LoRa, providing device management, adaptive
data rate (ADR), and AES-128 encryption with message integrity. The MAC overhead
consumes approximately 13 bytes of the LoRa PHY payload (MHDR, DevAddr, FCtrl,
FCnt, FPort, MIC), reducing the available application payload.

The maximum LoRaWAN application payload by data rate (EU868):

| Data Rate | Modulation | Max payload | Full iotdata (24B) | Headroom |
| --------- | ---------- | ----------- | ------------------ | -------: |
| DR0       | SF12/125   | 51 bytes    | fits               |     27 B |
| DR1       | SF11/125   | 51 bytes    | fits               |     27 B |
| DR2       | SF10/125   | 51 bytes    | fits               |     27 B |
| DR3       | SF9/125    | 115 bytes   | fits               |     91 B |
| DR4       | SF8/125    | 222 bytes   | fits               |    198 B |
| DR5       | SF7/125    | 222 bytes   | fits               |    198 B |

Full iotdata telemetry (24 bytes) fits comfortably at all LoRaWAN data rates,
with at least 27 bytes of headroom even at the lowest data rate.

Note that the AWS LoRaWAN documentation identifies 11 bytes as the safe
universal application payload across all global frequency plans and data rates.
The iotdata protocol's typical packet (battery + environment + depth) is 10
bytes, falling within this universal limit.

LoRaWAN's AES-128 encryption and MIC address the security considerations
discussed in Section 14. Deployments using LoRaWAN inherit these protections
without any additional work at the iotdata protocol layer.

### D.4. Sigfox

Sigfox imposes the tightest constraints of any common LPWAN medium: a maximum
uplink payload of **12 bytes** and a limit of **140 messages per day**
(approximately one every 10 minutes for uniform distribution).

| Packet configuration    | Bytes | Fits Sigfox? |
| ----------------------- | ----- | :----------: |
| Minimal (battery)       | 6     |      ✓       |
| Typical (bat+env+depth) | 10    |      ✓       |
| With link+flags         | 12    |  ✓ (exact)   |
| With position           | 19    |      ✗       |
| Full telemetry          | 24    |      ✗       |

The protocol's core telemetry packets fit within the 12-byte Sigfox limit.
Position and datetime, which require the extension byte and add 6-9 bytes, do
not fit alongside a full complement of sensor fields.

For Sigfox deployments, implementers SHOULD use a **field rotation strategy**:
transmit core telemetry (battery, environment, depth) on every message, and
rotate in less-frequently-needed fields across separate messages. For example:

- Every 10 minutes: battery + environment + depth (10 bytes)
- Once per hour: battery + position (12 bytes)
- Once per day: battery + datetime + flags (10 bytes)

The presence flag mechanism supports this natively — each packet is
self-describing, so the receiver assembles a complete picture from multiple
packets without any out-of-band configuration.

Sigfox provides its own authentication and anti-replay mechanisms at the network
level, but does not encrypt the payload. Implementers requiring payload
confidentiality on Sigfox must implement application-layer encryption within the
12-byte constraint.

### D.5. IEEE 802.11ah (Wi-Fi HaLow)

IEEE 802.11ah operates in the sub-GHz ISM bands (typically 868 MHz in Europe,
902-928 MHz in the US) and targets IoT applications with range up to 1 km.
Unlike LoRa and Sigfox, it is IP-based and supports standard Ethernet-class
MSDUs (up to 1500 bytes payload per frame), with A-MPDU aggregation for larger
transfers.

Packet size is not a meaningful constraint for iotdata on 802.11ah. However, the
efficiency argument still applies:

- **Power consumption** scales with transmission duration. 802.11ah introduced a
  reduced MAC header (18 bytes vs 28 bytes in legacy 802.11) specifically to
  reduce overhead for small IoT payloads. A 10-byte iotdata payload benefits
  from this optimisation more than a 200-byte JSON payload would.

- **EU duty cycle** regulations apply to the sub-GHz bands used by 802.11ah,
  though the specific constraints differ from LoRa (802.11ah typically uses
  listen-before-talk rather than pure duty cycle limits).

- **Contention** in dense deployments is reduced by shorter frame durations,
  improving effective throughput for all stations.

The iotdata payload would typically be carried as a UDP datagram within the
802.11ah frame. The receiver-side JSON conversion is well suited to 802.11ah
gateways, which have IP connectivity and typically run on more capable hardware.

### D.6. Cellular (NB-IoT, LTE-M, SMS)

Cellular technologies provide reliable, wide-area connectivity with
operator-managed infrastructure. Three cellular transports are relevant for
iotdata:

**NB-IoT and LTE-M** are purpose-built cellular IoT standards. NB-IoT supports
payloads of approximately 1600 bytes per message; LTE-M supports standard IP MTU
sizes. Payload size is not a constraint. Both provide encryption, integrity, and
authentication at the network layer, fully addressing the security
considerations of Section 14.

**SMS** is a widely overlooked but practical transport for low-rate telemetry.
The GSM 03.40 specification defines an 8-bit binary encoding mode (selected via
the TP-DCS field) that provides **140 bytes** of raw octet payload per message —
more than enough for any iotdata packet. Binary SMS is sent via AT commands in
PDU mode, which every GSM/3G/4G modem supports (SIM800, u-blox SARA, Quectel,
etc.).

| Cellular transport | Max payload | Full iotdata | IP stack needed |
| ------------------ | ----------- | :----------: | :-------------: |
| NB-IoT             | ~1600 B     |      ✓       |       Yes       |
| LTE-M              | ~MTU        |      ✓       |       Yes       |
| SMS (8-bit binary) | 140 B       |      ✓       |       No        |

SMS has several properties that distinguish it from IP-based cellular
transports:

- **Near-universal coverage.** SMS operates on the GSM signalling channel and
  works in 2G-only areas where NB-IoT or LTE-M may not be deployed.

- **Store-and-forward.** The SMSC holds messages if the receiver is temporarily
  unreachable, providing inherent buffering that IP-based transports must
  implement at the application layer.

- **No IP stack.** The sensor MCU needs only a UART connection to a GSM modem
  and a handful of AT commands (`AT+CMGS` in PDU mode). This significantly
  reduces firmware complexity compared to a full IP/CoAP/DTLS stack.

- **No data plan.** SMS-only SIM plans are available at low cost, avoiding the
  complexity of cellular data provisioning.

- **Fallback resilience.** SMS uses the control plane rather than the data
  plane, so it typically remains functional during network congestion that would
  affect data services.

The primary disadvantages of SMS are per-message cost (unlike LoRa which is
free, or bulk-metered data plans), latency (typically 1-30 seconds, occasionally
longer during congestion), and receiving-side complexity (the gateway requires
either a GSM modem or an SMS-to-HTTP gateway service). SMS provides no payload
encryption; content is visible to the carrier network.

The bit-packing efficiency of iotdata remains beneficial across all cellular
transports for two reasons:

1. **Energy per byte.** Cellular radio transmission energy is roughly
   proportional to transmission time. Shorter payloads mean shorter active radio
   periods and longer battery life.

2. **Data and message cost.** For IP-based transports, reducing payload from 200
   bytes (JSON) to 10 bytes (iotdata) reduces per-message data consumption by
   95%. For SMS, keeping packets within a single 140-byte message avoids the
   overhead and cost of concatenated multi-part SMS.

SMS is particularly well suited as a **fallback transport**: a sensor that
normally transmits via LoRa could fall back to SMS when connectivity is lost or
when an alarm condition requires guaranteed delivery via a different path. The
same encoded packet is valid on both transports without modification.

### D.7. Medium Selection Summary

| Medium   | Max payload | Full iotdata | Primary constraint      | Encryption | Notes                         |
| -------- | ----------- | :----------: | ----------------------- | :--------: | ----------------------------- |
| LoRa     | 255 B       |      ✓       | Duty cycle / airtime    |     No     | Primary target medium         |
| LoRaWAN  | 51-222 B    |      ✓       | Duty cycle / airtime    |  AES-128   | Managed network               |
| Sigfox   | 12 B        |   Partial    | Hard payload limit      | Auth only  | Field rotation needed         |
| 802.11ah | 1500 B      |      ✓       | Duty cycle (EU) / power |   WPA2/3   | IP-based, UDP transport       |
| NB-IoT   | ~1600 B     |      ✓       | Energy / data cost      |    Yes     | Operator infrastructure       |
| LTE-M    | ~MTU        |      ✓       | Energy / data cost      |    Yes     | Operator infrastructure       |
| SMS      | 140 B       |      ✓       | Per-message cost        |     No     | Fallback / universal coverage |

The protocol's presence flag mechanism makes medium-aware field selection a
runtime decision rather than a compile-time decision. The same encoder can
produce a 10-byte packet for Sigfox and a 24-byte packet for LoRa, with the
receiver handling both identically.

## Appendix E. System Implementation Considerations

### E.1. Microcontroller Class Taxonomy

The iotdata protocol is designed to run on a wide range of microcontrollers
(MCU), but the appropriate implementation strategy varies significantly by
device class:

| Class | Examples                  | RAM       | Flash      | FPU  | Typical role                   |
| ----- | ------------------------- | --------- | ---------- | ---- | ------------------------------ |
| 1     | PIC16F, ATtiny, MSP430G   | 256B-2KB  | 4-16KB     | No   | Sensor (encode only)           |
| 2     | PIC18F, STM32L0, nRF52    | 2-64KB    | 32-256KB   | No\* | Sensor (encode + basic decode) |
| 3     | ESP32-C3, STM32F4, RP2040 | 256-520KB | 384KB-16MB | Yes  | Sensor + gateway               |
| 4     | Raspberry Pi, Linux SBC   | 256MB+    | SD/eMMC    | Yes  | Gateway / server               |

\*nRF52840 has an FPU; most Class 2 devices do not.

The reference implementation currently targets Class 3 and 4 devices. Class 1
and 2 devices require implementation strategies discussed below. The design is
specifically intended to be extended to them.

### E.2. Memory Footprint

The reference implementation's data structures have the following sizes
(measured on a 64-bit platform; 32-bit targets will be smaller due to pointer
size):

| Structure           | Size    | Purpose                                     |
| ------------------- | ------- | ------------------------------------------- |
| `iotdata_encoder_t` | ~300 B  | Encoder context (all fields + TLV pointers) |
| `iotdata_decoder_t` | ~2000 B | Decoded packet (includes TLV data buffers)  |

The encoder context (~300 bytes) is dominated by the TLV pointer array (8
entries × 2 pointers × 8 bytes = 128 bytes on 64-bit). The core sensor fields
occupy approximately 60 bytes. On a 32-bit MCU:

- Full encoder context: ~200 bytes
- Encoder context without TLV: ~72 bytes
- Core sensor fields alone: ~50 bytes

The decoded struct (~2000 bytes) is dominated by the TLV data buffers (8 entries
× ~256 bytes = 2048 bytes). This structure is designed for gateway/server use
and is NOT appropriate for Class 1 or 2 devices. A minimal decoder that ignores
TLV data needs approximately 60 bytes.

TLV support can be excluded from the encoder, which would yield the most
considerable level of savings if resource constrained.

### E.3. Encoder Architecture: Store-Then-Pack

The current encoder uses a "store then pack" strategy:

```c
iotdata_encode_begin(&enc, buf, sizeof(buf), variant, station, seq);
iotdata_encode_battery(&enc, 84, false);    /* stores values */
iotdata_encode_environment(&enc, 21.5f, 1013, 45);
iotdata_encode_end(&enc, &out_len);         /* packs all at once */
```

**Advantages:**

- Fields can be added in any order — the variant table determines wire order at
  pack time.
- Validation happens eagerly, at the `encode_*()` call site.
- The presence bytes are computed once the full field set is known, avoiding
  backfill.

**Disadvantage:**

- The full encoder context must be held in RAM simultaneously: all field values
  plus the output buffer. On Class 3+ devices this is negligible; on Class 1
  devices (256 bytes RAM) it is prohibitive without stripping.

### E.4. Encoder Alternative: Pack-As-You-Go

For severely RAM-constrained devices (Class 1), a pack-as-you-go encoder could
eliminate the context struct entirely. The encoder would write bits directly to
the output buffer as each field is supplied.

The challenge is that the presence bytes (at bit offsets 32-39 and optionally
40-47) must appear in the wire format before the data fields they describe, but
the encoder does not know which fields will be present until all `encode_*()`
calls have been made.

Two approaches can resolve this:

**Approach A: Presence byte backfill.** Reserve the presence byte positions in
the output buffer (write zeros), then pack each field's bits immediately. After
the last field, go back and write the correct presence bytes. This requires
fields to be supplied in strict field order (S0, S1, S2...) so that the bit
cursor advances correctly.

```text
/* Pseudocode for pack-as-you-go with backfill */
write_header(buf, variant, station, seq);   /* bits 0-31 */
pres0_offset = 32;                          /* remember position */
skip_bits(8);                               /* reserve pres0 */
/* caller must add fields in field order */
add_battery(buf, &cursor, level, charging); /* pack immediately */
add_environment(buf, &cursor, t, p, h);
add_depth(buf, &cursor, cm);
/* after all fields: */
backfill_presence(buf, pres0_offset, fields_present);
```

**Approach B: Two-pass encode.** First pass: call all `encode_*()` functions
which simply set bits in a `fields_present` bitmask (1 byte of RAM). Second
pass: iterate the variant field table and pack only the fields that are present.
This requires the field values to be available again on the second pass, either
from global/static variables or by re-reading the sensors.

**Trade-offs:**

| Property             | Store-then-pack | Pack-as-you-go (A) | Two-pass (B) |
| -------------------- | :-------------: | :----------------: | :----------: |
| RAM (no TLV)         |   ~72 B + buf   |     ~4 B + buf     | ~4 B + buf\* |
| Field order          |       Any       | Strict field order |     Any      |
| Code complexity      |       Low       |       Medium       |    Medium    |
| Re-read sensors      |       No        |         No         |     Yes      |
| Suitable for Class 1 |    Marginal     |        Yes         |     Yes      |

\*Two-pass requires field values to be available on the second pass, either
stored elsewhere or re-read from hardware.

The reference implementation uses store-then-pack because it is the most
developer-friendly and the target devices (ESP32-C3, Class 3) have ample RAM.
Implementers targeting Class 1 devices SHOULD consider pack-as-you-go with
backfill. Approach A maybe be provided in a future version of the reference
implementation.

### E.5. Compile-Time Field Stripping (#ifdef)

The reference implementation factors all per-field operations into static inline
functions specifically to enable compile-time stripping:

```c
#ifdef IOTDATA_ENABLE_SOLAR
static inline void pack_solar(uint8_t *buf, size_t *bp,
                              const iotdata_encoder_t *enc) {
    bits_write(buf, bp, enc->solar_irradiance, 10);
    bits_write(buf, bp, enc->solar_ultraviolet, 4);
}
static inline void unpack_solar(...) { ... }
/* ...4 more functions... */
#endif
```

Each field type has 6 functions (pack, unpack, json_add, json_read, dump,
print). On an embedded sensor that only transmits battery, environment, and
depth:

| Component                     | Functions | Approx. code size |
| ----------------------------- | :-------: | ----------------: |
| Core (header, presence, bits) |     —     |             ~1 KB |
| Battery (6 functions)         |     6     |            ~400 B |
| Environment (6 functions)     |     6     |            ~500 B |
| Depth (6 functions)           |     6     |            ~350 B |
| **Included total**            |    18     |       **~2.2 KB** |
| Solar (excluded)              |     6     |            -400 B |
| Link quality (excluded)       |     6     |            -350 B |
| Flags (excluded)              |     6     |            -300 B |
| Position (excluded)           |     6     |            -500 B |
| Datetime (excluded)           |     6     |            -400 B |
| JSON functions (excluded)     |    ~20    |             -2 KB |
| Print/dump (excluded)         |    ~20    |           -1.5 KB |

A fully stripped sensor-only build (encode path only, 3 field types, no
JSON/print/dump) can fit in approximately 2-3 KB of flash. This is achievable on
Class 1 devices.

### E.6. Floating Point Considerations

Several encode functions accept floating-point parameters:

- `iotdata_encode_environment()` takes `float` temperature
- `iotdata_encode_position()` takes `double` latitude/longitude
- `iotdata_encode_link()` takes `float` SNR

On MCUs without a hardware FPU (most Class 1 and many Class 2 devices),
floating-point arithmetic is emulated in software, which is both slow (~50-100
cycles per operation) and adds code size (~2-5 KB for soft-float library).

For Class 1 targets, implementers SHOULD consider:

- **Integer-only temperature API:** Accept temperature as a fixed-point integer
  in units of 0.25°C offset from -40°C. The caller performs
  `q = (temp_raw - (-40*4))` and passes `q` directly. No floating point needed.

- **Integer-only position API:** Accept pre-quantised 24-bit latitude and
  longitude values. The caller uses the GNSS receiver's native integer output
  and scales appropriately.

- **Integer-only SNR:** Accept SNR as an integer in dB.

The reference implementation uses float/double for developer convenience on
Class 3+ targets. However, the compile-time configurations of
`IOTDATA_NO_FLOATING_DOUBLES` or `IOTDATA_NO_FLOATING` can be used to remove
floating point operations and provide replacement integer-only entry points such
as temperature encoders that take values multiplied by 100, e.g. 152 to
represent a temperature of 15.2°C.

### E.7. Dependencies and Portability

The reference implementation has the following dependency profile:

| Component       | Dependencies                                          | Required for            |
| --------------- | ----------------------------------------------------- | ----------------------- |
| Encoder         | `<stdint.h>`, `<stdbool.h>`, `<stddef.h>`, `<math.h>` | All builds              |
| Decoder         | Same as encoder                                       | Gateway / bidirectional |
| JSON conversion | `libcjson`                                            | Gateway / server        |
| Print / dump    | `<stdio.h>`                                           | Debug / gateway         |

The core encoder has no external library dependencies. The `<math.h>` dependency
is for `round()` and `floor()` in the quantisation functions; on platforms where
`<math.h>` is unavailable or expensive, these can be replaced with integer
arithmetic equivalents:

```c
/* Integer-only round for non-negative values */
static inline uint16_t int_round(uint32_t num, uint32_t denom) {
    return (uint16_t)((num + denom / 2) / denom);
}
```

The `libcjson` dependency exists only for the JSON serialisation functions and
SHOULD be excluded from embedded builds via `#ifdef IOTDATA_NO_JSON`.

### E.8. Stack vs Heap Allocation

The encoder and decoder are designed for **stack allocation only**. No
`malloc()` or `free()` calls are made in any encode or decode path. This is
critical for:

- **Bare-metal systems** without a heap allocator.
- **RTOS environments** (FreeRTOS, Zephyr) where heap fragmentation must be
  avoided and stack sizes are fixed.
- **Safety-critical systems** where dynamic allocation is prohibited by coding
  standards (MISRA C, etc.).

The JSON conversion functions (`iotdata_decode_to_json`) do allocate heap memory
via `cJSON_CreateObject()` and return a `malloc`'d string. These functions are
gateway/server-only and are not intended for embedded use.

### E.9. Endianness

The bit-packing functions operate on individual bytes via bit manipulation and
are **endian-agnostic**. The `bits_write()` and `bits_read()` functions address
the output buffer byte-by-byte, computing bit offsets explicitly. No multi-byte
loads or stores are used in the packing/unpacking path.

This means the same code runs correctly on:

- Little-endian ARM (ESP32, STM32, RP2040)
- Big-endian PIC (in standard configuration)
- Any other byte-addressable architecture

No byte-swap operations are needed when moving between platforms.

### E.10. Real-Time Considerations

The encoder's `encode_end()` function performs a single linear pass over the
variant field table and packs all present fields. The execution time is bounded
and predictable:

- **Minimum** (battery only): ~50 bit operations
- **Maximum** (all fields + TLV): ~300 bit operations

Each bit operation is a constant-time shift-and-mask. There are no loops with
data-dependent iteration counts (except TLV string encoding, which iterates over
the string length). The encoder is suitable for use in interrupt service
routines or time-critical sections, though calling it from a main-loop context
is more typical.

### E.11. Platform-Specific Notes

**Raspberry Pi / Linux (Class 4):** The full implementation runs unmodified,
including JSON conversion, print, dump, and all 8 field types. Typically used as
a gateway, receiving packets via LoRa HAT or USB-connected radio module,
decoding to JSON, and forwarding via MQTT, HTTP, or database insertion.

**ESP32-C3 (Class 3, primary target):** The reference implementation runs
unmodified. The ESP32-C3 has 400 KB SRAM and 4 MB flash but no hardware FPU —
both single- and double-precision arithmetic is software-emulated. Use
`IOTDATA_NO_FLOATING` for best performance. Both the encoder and decoder,
including JSON functions, fit comfortably. The ESP-IDF build system supports
`#ifdef` stripping via `menuconfig` or `sdkconfig` defines.

**STM32L0 (Class 2):** With 20 KB RAM and 64-192 KB flash, both the encoder
context (328 bytes) and decoded struct (2176 bytes) fit on the stack
comfortably. Exclude JSON, print, and dump functions. Use
`-ffunction-sections -fdata-sections -Wl,--gc-sections` to strip unused code.

**PIC18F (Class 2):** Similar constraints to STM32L0 but with typically less RAM
(2-4 KB) and a more limited C compiler. The reference implementation's use of
`static inline` functions may need to be adjusted (some PIC compilers do not
inline effectively). Consider the pack-as-you-go approach (Section E.4) for
minimal RAM usage.

**PIC16F (Class 1):** With as little as 256 bytes of RAM, the full encoder
context does not fit. Use pack-as-you-go with backfill (Section E.4, Approach
A), integer-only APIs (Section E.6), and aggressive `#ifdef` stripping (Section
E.5). Target flash budget: 2-3 KB for a battery + environment + depth encoder.

### E.12. Class 1 Hand-Rolled Encoder Example

[flag: contains a full code example (spec-like) inside platform guidance]

On devices with as little as 256 bytes of RAM (PIC16F, ATtiny), the library's
table-driven architecture is unnecessary overhead. The following self-contained
function encodes a weather station packet with battery, environment, and two TLV
entries directly into a caller-provided buffer. No structs, no function
pointers, no library linkage — just bit-packing arithmetic. The full weather
station packet requires a buffer of no more than 32 bytes (without TLV). If
really necessary, the implementation can avoid buffers and use ws_bits to write
directly to an output (e.g. serial port).

```c
#define WS_VARIANT       0    /* Built-in weather station */
#define WS_STATION       1    /* 0 to 4095 */
#define WS_PRES0_FIELDS  6    /* battery, link, environment, wind, rain, solar */

#define WS_PRES_EXT      0x80
#define WS_PRES_TLV      0x40

static void ws_bits(uint8_t *buf, uint16_t *bp, uint32_t val, uint8_t n) {
    for (int8_t i = n - 1; i >= 0; i--, (*bp)++)
        if (val & (1UL << i))
            buf[*bp >> 3] |= (1U << (7 - (*bp & 7)));
}

/*
 * Encodes: battery, environment, one raw TLV, one string TLV.
 * All integer arithmetic.  No floating point.  No malloc.
 *
 * Parameters:
 *   buf        — output buffer, must be zeroed by caller, >= 32 bytes
 *   sequence   — 16-bit sequence number
 *   batt_pct   — battery level 0-100
 *   charging   — 0 or 1
 *   temp100    — temperature in centidegrees (-4000 = -40.00 C)
 *   press_hpa  — pressure in hPa (850-1105)
 *   humid_pct  — humidity 0-100
 *   tlv0_type  — first TLV type (0-63)
 *   tlv0_data  — first TLV raw data pointer
 *   tlv0_len   — first TLV data length
 *   tlv1_type  — second TLV type (0-63)
 *   tlv1_str   — second TLV string (6-bit charset: a-z 0-9 A-Z space)
 *   tlv1_len   — second TLV string length
 *
 * Returns: packet size in bytes
 */
static uint8_t ws_encode(
    uint8_t *buf, uint16_t sequence,
    uint8_t batt_pct, uint8_t charging,
    int16_t temp100, uint16_t press_hpa, uint8_t humid_pct,
    uint8_t tlv0_type, const uint8_t *tlv0_data, uint8_t tlv0_len,
    uint8_t tlv1_type, const char *tlv1_str, uint8_t tlv1_len)
{
    uint16_t bp = 0;

    /* --- Header: 4-bit variant + 12-bit station + 16-bit sequence --- */
    ws_bits(buf, &bp, WS_VARIANT, 4);
    ws_bits(buf, &bp, WS_STATION, 12);
    ws_bits(buf, &bp, sequence, 16);

    /* --- Presence byte 0: ext=0, tlv=1, battery=1, link=0,
           environment=1, wind=0, rain=0, solar=0 --- */
    /*   bit 7: ext         = 0  (no pres1)
     *   bit 6: tlv         = 1  (TLV present)
     *   bit 5: battery     = 1
     *   bit 4: link        = 0
     *   bit 3: environment = 1
     *   bit 2: wind        = 0
     *   bit 1: rain        = 0
     *   bit 0: solar       = 0
     */
    ws_bits(buf, &bp, 0x68, 8);   /* 0b01101000 */

    /* --- Battery: 5-bit level + 1-bit charging --- */
    ws_bits(buf, &bp, ((uint32_t)batt_pct * 31 + 50) / 100, 5);
    ws_bits(buf, &bp, charging ? 1 : 0, 1);

    /* --- Environment: 9-bit temp + 8-bit pressure + 7-bit humidity --- */
    ws_bits(buf, &bp, (uint32_t)((temp100 - (-4000)) + 12) / 25, 9);
    ws_bits(buf, &bp, (uint32_t)(press_hpa - 850), 8);
    ws_bits(buf, &bp, (uint32_t)humid_pct, 7);

    /* --- TLV 0: raw --- */
    ws_bits(buf, &bp, 0, 1);           /* format: raw */
    ws_bits(buf, &bp, tlv0_type, 6);   /* type */
    ws_bits(buf, &bp, 1, 1);           /* more: yes */
    ws_bits(buf, &bp, tlv0_len, 8);    /* length */
    for (uint8_t i = 0; i < tlv0_len; i++)
        ws_bits(buf, &bp, tlv0_data[i], 8);

    /* --- TLV 1: 6-bit string --- */
    ws_bits(buf, &bp, 1, 1);           /* format: string */
    ws_bits(buf, &bp, tlv1_type, 6);   /* type */
    ws_bits(buf, &bp, 0, 1);           /* more: no */
    ws_bits(buf, &bp, tlv1_len, 8);    /* length */
    for (uint8_t i = 0; i < tlv1_len; i++) {
        char c = tlv1_str[i];
        uint8_t v = (c == ' ') ? 0 :
                    (c >= 'a' && c <= 'z') ? 1 + (c - 'a') :
                    (c >= '0' && c <= '9') ? 27 + (c - '0') :
                    (c >= 'A' && c <= 'Z') ? 37 + (c - 'A') : 0;
        ws_bits(buf, &bp, v, 6);
    }

    return (uint8_t)((bp + 7) >> 3);
}
```

**Resource usage:** This function requires approximately 20 bytes of stack (loop
counters, bit pointer, temporaries) plus the output buffer (or not, if directly
writing to serial output). The code compiles to under 400 bytes on PIC18F or
AVR. The caller can reuse the output buffer between transmissions.

**Adapting for other variants:** Copy the function, change the presence byte
constant, and add or remove the field sections. Each field is a self-contained
block of `ws_bits` calls — the protocol document (Sections 4-6) gives the bit
widths and quantisation formulae for every field type.

## Appendix F. Example Weather Station Output

[moved]

## Appendix G. Mesh Protocol

### Overview

The iotdata mesh protocol extends the reach of sensor networks by allowing
dedicated relays to forward sensor data across multiple relays toward one or
more gateways. The protocol is designed to be **seamless** — existing sensors
require no firmware changes, the system works without mesh infrastructure, and
relays can be inserted into a live deployment to fill coverage gaps.

The mesh layer is carried within the existing iotdata wire format using variant
ID 15 (0x0F) for all control-plane traffic. This means mesh packets share the
same 4-byte header structure as sensor data, can coexist on the same radio
channel, and are handled by the same receive path up to the point of variant
dispatch. Relay nodes have a dedicated station ID and can also convey sensor
data under that ID.

### G.1. Use Cases and System Roles

#### G.1.1. The Problem

LoRa radio links between sensors and gateways are subject to terrain,
vegetation, buildings, and seasonal variation. A sensor that works reliably in
winter may become intermittent when foliage returns in spring. A sensor placed
in a valley or behind a structure may never reach the gateway directly.
Increasing transmit power or antenna height is not always practical or
permitted.

#### G.1.2. The Solution

Rather than requiring all sensors to participate in a mesh network (which adds
complexity, power consumption, and firmware requirements), the protocol
introduces a separate class of mesh-aware relays that transparently extend
range. Sensors remain simple transmit-only devices. The mesh is an overlay
infrastructure.

#### G.1.3. System Roles

The protocol defines three roles. A single physical device may implement one or
two of these roles simultaneously.

**Sensor** — A device that periodically transmits iotdata-encoded packets
containing measurement data. Sensors are transmit-only, fire-and-forget. They
have no awareness of the mesh, do not listen for packets, and do not participate
in routing. A sensor's firmware is identical whether or not mesh infrastructure
is deployed. Sensors use iotdata variant IDs 0–14 as defined by their
measurement type. Sensors are typically power constrained.

**Relay** — A mesh-aware device that listens for both sensor packets and mesh
control packets. Its primary function is to forward sensor data toward a gateway
when the sensor cannot reach the gateway directly. Relay nodes form a
self-organising tree topology rooted at gateways, using periodic beacon messages
to discover routes. A relay treats sensor payloads as opaque byte sequences — it
never inspects or interprets measurement fields. A relay may optionally also
function as a sensor (dual-role), transmitting its own measurement data (e.g.
position, battery level, environment) using a standard iotdata variant alongside
its mesh traffic on variant 15. Relays have higher power demand than sensors,
but are still unlikely to be mains powered.

**Gateway** — A mesh-aware device that receives sensor data (directly or via
relays) and delivers it to upstream systems for processing, storage, and
display. Gateways originate beacon messages that define the routing topology. A
deployment may include multiple gateways for redundancy or to cover a wide area.
Each gateway is identified by a unique station ID. Gateways perform duplicate
suppression — if the same sensor packet arrives both directly and via a relay,
only the first arrival is processed. Gateways are typically mains powered and
likely to be connected to network and internet infrastructure.

#### G.1.4. Role Capabilities

| Capability                       | Sensor            | Relay                | Gateway            |
| -------------------------------- | ----------------- | -------------------- | ------------------ |
| Transmits own sensor data        | yes               | optional (dual-role) | no                 |
| Listens for packets              | no                | yes                  | yes                |
| Forwards sensor data             | no                | yes                  | no (endpoint)      |
| Participates in mesh routing     | no                | yes                  | yes (root)         |
| Originates beacons               | no                | no                   | yes                |
| Rebroadcasts beacons             | no                | yes                  | no                 |
| Requires iotdata field knowledge | yes (own variant) | no (opaque relay)    | yes (all variants) |
| Firmware changes for mesh        | none              | mesh-specific        | mesh additions     |

### G.2. Design Principles

#### G.2.1. Seamless Operation

The mesh layer is an optional enhancement, not a prerequisite. A deployment
consisting only of sensors and gateways works exactly as it does today. Mesh
infrastructure can be added incrementally — deploying a relay between a
struggling sensor and the gateway immediately improves reliability without
touching the sensor.

#### G.2.2. Protocol Integration

Mesh control packets use iotdata variant ID 15 (0x0F). This reserves the final
variant slot for mesh traffic while leaving variants 0–14 available for sensor
data definitions. The 4-byte iotdata header (variant, station_id, sequence) is
shared by all packet types, meaning mesh packets are structurally valid iotdata
packets with a different interpretation of the payload.

Byte 4, which serves as a presence bitmap in sensor variants, serves as a
control type field in variant 15 packets. The upper nibble identifies the mesh
packet type (4 bits, supporting up to 16 types). This allows the receive path to
branch on variant ID alone: variants 0–14 route to the sensor data decoder,
variant 15 routes to the mesh handler.

#### G.2.3. Opaque Forwarding

Relays never inspect the contents of sensor packets beyond the 4-byte iotdata
header. The header is read only to extract the originating sensor's station_id
and sequence number for duplicate suppression. All remaining bytes are treated
as an opaque blob, copied verbatim during forwarding. This means the mesh layer
has zero coupling to field definitions, variant suites, encoding formats, or any
future changes to the iotdata measurement schema.

The sole structural dependency is the position and size of station_id (12 bits
at bytes 0–1) and sequence (16 bits at bytes 2–3) in the iotdata header. This is
the most stable contract in the protocol and is not expected to change.

#### G.2.4. Multiple Gateway Support

Each gateway originates its own beacon stream identified by its station_id
(carried as `gateway_id` in the beacon). Relays independently track which
gateway trees they belong to and select the best gateway by cost (relay count),
breaking ties by received signal strength. If a gateway fails, its beacons
cease, and so relays in its tree will time out after a configurable number of
missed beacon rounds, and automatically adopt an alternative gateway's tree.

#### G.2.5. Gradient-Based Routing

The mesh uses a simplified distance-vector approach where each relay knows its
cost (number of relays to reach the gateway) and forwards data toward lower-cost
neighbours. This is conceptually similar to RPL (RFC 6550, Routing Protocol for
Low-Power and Lossy Networks) but dramatically simplified — no full topology
state, no Directed Acyclic Graph computation, no IPv6 dependency. Each node
stores only its parent, a backup parent, and a small neighbour table.

### G.3. Protocol Flows

[moved]

#### G.3.1. Topology Discovery

[moved]

#### G.3.2. Sensor Data Forwarding

[moved]

#### G.3.3. Relay-by-Relay Acknowledgement

[moved]

#### G.3.4. Fast Failover

[moved]

#### G.3.5. Network Monitoring

[moved]

#### G.3.6. Reachability Testing (v2)

[moved]

### G.4. Packet Structures

[moved]

#### G.4.1. Standard iotdata Header (all packets, all variants)

[moved]

#### G.4.2. Variant 15 Common Header

[moved]

#### G.4.3. BEACON (ctrl_type 0x0)

[moved]

#### G.4.4. FORWARD (ctrl_type 0x1)

[moved]

#### G.4.5. ACK (ctrl_type 0x2)

[moved]

#### G.4.6. ROUTE_ERROR (ctrl_type 0x3)

[moved]

#### G.4.7. NEIGHBOUR_REPORT (ctrl_type 0x4)

[moved]

#### G.4.8. PING (ctrl_type 0x5) — v2

[moved]

#### G.4.9. PONG (ctrl_type 0x6) — v2

[moved]

#### G.4.10. Reserved (ctrl_type 0x7–0xF)

[moved]

#### D.11 Packet Summary

[moved]

### G.5. Node Operation and Requirements

[moved]

#### G.5.1. Relay Node State

[moved]

#### G.5.2. Relay Node Main Loop

[moved]

#### G.5.3. Gateway Additions

[moved]

#### G.5.4. Duplicate Suppression

[moved]

#### G.5.5. Parent Selection and Failover

[moved]

#### G.5.6. Beacon Rebroadcast Rules

[moved]

#### G.5.7. Forward Suppression (Trickle)

[moved]

### G.6. Deployment Considerations

[flag: deployment considerations - borderline TARGETING/SPECIFICATION]

#### G.6.1. Hardware

The mesh protocol is designed for off-the-shelf LoRa modules (e.g. Semtech
SX1262-based modules like Ebyte E22-900T22D) connected to ESP32-class
microcontrollers. No specialised radio hardware is required.

Recommended relay hardware:

- ESP32-C3 or ESP32-S3 (low cost, low power, sufficient RAM and compute)
- LoRa module with RSSI reporting (for neighbour quality assessment)
- Reliable power: mains, solar + battery, or PoE where available
- Weatherproof enclosure with external antenna for outdoor deployment

Relays should have reliable power supplies. Unlike sensors which can sleep
between transmissions, relays must listen continuously. Solar + battery is
viable in most climates with an appropriately sized panel (5–10W) and battery
(5–10Ah).

#### G.6.2. Range and Relay Budgets

Typical LoRa range per relay in different environments at commonly used settings
(SF7–SF9, 125kHz bandwidth, 14–22 dBm transmit power):

| Environment               | Typical range per relay | Notes                                       |
| ------------------------- | ----------------------- | ------------------------------------------- |
| Open farmland             | 2–8 km                  | Line-of-sight, minimal obstructions         |
| Rolling hills             | 1–4 km                  | Terrain shadowing, partial LOS              |
| Forest / dense vegetation | 0.5–2 km                | Significant attenuation, seasonal variation |
| Urban / buildings         | 0.3–1.5 km              | Multipath, reflection, penetration loss     |
| Indoor-to-outdoor         | 50–500 m                | Wall penetration, highly variable           |

With a maximum TTL of 255 relays, the theoretical network span is enormous
(hundreds of kilometres). In practice, latency and airtime constraints limit
useful depth to 5–10 relays in most deployments. Beyond 10 relays, per-packet
latency (including backoff, transmission time, and ACK round-trips) accumulates
significantly.

#### G.6.3. Latency Budget

Per-relay latency for a forwarded packet:

| Component                    | Typical     | Notes                                     |
| ---------------------------- | ----------- | ----------------------------------------- |
| Trickle backoff              | 200–1000 ms | Random, first-relay only for sensor→relay |
| LoRa TX time (30 bytes, SF7) | ~50 ms      | Higher SF increases proportionally        |
| ACK wait                     | 0–500 ms    | Timeout before retry                      |
| Processing overhead          | < 1 ms      | Negligible on ESP32                       |

Estimated end-to-end latency by relay count:

| Relays | Best case | Typical |
| ------ | --------- | ------- |
| 1      | ~300 ms   | ~700 ms |
| 2      | ~400 ms   | ~1.2 s  |
| 3      | ~500 ms   | ~1.8 s  |
| 5      | ~700 ms   | ~3.0 s  |
| 10     | ~1.2 s    | ~6.0 s  |

For environmental sensor data with transmission intervals of 5–15 seconds, these
latencies are entirely acceptable. The data is not time-critical — a few seconds
of additional delay has no impact on the value of temperature, moisture, or
depth readings.

#### G.6.4. Airtime and Duty Cycle

Every relay consumes airtime. A packet forwarded across 3 relays uses 3× the
airtime of a direct transmission plus ACK overhead. In regions with regulatory
duty cycle limits (e.g. 1% in EU 868MHz sub-band h1.4), this constrains the
aggregate throughput.

Example: 16 sensors transmitting every 10 seconds, average packet 25 bytes.

| Scenario                        | Packets/sec             | Airtime/sec (SF7) | Effective duty cycle |
| ------------------------------- | ----------------------- | ----------------- | -------------------- |
| All direct to gateway           | 1.6                     | ~80 ms            | ~0.8%                |
| All via 1 relay                 | 1.6 × 2 (fwd+ack)       | ~210 ms           | ~2.1%                |
| All via 2 relays                | 1.6 × 4 (2×fwd + 2×ack) | ~420 ms           | ~4.2%                |
| Mixed (50% direct, 50% 1-relay) | ~2.4                    | ~145 ms           | ~1.5%                |

In practice, most sensors will be direct to gateway. Only sensors with poor
direct links use mesh relays. A typical deployment with 3–5 sensors using
1-relay stays well within 1% duty cycle for the relay and gateway.

For deployments requiring higher throughput, use the 915MHz ISM band (Americas,
Australia) which has more relaxed duty cycle requirements, or use LoRa spreading
factor 7 (fastest airtime) with forward error correction.

#### G.6.5. Recommended Maximum Configuration Per Deployment

| Parameter                      | Recommended limit | Hard limit                                 |
| ------------------------------ | ----------------- | ------------------------------------------ |
| Sensors per gateway            | 50–100            | 4095 (station_id space)                    |
| Relays per gateway             | 10–20             | No hard limit, bounded by airtime          |
| Maximum useful relays          | 5–10              | 255 (TTL field)                            |
| Network span                   | 10–50 km          | Limited by relay count × range             |
| Gateways per deployment        | 2–5               | No hard limit (each runs independent tree) |
| Neighbour table size per relay | 8–16 typical      | 63 (protocol limit)                        |
| Total nodes (sensors + relays) | 100–200           | 4095 (station_id space)                    |

### G.7. Example Deployments

[flag: example deployments - borderline TARGETING/SPECIFICATION]

#### G.7.1. Moderate Farm (Mixed Arable and Livestock)

A 500-hectare farm with a central farmhouse, outlying barns, fields extending
2–3 km in each direction, and a river valley at the property boundary.

**Challenge:** Sensors in low-lying fields near the river are 3 km from the
farmhouse with a ridge blocking line-of-sight. A weather station on a
north-facing hillside has intermittent connectivity.

**Deployment:**

```text
                [WS-4] weather station (hilltop)
                              |
                           direct
                             |
[SM-1] soil ──direct──> [GATEWAY] farmhouse
[SM-2] soil ──direct──/      |
[WL-1] water ─direct──/      |
                             |
                         direct
                             |
                    [HOP-A] barn roof (solar powered)
                             |
                         1 relay
                         /       \
               [SM-3] soil    [SM-4] soil      (low field, behind ridge)
               [WS-5] weather station          (river valley)
               [WL-2] water level              (river gauge)
```

Configuration: 1 gateway, 1 relay, 8 sensors. Relay A sits on a barn roof at
mid-elevation with clear line-of-sight to both the gateway and the river valley.
Sensors SM-3, SM-4, WS-5, and WL-2 transmit normally. Relay A hears them (they
cannot reach the gateway directly), wraps their data in FORWARD messages, and
sends upstream to the gateway. Total relay load on Relay A: 4 sensors × ~0.1
packets/sec = ~0.4 FORWARD packets/sec. Well within capacity.

**Seasonal variation:** In summer, tree canopy growth may degrade Relay A's link
to WS-5. If WS-5's data becomes intermittent, deploy Relay B near the river to
provide a 2-relay path: WS-5 → Relay B → Relay A → Gateway. Relay B
automatically integrates — it hears Relay A's rebroadcast beacon (cost=1), sets
itself as cost=2, and begins forwarding.

#### G.7.2. Forest Research Station

A 2000-hectare managed forest with environmental sensors distributed along
trails and watercourses. Dense canopy limits per-relay range to 500m–1.5km. A
research cabin at the forest edge serves as the gateway location, with a second
gateway at a fire lookout tower 4 km away.

**Challenge:** Sensors deep in the forest are 3–5 km from either gateway with no
line-of-sight. Canopy attenuation is severe.

**Deployment:**

```text
[GATEWAY-1] research cabin          [GATEWAY-2] fire tower
      |                                    |
   direct                              direct
      |                                    |
  [HOP-1] ridge clearing              [HOP-4] trail junction
      |                                    |
  1 relay from GW-1                     1 relay from GW-2
      |                                    |
  [HOP-2] stream crossing             [HOP-5] canopy gap
      |                                    |
  2 relays from GW-1                    2 relays from GW-2
      |                                    |
  [SM-1..4] soil sensors              [ENV-1..3] environment
  [WL-1] water level                  [WS-1] weather station
      |
  [HOP-3] deep forest
      |
  3 relays from GW-1
      |
  [SM-5..8] deep soil sensors
  [AQ-1] air quality
```

Configuration: 2 gateways, 5 relays, ~16 sensors. Maximum depth is 3 relays.
Relays are placed at natural clearings, ridge lines, trail junctions, and stream
crossings where canopy gaps improve radio propagation.

**Redundancy:** HOP-2 can hear beacons from both GW-1 (via HOP-1, cost=2) and
GW-2 (via HOP-4 and HOP-5, cost=3). It normally routes via GW-1 (lower cost). If
HOP-1 fails, HOP-2 receives no beacons from GW-1's tree, times out after 3
rounds, and adopts GW-2's tree via HOP-5 at cost=3. Sensors SM-1..4 and WL-1
continue to operate without interruption — they are unaware of the topology
change.

#### G.7.3. Considerations for Moving Sensors

If a sensor is mounted on a vehicle (e.g. a tractor, livestock tracker, or
patrol vehicle) that moves between coverage areas, the protocol handles this
naturally because sensors are not mesh-aware.

**How it works:** The vehicle-mounted sensor transmits periodically as always.
As it moves, different relays (or the gateway directly) hear its transmissions.
Whichever relay hears the packet forwards it. If multiple relays hear it,
Trickle suppression ensures only one forwards. As the vehicle moves out of one
relay's range and into another's, forwarding seamlessly transitions.

**What works well:** Slow-moving vehicles (tractors, livestock) that spend
minutes to hours within each relay's coverage area. The sensor's transmission
interval (5–15 seconds) means multiple packets are sent during each coverage
window.

**What works less well:** Fast-moving vehicles passing through a relay's range
in seconds. If the sensor's transmission interval is longer than the transit
time through coverage, packets may be missed during the transition between
nodes. This is inherent to any non-continuous transmission scheme and is not
specific to the mesh protocol.

**The sensor firmware needs no changes.** The mesh adapts to the sensor's
location in real time. The gateway sees the same station_id and sequence numbers
regardless of which relay forwarded the data. Duplicate suppression handles
cases where the sensor is within range of multiple relays simultaneously.

### G.8. Protocol Version History

[moved]

### G.9. Reserved Identifiers

[moved]

### G.10. Future Considerations

#### G.10.1. Cross-Gateway Duplicate Suppression

In multi-gateway deployments, a sensor positioned between two gateways (or a
relay node with paths to both) may deliver the same packet to multiple gateways.
Each gateway performs local dedup correctly, but the upstream system (database,
MQTT broker, dashboard) receives the same measurement data twice from different
gateway sources.

**Lightweight solution: UDP dedup broadcast.** Each gateway UDP-broadcasts a
compact dedup notification on the local network whenever it processes a sensor
packet. The notification contains only the dedup key — no measurement data:

```text
[gateway_id]        2 bytes     (12-bit station_id of the broadcasting gateway)
[num_entries]       1 byte      (number of dedup tuples in this batch, 1–32)
[entries...]        4 bytes each:
    [station_id]    2 bytes     (12-bit origin sensor, zero-padded to 16 bits)
    [sequence]      2 bytes     (16-bit origin sequence)
```

Maximum batch: 32 entries × 4 bytes + 3 byte header = 131 bytes per UDP
datagram.

On receipt, other gateways add these tuples to their local dedup ring. If a
subsequent FORWARD or direct sensor packet arrives with a station_id and
sequence already in the ring (whether from local receive or cross-gateway
notification), it is suppressed.

**Timing:** On a LAN, UDP broadcast latency is under 1ms. LoRa packet
transmission plus relay backoff is typically 200–1000ms. This means the
cross-gateway notification almost always arrives before the second copy of the
sensor data, achieving reliable suppression. In the rare case where two gateways
receive the same packet within 1ms of each other (both heard the sensor
directly), both may process it. This is acceptable — the upstream system can
perform its own dedup on {station_id, sequence} as a final safety net.

**Implementation:** This is entirely optional. A deployment with a single
gateway has no need for it. A multi-gateway deployment works correctly without
it — the upstream system simply sees occasional duplicates. The UDP broadcast
layer can be added to gateways independently of the mesh protocol and requires
no changes to relays or sensors.

**Alternative approaches:**

- **Shared MQTT topic** — gateways publish dedup tuples to a topic such as
  `iotdata/mesh/dedup/{gateway_id}`. Other gateways subscribe. Adds dependency
  on the MQTT broker being available but piggybacks on infrastructure that
  likely already exists for sensor data delivery.
- **Upstream dedup only** — skip gateway-to-gateway coordination entirely. The
  database or ingestion layer deduplicates on {station_id, sequence,
  time_window}. Simplest to implement, slightly higher upstream load from
  duplicate records.
- **Station-id range assignment** — the operator assigns non-overlapping
  station_id ranges to gateways. A gateway only processes packets from its
  assigned range. Simple but inflexible — a sensor that moves or a relay that
  reroutes to a different gateway may fall outside the assigned range. Not
  recommended for dynamic mesh deployments.

#### G.10.2. Potential Additional Control Packet Types

The ctrl_type field has 10 unused values (0x7–0xF, plus 0x5–0x6 reserved for
PING/PONG v2). Future protocol revisions may define additional packet types. The
following have been identified as candidates:

**CONFIG_PUSH (candidate: 0x7)** — Gateway pushes configuration parameters to a
specific relay. Routed downstream like PING. Enables remote adjustment of beacon
intervals, transmit power, parent selection thresholds, and reporting frequency
without physical access to the node.

Possible payload: target_station(12), config_key(8), config_value(16). Config
keys could include:

| Key  | Meaning                               | Value range         |
| ---- | ------------------------------------- | ------------------- |
| 0x01 | Beacon rebroadcast interval (seconds) | 10–600              |
| 0x02 | Forward retry count                   | 0–15                |
| 0x03 | Forward ACK timeout (ms / 100)        | 1–50 (100ms–5000ms) |
| 0x04 | Parent timeout (missed beacon rounds) | 1–15                |
| 0x05 | Neighbour report interval (minutes)   | 1–60                |
| 0x06 | Transmit power level                  | Module-specific     |
| 0x07 | Force rejoin (clear routing state)    | 1 = trigger         |

**CONFIG_ACK (candidate: 0x8)** — Target node acknowledges receipt of
CONFIG_PUSH. Routes upstream. Confirms the configuration was applied.

**PATH_TRACE (candidate: 0x9)** — Diagnostic packet that records the station_id
of every relay it traverses, building a full path trace from sensor to gateway.
A relay appends its own station_id (2 bytes) to the payload before forwarding.
The gateway receives a complete ordered list of the relay chain.

Possible payload: origin_station(12), ttl(8), relay_count(8), then relay_count ×
station_id(12) entries packed sequentially. Maximum path of 15 relays = 5 + 1 +
1 + 23 = 30 bytes. This would be triggered by the gateway wrapping a specific
sensor's next FORWARD in a PATH_TRACE envelope, or by a relay when it detects a
new sensor (first time seeing a station_id).

**NETWORK_RESET (candidate: 0xA)** — Gateway broadcasts a command for all relays
to flush routing state and re-discover the topology from scratch. Nuclear option
for when the mesh becomes wedged in a suboptimal configuration. Payload: just a
confirmation nonce to prevent accidental triggering.

**TIME_SYNC (candidate: 0xB)** — If a future deployment requires coordinated
sleep windows or TDMA-style channel access, a dedicated time synchronisation
packet could carry a high-resolution timestamp from the gateway. Relays would
estimate propagation delay from relay count and adjust their local clocks.
However, for the current protocol's CSMA-based channel access, this is
unnecessary.

**GROUP_FORWARD (candidate: 0xC)** — Aggregation packet where a relay bundles
multiple small sensor packets into a single transmission to reduce per-packet
overhead and ACK traffic. Payload: num_packets(6), then concatenated inner
packets with 1-byte length prefixes. Most useful in dense deployments where a
single relay forwards for many sensors. Trade-off: increases single-packet
airtime and failure blast radius (losing one aggregated transmission loses
multiple sensor readings).

#### G.10.3. Extended Neighbour Metrics

The current NEIGHBOUR_REPORT carries cost, RSSI, and station_id per neighbour.
Future revisions may extend neighbour entries with additional quality metrics:

- **Packet delivery ratio (PDR)** — percentage of expected packets actually
  received from this neighbour over a window. 4 bits (16 levels, ~6%
  granularity) would suffice. Better parent selection metric than instantaneous
  RSSI.
- **Asymmetric link detection** — a flag indicating whether the neighbour has
  acknowledged hearing this node. A neighbour with good inbound RSSI but no
  evidence of hearing this node's transmissions is a poor parent candidate
  (asymmetric link, common with differing antenna heights or transmit powers).
- **Neighbour role** — 2 bits indicating whether the neighbour is a gateway,
  relay, or sensor. Currently inferred from behaviour (gateways originate
  beacons, relays rebroadcast, sensors don't participate), but an explicit role
  field simplifies topology visualisation.

These extensions would increase neighbour entry size from 3 to 4 bytes. The
num_neighbours field (6 bits, max 63) and LoRa payload limits (222 bytes at SF7)
would support up to 53 extended entries — still more than sufficient.

#### G.10.4. Security Considerations

The v1 protocol includes no authentication or encryption. For agricultural and
environmental monitoring deployments, the threat model is typically low — the
data has no commercial sensitivity and the radio channel is shared ISM spectrum.
However, for deployments where integrity matters:

**Packet authentication** — a shared 4-byte key (pre-configured on all mesh
nodes and gateways) could be used to compute a 2-byte truncated HMAC appended to
every mesh control packet. This prevents rogue nodes from injecting false
beacons or FORWARD packets. The key would be distributed during provisioning and
is not expected to change frequently. 2 bytes provides 65536 possible values —
sufficient to prevent casual injection, though not secure against a determined
attacker with radio access.

**Replay protection** — the existing sequence number provides partial replay
protection. A replayed packet with a previously-seen sequence number is caught
by dedup. However, a replayed BEACON with a valid-looking generation could
disrupt routing. Binding the HMAC to the generation counter and gateway_id
prevents this.

**Encryption** — encrypting the inner packet within FORWARD would prevent
eavesdropping on sensor data. AES-128 in CTR mode adds zero overhead to the
packet size (ciphertext is same length as plaintext) and requires only a shared
key and a nonce derivable from {station_id, sequence}. However, this adds
computational cost at every relay (decrypt to verify, re-encrypt to forward)
which is unnecessary if the relay treats the inner packet as opaque — the relay
does not need to read the inner packet's contents, so the inner packet can
remain encrypted end-to-end between sensor and gateway with no relay
involvement. The sensor would encrypt before transmission, the gateway would
decrypt after receipt, and relay nodes would forward the encrypted blob
unchanged.

#### G.10.5. Power Management for Relay Nodes

Relays must listen continuously, which prevents the aggressive sleep modes used
by transmit-only sensors. Typical listen-mode current for an SX1262 LoRa module
is 5–8 mA; combined with an ESP32-C3 in active mode (~25–50 mA average with WiFi
disabled), total system draw is 30–60 mA.

**Solar viability:** At 12V with a 30mA average draw, daily consumption is
~8.6Wh. A 10W solar panel in temperate latitudes produces 20–40Wh/day (seasonal
variation). A 10Ah 12V battery provides ~3–4 days of autonomy in complete cloud
cover. This is viable for most deployments.

**Duty-cycled listening (future optimisation):** If beacon intervals are
synchronised, relays could sleep between expected beacon windows and wake only
during scheduled listen slots. This requires the TIME_SYNC mechanism described
in J.2 and adds complexity to the parent selection logic (must account for clock
drift during sleep). Not recommended for v1 but noted as a path to lower power
consumption if needed.

**Low-power relay mode:** A relay with no downstream children (leaf relay — only
forwarding for sensors it directly hears) could adopt a semi-synchronised
schedule: listen for a window after each expected sensor transmission, forward
any received packets, then sleep until the next expected window. This requires
the relay to learn sensor transmission intervals through observation, which is
feasible since sensors typically transmit at regular (if slightly randomised)
intervals.

#### G.10.6. Network Capacity Planning

The mesh protocol's capacity is fundamentally limited by shared airtime on the
LoRa channel. All nodes — sensors, relays, and gateways — share a single
frequency and spreading factor (assuming no frequency planning).

**Single-channel capacity at SF7/125kHz:**

A 30-byte LoRa packet at SF7/125kHz takes approximately 50ms of airtime. At 1%
duty cycle (EU 868MHz regulatory limit), one device can transmit 20 packets per
100 seconds, or 0.2 packets/second sustained.

In a mesh deployment, the bottleneck is the gateway's immediate neighbourhood —
all forwarded packets must pass through the last relay to the gateway. If 10
relay paths converge on a single relay-1 node, that node must forward 10× the
traffic of any individual sensor. With 100 sensors at 10-second intervals
producing 10 packets/second aggregate, the relay-1 node must forward all 10 plus
transmit ACKs, consuming ~1 second of airtime per second — impossible under any
duty cycle regulation.

**Mitigation strategies:**

- **Multiple relay-1 nodes** — deploy 2–4 relays within direct range of the
  gateway, each serving a different angular sector or downstream branch.
  Distributes the forwarding load.
- **Multiple gateways** — each gateway serves a subset of the network.
  Cross-gateway dedup (J.1) prevents duplicate processing.
- **Frequency planning** — assign different LoRa channels to different branches
  of the mesh. Requires relays to manage multiple frequencies, adding hardware
  or scheduling complexity.
- **Adaptive transmission intervals** — sensors or the CONFIG_PUSH mechanism
  could adjust transmission rates based on network load. Sensors deeper in the
  mesh (more relay) could transmit less frequently.
- **Aggregation** — the GROUP_FORWARD packet type (J.2) could reduce per-packet
  overhead and ACK count at the cost of increased single-transmission airtime.

**Rule of thumb:** For a single LoRa channel at SF7 with 1% duty cycle, plan for
no more than 30–50 sensors per gateway, with no more than 20 forwarded through
any single relay-1 relay. Scale beyond this by adding gateways, not by deepening
the mesh.

#### G.10.7. Interoperability and Versioning

The protocol currently has no version negotiation mechanism. All mesh nodes are
expected to run the same protocol version. For future-proofing:

- **Reserved ctrl_types (0x7–0xF)** should be silently discarded by nodes that
  do not recognise them. This allows new packet types to be deployed
  incrementally — gateways can be updated first, followed by relays, without
  causing errors on nodes still running older firmware.
- **Reserved flag bits** in the BEACON packet provide a forward-compatible
  extension point. A new capability can be signalled by setting a flag bit.
  Older nodes ignore unknown flags but continue to process the beacon normally.
- **The FORWARD packet is inherently version-agnostic** — relay nodes do not
  interpret the inner payload, so changes to iotdata sensor variants, field
  definitions, or encoding formats require no mesh firmware updates.

If a formal version field becomes necessary, it could be encoded in the BEACON's
reserved flag bits (e.g. flags bits 2–3 as a 2-bit protocol version, supporting
4 versions). Alternatively, a VERSION_ANNOUNCE packet type could be defined
using one of the reserved ctrl_type slots.

## Appendix H. System Architecture Considerations

This appendix discusses system-level concerns that fall outside the wire
protocol but are essential for reliable deployment. The protocol defines how
data is encoded and transmitted; this appendix addresses how the broader system
around it should be designed and operated.

### H.1. Transmission Scheduling

The protocol does not define or constrain the sensor's transmission interval. In
practice, the interval is a deployment parameter balancing data freshness
against power consumption and airtime budget.

#### Interval Selection

Typical intervals for environmental monitoring range from 5 seconds (high-rate
weather stations during storm events) to 3600 seconds (daily check-in for
dormant sensors). The most common operational range is 30–300 seconds.

The interval SHOULD be chosen with awareness of the regulatory duty cycle limit.
At 1% duty cycle (EU 868MHz), a 30-byte packet at SF7 (~50ms airtime) can be
transmitted every 5 seconds. At SF12 (~1.5s airtime), the minimum interval rises
to 150 seconds. Implementations that transmit more frequently than their duty
cycle permits risk regulatory non-compliance and may interfere with other users
of the shared ISM band.

#### Jitter

Sensors that transmit at a fixed interval risk persistent collisions if multiple
sensors are powered on simultaneously (e.g. after a site-wide power restoration
or batch deployment). Two sensors with identical 60-second intervals that happen
to align will collide on every transmission indefinitely.

Sensors SHOULD add uniformly distributed random jitter to each transmission
interval. A jitter of ±10% of the base interval is sufficient to decorrelate
sensors within a few transmission cycles. For example, a 60-second base interval
should use a random delay of 54–66 seconds per cycle.

The jitter SHOULD be re-randomised for each transmission, not fixed at boot. A
fixed offset (e.g. "this sensor always transmits at base + 3 seconds") reduces
collision probability at boot but does not eliminate persistent collisions
between sensors that happen to draw similar offsets.

#### Adaptive Intervals

Some deployments benefit from event-driven interval changes:

- **Storm mode.** A weather station that detects rapidly changing pressure or
  high wind speeds may temporarily reduce its interval (e.g. from 60s to 10s) to
  capture the event at higher resolution.

- **Low battery mode.** A sensor below a battery threshold may increase its
  interval to extend operational life.

- **Quiet mode.** A sensor that detects no change in its readings over several
  cycles may increase its interval. The presence bit mechanism ensures that
  unchanged fields can be omitted entirely, further reducing airtime.

These behaviours are deployment-specific and are not standardised by this
protocol. The CONFIG TLV (Section 9.5.4) can report the current interval to the
gateway for fleet monitoring.

### H.2. Gateway Architecture

A gateway receives iotdata packets (directly or via mesh relays), decodes them,
and delivers the data to upstream systems. The gateway is the protocol's
termination point — upstream of the gateway, data is typically represented as
JSON, stored in a time-series database, or forwarded via MQTT or HTTP.

#### Receive Path

The gateway's receive path should be structured as a pipeline:

1. **Radio receive.** The LoRa (or other) radio delivers a raw byte buffer and
   link metadata (RSSI, SNR, frequency, spreading factor).

2. **Duplicate suppression.** Check {station_id, sequence} against a ring buffer
   of recently processed packets. Discard duplicates. See Section E.4 of
   Appendix G for implementation details; this mechanism applies equally to
   non-mesh deployments where a sensor may be heard by multiple gateways.

3. **Decode.** Decode the binary packet to the internal representation or
   directly to JSON. Discard malformed packets per Section 11.6.

4. **Enrich.** Attach gateway-side metadata: receive timestamp (from the
   gateway's clock, independent of any datetime field in the packet), gateway
   identity, link quality metrics, and any cached state for this station (last
   known position, firmware version, etc.).

5. **Deliver.** Forward the enriched record to upstream systems via MQTT, HTTP
   POST, database insertion, or local storage.

#### State Management

A gateway SHOULD maintain per-station state for:

- **Last sequence number.** For gap detection and duplicate suppression.

- **Last known position.** Cached from the most recent packet containing a
  position field. Associated with subsequent packets that omit position (see
  Section 11.2).

- **Last known datetime offset.** For stations that transmit datetime
  infrequently, the gateway can estimate the sensor's clock drift by comparing
  the sensor's datetime field against the gateway's receive timestamp.

- **Firmware version and configuration.** Cached from VERSION and CONFIG TLV
  entries. Used for fleet management and diagnostics.

- **Cumulative statistics.** Packet count, gap count, average RSSI, last heard
  timestamp. See Section H.3.

This state may be held in memory (sufficient for small deployments), in a local
key-value store (e.g. SQLite, Redis), or in the upstream database.

### H.3. Operational Monitoring

Operators SHOULD monitor the following metrics to maintain system health. These
metrics are derived from the packet stream and gateway state, not from any
specific protocol field.

#### Per-Station Metrics

| Metric                   | Derivation                                           | Alerts on                                                                  |
| ------------------------ | ---------------------------------------------------- | -------------------------------------------------------------------------- |
| Packets per interval     | Count packets per station per time window            | Station silent for >2× expected interval                                   |
| Sequence gap rate        | Count gaps in sequence number per station            | Gap rate exceeding expected packet loss for the link                       |
| RSSI trend               | Moving average of link RSSI per station              | Sustained decline indicating antenna, obstruction, or hardware degradation |
| Battery trend            | Track battery level over time                        | Level below deployment-specific threshold; unexpected discharge rate       |
| Decode error rate        | Count packets that fail decoding per station         | Non-zero rate from a previously healthy station                            |
| TLV diagnostic frequency | Count DIAGNOSTIC TLV entries per station per window  | Sudden increase indicating sensor fault                                    |
| Restart frequency        | Track restart count from STATUS TLV                  | Restarts with WATCHDOG or PANIC reason                                     |
| Clock drift              | Compare sensor datetime against gateway receive time | Drift exceeding 30 seconds (may indicate RTC failure)                      |

#### System-Wide Metrics

| Metric              | Derivation                                               | Alerts on                                       |
| ------------------- | -------------------------------------------------------- | ----------------------------------------------- |
| Active stations     | Count stations heard within the last N intervals         | Station count drops below expected fleet size   |
| Duplicate rate      | Count packets suppressed by dedup as a fraction of total | High rate may indicate unnecessary mesh overlap |
| Gateway packet rate | Total packets processed per second across all stations   | Approaching processing or duty cycle capacity   |
| Airtime utilisation | Sum of received packet airtimes per time window          | Approaching regulatory duty cycle limit         |
| Decode failure rate | Aggregate decode errors across all stations              | Spike indicating firmware rollout issues        |

These metrics can be derived from the gateway's receive path with minimal
overhead. The enrichment step (Section H.2) is the natural point to update
counters and evaluate alert conditions.

#### Alerting

Alerting thresholds are deployment-specific. A remote weather station in a
mountain location with marginal link budget has different expectations than a
soil sensor 50 metres from the gateway. Operators SHOULD configure per-station
or per-station-class thresholds rather than global defaults.

The most critical alert is **station silence** — a station that stops
transmitting entirely. This may indicate hardware failure, power exhaustion,
theft, or catastrophic link degradation. The alert threshold should be set to
2–3× the expected transmission interval to avoid false positives from normal
jitter and occasional packet loss.

### H.4. Time Synchronisation

The protocol's datetime field (Section 8.8) encodes time relative to the start
of the year with no absolute time reference. The receiver resolves the year
(Section 11.1). This design assumes that the receiver has an accurate clock.

For gateways, this is typically satisfied by NTP synchronisation over an
internet connection, or by a local GNSS receiver. Gateways that lack both SHOULD
use the receive timestamp as the primary time reference and treat the sensor's
datetime field as a secondary indicator, useful for detecting transmission
delays or buffered transmissions but not as the authoritative timestamp.

For sensors, the time source determines the accuracy of the datetime field:

| Source             | Typical accuracy      | Drift                       |
| ------------------ | --------------------- | --------------------------- |
| GNSS (GPS/Galileo) | < 1 second            | None (re-acquired each fix) |
| NTP (via WiFi)     | < 100 ms              | None (re-synchronised)      |
| RTC (crystal)      | ±2 ppm (good crystal) | ~1 minute per year          |
| RTC (internal RC)  | ±1-5% (uncalibrated)  | Minutes per day             |

Sensors using a free-running RTC with no external synchronisation will
accumulate drift. The 5-second resolution of the datetime field means that RTC
drift below 5 seconds is invisible, but over days or weeks the drift becomes
significant. The gateway can detect and report drift by comparing the sensor's
datetime against its own receive timestamp (Section H.3).

### H.5. Data Pipeline Considerations

The gateway's output — typically JSON records — feeds into a data pipeline for
storage, analysis, and visualisation. The following considerations apply to the
pipeline design:

**Idempotent ingestion.** The combination of {station_id, sequence} is unique
per transmission. The ingestion layer SHOULD use this as a deduplication key,
ensuring that duplicate deliveries (from multi-gateway deployments, message
broker retries, or pipeline replays) do not create duplicate records.

**Schema evolution.** When new field types or variants are introduced, the
upstream schema must accommodate new JSON keys. A schema-on-read approach (e.g.
storing the full JSON document in a document store or a JSONB column) is more
resilient to evolution than a rigid relational schema with a column per field.

**Backfill and reprocessing.** If the binary packets are stored alongside (or
instead of) the decoded JSON, the pipeline can be reprocessed when decoder bugs
are fixed or new field interpretations are added. Storing the raw hex alongside
the decoded output is inexpensive (typically 16–32 bytes per record) and
provides an authoritative source of truth.

**Retention.** Environmental monitoring data is typically retained for years or
decades. At one packet per minute per station, a 16-station deployment produces
approximately 8.4 million records per year — modest by time-series database
standards.

### H.6. Multi-Gateway Deployments

Deployments with multiple gateways provide redundancy and extended coverage but
introduce coordination requirements.

**Duplicate suppression.** A sensor within range of two gateways will be
received by both. Each gateway independently decodes and delivers the data,
producing duplicate records upstream. See Appendix G, Section J.1 for
gateway-to-gateway dedup strategies. For non-mesh deployments, upstream dedup on
{station_id, sequence} in the ingestion layer is the simplest and most robust
approach.

**Gateway identity.** Each gateway SHOULD tag its output with its own identity
(station_id, hostname, or other identifier) so that the upstream system can
distinguish which gateway received each packet. This is essential for link
quality analysis — a packet received by gateway A at -90 dBm and gateway B at
-110 dBm indicates the sensor is closer to A.

**Failover.** If one gateway fails, sensors within range of both gateways
continue to be received by the surviving gateway with no data loss. Sensors
within range of only the failed gateway are lost until the gateway is restored
or a mesh relay is deployed to bridge the gap.

## Appendix I. Comparison with Alternative Encodings and Embedded Libraries

This appendix compares the iotdata protocol along two dimensions: first, against
alternative wire encodings for the same sensor payload; second, against
established embedded C libraries that share iotdata's cross-platform,
low-resource design philosophy even though they solve different problems.

### I.1. Test Payload

The following sensor reading is used for all encoding comparisons:

| Field            | Value    |
| ---------------- | -------- |
| Battery level    | 84%      |
| Battery charging | false    |
| Link RSSI        | -96 dBm  |
| Link SNR         | 10 dB    |
| Temperature      | 21.5°C   |
| Pressure         | 1013 hPa |
| Humidity         | 45%      |
| Wind speed       | 5.0 m/s  |
| Wind direction   | 180°     |
| Wind gust        | 8.5 m/s  |
| Rain rate        | 3 mm/hr  |
| Rain drop size   | 2.5 mm   |
| Solar irradiance | 850 W/m² |
| UV index         | 7        |

This is a Presence Byte 0 only packet (no position, datetime, radiation, or TLV
data). It represents the most common transmission for a weather station.

### I.2. Generic Serialisation Formats

#### I.2.1. iotdata (this protocol)

Header (32 bits) + Presence Byte 0 (8 bits) + Battery (6) + Link (6) +
Environment (24) + Wind (22) + Rain (12) + Solar (14) = 124 bits = **16 bytes**
(after zero-padding the final byte).

```text
Hex: 00 2A C3 50 FF D5 EB 95 BA 2F 52 8A 35 28 70 00
     [header ] [p][bat+lnk+environment ][wind     ]...
```

#### I.2.2. JSON (compact, no whitespace)

```json
{
  "battery": { "level": 84, "charging": false },
  "link": { "rssi": -96, "snr": 10.0 },
  "environment": { "temperature": 21.5, "pressure": 1013, "humidity": 45 },
  "wind": { "speed": 5.0, "direction": 180, "gust": 8.5 },
  "rain": { "rate": 3, "size": 2.5 },
  "solar": { "irradiance": 850, "ultraviolet": 7 }
}
```

**261 bytes.** With keys shortened to single characters: ~130 bytes. Even
aggressively minified JSON is 8× larger than iotdata.

#### I.2.3. CBOR (Concise Binary Object Representation, RFC 8949)

CBOR encodes the same structure as a map of maps with integer keys. Using
single-byte integer keys for all fields:

- Map overhead: ~14 bytes (outer map + 6 inner maps)
- Values: ~40 bytes (integers and floats in their minimal CBOR representation)
- Keys: ~12 bytes (single-byte integer keys)

**~66 bytes.** CBOR's self-describing nature adds per-field type tags and
lengths. It is approximately 4× larger than iotdata for this payload.

#### I.2.4. Protocol Buffers (Protobuf, varint encoding)

A Protobuf message with field numbers and varint/fixed encoding:

- Field tags: ~12 bytes (one per field, varint-encoded field number + wire type)
- Values: ~30 bytes (varints for integers, fixed32 for floats)
- No nested message overhead if flattened

**~42 bytes** (flattened). With nested messages matching the JSON structure:
**~52 bytes.** Protobuf is approximately 2.5–3× larger than iotdata. The
overhead comes from per-field tags and byte-aligned varint encoding.

#### I.2.5. MessagePack

MessagePack with integer keys produces results comparable to CBOR:

**~62 bytes.** Slightly smaller than CBOR due to more compact map headers.
Approximately 4× larger than iotdata.

#### I.2.6. Raw C struct (packed)

```c
struct __attribute__((packed)) weather_packet {
    uint8_t  battery_level;     /* 1 byte (wastes 8 bits for a 0-100 value) */
    uint8_t  battery_charging;  /* 1 byte (wastes 7 bits for a boolean)     */
    int8_t   link_rssi;         /* 1 byte */
    int8_t   link_snr;          /* 1 byte */
    int16_t  temperature;       /* 2 bytes (×100 fixed point) */
    uint16_t pressure;          /* 2 bytes */
    uint8_t  humidity;          /* 1 byte */
    uint16_t wind_speed;        /* 2 bytes (×100) */
    uint16_t wind_direction;    /* 2 bytes */
    uint16_t wind_gust;         /* 2 bytes (×100) */
    uint8_t  rain_rate;         /* 1 byte */
    uint8_t  rain_size;         /* 1 byte (×10) */
    uint16_t solar_irradiance;  /* 2 bytes */
    uint8_t  uv_index;          /* 1 byte */
};
```

**20 bytes** (plus typically 4 bytes of header for station ID and sequence, = 24
bytes total). The packed C struct is the closest generic competitor in size.
However, it wastes bits on byte alignment (the boolean charging flag consumes 8
bits instead of 1, battery level uses 8 bits instead of 5, etc.) and has no
presence flag mechanism — all fields are always transmitted regardless of
whether they have changed or are relevant.

### I.3. IoT-Specific Encodings

The generic formats above (JSON, CBOR, Protobuf) are general-purpose
serialisation. The following libraries and formats were designed specifically
for IoT sensor telemetry or for bit-efficient encoding on constrained devices.

#### I.3.1. CayenneLPP

CayenneLPP (Cayenne Low Power Payload) is the de facto standard payload format
for LoRaWAN sensor devices, natively supported by The Things Network and
myDevices Cayenne. It uses a byte-aligned TLV structure where each measurement
is prefixed with a 1-byte channel identifier and a 1-byte IPSO-derived type
code.

CayenneLPP defines standard types for temperature (2 bytes, 0.1°C), barometric
pressure (2 bytes, 0.1 hPa), relative humidity (1 byte, 0.5%), luminosity (2
bytes, 1 lux), and GPS (9 bytes). Fields without a standard type must use the
generic analog input (2 bytes, 0.01 resolution).

Encoding the test payload:

| Field            | CayenneLPP type   | Bytes (ch+type+data) |
| ---------------- | ----------------- | -------------------: |
| Battery level    | Analog Input      |                    4 |
| Battery charging | Digital Input     |                    3 |
| Link RSSI        | Analog Input      |                    4 |
| Link SNR         | Analog Input      |                    4 |
| Temperature      | Temperature       |                    4 |
| Pressure         | Barometric Press. |                    4 |
| Humidity         | Rel. Humidity     |                    3 |
| Wind speed       | Analog Input      |                    4 |
| Wind direction   | Analog Input      |                    4 |
| Wind gust        | Analog Input      |                    4 |
| Rain rate        | Analog Input      |                    4 |
| Rain size        | Analog Input      |                    4 |
| Solar irradiance | Luminosity        |                    4 |
| UV index         | Analog Input      |                    4 |

**~54 bytes.** No header (station ID and sequencing are provided by LoRaWAN).
Adding an equivalent 4-byte header for a fair comparison gives ~58 bytes.

CayenneLPP's strengths are its native integration with the LoRaWAN ecosystem
(The Things Network decodes CayenneLPP payloads automatically, with no custom
code) and its simplicity (a C++ Arduino library with `lpp.addTemperature()`
calls). Its weaknesses are the 2-byte overhead per field (channel + type), byte
alignment that wastes bits, no sub-byte field packing, and reliance on the
generic analog input type for any measurement not in the IPSO standard set —
which loses semantic meaning at the gateway.

The CayenneLPP constructor (`CayenneLPP lpp(51)`) uses `malloc` for its internal
buffer. The library is Arduino/C++ and is not straightforwardly portable to
bare-metal C on Class 1/2 MCUs.

#### I.3.2. Nanopb (Protocol Buffers for Embedded C)

Nanopb is a widely-used Protocol Buffers implementation targeting embedded
systems, written in ANSI C. It supports static allocation (no `malloc` at
runtime when configured with `.options` files), compiles to 2–10 KB ROM and ~300
bytes–1 KB RAM, and runs on STM32, AVR, ARM Cortex-M, and Linux.

The wire format is standard Protobuf: varint-encoded field tags and byte-aligned
varint/fixed-width values. For the test payload, the encoding size is identical
to the generic Protobuf figure:

**~42 bytes** (flattened message), **~52 bytes** (nested messages).

Nanopb's strengths are its maturity (widely deployed, well-tested), its
interoperability with the broader Protobuf ecosystem (the same `.proto` schema
can generate code for C, Python, Go, Java, etc.), and its static allocation
mode. Its weaknesses for this use case are the requirement for a code generator
(`protoc` plus a Python plugin), byte-aligned encoding (no sub-byte fields), and
the overhead of per-field tags — which exist to support schema evolution but are
redundant in a closed deployment where both sides know the schema.

Nanopb is the strongest alternative for deployments that require
interoperability with cloud services or multi-language environments where the
Protobuf ecosystem is already established. For closed LoRa deployments where
every bit matters, the 2.5–3× size overhead relative to iotdata is significant.

#### I.3.3. Bitproto

Bitproto is a bit-level serialisation format with a Protobuf-like schema
language. It is the closest existing tool to iotdata's approach: fields are
specified at arbitrary bit widths (`uint3`, `uint5`, `uint11`, etc.), and the
generated C encoder/decoder uses zero dynamic allocation and copies bits between
structures and buffers without padding or gaps.

If the test payload were defined in Bitproto with the same bit widths as iotdata
(5-bit battery, 1-bit charging, 4-bit RSSI, etc.), the data portion would occupy
a similar number of bits. However, Bitproto adds a 2-byte message size header
when extensibility is enabled, and does not provide a built-in header (variant,
station ID, sequence) or presence flags.

Encoding the test payload with equivalent bit widths:

**~14 bytes** (data bits only, no extensibility header), **~16 bytes** (with
2-byte size header). Adding the equivalent 4-byte iotdata header and a 1-byte
presence byte gives ~21 bytes — comparable to iotdata's 16 bytes, with the
difference coming from the lack of presence flags (all fields are always
transmitted) and the size header.

Bitproto's strengths are its bit-level granularity (identical to iotdata in
principle), zero dynamic allocation in C, and the availability of a code
generator for C, Go, and Python. Its limitations for this use case are:

- **Little-endian only.** Bitproto encodes in little-endian byte order,
  requiring byte-swap logic on big-endian platforms (some PIC configurations,
  network-order protocols). iotdata is endian-agnostic via explicit bit-by-bit
  packing.
- **No presence flags.** Every field defined in the schema is always encoded.
  There is no mechanism equivalent to iotdata's presence bytes, so a
  battery-only packet is the same size as a full-telemetry packet.
- **No variable-length fields.** All types are fixed-width. Bitproto cannot
  express variable-length constructs such as iotdata's TLV section or the Air
  Quality PM/Gas fields with per-channel presence masks.
- **No quantisation.** Bitproto packs raw bit fields; the quantisation (mapping
  physical values to reduced-bit representations) must be implemented by the
  application. iotdata defines the quantisation as part of the protocol.
- **Requires a code generator.** The `bitproto` compiler (Python) must be run to
  generate C source files from `.bitproto` schema files. iotdata's reference
  implementation is self-contained C with no code generation step.
- **No domain awareness.** Bitproto is a generic bit-packing tool. It has no
  concept of sensor types, field bundles, or variant maps.

#### I.3.4. TinyCBOR and QCBOR

TinyCBOR (developed by Intel) and QCBOR (developed by Qualcomm/Laurence
Lundblade) are embedded-focused CBOR implementations. Both avoid `malloc` in
their core encode/decode paths (TinyCBOR operates on caller-provided buffers;
QCBOR uses a similar model with richer error handling). QCBOR is approximately
25% larger in code size than TinyCBOR but provides more complete CBOR support.

The wire format is standard CBOR (RFC 8949), so the encoding size for the test
payload is the same as the generic CBOR figure: **~66 bytes.** The advantage of
these implementations over generic CBOR libraries is their suitability for
embedded targets: no heap allocation, bounded stack usage, and small code
footprint (~4–8 KB for TinyCBOR, ~10–15 KB for QCBOR).

CBOR's self-describing nature (every value carries its type and length) is the
opposite of iotdata's approach. This makes CBOR ideal for schemaless systems
where the receiver may not know the sender's data model, but adds 4× overhead
for the structured, schema-known payloads that iotdata targets.

### I.4. Encoding Summary

| Encoding          | Bytes | Ratio vs iotdata | Presence flags | Self-describing | Byte-aligned | Code gen required |
| ----------------- | ----: | ---------------: | :------------: | :-------------: | :----------: | :---------------: |
| **iotdata**       |    16 |             1.0× |       ✓        |        ✗        |      ✗       |         ✗         |
| Bitproto†         |   ~16 |            ~1.0× |       ✗        |        ✗        |      ✗       |         ✓         |
| Raw C struct      |    24 |             1.5× |       ✗        |        ✗        |      ✓       |         ✗         |
| Protobuf / Nanopb |    42 |             2.6× |      ✓\*       |     Partial     |      ✓       |         ✓         |
| CayenneLPP        |    54 |             3.4× |       ✗        |        ✓        |      ✓       |         ✗         |
| MessagePack       |    62 |             3.9× |       ✗        |        ✓        |      ✓       |         ✗         |
| CBOR / TinyCBOR   |    66 |             4.1× |       ✗        |        ✓        |      ✓       |         ✗         |
| JSON (compact)    |   261 |            16.3× |       ✗        |        ✓        |      ✓       |         ✗         |

\*Protobuf omits default/zero values, which functions as implicit presence for
non-zero fields.

†Bitproto data fields only, with equivalent bit widths. With the addition of an
iotdata-equivalent header (4 bytes) and extensibility header (2 bytes), the
total rises to ~22 bytes. Bitproto does not support presence flags, so this
figure applies to every packet regardless of which fields have changed.

### I.5. Analysis

The size advantage of iotdata comes from three sources:

1. **Sub-byte field packing.** Fields are packed to the exact number of bits
   required. A boolean is 1 bit, a battery level is 5 bits, a wind direction is
   8 bits. Only Bitproto matches this capability among the compared formats.

2. **No per-field metadata.** There are no field tags, type indicators, length
   prefixes, or key strings. The field layout is determined entirely by the
   variant table, which both sides know at compile time. CayenneLPP's 2-byte
   per-field overhead (channel + type) and Protobuf's varint field tags exist to
   support self-description and schema evolution — valuable properties, but
   expensive when every bit counts.

3. **Quantisation to operational resolution.** Temperature is quantised to
   0.25°C steps, fitting in 9 bits. A Protobuf float or CBOR float uses 32 bits
   for the same value. The quantisation is chosen to be within or below the
   sensor's own measurement accuracy, so no operationally useful information is
   lost.

The trade-off is the loss of self-description. An iotdata packet cannot be
decoded without prior knowledge of the variant table. For the target use case —
closed deployments where the operator controls all devices — this is acceptable.
For open or interoperable systems, a self-describing format like CBOR or
Protobuf would be more appropriate, at the cost of 3–4× larger payloads. The
CayenneLPP format occupies an intermediate position: it is self-describing and
has LoRaWAN ecosystem support, but at 3.4× the size of iotdata, the cost is
significant for duty-cycle-constrained deployments.

Alternatively, the specific variant table could be determined from the device's
VERSION or CONFIG information, as transmitted at startup.

### I.6. Embedded Library Design Comparison

The preceding sections compare iotdata against alternative _encodings_ — they
answer "what else could I use to pack sensor telemetry onto a wire?" This
section addresses a different question: "as an embedded C library designed to
work from Cortex-M0 to Linux, how does iotdata's architecture compare to the
best-practice embedded libraries that solve other problems?"

The embedded C ecosystem contains several libraries that are widely regarded as
exemplars of portable, resource-conscious design. Although they address entirely
different domains — networking, filesystems, compression, cryptography, parsing
— they share a set of architectural principles that iotdata also follows. This
comparison positions iotdata's design choices within that tradition and
identifies where iotdata conforms to, or deviates from, established practice.

#### I.6.1. Reference Libraries

**lwIP** (lightweight IP) is an open-source TCP/IP stack created by Adam
Dunkels, targeting embedded systems with tens of kilobytes of RAM. It is used by
Espressif (ESP32/ESP-IDF), STMicroelectronics, NXP, Xilinx, and many others.
Code size is approximately 40 KB ROM; RAM usage is 10–30 KB depending on
configuration. Configuration is via a user-provided `lwipopts.h` header that
selects protocols, buffer sizes, and API style at compile time.

**littlefs** is a fail-safe filesystem designed for microcontrollers and
NOR/NAND flash. It provides power-loss resilience, dynamic wear levelling, and
bad block detection while maintaining strictly bounded RAM and ROM usage. It
avoids recursion, limits dynamic memory to configurable caller-provided buffers,
and at no point stores an entire storage block in RAM.

**heatshrink** is an LZSS-based compression library for embedded and real-time
systems. It operates with as little as 50 bytes of RAM, processes data
incrementally in arbitrarily small chunks, supports both static and dynamic
allocation, and separates the encoder and decoder into independently compilable
units. iotdata already uses heatshrink for image compression (Section 8.27.2).

**Mbed TLS** (formerly PolarSSL) is a C implementation of TLS/DTLS and
cryptographic primitives. Its minimum TLS stack requires under 60 KB ROM and
under 64 KB RAM. It is highly modular: individual cryptographic algorithms can
be used independently of the TLS stack. Feature selection is via a compile-time
configuration header (`mbedtls_config.h`).

**wolfSSL / wolfCrypt** is an embedded SSL/TLS library written in ANSI C,
targeting RTOS and resource-constrained environments. With the LeanPSK
configuration, it compiles to as little as 20 KB. It supports extensive hardware
cryptographic acceleration and compile-time algorithm selection.

**minmea** is a minimalistic GPS NMEA 0183 parser in pure ISO C99. It consists
of a single source file and header, uses no dynamic memory allocation, performs
no floating-point arithmetic in the core library (offering both fixed-point and
float output), and runs on embedded ARM, Linux, macOS, and Windows.

#### I.6.2. Design Principle Comparison

The following table maps six architectural principles common to high-quality
embedded C libraries against the reference libraries and iotdata.

| Principle                        | lwIP               | littlefs                  | heatshrink               | Mbed TLS             | minmea               | iotdata                  |
| -------------------------------- | ------------------ | ------------------------- | ------------------------ | -------------------- | -------------------- | ------------------------ |
| **Compile-time modularity**      | `lwipopts.h`       | Config struct             | `heatshrink_config.h`    | `mbedtls_config.h`   | Linker GC            | `IOTDATA_ENABLE_*`       |
| **Zero malloc / caller buffers** | Pool allocator     | Caller-provided           | Static or dynamic        | Caller-provided      | Stack only           | Stack only, no malloc    |
| **Integer-only capability**      | N/A (networking)   | Yes (all integer)         | Yes (all integer)        | Yes (bignum library) | Core is integer-only | `IOTDATA_NO_FLOATING`    |
| **Separable components**         | Raw/Netconn/Socket | Single unit               | Encode ≠ Decode          | Crypto ≠ TLS ≠ X.509 | Single unit          | Encode ≠ Decode ≠ JSON   |
| **Platform abstraction**         | OS emulation layer | Block device API          | None needed              | Platform ALT layer   | Compat headers       | `<stdint.h>` only        |
| **Bounded resource usage**       | Configurable pools | No recursion, bounded RAM | Incremental, bounded CPU | Configurable buffers | Fixed struct sizes   | Compile-time-known sizes |

**Compile-time modularity.** The ability to include only the features a
deployment needs, with the compiler eliminating unused code. lwIP pioneered this
with `lwipopts.h`, a user-created header that overrides defaults for every
tunable parameter. Mbed TLS uses the same pattern with `mbedtls_config.h`.
iotdata follows this tradition with `IOTDATA_ENABLE_SELECTIVE` and per-field
`IOTDATA_ENABLE_*` defines, plus functional subsetting (`IOTDATA_NO_DECODE`,
`IOTDATA_NO_JSON`, etc.). The result is that a minimal iotdata encoder
(battery + environment only) compiles to 768 bytes on ESP32-C3 — comparable to
heatshrink's decoder at ~1 KB on AVR.

**Zero malloc / caller-provided buffers.** Dynamic memory allocation is avoided
or prohibited because it introduces fragmentation, non-deterministic timing, and
failure modes that are unacceptable in embedded systems (particularly those
governed by MISRA C or similar standards). littlefs achieves this by requiring
the caller to provide a configuration struct with buffer pointers. heatshrink
supports both modes — static allocation for embedded, dynamic for convenience on
hosted platforms. iotdata's encode and decode paths perform no `malloc` or
`free` calls; the `iotdata_encoder_t` context is allocated on the caller's stack
or as a static variable. The only heap allocation in the library is within the
JSON conversion functions (`cJSON_CreateObject`), which are gateway-only and
excluded from embedded builds via `IOTDATA_NO_JSON`.

**Integer-only capability.** Many Class 1 and Class 2 MCUs lack a hardware FPU.
Software floating-point emulation adds 2–5 KB of code and ~50–100 cycles per
operation. minmea addresses this by keeping its core parser integer-only, using
a `struct minmea_float` that stores values as a numerator/denominator pair
(`int_least32_t`) and offering an explicit `minmea_tocoord()` conversion for
callers that need floating-point output. iotdata's `IOTDATA_NO_FLOATING` mode
follows the same philosophy: all values are passed as scaled integers
(temperature as centidegrees, position as degrees×10⁷), eliminating all
floating-point dependencies.

**Separable components.** heatshrink's encoder and decoder are independently
compilable — an embedded device that only compresses data need not include the
decompressor. Mbed TLS separates into three libraries: `libtfpsacrypto` (raw
cryptographic primitives), `libmbedx509` (certificate handling), and
`libmbedtls` (TLS/DTLS protocol). An application that needs only AES encryption
links against the crypto library alone. iotdata provides the same separation:
`IOTDATA_NO_DECODE` excludes the decoder, `IOTDATA_NO_JSON` excludes JSON, and
`IOTDATA_NO_PRINT` / `IOTDATA_NO_DUMP` exclude diagnostic output. An
encoder-only build for a sensor node includes none of the decoder, print, dump,
or JSON machinery.

**Platform abstraction without OS dependency.** lwIP defines an operating system
emulation layer (`sys_arch`) that provides semaphores, mailboxes, and threads,
with a bare-metal implementation that uses polling. littlefs abstracts storage
behind a block device API (`lfs_config` with read/prog/erase function pointers).
iotdata has the lightest abstraction of all: it depends only on `<stdint.h>`,
`<stdbool.h>`, `<stddef.h>`, and optionally `<math.h>` (for `round()`/`floor()`
in floating-point mode). No OS services, no file I/O, no timers, no threading.
The bit-packing core operates on a caller-provided `uint8_t` buffer and a bit
offset — it is portable to any byte-addressable architecture.

**Bounded, predictable resource usage.** littlefs guarantees that its RAM
consumption is bounded regardless of filesystem size — it never stores an entire
flash block in RAM, avoids recursion (which would produce data-dependent stack
growth), and limits dynamic memory to configurable buffers. heatshrink processes
data in incremental chunks with bounded CPU time per call, making it suitable
for hard real-time contexts. iotdata's `encode_end()` function performs a single
linear pass over the variant field table; its execution time is proportional to
the number of present fields (maximum ~300 bit operations for all 12 fields),
with no data-dependent loops except TLV string encoding.

#### I.6.3. Positioning

iotdata sits at an intersection that is unusual in the embedded library
landscape. Most IoT encoding libraries (CayenneLPP, Nanopb) do not follow all of
the embedded design principles above — CayenneLPP uses `malloc`, Nanopb requires
a code generator and an external toolchain dependency. Conversely, most
libraries that rigorously follow these principles (lwIP, littlefs, heatshrink)
are not encoding libraries — they solve networking, storage, or compression
problems.

iotdata applies the architectural discipline of the best embedded C libraries to
the specific problem of sensor telemetry encoding. The design choices — compile-
time field selection, caller-provided buffers, integer-only mode, separable
encode/decode, no OS dependency, bounded resource usage — are individually
unremarkable (they are standard practice in the embedded ecosystem). Their
combination in a sensor telemetry protocol is less common, because the IoT
encoding space has historically been dominated by formats designed for
flexibility and interoperability (Protobuf, CBOR, CayenneLPP) rather than for
the architectural constraints that govern embedded library design.

This positioning has costs. iotdata lacks the ecosystem integration of
CayenneLPP (no native TTN/Cayenne support), the schema evolution guarantees of
Protobuf (no field tags, no wire-level versioning), the self-description of CBOR
(packets cannot be decoded without the variant table), and the multi-language
code generation of Bitproto and Nanopb (the reference implementation is C only).
These trade-offs are deliberate: they are the price of a library that compiles
to 768 bytes on a RISC-V MCU and produces 16-byte packets for a full weather
station reading.

### I.7. Impact on LoRa Airtime and Battery Life

The size difference has direct operational consequences on LoRa:

| Encoding   | Bytes | Airtime (SF7/125kHz) | Airtime (SF10/125kHz) | Fits SF12? |
| ---------- | ----: | -------------------: | --------------------: | :--------: |
| iotdata    |    16 |               ~36 ms |               ~247 ms |     ✓      |
| Bitproto†  |   ~22 |               ~41 ms |               ~289 ms |     ✓      |
| C struct   |    24 |               ~46 ms |               ~370 ms |     ✓      |
| Protobuf   |    42 |               ~72 ms |               ~617 ms |    ✓\*     |
| CayenneLPP |    54 |               ~92 ms |               ~781 ms |     ✗      |
| CBOR       |    66 |              ~107 ms |               ~925 ms |     ✗      |
| JSON       |   261 |              ~369 ms |                     ✗ |     ✗      |

\*Protobuf at 42 bytes fits the SF12/125kHz maximum payload of 51 bytes, but
with minimal room for header overhead.

†Bitproto with iotdata-equivalent header and extensibility header.

At SF10 with a 1% duty cycle, the minimum transmission interval for iotdata is
25 seconds. For CayenneLPP, it is 78 seconds — over 3× longer between
transmissions for the same regulatory budget. For battery-powered sensors where
radio transmission dominates power consumption, this difference translates
directly to battery life.

CayenneLPP's 54-byte payload exceeds the SF12/125kHz maximum of 51 bytes,
meaning it cannot be used at the highest spreading factor for the test payload
without dropping fields. iotdata's 16 bytes fit comfortably at all spreading
factors, with headroom for TLV data even at SF12.

## Appendix J. Known Limitations and Open Issues

This appendix documents known limitations, unresolved design questions, and
areas where the current specification reflects engineering judgement rather than
systematic analysis. These items are recorded to inform implementers of known
risks and to scope the work required before the protocol advances beyond its
current pre-release status.

The protocol is currently at version 0.90 (alpha/beta). Breaking changes to
field encodings, header layout, and quantisation parameters are expected before
the protocol is finalised as version 1.0. Implementers SHOULD be aware that
deploying the current specification may require firmware updates when these
issues are resolved. The items in this appendix will be systematically addressed
— resolved, accepted with justification, or deferred — before the protocol is
submitted as an Internet-Draft.

### J.1. Pressure Range

**Issue:** The pressure field (Section 8.3, 8.10) encodes 850–1105 hPa in 8 bits
at 1 hPa resolution. This range excludes altitudes above approximately 1,500
metres, where standard atmospheric pressure falls below 850 hPa.

**Impact:** Deployments at moderate altitude are affected. A weather station at
1,600 metres (e.g. Davos, Switzerland, ~840 hPa) cannot encode its pressure
readings. Mountain agriculture, ski resort monitoring, and high-altitude
research stations — all plausible use cases for this protocol — are excluded by
the current range.

**Context:** The BME280 sensor, explicitly named in the specification, measures
300–1100 hPa. The protocol's range covers only the upper third of the sensor's
capability.

**Options under consideration:**

| Option            | Bits | Range        | Resolution | Notes                                                     |
| ----------------- | ---- | ------------ | ---------- | --------------------------------------------------------- |
| Current           | 8    | 850–1105 hPa | 1 hPa      | Excludes altitudes above ~1,500m                          |
| Wider range       | 10   | 300–1100 hPa | ~0.8 hPa   | Covers full BME280 range; 2 extra bits                    |
| Wider, coarser    | 8    | 300–1100 hPa | ~3.1 hPa   | Same bit width; resolution exceeds BME280 ±1 hPa accuracy |
| Wider, compromise | 9    | 540–1066 hPa | 1 hPa      | Covers altitudes to ~5,000m; 1 extra bit                  |

**Recommendation:** This is a likely breaking change before v1.0. The current
range is too restrictive for the protocol's stated deployment scenarios.

### J.2. Wind Speed Range and Resolution

**Issue:** Wind speed (Section 8.12, 8.13) is encoded as 0–63.5 m/s in 7 bits at
0.5 m/s resolution. The maximum corresponds to the onset of Beaufort 12
(hurricane force). Actual hurricane and cyclone wind speeds reach 85+ m/s;
tornado wind speeds exceed 100 m/s.

**Impact:** Weather stations that survive extreme wind events will saturate the
field. The encoder clamps to 63.5 m/s, and the receiver cannot distinguish "63.5
m/s" from "90 m/s." For stations deployed specifically to monitor severe
weather, this is a data loss.

**Analysis needed:** Whether the 0.5 m/s resolution is operationally justified.
Most consumer and semi-professional anemometers (Davis Vantage Pro2, Inspeed
Vortex) have ±1 m/s accuracy or ±5% of reading. At 10 m/s, ±5% is ±0.5 m/s, so
the 0.5 m/s resolution is at the sensor's accuracy limit. At 30 m/s, ±5% is ±1.5
m/s, and the 0.5 m/s resolution is well below the noise floor.

**Options under consideration:**

| Option            | Bits | Range       | Resolution | Notes                                      |
| ----------------- | ---- | ----------- | ---------- | ------------------------------------------ |
| Current           | 7    | 0–63.5 m/s  | 0.5 m/s    | Saturates at hurricane onset               |
| Extended range    | 8    | 0–127.5 m/s | 0.5 m/s    | Covers all terrestrial wind; 1 extra bit   |
| Extended, coarser | 7    | 0–127 m/s   | 1.0 m/s    | Same bit width; resolution matches sensors |
| Non-linear        | 7    | 0–127+ m/s  | Variable   | Finer below 30 m/s, coarser above; complex |

The same analysis applies to the wind gust field (Section 8.15), which shares
the encoding.

### J.3. Linear Quantisation vs. Real-World Distributions

**Issue:** All field encodings use linear quantisation, allocating equal
resolution across the entire range. Many environmental measurements have heavily
right-skewed distributions where most readings cluster at low values with rare
high excursions.

**Affected fields:**

- **Wind speed:** Most readings are 0–10 m/s; extreme values above 30 m/s are
  rare but operationally significant.
- **Rain rate:** Most readings are 0–10 mm/hr; extreme events reach 100+ mm/hr.
- **Radiation CPM:** Background is 10–50 CPM; elevated readings are 100+;
  emergency-level readings are 1,000+.
- **Radiation dose:** Background is 0.05–0.20 µSv/h; the 0.01 µSv/h resolution
  provides only 5–20 distinguishable levels in the normal background range,
  while 16,363 of the 16,383 steps cover values that will never be observed in
  normal operation.
- **Air quality PM:** Background PM2.5 in clean environments is 0–10 µg/m³; the
  5 µg/m³ resolution provides only 2 distinguishable levels (0 and 5) in this
  range.

**Analysis needed:** For each field, characterise the real-world distribution of
values (from public meteorological and environmental datasets) and compute the
information content per quantisation step. Compare linear quantisation against
logarithmic, square-root, and piecewise-linear alternatives in terms of
mean-squared quantisation error for typical measurement distributions.

**Trade-off:** Non-linear quantisation improves effective resolution in common
value ranges but adds implementation complexity. Linear quantisation requires
one multiply and one add; logarithmic quantisation requires a lookup table or a
log/exp computation. On Class 1 MCUs (Section E.1), this is a meaningful cost.
Additionally, non-linear quantisation violates the principle of least surprise —
a user examining the raw quantised value cannot easily estimate the physical
value without consulting the transfer function.

**Current position:** Linear quantisation is retained in v0.90 for simplicity. A
systematic analysis of the quantisation error budget against real sensor data is
planned before v1.0.

### J.4. Datetime Resolution vs. Bit Allocation

**Issue:** The datetime field (Section 8.8) uses 24 bits at 5-second resolution,
covering 971 days. The 5-second tick was chosen because 24 bits at 1-second
resolution covers only 194 days — insufficient for a calendar year.

**Observation:** The protocol operates at arbitrary bit boundaries throughout. A
25-bit datetime field at 1-second resolution covers 388 days — sufficient for a
full year with 23 days of margin. The cost is 1 additional bit per packet when
the datetime field is present. For sensors with GNSS time sources (sub-second
accurate), the current design degrades accuracy by up to 4 seconds to preserve a
round bit count that has no structural significance in a bit-packed protocol.

**Counter-argument:** The 5-second resolution exceeds the typical sensor
observation cycle (tens of seconds to minutes). For a sensor that wakes, reads,
encodes, and transmits once per minute, a ±4 second timestamp error is
negligible. The 24-bit width also provides 971-day coverage, which is convenient
for resolving year boundaries (Section 11.1) — a 25-bit field at 1-second
resolution would require the year-boundary algorithm to handle timestamps up to
23 days into the next year rather than 241 days.

**Analysis needed:** Survey of sensor timing requirements across target use
cases. Determine whether any use case benefits materially from 1-second vs.
5-second resolution, given that the protocol does not provide sub-second
timestamps in either case.

### J.5. Header Bit Allocation

**Issue:** The 32-bit header allocates 4 bits to variant (16 values), 12 bits to
station ID (4,096 values), and 16 bits to sequence (65,536 values). The
specification recommends 100–200 nodes per deployment, meaning approximately 95%
of the station ID space is typically unused. The sequence field wraps every ~3.6
days at 5-second intervals or ~45 days at 60-second intervals.

**Observation:** A 24-bit header with 4 bits variant, 8 bits station ID (256
values), and 12 bits sequence (4,096 values before wrap) would save 8 bits per
packet — a 17% reduction in minimum packet size (from 46 to 38 bits). The
reduced station ID space (256) still exceeds the recommended deployment size.
The reduced sequence space wraps every ~5.7 hours at 5-second intervals, which
is adequate for dedup (where the relevant window is seconds, not hours) but
reduces the gap detection window.

**Counter-argument:** The 32-bit header aligns to a 4-byte boundary, which
simplifies implementation on platforms where the header is read as a single
uint32_t. It also provides headroom for larger deployments and longer gap
detection windows without protocol changes. The 8-bit saving per packet is
meaningful for the shortest packets (battery-only heartbeats) but diminishes in
significance for typical 16–32 byte weather station packets.

**Analysis needed:** Survey of real deployment sizes and transmission intervals
to determine whether the current allocation is well-matched or over-provisioned.
Consider whether a configurable header format (selected by variant or by a
deployment-wide parameter) could serve both small and large deployments without
a fixed compromise.

The specification already acknowledges this issue (Section 5, final paragraph)
and notes that a reduction is "not contemplated in this version."

### J.6. Presence Byte TLV Bit Placement

**Issue:** Bit 6 of Presence Byte 0 is permanently allocated to the TLV flag.
Every packet pays this bit regardless of whether TLV data is present. In the
common case (no TLV data), this bit is always 0 and conveys no information.

**Impact:** Presence Byte 0 has 6 data field slots (bits 0–5) rather than 7. A
variant with 7 frequently-transmitted fields must use an extension byte for the
seventh field, adding 8 bits to every packet. If the TLV bit were relocated, 7
fields could fit in a single presence byte.

**Alternatives:**

- **TLV as an extension byte sentinel.** TLV data could be signalled by a
  specific extension bit pattern (e.g. an extension byte where all data field
  bits are 0) rather than a dedicated bit.
- **TLV as a field type.** A field type IOTDATA_FIELD_TLV could be defined,
  occupying a field slot in the variant table. Variants that need TLV would
  allocate a slot; variants that don't would reclaim the slot.
- **Accept the cost.** The current design is simple, unambiguous, and costs at
  most 1 bit per packet. The number of deployments that need exactly 7 fields in
  pres0 may be small.

**Analysis needed:** Survey of planned and potential variant definitions to
determine how many would benefit from a 7th slot in pres0.

### J.7. Rain Drop Size Semantics

**Issue:** The rain size field (Section 8.18) encodes a value in the range 0–6.0
mm at 0.25 mm resolution, but the specification does not define what physical
quantity this represents.

Meteorological raindrop measurements come in several forms:

- **Median volume diameter (D₀ or D₅₀):** The drop diameter at which half the
  total volume is in smaller drops and half in larger. This is the standard
  descriptor for a raindrop size distribution.
- **Mean diameter:** Arithmetic mean of all detected drops.
- **Maximum detected diameter:** The largest single drop in the observation
  period.
- **Modal diameter:** The most common drop size.

These quantities differ significantly for the same rain event. A moderate
stratiform rain might have D₅₀ = 1.5 mm, mean = 1.0 mm, and maximum = 3.5 mm.

**Impact:** Without a defined physical quantity, the field is ambiguous. Two
different sensor implementations might encode different quantities using the
same field, producing incomparable data.

**Additionally:** The 0.25 mm resolution is coarse for the scientifically
interesting region below 2 mm, where the Marshall-Palmer distribution
concentrates most drops. The field's utility for meteorological research is
limited unless the resolution is improved or the target application (coarse
operational classification rather than scientific measurement) is stated.

**Recommendation:** Define the field as encoding the median volume diameter
(D₀), or explicitly state that the quantity is implementation-defined and
intended for coarse classification rather than research-grade measurement.

### J.8. UV Index Range

**Issue:** The UV index field (Section 8.4) is encoded as 4 bits, range 0–15.
The WHO UV Index scale is nominally 0–11+ with values above 11 classified as
"extreme." Measured UV indices above 15 are documented at high altitude near the
equator (Andes, Tibetan Plateau), with readings up to 20+ recorded by research
stations.

**Impact:** Minimal for the majority of deployments. The ceiling of 15
accommodates all but the most extreme high-altitude equatorial conditions.
Deployments targeting such environments would saturate the field.

**Recommendation:** Accept as a known limitation for v1.0, with a note that
5-bit encoding (range 0–31) would cover all documented terrestrial UV conditions
at a cost of 1 additional bit.

### J.9. Information-Theoretic Efficiency

**Issue:** The protocol claims bit efficiency as its primary design goal, and
the comparison with alternative encodings (Appendix I) demonstrates that iotdata
is significantly more compact than JSON, CBOR, Protobuf, and packed C structs.
However, the specification does not establish how close the encoding is to the
theoretical minimum.

**Analysis needed:** For a representative weather station payload (the variant 0
default), compute the Shannon entropy of each field given real-world measurement
distributions. Sum the per-field entropies to obtain the theoretical minimum
number of bits required to represent a typical reading. Compare this against the
actual bit allocation.

For example, if the entropy of a typical reading is 90 bits, then the current
124-bit encoding has 38% overhead and there may be significant room for
improvement. If the entropy is 115 bits, the encoding is within 8% of optimal
and further compression would yield diminishing returns.

This analysis would also identify which fields contribute the most overhead
relative to their information content, guiding any future optimisation of bit
allocations.

**Context:** The protocol deliberately avoids variable-length or
entropy-optimised encodings (Huffman, arithmetic coding) for implementation
simplicity. A fixed bit-packed encoding can never reach the Shannon limit for
variable-entropy data. The analysis would quantify the cost of this design
choice and determine whether it is a few percent (acceptable) or tens of percent
(worth revisiting).

### J.10. Sensor-Specific Field Semantics

**Issue:** Several fields encode physical quantities without specifying the
measurement conditions, averaging periods, or sensor calibration assumptions
that affect their interpretation.

**Examples:**

- **Temperature:** Dry-bulb? Wet-bulb? Aspirated or unaspirated sensor housing?
  The difference between an aspirated and unaspirated temperature reading in
  direct sunlight can be 5–10°C — well above the 0.25°C quantisation resolution.
- **Wind speed and gust:** What averaging period? WMO standard is 10-minute mean
  speed and 3-second gust. The National Weather Service uses 2-minute mean and
  5-second gust. The protocol does not specify, so two stations with different
  averaging periods produce incomparable data.
- **Humidity:** Relative humidity depends on temperature. Is the temperature
  co-located and simultaneous? The Environment bundle (Section 8.3) implies yes,
  but standalone Humidity (Section 8.11) makes no such guarantee.
- **Rain rate:** Instantaneous rate? 1-minute average? 10-minute average?
  Tipping-bucket gauges report discrete tip events; the computed "rate" depends
  entirely on the averaging algorithm.

**Impact:** For closed deployments where the operator controls all sensors and
understands their characteristics, this ambiguity is manageable. For any form of
data sharing or comparison between deployments, it undermines data quality.

**Relationship to Section 11.3 and Section 15:** The specification acknowledges
this gap and identifies sensor metadata TLVs (types 0x10–0x1F) as future work.
This appendix entry records the specific fields where the ambiguity is most
significant to guide the design of those TLVs.

### J.11. Irradiance Range

**Issue:** The solar irradiance field (Section 8.4) encodes 0–1023 W/m² in 10
bits. The solar constant (top-of-atmosphere irradiance) is approximately 1,361
W/m². At the Earth's surface, clear-sky irradiance rarely exceeds 1,100 W/m²,
but localised reflections from clouds (the "cloud enhancement" or "lensing"
effect) can produce transient readings of 1,200–1,400 W/m² as measured by
high-quality pyranometers.

**Impact:** Stations equipped with research-grade pyranometers in environments
prone to cloud enhancement (e.g. tropical cumulus, mountain environments) may
occasionally saturate the field. For most operational deployments with standard
silicon-cell sensors, the 1,023 W/m² ceiling is adequate.

**Recommendation:** Consider 11 bits (0–2,047 W/m², 1 W/m² resolution) for
headroom, or accept the current range with a documented limitation. The
additional bit is a minor cost.

### J.12. Image Field Practicality

**Issue:** The Image field (Section 8.27) is the most complex and largest field
type in the protocol. It supports multiple pixel formats, compression methods,
and sizes, with payload budgets up to 254 bytes. This sits uneasily in a
protocol designed around 16–32 byte packets for resource-constrained sensors.

**Concerns:**

- **Payload budget conflict.** A 96-byte BILEVEL image at 32×24 plus a 16-byte
  weather station payload totals 112 bytes. This fits at SF7 (222 byte limit)
  but not at SF9 (115 bytes) or above. The image effectively precludes higher
  spreading factors, which are often needed precisely in the remote deployments
  where camera-equipped sensors might be deployed.
- **Compression complexity.** Heatshrink decompression requires ~256 bytes of
  RAM — feasible on Class 3 devices but prohibitive on Class 1 and marginal on
  Class 2. This creates a field type that cannot be decoded on the same device
  classes the protocol targets for encoding.
- **Use case validation.** The field was designed for motion-detection
  thumbnails (wildlife, livestock, security). It is not clear whether a 32×24
  1-bit image transmitted over LoRa provides sufficient utility to justify the
  protocol complexity. Alternative approaches (transmitting a motion-detected
  flag bit and storing full images locally for periodic retrieval) may be more
  practical.

**Current position:** The Image field is included in v0.90 as an experimental
capability. Its inclusion in v1.0 will be reviewed based on implementation
experience and demonstrated operational utility.

### J.13. Absence of Test Vectors

**Issue:** The specification does not include normative test vectors (known
inputs with expected binary outputs). The reference implementation test suite
provides de facto test cases, but these are not reproduced in the specification
document.

**Impact:** An independent implementer working from the specification alone
cannot verify conformance without access to the reference implementation. This
is a significant gap for a protocol intended for independent implementation.

**Plan:** A set of normative test vectors covering the following cases will be
added before v1.0:

- Minimum valid packet (header + empty presence byte).
- Battery-only heartbeat (minimum useful packet).
- Full weather station packet (variant 0, all pres0 fields).
- Full weather station packet with pres1 fields (position, datetime, flags).
- Packet with TLV data (VERSION + STATUS).
- Edge cases: all fields at minimum values, all fields at maximum values, all
  fields at quantisation boundary values where round-trip error is maximised.

### J.14. Formal Decode Specification

**Issue:** The specification describes the packet structure through bit
diagrams, encode/decode formulae, and narrative text. It does not include a
formal or pseudocode description of the complete decode procedure as a single
algorithm.

**Impact:** The decode path requires synthesising information from Sections 4,
5, 6, 7, and 8. A reader must mentally reconstruct the decode loop: read header,
look up variant, read presence bytes, iterate field table, for each present
field read the appropriate number of bits and apply the decode formula. This is
straightforward but is not stated as an explicit algorithm anywhere in the
document.

**Plan:** A pseudocode decode algorithm (comparable to the encoder example in
Appendix C) will be added before v1.0, likely as an additional appendix.

### J.15. Bundle vs. Standalone Asymmetry

**Issue:** Some sensor groupings have both bundle and standalone forms
(Environment, Wind, Rain, Air Quality, Radiation), while Solar (irradiance + UV)
exists only as a bundle with no standalone components. Section 8 acknowledges
this: "some bundles have no standalone forms ... this may be addressed in future
versions."

**Impact:** A variant that needs irradiance without UV, or UV without
irradiance, must either waste bits on the unwanted component or leave the field
absent entirely. This is inconsistent with the protocol's bit-efficiency
principle.

**Recommendation:** Add standalone Irradiance (10 bits) and standalone UV Index
(4 bits) field types for completeness before v1.0.

### J.16. Cloud Cover Resolution

**Issue:** The clouds field (Section 8.26) uses 4 bits to encode 0–8 okta,
leaving 7 of 16 possible values unused. The okta scale is inherently coarse (9
levels for the entire range of cloud cover), and the 4-bit encoding wastes
nearly half its capacity.

**Alternatives:**

- **3 bits (0–7) with 8 mapped to 7.** Saves 1 bit but conflates "overcast" (8
  okta) with "nearly overcast" (7 okta), which is a meaningful distinction in
  meteorology.
- **4 bits (0–15) with extended resolution.** Use 0–16 to represent cloud cover
  in sixteenths rather than eighths, providing ~6% resolution. This matches some
  automated ceilometer outputs more closely than okta.
- **Accept as-is.** The 4-bit allocation is the smallest whole unit that
  accommodates 9 values. The waste is 3.17 bits of entropy in a 4-bit field —
  less than 1 bit of overhead.

**Current position:** Accepted as-is. The overhead is negligible and the okta
scale is the established meteorological standard.

### J.17. Additional Sensor Field Types

The current field type inventory covers core meteorological and environmental
measurements. The following field types have been identified as absent but
relevant to the protocol's stated deployment scenarios (farming, forestry,
outdoor commercial/industrial, water monitoring). None require new encoding
techniques — they are straightforward linear ranges that fit in 7–10 bits and
can be added as new field types without structural protocol changes.

The quantisation ranges listed below are preliminary and subject to the same
systematic analysis discussed in Section J.3 before inclusion in v1.0.

#### J.17.1. Soil Moisture and Conductivity

**Priority: High.** Soil moisture is arguably the most widely deployed
agricultural sensor type after temperature. Its absence is conspicuous for a
protocol targeting farm deployments.

**Soil moisture** is typically reported as volumetric water content (VWC), a
percentage of the soil volume occupied by water. The standalone Humidity field
(Section 8.11) could be repurposed via variant labelling, as the 0–100% range at
1% resolution is a reasonable fit for VWC. However, some capacitive and TDR
sensors (e.g. Meter EC-5, Teros 12) report raw dielectric permittivity rather
than calibrated VWC, with a range of approximately 1–80. A dedicated field type
would allow the variant to distinguish calibrated VWC from raw permittivity.

**Soil electrical conductivity (EC)** measures the ability of the soil solution
to conduct electrical current, serving as a proxy for salinity and dissolved ion
concentration. This is critical for irrigation management in arid and semi-arid
agriculture, where soil salinisation is a primary crop yield limiter. Typical
range: 0–5,000 µS/cm for most agricultural soils (some saline soils reach
20,000+ µS/cm). Resolution of 10–20 µS/cm is adequate for management decisions.
A 10-bit field at 20 µS/cm resolution covers 0–20,460 µS/cm.

**Soil temperature** is already available via the standalone Temperature field
(Section 8.9) with a variant label such as `"soil_temp"`. No new field type is
needed.

A **soil bundle** (moisture + conductivity + temperature) would be natural,
mirroring the Environment bundle pattern, for sensors like the Teros 12 and
Meter TEROS 21 that output all three simultaneously.

| Candidate field     | Bits | Range           | Resolution | Sensor examples        |
| ------------------- | ---- | --------------- | ---------- | ---------------------- |
| Soil moisture (VWC) | 7    | 0–100%          | 1%         | Teros 12, EC-5, SHT40  |
| Soil EC             | 10   | 0–20,460 µS/cm  | 20 µS/cm   | Teros 12, Teros 21     |
| Soil bundle         | 26   | VWC + EC + temp | As above   | Combined probe outputs |

#### J.17.2. Water Quality: pH

**Priority: High.** pH is the most fundamental water quality measurement,
relevant to aquaculture, river and lake monitoring, irrigation water assessment,
and water treatment. The measurement is well-defined (hydrogen ion activity on a
logarithmic scale), universally understood, and reported by a wide range of
sensors.

Range: 0.00–14.00. Resolution: 0.1 pH units is standard for field instruments;
0.01 is available from laboratory-grade probes but rarely meaningful in
continuous outdoor monitoring due to drift and fouling.

| Candidate field | Bits | Range  | Resolution | Sensor examples                |
| --------------- | ---- | ------ | ---------- | ------------------------------ |
| pH (×10)        | 8    | 0–14.0 | ~0.06      | Atlas Scientific EZO-pH, Hanna |
| pH (×100)       | 10   | 0–14.0 | ~0.014     | Laboratory probes              |

The 8-bit encoding (0–255, mapped to 0–14.0 at 255 steps ≈ 0.055 resolution) is
adequate for field deployment and matches the ±0.1 accuracy of most submersible
pH probes.

#### J.17.3. Water Quality: Electrical Conductivity

**Priority: High.** Water EC measures dissolved ion concentration, serving as a
proxy for total dissolved solids (TDS). It is the primary measurement for
monitoring water quality in rivers, lakes, aquaculture ponds, and water
treatment systems.

The range varies enormously by application: freshwater rivers are typically
50–1,500 µS/cm, drinking water up to 2,500 µS/cm, brackish water 2,500–30,000
µS/cm, and seawater ~50,000 µS/cm. This wide range is a candidate for non-linear
quantisation (Section J.3) or a configurable range per variant.

| Candidate field | Bits | Range           | Resolution | Notes                          |
| --------------- | ---- | --------------- | ---------- | ------------------------------ |
| EC (freshwater) | 10   | 0–5,115 µS/cm   | 5 µS/cm    | Rivers, lakes, irrigation      |
| EC (wide range) | 10   | 0–51,150 µS/cm  | 50 µS/cm   | Covers brackish and seawater   |
| EC (log scale)  | 8    | 1–100,000 µS/cm | ~12%/step  | Single field for all use cases |

Atlas Scientific EZO-EC probes cover 0.07–500,000 µS/cm. A deployment-selected
range (freshwater vs. wide) via variant label may be the most practical
approach, using the same bit encoding with different scaling.

Note that soil EC (Section J.17.1) and water EC are the same physical
measurement with different typical ranges. A single EC field type with
variant-defined scaling could serve both.

#### J.17.4. Water Quality: Dissolved Oxygen

**Priority: Medium.** Dissolved oxygen (DO) is critical for aquaculture (fish
require >5 mg/L; below 3 mg/L is lethal for most species) and river ecology
(regulatory thresholds for water quality classification). It is also relevant to
wastewater treatment monitoring.

Typical range: 0–20 mg/L. Resolution: 0.1 mg/L is adequate for management
decisions and matches the ±0.1–0.2 mg/L accuracy of optical DO probes (Atlas
Scientific EZO-DO, In-Situ RDO).

Some sensors also report oxygen saturation as a percentage (0–100%+ ;
supersaturation above 100% occurs in algae-rich water). This could be encoded as
a standalone humidity-style field with a variant label.

| Candidate field   | Bits | Range  | Resolution | Sensor examples        |
| ----------------- | ---- | ------ | ---------- | ---------------------- |
| DO (mg/L, ×10)    | 8    | 0–25.5 | 0.1 mg/L   | Atlas EZO-DO, RDO Pro  |
| DO saturation (%) | 7    | 0–100% | 1%         | Same sensors, % output |

#### J.17.5. Water Quality: Turbidity

**Priority: Medium.** Turbidity measures the optical clarity of water, reported
in Nephelometric Turbidity Units (NTU). It is a proxy for suspended sediment,
algal concentration, and general water quality. Relevant for river monitoring
(sediment transport during flood events), water treatment (intake turbidity),
and aquaculture (pond clarity).

The distribution is heavily right-skewed: clean water is 0–5 NTU, typical rivers
10–100 NTU, flood events 1,000+ NTU, and extremely turbid water can exceed
10,000 NTU. This is a strong candidate for non-linear quantisation.

| Candidate field    | Bits | Range        | Resolution | Notes                         |
| ------------------ | ---- | ------------ | ---------- | ----------------------------- |
| Turbidity (linear) | 10   | 0–1,023 NTU  | 1 NTU      | Adequate for clean water only |
| Turbidity (linear) | 10   | 0–10,230 NTU | 10 NTU     | Covers flood events; coarse   |
| Turbidity (log)    | 8    | 0.1–10,000   | ~12%/step  | Matches sensor dynamic range  |

#### J.17.6. Water Quality: ORP (Oxidation-Reduction Potential)

**Priority: Low-Medium.** ORP measures the tendency of a solution to oxidise or
reduce, reported in millivolts. It is used in water treatment, aquaculture, and
pool/spa monitoring as an indicator of disinfection effectiveness and water
chemistry. Atlas Scientific produces an EZO-ORP module for this measurement.

Range: -1,000 to +1,000 mV (most natural water: +200 to +600 mV). Resolution:
1–5 mV is adequate.

| Candidate field | Bits | Range             | Resolution | Sensor examples |
| --------------- | ---- | ----------------- | ---------- | --------------- |
| ORP (mV)        | 10   | -999 to +1,024 mV | ~2 mV      | Atlas EZO-ORP   |

#### J.17.7. Water Flow / Discharge

**Priority: Medium.** Flow rate is relevant for river discharge monitoring,
irrigation flow measurement, and water distribution systems. Unlike water level
(which can be encoded as depth), flow rate is a derived quantity with a wide
dynamic range: a small irrigation pipe might carry 0.1 L/s, while a river gauge
might report 500 m³/s.

The wide dynamic range and the diversity of units (L/s, m³/s, gallons/min)
suggest that this measurement may be better handled as a variant-specific custom
encoding or via TLV, rather than as a fixed field type. Alternatively, a generic
flow field with variant-defined units and a logarithmic encoding could cover the
range.

**Note:** Many flow measurements are derived from water level via a
stage-discharge curve (Manning's equation or a calibrated rating curve). In
these cases, the sensor transmits level (depth field) and the gateway computes
flow. A flow field type is primarily useful for sensors that output flow
directly (ultrasonic transit-time meters, electromagnetic flow meters, weir
gauges with integrated computation).

#### J.17.8. Leaf Wetness / Surface Moisture

**Priority: Low.** Leaf wetness sensors detect the presence and quantity of
surface moisture on vegetation. They are used in precision viticulture, orchard
management, and crop disease prediction models (e.g. downy mildew in grapes,
late blight in potatoes). The measurement is typically reported as a coarse
categorical scale (dry / dew / wet / saturated) or as a percentage (0–100%) from
resistive or capacitive sensors.

A 2-bit categorical field (4 levels) or a repurposed 7-bit humidity field with a
variant label would suffice. The measurement is niche but falls within the
farming use case.

#### J.17.9. Thermal Rate of Change and Fire Detection

**Priority: Medium.** The protocol encodes instantaneous values: a temperature
reading is a snapshot at the moment of transmission. For fire detection and
frost early warning, the rate of temperature change is often more informative
than the absolute value.

**Fire detection context:** A wildfire approaching a sensor produces a
characteristic thermal signature: ambient temperature rises rapidly (10–30°C
over 1–5 minutes) before the absolute temperature reaches alarming levels. By
the time the temperature reads 60°C, the fire is already at the sensor. Early
detection depends on recognising the rate of change while the absolute
temperature is still in the 25–40°C range.

Relevant measurements for fire/thermal anomaly detection:

**Temperature rate of change (°C/min).** A signed value indicating heating or
cooling rate. Range: -10 to +30 °C/min covers both frost events (slow cooling,
-1 to -5 °C/min) and fire approach (rapid heating, +5 to +30 °C/min). 8 bits at
0.2 °C/min resolution (-25.6 to +25.4 °C/min with signed encoding) would
suffice.

This measurement is computed by the sensor from consecutive temperature readings
and the interval between them. It requires no additional hardware — only
firmware logic and retention of the previous reading. The averaging period
should be stated (e.g. "rate computed over the most recent two readings" or "60-
second moving average") and could be communicated via a CONFIG TLV.

**Smoke / particulate spike.** The existing Air Quality PM fields (Section 8.21)
encode absolute particulate concentration, which can indicate smoke. However,
fire-relevant smoke detection is better characterised by rate of change in PM2.5
(a sudden spike from baseline) rather than absolute level, since baseline varies
by location and season. A rate-of-change field for PM2.5 (similar to the thermal
rate) could complement the absolute PM reading. Alternatively, the sensor
firmware can set a flag bit (Section 8.6) when it detects a PM spike, leaving
the algorithm sensor-side.

**Carbon monoxide.** Already available in the Air Quality Gas field (Section
8.22, slot 3). CO is an early indicator of smouldering combustion. No new field
type is needed, but the CO slot's 1 ppm resolution and 0–1,023 ppm range are
well suited to fire detection (dangerous levels are 50+ ppm).

**Summary of fire-relevant capabilities:**

| Measurement          | Current status        | Gap                                        |
| -------------------- | --------------------- | ------------------------------------------ |
| Absolute temperature | Available (8.3, 8.9)  | None — but absolute alone is a late signal |
| Temperature rate     | Not available         | New field type needed                      |
| PM2.5 absolute       | Available (8.21)      | None                                       |
| PM2.5 rate           | Not available         | New field type or flag-based approach      |
| CO                   | Available (8.22)      | None                                       |
| Humidity drop        | Available (8.3, 8.11) | Rapid humidity drop precedes fire front    |
| IR flame detection   | Not available         | Specialised sensor; out of scope for v1.0  |

**Design question:** Should rate-of-change be a generic modifier applicable to
any field type, or a standalone field type? A generic approach (e.g. a "delta"
flag or a companion field type that encodes the first derivative of any
measurement) would be more flexible but adds protocol complexity. A standalone
`TEMPERATURE_RATE` field is simpler and covers the primary use case.

**Recommendation:** Add a standalone temperature rate-of-change field for v1.0
(simple, no new encoding concepts, high value for fire and frost detection).
Defer generic rate-of-change mechanisms to a future version.

| Candidate field        | Bits | Range                  | Resolution  | Use case            |
| ---------------------- | ---- | ---------------------- | ----------- | ------------------- |
| Temp rate (°C/min)     | 8    | -25.6 to +25.4 °C/min  | 0.2 °C/min  | Fire, frost warning |
| PM2.5 rate (µg/m³/min) | 8    | -127 to +128 µg/m³/min | 1 µg/m³/min | Smoke detection     |

### J.18. Silent Decode of Corrupted Payloads

**Issue:** The iotdata wire format contains no structural markers — no field
tags, type indicators, length prefixes, or sentinel values — between data
fields. The decoder determines field boundaries entirely from the variant table
and presence bytes. If either the presence bytes or a data field are corrupted
by a bit error that is not caught by the transport layer, the decoder will
produce a structurally valid but semantically wrong result with no indication of
error.

Consider a single bit-flip in Presence Byte 0 that sets the wind bit (S3) when
wind data was not transmitted. The decoder now attempts to read 22 bits of wind
data from what is actually the rain, solar, and beginning of any extension byte
or TLV data. Every field boundary after the corrupted presence byte is shifted,
and every subsequent decoded value is wrong. The decoder reports success.

**Comparison:** Protobuf and CBOR include per-field type and length information
that acts as structural redundancy. A corrupted Protobuf field tag will
typically produce an invalid wire type or an impossibly large field number,
causing the decoder to reject the packet. CayenneLPP's 2-byte channel+type
prefix per field provides similar structural checkpoints — a corrupted type byte
will fail to match any known sensor type. These formats pay a wire-size cost for
this property, but gain detection of mid-payload corruption that survives the
transport CRC.

**Impact:** In practice, this risk is low for deployments using LoRa CRC (which
catches most bit errors) or LoRaWAN MIC (which provides cryptographic
integrity). The risk is higher for deployments on transports without integrity
checks, or where the LoRa CRC is disabled for range extension (a practice used
by some long-range deployments at high spreading factors).

The most dangerous failure mode is a corrupted presence byte, because it shifts
all subsequent field boundaries and can cause every decoded value to be
plausible but wrong. A corrupted data field is less dangerous — it affects only
that field and subsequent fields are still correctly aligned (because the
corrupt field still occupies its expected bit width).

**Mitigation options:**

1. **Transport-layer CRC (current approach).** Rely on LoRa CRC, LoRaWAN MIC, or
   equivalent link-layer integrity. This is the protocol's stated design choice
   (Section 3.7). For most deployments it is sufficient.

2. **Application-layer checksum.** Reserve a TLV type for a packet checksum
   (e.g. CRC-8 over the header and data fields). The TLV section appears after
   all data fields, so a checksum TLV allows the receiver to verify the entire
   data section. Cost: 3 bytes (16-bit TLV header + 8-bit CRC). This is
   available today using a proprietary TLV type (0x20+) and requires no protocol
   changes.

3. **Range validation.** The decoder can validate each decoded value against the
   field's defined range (e.g. humidity must be 0–100, pressure must be
   850–1105). Out-of-range values indicate corruption. This catches some
   corruptions but not all — a corrupted temperature of 22.5°C when the true
   value was 18.0°C passes range validation. The reference implementation does
   not currently perform post-decode range validation, though Section 11.6
   identifies this as a non-fatal anomaly.

4. **Statistical anomaly detection.** The gateway can flag decoded values that
   are statistically inconsistent with recent history for the same station (e.g.
   a 30°C temperature jump between consecutive transmissions). This is a
   receiver-side strategy that requires no wire changes but cannot distinguish
   corruption from genuine rapid change.

**Current position:** The protocol's transport-delegated integrity model
(option 1) is retained for v1.0. Deployments requiring stronger guarantees
SHOULD use LoRaWAN MIC or add an application-layer checksum via a proprietary
TLV. The specification should note this failure mode explicitly in Section 11.6
as a receiver consideration.

### J.19. No Schema Tooling or Code Generation

**Issue:** Nanopb, Bitproto, and Protobuf all provide code generators that
produce encoder/decoder source code from a schema definition file (`.proto`,
`.bitproto`). This ensures that both sides of a communication link use an
identical, machine-generated interpretation of the data layout. Schema changes
are made in one place and propagated automatically.

iotdata's variant tables are hand-coded C arrays. The reference implementation
defines them as `iotdata_variant_def_t` structs with manually specified field
types, labels, and presence byte counts. Custom variants are created by writing
C code (Section 7, "Custom Variant Maps"). There is no schema definition
language, no code generator, and no tooling to verify that a transmitter's
variant table matches a receiver's.

**Impact:** In a small deployment (5–20 sensors, one operator), this is
manageable — the operator compiles the same variant definition into both sensor
and gateway firmware. In larger deployments, or deployments with separate teams
responsible for sensor and gateway software, the risk of variant table mismatch
increases. A mismatch produces silently wrong data (compounded by J.18 — there
are no structural markers to detect the misalignment).

The absence of a schema language also means there is no machine-readable variant
definition that could be used to auto-generate decoders in other languages
(Python, JavaScript, Go), to validate variant definitions for consistency, or to
produce documentation from the schema.

**Comparison:** Nanopb's workflow is: edit `.proto` file → run `protoc` with the
nanopb plugin → get `.pb.c` and `.pb.h` → compile into both sensor and gateway.
The schema file is the single source of truth. Bitproto follows an identical
pattern. iotdata's workflow is: edit C source on both sensor and gateway,
manually ensuring consistency.

**Options under consideration:**

1. **Schema definition file.** Define a simple text format for variant tables
   (field type, label, presence byte assignment) and write a generator that
   produces C source for the reference implementation. This could also generate
   Python/JavaScript decoders for gateway use. The schema file becomes the
   single source of truth for the variant definition.

2. **Variant table in VERSION TLV.** Encode a compact representation of the
   variant table in the VERSION TLV (Section 9.5.1), transmitted at boot. The
   gateway auto-discovers the field layout from the first packet. This adds wire
   overhead but eliminates the need for pre-shared variant definitions. See also
   J.22.

3. **Accept the limitation.** The protocol explicitly disclaims global
   interoperability (Section 3.8). For closed deployments where one build system
   compiles both sensor and gateway, the risk of mismatch is low. A shared C
   header included by both sides achieves consistency without a separate
   toolchain.

**Recommendation:** Option 1 (a lightweight schema tool) is the most practical
improvement and would also enable option 3 of J.22 (variant advertisement).
Option 3 (accept the limitation) is the appropriate baseline for v1.0, with a
shared C header as a documented best practice for multi-target builds. A schema
tool is deferred to post-v1.0 tooling.

### J.20. C-Only Reference Implementation

**Issue:** The reference implementation is written in C11 and provides a static
library (`libiotdata.a`). There are no bindings, ports, or reference
implementations in other languages. A gateway or server written in Python, Go,
JavaScript, or Rust must reimplement the decoder from the specification document
(this README), using the C code as an informal reference.

**Comparison:** Nanopb generates C code, but the `.proto` schema it consumes is
shared with the wider Protobuf ecosystem — any Protobuf library in any language
can decode a Nanopb-encoded message. Bitproto generates C, Go, and Python from a
single schema. CayenneLPP has implementations in C++ (Arduino), Python, and
JavaScript, and is natively decoded by The Things Network and ChirpStack without
any user code.

**Impact:** For an all-C deployment (ESP32 sensor + Raspberry Pi gateway using
the same `libiotdata.a`), this is not a limitation. For deployments where the
gateway or backend is written in Python, Go, or JavaScript — which is the common
case for cloud-connected IoT platforms — the absence of a reference decoder in
those languages is a significant adoption barrier. Reimplementing the
bit-packing, quantisation, variant dispatch, and TLV parsing is non-trivial and
error-prone, particularly for the variable-length fields (Air Quality, Image)
and the 6-bit packed string format.

**Options:**

1. **Python reference decoder.** A Python implementation of the decoder would
   cover the most common gateway/server language and could also serve as a test
   oracle for the C implementation. The bit-packing logic is straightforward in
   Python; the primary work is replicating the variant table dispatch and
   quantisation formulae.

2. **JavaScript/TypeScript decoder.** For LoRaWAN deployments, a JavaScript
   decoder function is directly usable as a TTN/ChirpStack payload formatter,
   addressing J.21 simultaneously.

3. **Generated decoders.** If a schema tool is developed (J.19), it could
   generate decoders in multiple languages from the variant definition.

**Recommendation:** A Python reference decoder and a JavaScript payload
formatter are identified as high-value post-v1.0 deliverables. The C
implementation remains the normative reference.

### J.21. No LoRaWAN Ecosystem Integration

**Issue:** CayenneLPP's primary competitive advantage is not its wire efficiency
(it is 3.4× larger than iotdata for the test payload) but its zero-
configuration integration with the LoRaWAN ecosystem. The Things Network,
ChirpStack, and myDevices Cayenne all decode CayenneLPP payloads automatically —
the operator selects "CayenneLPP" as the payload formatter and sensor data
appears in the dashboard with correct field names, units, and types. No custom
code is required.

iotdata has no equivalent integration. An operator deploying iotdata on a
LoRaWAN network server must write a custom payload formatter (typically in
JavaScript) that reimplements the decoder for their specific variant. This
formatter must be maintained alongside the sensor firmware and updated whenever
the variant definition changes.

**Impact:** For operators already invested in the LoRaWAN ecosystem and using
TTN or ChirpStack with Cayenne dashboards, CayenneLPP's ecosystem integration
may outweigh iotdata's wire efficiency advantage. The 3.4× payload size
difference matters most at high spreading factors and under tight duty cycle
budgets; at SF7 with modest duty cycle pressure, the operational impact of
larger payloads is tolerable, and the operational simplicity of CayenneLPP
becomes the dominant factor.

For operators using custom gateway software (direct LoRa, non-LoRaWAN),
iotdata's wire efficiency advantage applies fully and CayenneLPP's ecosystem
integration is irrelevant.

**Options:**

1. **TTN/ChirpStack payload formatter.** Provide a JavaScript decoder function
   that can be pasted into the TTN or ChirpStack payload formatter
   configuration. This would need to be parameterised by variant (either a
   generic decoder that reads the variant from the header and looks up a
   JavaScript variant table, or a generated per-variant formatter). See also
   J.20 option 2.

2. **MQTT auto-decode.** For gateways that forward raw packets via MQTT, provide
   a lightweight MQTT-to-JSON bridge (Python or Node.js) that subscribes to raw
   packet topics, decodes iotdata, and republishes as JSON. This is
   architecturally equivalent to CayenneLPP's network server integration but
   operates at the application layer.

3. **Accept the limitation.** iotdata's design philosophy prioritises wire
   efficiency for constrained links over ecosystem convenience. Operators
   choosing iotdata accept the cost of custom integration in exchange for
   smaller payloads and longer battery life.

**Recommendation:** A reference JavaScript payload formatter for TTN/ChirpStack
is identified as a high-value deliverable that would substantially reduce the
adoption barrier for LoRaWAN deployments, and could be produced as a companion
artifact alongside a Python decoder (J.20). The protocol itself requires no
changes.

### J.22. No Variant Table Discovery or Advertisement

**Issue:** An iotdata packet is not self-describing. The receiver must possess
the transmitter's variant table before it can decode any data fields. If a
receiver encounters a packet from a station whose variant definition it does not
have, it cannot determine the field types, field widths, or field order. The
packet is opaque.

This is a deliberate design choice (Section 3.8: "it is expressly not a goal to
support interoperability between implementations"). However, the absence of any
mechanism for a receiver to discover a transmitter's variant definition creates
operational friction in several scenarios:

- **New sensor deployment.** When a new sensor is added to an existing
  deployment, the gateway must be reconfigured with the sensor's variant
  definition before it can decode the sensor's data.

- **Multi-operator environments.** If two operators share a gateway or mesh
  infrastructure, each must ensure the gateway has variant definitions for all
  sensors from both operators.

- **Diagnostic and debugging.** A technician with a generic LoRa receiver cannot
  inspect packets from an unknown deployment without obtaining the variant
  definition out-of-band.

- **Mesh relay transparency.** Mesh relays (Appendix G) forward sensor packets
  as opaque blobs, but gateways must decode them. If a relay forwards a packet
  from a sensor using an unknown variant, the gateway cannot decode it.

**Comparison:** CayenneLPP payloads are fully self-describing — every field
carries its channel and type. CBOR and Protobuf carry structural metadata that
enables generic tools to display the data structure even without a schema (e.g.
`protoc --decode_raw`). Nanopb-encoded data can be decoded by any Protobuf
library with the `.proto` schema — and the schema is a portable text file, not
compiled into a specific target.

**Options under consideration:**

1. **Accept the limitation (current position).** For closed deployments where
   one operator controls all devices, the variant table is a compile-time
   artefact shared between sensor and gateway firmware. No wire-level discovery
   is needed.

2. **Variant definition in VERSION TLV.** The existing VERSION TLV (type 0x01,
   string format) carries firmware and hardware identification. A compact
   encoding of the variant table could be added — either as additional key-value
   pairs in the VERSION TLV (e.g. `V0 BAT LNK ENV WND RAN SOL`) using short
   field type mnemonics, or as a new dedicated TLV type in the reserved
   0x10–0x1F range.

   The variant definition is static per firmware build, so it would be
   transmitted once at boot alongside the VERSION TLV. The gateway caches it per
   station_id and uses it to decode subsequent packets. Cost: approximately
   20–40 bytes once per boot cycle — negligible amortised across thousands of
   subsequent data packets.

3. **Schema file distribution.** If a schema definition language is developed
   (J.19), variant definitions could be distributed as files (alongside firmware
   images, via OTA manifest, or published to a repository). The gateway loads
   schema files for the variants it expects to encounter. This is an out-of-band
   mechanism that requires no wire changes.

4. **Well-known variant registry.** Publish a set of standardised variant
   definitions (weather station, soil sensor, water quality, snow depth) with
   assigned variant IDs. Receivers that implement the registry can decode any
   sensor using a registered variant without per-deployment configuration. This
   conflicts with the current non-interoperability stance (Section 3.8) but
   could be offered as an optional extension for operators who want
   plug-and-play behaviour.

**Recommendation:** Option 1 (accept the limitation) is appropriate for v1.0,
consistent with the protocol's design philosophy. Option 2 (variant definition
in a TLV) is the most promising future mechanism because it requires no
out-of-band coordination and leverages existing protocol features. The design of
a compact variant table encoding is deferred to post-v1.0 but is noted as a
prerequisite for any future interoperability work. Option 4 (registered
variants) may be revisited if the protocol achieves adoption beyond single-
operator deployments.

### J.23. Relationship to ASN.1 Packed Encoding Rules (UPER)

**Issue:** ASN.1 with Unaligned Packed Encoding Rules (UPER, ITU-T X.691) is a
standardised bit-packing encoding that operates on the same principle as
iotdata's core encoding: constrained integer ranges are mapped to
minimum-bit-width representations, fields are not byte-aligned, and optional
fields are indicated by a presence bitmap at the start of the SEQUENCE. UPER is
deployed at scale in 3GPP signalling (LTE RRC, 5G NR), aviation (ADS-B uses a
fixed-layout bit-packed format with the same philosophy), automotive V2X
(CAM/DENM messages), and space telemetry (ESA's Packet Utilisation Standard via
ASN1SCC). An informed reviewer of iotdata will immediately ask: "why not define
an ASN.1 schema and use UPER?"

**Comparison:** An ASN.1 schema for the iotdata test payload would look
approximately like:

```asn1
WeatherStation ::= SEQUENCE {
    battery       INTEGER (0..100)   OPTIONAL,  -- 7 bits
    linkQuality   INTEGER (0..100)   OPTIONAL,  -- 7 bits
    temperature   INTEGER (-400..850) OPTIONAL,  -- 11 bits (range 1251)
    humidity      INTEGER (0..100)   OPTIONAL,  -- 7 bits
    pressure      INTEGER (8500..11050) OPTIONAL, -- 9 bits (range 2551)
    ...
}
```

UPER would encode constrained integers in minimum bits (identical to iotdata),
prefix the SEQUENCE with a presence bitmap (identical to iotdata's presence
bytes), and pack fields without byte alignment. The wire encoding of the data
fields would be nearly identical in size — UPER's encoding of this schema would
produce a payload within 1–2 bits of iotdata's 16-byte test payload.

**Why iotdata does not use ASN.1 UPER:**

1. **Toolchain weight.** ASN.1 compilers are substantial tools. The open-source
   ASN1SCC (ESA) generates C and SPARK/Ada from ASN.1 grammars with zero-malloc
   guarantees and is suitable for embedded targets. However, ASN1SCC itself
   requires .NET 9 and Java JRE to run, and the generated code includes a
   runtime library (asn1crt.c, encoding helpers) that adds several KB of ROM.
   Commercial ASN.1 compilers (OSS Nokalva, Objective Systems) are expensive and
   typically licensed per-seat. The Python `asn1tools` package can generate UPER
   C source but supports only a subset of ASN.1. For a project targeting
   ESP32-C3 with 400 KB flash, the toolchain overhead and generated code size
   are non-trivial compared to iotdata's single .c/.h with no external
   dependencies.

2. **No domain-specific quantisation.** UPER encodes constrained integers in
   minimum bits, but the constraint must be expressed as an integer range. To
   encode temperature as 0.1°C resolution over -40.0°C to +85.0°C, the ASN.1
   schema must define the field as `INTEGER (-400..850)` and the application
   must perform the ×10 scaling on both sides. UPER provides the bit-packing but
   not the semantic quantisation — the schema does not express "this field is a
   temperature in °C with 0.1 resolution." iotdata's field type system encodes
   the physical meaning, resolution, and range as a single declaration, and the
   reference implementation performs quantisation and dequantisation
   automatically.

3. **No presence-byte-driven variable layout.** UPER's OPTIONAL bitmap is fixed
   at schema definition time — every OPTIONAL field in the SEQUENCE gets a bit
   in the preamble, always. iotdata's variant system allows different
   deployments to define different field sets (variants) with different presence
   byte layouts, and the presence bytes serve double duty as both optional-field
   indicators and variant-specific field selectors. ASN.1 would require a
   separate schema (or CHOICE type) per variant, and the decoder would need to
   know which schema to apply — reintroducing the variant-selection problem at
   the ASN.1 level.

4. **No TLV extension mechanism.** iotdata's TLV section (Section 9.5) allows
   arbitrary typed extensions (firmware version, GPS coordinates, text labels,
   image data) to be appended to any packet without schema changes. ASN.1
   supports extensibility via the `...` extension marker, but extending a
   UPER-encoded SEQUENCE requires the extension to be defined in the schema and
   recompiled. iotdata's TLV section is deliberately schema-free.

5. **Specification complexity.** The ASN.1 standard spans ITU-T X.680–X.683
   (notation) and X.690–X.696 (encoding rules). UPER alone (X.691) is a 107-page
   specification with complex rules for fragmentation, length determinants, and
   constraint visibility. iotdata's encoding rules fit in a single README
   section and can be implemented from scratch in an afternoon. For a
   single-purpose IoT sensor protocol, the full generality of ASN.1 is
   unnecessary overhead.

**What ASN.1 UPER does better:**

- Formal schema language with decades of tooling, validation, and
  interoperability testing.
- Automatic code generation for C, Ada, Python, Java, Go, and Rust.
- Proven at enormous scale (every LTE/5G device on earth uses UPER for RRC).
- Schema evolution via extension markers — forward and backward compatibility is
  a solved problem.
- Interface Control Document (ICD) generation from the schema.

**Position:** iotdata's encoding is philosophically identical to ASN.1 UPER but
trades generality for simplicity, domain awareness, and minimal toolchain
dependency. For deployments where ASN.1 tooling is already available (e.g. space
systems, automotive V2X), UPER is the superior choice. For bare-metal IoT
sensors where the entire firmware fits in 256 KB and the developer does not have
access to (or budget for) an ASN.1 compiler, iotdata provides the same wire
efficiency with a fraction of the toolchain complexity.

The existence of ASN1SCC (open-source, zero-malloc, ESA-funded) narrows this gap
considerably. A future version of iotdata could offer an ASN.1 schema as an
alternative interface to the same wire format, allowing ASN.1-equipped teams to
use their preferred toolchain while remaining wire-compatible with the C
reference implementation.

### J.24. Relationship to SenML and LwM2M

**Issue:** SenML (Sensor Measurement Lists, IETF RFC 8428) and LwM2M
(Lightweight M2M, OMA SpecWorks) are the IETF/OMA standards for IoT sensor data
representation and device management respectively. CayenneLPP's type codes are
derived from LwM2M/IPSO Smart Object IDs. Any IoT data format should be
positioned relative to these standards.

**SenML overview:** SenML defines a data model for sensor measurements as a list
of records, each containing a name, value, unit, and optional timestamp. It
supports JSON, CBOR, XML, and EXI representations. The CBOR representation uses
integer keys for compactness (e.g. key -2 for Base Name, key 2 for Value). A
minimal SenML+CBOR record for one temperature reading is approximately 15–20
bytes (CBOR map with name string, unit string, and double-precision value). A
full weather station payload (6 sensor readings) would be approximately 90–120
bytes in SenML+CBOR — 6–7× larger than iotdata's 16-byte encoding.

**LwM2M overview:** LwM2M defines a device management and service enablement
protocol built on CoAP. It uses an object/resource model where standardised
Object IDs (e.g. 3303 = Temperature, 3304 = Humidity, 3323 = Pressure) identify
sensor types. LwM2M operates over CoAP/UDP with DTLS security and requires a
LwM2M server. It is designed for bidirectional device management (firmware
update, configuration, observation) rather than unidirectional sensor data
streaming.

**Why iotdata does not use SenML or LwM2M:**

1. **Wire overhead.** SenML's self-describing records carry field names, units,
   and full-precision values per reading. Even in CBOR, this is 6–7× larger than
   iotdata. For LoRa at SF12 (51-byte maximum payload), a SenML+CBOR weather
   station payload would not fit in a single packet.

2. **Protocol weight.** LwM2M requires CoAP, DTLS, and a server-side LwM2M
   implementation. This is a full application-layer stack unsuitable for
   bare-metal LoRa devices with no IP connectivity. SenML as a data format is
   lighter but still assumes a transport capable of carrying its CBOR/JSON
   payloads.

3. **Unidirectional design.** iotdata is designed for fire-and-forget sensor
   telemetry on unidirectional or asymmetric links. LwM2M's observation model
   (where the server subscribes to resources and receives notifications) assumes
   bidirectional connectivity.

**What SenML/LwM2M do better:**

- Standardised sensor type identifiers (IPSO Object IDs) with IANA-registered
  units — a solved namespace problem.
- Self-describing payloads with no out-of-band schema requirement.
- Ecosystem integration with IoT platforms (AWS IoT, Azure IoT Hub, Thingsboard)
  that natively parse SenML.
- Formal extensibility through IANA registries.

**Relevance to iotdata:** If iotdata adopts global field type IDs (see J.27),
aligning those IDs with IPSO/LwM2M Object IDs where possible would provide
semantic interoperability without wire overhead. A gateway decoding iotdata
could map field type 0x03 (IOTDATA_FIELD_ENVIRONMENT) to LwM2M Objects 3303
(Temperature) + 3304 (Humidity) + 3315 (Barometer), enabling integration with
LwM2M-aware platforms at the application layer.

### J.25. Energy and Battery Life Impact

**Issue:** Appendix I.7 presents airtime comparisons across encoding formats and
spreading factors, but does not translate these into the operational metric that
matters most for battery-powered deployments: projected battery life.

**Worked example:** Consider a solar-powered weather station transmitting the
test payload (Section I.1) every 60 seconds using an SX1262 transceiver at +14
dBm on EU868 (125 kHz bandwidth).

Key parameters from the SX1262 datasheet:

- TX current at +14 dBm (DC-DC): ~45 mA
- RX current: 4.2 mA
- Sleep current (warm start, RTC running): 1.2 µA
- MCU (ESP32-C3) deep sleep: ~5 µA
- Total sleep current: ~6.2 µA

For each transmission cycle, the energy cost is dominated by the TX duration.
Using airtime values from Appendix I.7:

**At SF7 (short range, high data rate):**

| Format     | Payload | Airtime  | TX charge per cycle | TX charge/day |
| ---------- | ------- | -------- | ------------------- | ------------- |
| iotdata    | 16 B    | 46.3 ms  | 0.579 µAh           | 0.834 mAh     |
| Bitproto   | 20 B    | 51.5 ms  | 0.644 µAh           | 0.927 mAh     |
| CayenneLPP | 54 B    | 97.5 ms  | 1.219 µAh           | 1.755 mAh     |
| Protobuf   | 42 B    | 82.2 ms  | 1.028 µAh           | 1.480 mAh     |
| CBOR       | 66 B    | 113.2 ms | 1.415 µAh           | 2.037 mAh     |
| JSON       | 177 B   | 256.0 ms | 3.200 µAh           | 4.608 mAh     |

At SF7, the differences are small in absolute terms — all formats consume <5
mAh/day on TX alone. Sleep current (~0.149 mAh/day) and MCU active time
dominate. Battery life differences are negligible at this spreading factor.

**At SF12 (long range, low data rate):**

| Format     | Payload | Airtime    | TX charge per cycle | TX charge/day |
| ---------- | ------- | ---------- | ------------------- | ------------- |
| iotdata    | 16 B    | 1,482 ms   | 18.5 µAh            | 26.7 mAh      |
| CayenneLPP | 54 B    | 3,121 ms\* | 39.0 µAh            | 56.2 mAh      |
| JSON       | 177 B   | —\*\*      | —                   | —             |

\* CayenneLPP's 54 bytes exceeds the SF12 maximum payload of 51 bytes. The value
shown assumes the DR0 maximum is relaxed or the payload is split across two
packets (doubling actual TX cost).

\*\* JSON's 177 bytes far exceeds the SF12 maximum payload. Multiple packets
required.

On a 3000 mAh battery (e.g. 18650 LiPo), assuming 80% usable capacity (2400
mAh), with sleep current of ~0.149 mAh/day:

| Format     | SF7 battery life        | SF12 battery life       |
| ---------- | ----------------------- | ----------------------- |
| iotdata    | ~2,440 days (6.7 years) | ~89 days (2.9 months)   |
| CayenneLPP | ~2,410 days (6.6 years) | ~42 days\* (1.4 months) |

\* Assumes two-packet transmission to fit SF12.

**Analysis:** At SF7, encoding efficiency has minimal impact on battery life
because sleep current dominates. At SF12 — which is the regime where encoding
efficiency matters most — iotdata's 16-byte payload delivers roughly 2× the
battery life of CayenneLPP. For solar-powered deployments with marginal winter
charging, this difference can be the margin between continuous operation and
data gaps.

The battery life advantage scales with transmission frequency. A sensor
transmitting every 30 seconds at SF12 would halve all battery life figures,
making the encoding efficiency difference more pronounced.

**Limitation of this analysis:** These figures account only for TX energy. In
practice, MCU wake time (sensor reading, encoding, SPI transfer), RX windows
(for LoRaWAN Class A), and DC-DC converter efficiency also contribute. TX energy
is typically 60–80% of per-cycle energy at SF10+, making the airtime comparison
a reasonable proxy for total energy at high spreading factors.

### J.26. Delta and Differential Encoding

**Issue:** iotdata encodes every field as an absolute value on every
transmission. For sensor data with high temporal correlation (temperature
changes <0.5°C per minute, pressure changes <0.5 hPa per minute), this transmits
substantial redundant information. A 9-bit absolute temperature (0.1°C over -40
to +85°C) could often be replaced by a 4-bit signed delta (±0.7°C), reducing the
per-field cost from 9 bits to 4 bits for ~95% of consecutive readings.

**Information-theoretic context:** This observation connects to J.9
(information-theoretic encoding efficiency). iotdata's quantisation optimises
the per-field encoding to the minimum bits required for the field's static
range. Delta encoding would optimise for the dynamic range of consecutive
readings — the temporal entropy rather than the static entropy. For slowly
changing environmental data, temporal entropy is substantially lower than static
entropy.

**Comparison:** No existing IoT payload format in the comparison set
(CayenneLPP, Nanopb, Bitproto, TinyCBOR, SenML) implements delta encoding. This
is not a gap relative to competitors but an opportunity for iotdata to extend
its efficiency advantage.

Delta encoding is well-established in other domains: video codecs (I-frames vs
P-frames), audio codecs (ADPCM), GPS track compression (delta-of-deltas), and
time-series databases (Gorilla compression). The pattern is always the same:
transmit a full keyframe periodically and deltas between keyframes.

**Design sketch:**

A delta-capable iotdata variant would operate as follows:

1. Every N-th packet (e.g. N=10) is a **keyframe** — encoded identically to the
   current absolute format. The keyframe establishes the reference values for
   all fields.

2. Intermediate packets are **delta frames**. Each present field is encoded as a
   signed delta from the previous keyframe value, using a smaller bit width
   defined per field type:

   | Field          | Absolute bits | Delta bits | Delta range |
   | -------------- | ------------- | ---------- | ----------- |
   | Temperature    | 9             | 5          | ±1.5°C      |
   | Humidity       | 7             | 4          | ±7%         |
   | Pressure       | 9             | 5          | ±1.5 hPa    |
   | Wind speed     | 8             | 5          | ±1.5 m/s    |
   | Wind direction | 9             | 5          | ±15°        |
   | Rain           | 5             | 3          | ±0.3 mm     |
   | Solar          | 10            | 5          | ±15 W/m²    |

3. If a delta exceeds the representable range, the field falls back to absolute
   encoding for that packet (indicated by a flag bit, or by transmitting a
   keyframe).

**Estimated savings:** For the test payload at steady-state (6 sensor fields
present), absolute encoding uses ~55 data bits. Delta encoding would use ~27
data bits — approximately 50% reduction in the data section, saving ~3.5 bytes
per packet. Over a 10-packet keyframe cycle, 9 delta frames save ~31.5 bytes
total, at the cost of added decoder complexity and keyframe synchronisation
requirements.

**Challenges:**

1. **Keyframe synchronisation.** A receiver that misses the keyframe cannot
   decode subsequent delta frames. This is the same problem as joining a video
   stream mid-GOP. Mitigations: periodic keyframes at a rate faster than the
   expected packet loss rate; transmitting the keyframe index in the header so
   the receiver knows when to expect the next one; or a "request keyframe"
   mechanism for bidirectional links.

2. **Compounded by J.18.** A corrupted delta value produces a wrong reference
   for subsequent deltas, causing error accumulation until the next keyframe.
   This is strictly worse than absolute encoding's corruption behaviour (where
   each packet is independent).

3. **Complexity.** Both encoder and decoder must maintain per-field state across
   packets. The encoder must track the last keyframe values; the decoder must
   reconstruct absolute values from deltas. This adds RAM (one
   `iotdata_reading_t` per tracked station) and code complexity.

4. **Variant table expansion.** Delta bit widths would need to be defined per
   field type, adding another dimension to the variant table.

**Recommendation:** Delta encoding is deferred beyond v1.0. The complexity and
synchronisation challenges outweigh the 3–4 byte savings for most deployments.
However, for deployments transmitting at SF12 where every byte matters (see
J.25), delta encoding could reduce a 16-byte payload to ~12 bytes, potentially
allowing a lower spreading factor and substantially reducing airtime. The design
sketch above is preserved for future consideration. If implemented, it should be
a variant-level option (e.g. `IOTDATA_VARIANT_FLAG_DELTA`) rather than a
protocol-level change, keeping backward compatibility with absolute-only
decoders.

### J.27. Variant Map Transmission and Global Field Type Identifiers

**Issue:** J.22 identifies the lack of variant table discovery as a limitation.
J.19 identifies the absence of schema tooling. This item proposes a concrete
mechanism that addresses both: transmitting the variant map from the device as a
compact TLV, enabling any receiver to decode subsequent packets without
pre-shared configuration.

The core analogy is dictionary-based compression: zstd can transmit a dictionary
once and reference it for all subsequent frames. Similarly, iotdata could
transmit the variant definition once at boot (or periodically) and every
subsequent packet is decoded against the cached variant map.

**Prerequisite — Global Field Type Identifiers:**

For the variant map to be meaningful to any receiver, field types must have
globally unique, stable identifiers. Currently, field types
(`IOTDATA_FIELD_BATTERY`, `IOTDATA_FIELD_ENVIRONMENT`, etc.) are
implementation-internal enum values in the C reference implementation. Their
numeric values are not part of the specification and could change between
releases.

Promoting these to **protocol-level identifiers** means:

- Each field type is assigned a permanent numeric ID in the specification.
- The ID encodes the field's data layout (bit widths, quantisation, sub-field
  structure) — a decoder that knows ID 0x03 can decode an ENVIRONMENT field
  without any additional information.
- IDs are never reassigned or reused. New field types receive new IDs.
- The ID space is partitioned: 0x00–0x3F for specification-defined types,
  0x40–0x7F for user-defined types (with locally-scoped semantics).

A suggested initial assignment (illustrative, subject to specification review):

| ID   | Field Type     | Sub-fields                             |
| ---- | -------------- | -------------------------------------- |
| 0x01 | BATTERY        | voltage_pct (7 bits)                   |
| 0x02 | LINK_QUALITY   | rssi_pct (7 bits)                      |
| 0x03 | ENVIRONMENT    | temp (9) + humidity (7) + pressure (9) |
| 0x04 | WIND           | speed (8) + direction (9) + gust (5)   |
| 0x05 | RAIN           | accumulation (5 bits)                  |
| 0x06 | SOLAR          | irradiance (10 bits)                   |
| 0x07 | UV_INDEX       | uv (4 bits)                            |
| 0x08 | SOIL           | moisture (7) + temp (9)                |
| 0x09 | AIR_QUALITY    | pm2.5 (10) + pm10 (10) + aqi (8)       |
| 0x0A | WATER_QUALITY  | tds (10) + ph (7) + temp (9)           |
| 0x0B | SNOW_DEPTH     | depth (10 bits)                        |
| 0x0C | LEAF_WETNESS   | wetness (7 bits)                       |
| ...  | ...            | ...                                    |
| 0x40 | USER_DEFINED_0 | (layout defined by variant map)        |

**Variant Map TLV Design:**

A new TLV type (proposed: 0x10, within the reserved 0x10–0x1F sensor metadata
range) carries the variant definition:

```text
TLV Header:  [0x10][length]          -- 2 bytes (standard 6+10 bit TLV header)
Payload:     [num_presence_bytes:3]  -- 3 bits: number of presence bytes (1-7)
             [num_fields:5]          -- 5 bits: number of fields in variant (1-31)
             [field_0_id:7]          -- 7 bits per field: global field type ID
             [field_1_id:7]
             ...
             [field_N_id:7]
```

For the weather station test variant (2 presence bytes, 6 fields):

```text
Presence count: 2       →  010        (3 bits)
Field count:    6       →  00110      (5 bits)
Field IDs:      BATTERY →  0000001   (7 bits)
                LINK    →  0000010   (7 bits)
                ENV     →  0000011   (7 bits)
                WIND    →  0000100   (7 bits)
                RAIN    →  0000101   (7 bits)
                SOLAR   →  0000110   (7 bits)
```

Total: 3 + 5 + (6 × 7) = 50 bits = 7 bytes payload + 2 bytes TLV header = **9
bytes**. Transmitted once at boot alongside the VERSION TLV, then cached by the
gateway per station_id.

**Operational model:**

1. **Boot:** Sensor transmits a VERSION TLV (type 0x01) and a VARIANT MAP TLV
   (type 0x10) in the first packet after power-on or reset.

2. **Periodic refresh:** The variant map TLV is retransmitted every N packets
   (e.g. N=100, or once per hour) to handle gateway restarts and new receivers
   joining the network.

3. **Gateway caching:** The gateway maintains a map of
   `station_id → variant_definition`. When a variant map TLV is received, the
   gateway stores or updates the entry. Subsequent data packets from that
   station_id are decoded using the cached variant.

4. **Unknown station:** If a data packet arrives from a station_id with no
   cached variant, the gateway buffers the raw packet and waits for the next
   variant map TLV. Alternatively, the gateway can request a retransmission on
   bidirectional links.

**Relationship to IPSO/LwM2M:** If global field type IDs are aligned with IPSO
Smart Object IDs where possible (J.24), the variant map TLV provides enough
information for a gateway to not only decode the packet but also map each field
to a standardised semantic type — bridging iotdata's compact wire format to the
broader IoT standards ecosystem.

**Relationship to other J items:**

- J.19 (schema tooling): The global field type ID registry IS the schema. A
  schema tool generates variant tables from a list of field type IDs.
- J.20 (multi-language decoders): A decoder that knows the global field type
  registry can decode any variant map TLV and then decode any subsequent packet
  — no per-variant code generation needed.
- J.21 (LoRaWAN integration): A generic JavaScript payload formatter that parses
  the variant map TLV can decode any iotdata variant on TTN/ChirpStack without
  per-deployment configuration.
- J.22 (variant discovery): This item IS the concrete mechanism for variant
  discovery.

**Recommendation:** Global field type identifiers should be defined in the v1.0
specification even if the variant map TLV is deferred to a future version.
Locking the IDs now ensures forward compatibility — any variant tables created
today will be expressible as variant map TLVs in the future. The variant map TLV
itself is a low-risk addition (it uses the existing TLV mechanism, adds no
overhead to data packets, and is entirely optional) and could be included in
v1.0 as an OPTIONAL feature.

### J.28. Mesh Protocol Comparison

**Issue:** Appendix G defines a mesh relay protocol for multi-hop iotdata
delivery. This protocol should be compared against established mesh routing
approaches for low-power wireless networks to contextualise its design choices
and identify trade-offs.

**Comparison targets:**

**RPL (RFC 6550) — IPv6 Routing Protocol for Low-Power and Lossy Networks:**

RPL is the IETF standard for mesh routing in constrained networks. It builds a
Destination-Oriented Directed Acyclic Graph (DODAG) rooted at a border router,
using periodic DIO (DODAG Information Object) messages to construct and maintain
the topology. Key characteristics:

- **Full IP stack required.** RPL operates on 6LoWPAN/IPv6, requiring a 6LoWPAN
  adaptation layer, IPv6, ICMPv6, and the RPL control protocol. The Contiki-NG
  implementation uses approximately 30–50 KB ROM and 10–20 KB RAM.
- **Proactive routing.** Routes are maintained continuously via DIO/DIS/DAO
  control messages, even when no data is being sent. The Trickle timer reduces
  control traffic in stable topologies but still consumes airtime and energy.
- **Bidirectional.** Supports multipoint-to-point (sensor→gateway),
  point-to-multipoint (gateway→sensors), and point-to-point (sensor→sensor).
- **Topology-aware.** RPL maintains a routing table and selects routes based on
  an Objective Function (e.g. minimise hop count, maximise path ETX).
- **Target environment.** IEEE 802.15.4 networks (Zigbee-class), typically
  sub-100m range, hundreds to thousands of nodes, 250 kbps data rate.

**Meshtastic — LoRa Mesh Protocol:**

Meshtastic is an open-source LoRa mesh protocol designed for off-grid text
messaging. It uses managed flood routing (since v2.6) with distinct strategies
for broadcast and direct messages. Key characteristics:

- **Managed flooding.** Broadcast messages are rebroadcast by all receiving
  nodes (up to a configurable hop limit, default 3, max 7). Nodes use a
  rebroadcast scoring heuristic based on SNR, hop count, and role to decide
  whether to relay — nodes unlikely to improve coverage suppress their
  rebroadcast.
- **No routing tables.** Nodes do not maintain routes. Each packet carries a hop
  limit and nodes make independent forwarding decisions. This eliminates
  control-plane overhead entirely.
- **Protocol Buffers payload.** The packet header is raw bytes (for hardware
  filtering efficiency) but the payload is Protobuf-encoded. This adds encoding
  overhead but enables cross-vendor interoperability.
- **High duty cycle.** Nodes must listen continuously (or near-continuously) to
  participate in mesh relaying. This fundamentally conflicts with
  battery-powered sensor operation — Meshtastic nodes typically require USB
  power or frequent charging.
- **Target environment.** LoRa P2P at 868/915 MHz, 1–20 km range per hop, tens
  to hundreds of nodes, text messaging and telemetry.
- **Scalability concerns.** Flooding-based routing generates O(N) transmissions
  per message in an N-node network. Community experience suggests congestion
  issues beyond ~100 nodes in a single mesh, particularly on the default
  LONG_FAST preset with 10% EU duty cycle.

**Thread (IEEE 802.15.4 / 6LoWPAN):**

Thread is a low-power mesh networking protocol for home automation. It uses
6LoWPAN over IEEE 802.15.4 with RPL for routing and MLE (Mesh Link
Establishment) for network management. Thread is a full networking stack
(IP-based, with DTLS security, DNS-SD service discovery, and border router
integration) designed for always-powered or mains-powered devices. Its resource
requirements (64 KB ROM, 32 KB RAM minimum) and always-on radio make it
unsuitable for battery-powered LoRa sensors.

**Zigbee Mesh:**

Zigbee uses a hybrid routing approach (AODV reactive routing + tree routing)
over IEEE 802.15.4. Like Thread, it requires substantial stack resources and an
always-on radio for routing nodes. Zigbee's mesh is designed for dense,
short-range networks (10–100m) with mains-powered routers and battery-powered
end devices that do not participate in routing.

**iotdata Appendix G mesh — design positioning:**

| Dimension              | RPL              | Meshtastic      | Thread/Zigbee      | iotdata G        |
| ---------------------- | ---------------- | --------------- | ------------------ | ---------------- |
| Routing strategy       | Proactive DODAG  | Managed flood   | Proactive/reactive | Simple relay     |
| Control overhead       | DIO/DAO periodic | None (flooding) | MLE + RPL          | None             |
| Routing table          | Yes (per-node)   | No              | Yes                | No               |
| IP stack required      | Yes (6LoWPAN)    | No              | Yes                | No               |
| RAM for routing        | 10–20 KB         | ~1 KB           | 32+ KB             | ~100 bytes       |
| Payload awareness      | No (opaque)      | Protobuf        | No (opaque)        | iotdata-native   |
| Relay duty cycle       | Always-on        | Always-on       | Always-on          | Duty-cycled      |
| Battery-powered relays | Impractical      | Impractical     | Impractical        | Designed for     |
| Max hops (practical)   | 10+              | 3–7             | 10+                | 3–5              |
| Node scale             | 1000+            | ~100            | 250+               | 10–50            |
| Bidirectional          | Yes              | Yes             | Yes                | No (uplink only) |

**Key trade-offs in iotdata's approach:**

1. **Simplicity over optimality.** iotdata's mesh relay is a simple
   store-and-forward mechanism: a relay node receives a sensor packet, stores
   it, and retransmits it in a subsequent TX window. There is no route
   discovery, no topology management, and no routing table. This is viable
   because iotdata assumes a sparse, mostly-static topology with a small number
   of relay hops between sensor and gateway.

2. **Duty-cycled relays.** Unlike RPL, Meshtastic, Thread, and Zigbee — all of
   which require routing nodes to listen continuously — iotdata relays can
   operate on duty-cycled schedules. A relay wakes for a brief RX window,
   buffers any received packets, and retransmits them in the next TX window.
   This enables solar-powered or battery-powered relay nodes in locations
   without mains power.

3. **Uplink-only.** iotdata's mesh supports only sensor→gateway traffic. There
   is no downlink path (gateway→sensor) through the mesh. This eliminates the
   complexity of bidirectional routing but means that remote sensors cannot
   receive configuration updates, firmware, or acknowledgements via the mesh.

4. **Payload-native.** Relay nodes can optionally inspect and filter iotdata
   packets (e.g. suppress duplicate readings, aggregate multiple sensors into a
   single relay packet). RPL and Thread treat payloads as opaque IP packets.

5. **No scalability beyond sparse topologies.** The simple relay approach does
   not handle network congestion, route selection, or topology changes. For
   dense deployments (>50 nodes) or dynamic topologies (mobile sensors), RPL or
   a managed-flooding approach would be necessary.

**Recommendation:** iotdata's mesh relay is appropriate for its target use case:
sparse, static sensor networks with 3–5 hops where relay nodes must operate on
limited power budgets. For deployments requiring larger scale, bidirectional
communication, or dynamic topologies, an IP-based mesh (RPL over 6LoWPAN) or a
flooding-based mesh (Meshtastic-style) should be used as the transport layer,
with iotdata as the payload format within that transport. The two concerns (mesh
routing and payload encoding) are orthogonal — iotdata packets can be carried
over any transport, and Appendix G's relay protocol is an optional convenience
for deployments that do not need or cannot afford a full mesh networking stack.

---

_This document and the reference implementation are maintained at
[https://libiotdata.org]._
