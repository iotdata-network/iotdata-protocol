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

[moved]

## 2. Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in RFC 2119.

**Bit numbering:** All bit diagrams in this document use MSB-first (big-endian)
bit order. Bit 7 of a byte is the most significant bit and is transmitted first.
Multi-bit fields are packed MSB-first: the most significant bit of a field
occupies the earliest bit position in the stream.

**Bit offset:** Bit positions within a packet are numbered from 0, starting at
the MSB of the first byte. Bit 0 is the MSB of byte 0; bit 7 is the LSB of byte
0; bit 8 is the MSB of byte 1; and so on.

**Byte boundaries:** Fields are NOT byte-aligned unless they happen to fall on a
byte boundary. The packet is a continuous bit stream; byte boundaries have no
structural significance. The final byte is zero-padded in its least-significant
bits if the total bit count is not a multiple of 8.

**Quantisation:** The process of mapping a continuous or large-range value to a
reduced set of discrete steps that fit in fewer bits. All quantisation in this
protocol uses `round()` (round half away from zero), unless otherwise specified,
and can be carried out as floating-point or integer-only.

## 3. Design Principles

[moved]

## 4. Packet Structure Overview

An iotdata packet consists of the following sections, in order:

```text
+--------+------------+-------------+------------+
| Header | Presence   | Data Fields | TLV Fields |
| 32 bits| 8 to 32 b. | variable    | optional   |
+--------+------------+-------------+------------+
```

All sections are packed as a continuous bit stream with no alignment gaps
between them.

- **Header** (32 bits): Always present. Identifies the variant, station, and
  sequence number.

- **Presence** (8 to 32 bits): Always present. One to four presence bytes
  chained via extension bits indicate which data fields follow. data fields and
  TLV data follow.

- **Data fields** (variable): Zero or more sensor data fields, packed in the
  order defined by the variant's field table.

- **TLV fields** (variable, optional): Zero or more type-length-value data
  entries.

The minimum valid packet is 5 bytes (header + one presence byte with no fields
set), though such a packet carries no sensor data and serves only as a
heartbeat. In practice the minimum useful packet is 6 bytes (header + presence +
battery = 46 bits).

## 5. Header

The header is always the first 32 bits of a packet.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Var  |      Station ID       |           Sequence            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Variant** (4 bits, offset 0): Index into the variant field table (Section 7).
Values 0-14 are usable for sensor oriented data; value 15 is RESERVED for the
mesh protocol control messages. A non mesh capable device encountering variant
15 SHOULD reject the packet.

**Station ID** (12 bits, offset 4): Identifies the transmitting station. Range
0-4095. Station IDs are assigned by the deployment operator; this protocol does
not define an allocation mechanism.

**Sequence** (16 bits, offset 16): Monotonically increasing packet counter,
wrapping from 65535 to 0. The receiver MAY use this to detect lost packets. The
wrap-around is expected and MUST NOT be treated as an error.

The header could be reduced to 24 bits by a reduction in the Station ID (from 12
to 8 bits, saving 4 bits) and the Sequence (from 16 to 12 bits, saving 4 bits).
This would retain station diversity (at 256, rather than 4096) and loss
detection (at 4096 packet window, rather than 65536). Such a modification is not
contemplated in this version of the protocol.

## 6. Presence Bytes

Immediately following the header, one or more presence bytes indicate which data
fields are included in the packet. Presence bytes form an extension chain: each
byte has an extension bit that, when set, indicates another presence byte
follows.

### Presence Byte 0 (always present)

```text
 7   6   5   4   3   2   1   0
+---+---+---+---+---+---+---+---+
|Ext|TLV| S5| S4| S3| S2| S1| S0|
+---+---+---+---+---+---+---+---+
```

- **Ext** (bit 7): Extension flag. If set, Presence Byte 1 follows immediately.
  If clear, no further presence bytes exist.

- **TLV** (bit 6): TLV data flag. If set, one or more TLV entries (Section 9)
  follow after all data fields. Builds excluding TLV might use this as a Data
  field, but this is not contemplated in this version of the protocol.

- **S0-S5** (bits 5-0): Data fields 0 through 5. Each bit, when set, indicates
  that the corresponding field (as defined by the variant's field table) is
  present in the packet. The field data appears in field order: S0 first, then
  S1, S2, and so on.

### Presence Byte N (N ≥ 1, conditional)

Present only when the Ext bit in the preceding presence byte is set.

```text
 7   6   5   4   3   2   1   0
+---+---+---+---+---+---+---+---+
|Ext| S6| S5| S4| S3| S2| S1| S0|
+---+---+---+---+---+---+---+---+
```

- **Ext** (bit 7): Extension flag. If set, another presence byte follows. This
  allows chaining of an arbitrary number of presence bytes.

- **S0-S6** (bits 6-0): Data fields for this presence byte. The first extension
  byte (pres1) carries fields 6-12, the second (pres2) carries fields 13-19, and
  so on.

### Field Capacity

The maximum number of data fields available depends on the number of presence
bytes:

| Presence Bytes  | Total Data Fields | Formula       |
| --------------- | ----------------- | ------------- |
| 1 (pres0)       | 6                 | 6             |
| 2 (pres0+1)     | 13                | 6 + 7         |
| 3 (pres0+1+2)   | 20                | 6 + 7 + 7     |
| 4 (pres0+1+2+3) | 27                | 6 + 7 + 7 + 7 |

The reference implementation supports up to 4 presence bytes (27 data fields).
In practice, the default weather station variant uses 2 presence bytes for 12
data fields. It is unlikely an implementation would pratically require more than
2-3 presence bytes.

### Extension Byte Optimisation

The encoder only emits the minimum number of presence bytes needed for the
fields actually present. If all set fields fit in pres0 (fields 0-5), no
extension bytes are emitted, even if the variant defines fields in pres1. This
optimisation reduces packet size for common transmissions that include only the
most frequently updated fields.

### Field Ordering

Data fields are packed in strict field order. First, all set fields from
Presence Byte 0 are packed in order S0, S1, ..., S5. Then, if Presence Byte 1 is
present, fields S6 through S12 are packed. The TLV section (if present) always
comes last, after all data fields.

The meaning of each field position — which sensor field type it represents — is
determined entirely by the variant table (Section 7).

## 7. Variant Definitions

The variant field in the header selects a field mapping that determines which
field type occupies each presence bit position. This mechanism allows different
sensor types to prioritise their most commonly transmitted fields in Presence
Byte 0, while less frequent fields (such as position and datetime) occupy later
presence bytes and only trigger extension bytes when actually transmitted.

All field encodings (Section 8) are universal and independent of variant. The
variant affects only which encoding type is associated with which field
position, and which label is used in human-readable output and JSON
serialisation.

Fields may be repeated, such as to specify multiple temperature entries which
have different meanings (for example, the temperature of the microcontroller vs.
the temperature of the environment). This is supported by the protocol, but not
the current reference implementation (which will be modified at some future date
to do so).

### Variant Table Structure

In the reference implementation, each variant is defined as:

```c
typedef struct {
    iotdata_field_type_t  type;   /* encoding type for this field */
    const char           *label;  /* JSON key and display label  */
} iotdata_field_def_t;

typedef struct {
    const char          *name;
    uint8_t              num_pres_bytes;
    iotdata_field_def_t  fields[IOTDATA_MAX_DATA_FIELDS];
} iotdata_variant_def_t;
```

The `fields[]` array is flat: entries 0-5 map to Presence Byte 0, entries 6-12
to Presence Byte 1, entries 13-19 to Presence Byte 2, and so on. Unused trailing
fields should have type `IOTDATA_FIELD_NONE`.

### Default Variant: Weather Station

The built-in default variant (variant 0) is a general-purpose weather station
layout. It is enabled by defining `IOTDATA_VARIANT_MAPS_DEFAULT` at compile
time. It is illustrative and not mandated for this use case: there are no
standardised variants, as global interoperability is not a goal.

| Pres Byte | Field | Type              | Label       | Bits |
| --------- | ----- | ----------------- | ----------- | ---- |
| 0         | S0    | BATTERY           | battery     | 6    |
| 0         | S1    | LINK              | link        | 6    |
| 0         | S2    | ENVIRONMENT       | environment | 24   |
| 0         | S3    | WIND              | wind        | 22   |
| 0         | S4    | RAIN              | rain        | 12   |
| 0         | S5    | SOLAR             | solar       | 14   |
| 1         | S6    | CLOUDS            | clouds      | 4    |
| 1         | S7    | AIR_QUALITY_INDEX | air_quality | 9    |
| 1         | S8    | RADIATION         | radiation   | 28   |
| 1         | S9    | POSITION          | position    | 48   |
| 1         | S10   | DATETIME          | datetime    | 24   |
| 1         | S11   | FLAGS             | flags       | 8    |

This layout prioritises the most commonly transmitted weather data (battery,
environment, wind, rain, solar, link quality) in Presence Byte 0, minimising
packet size for routine transmissions. The less frequently updated fields
(position, datetime, radiation) are placed in Presence Byte 1 and only add to
the packet when present.

Note that the weather station variant uses the ENVIRONMENT, WIND, RAIN, and
RADIATION bundle types (see Sections 8.3, 8.12, 8.16, 8.23) rather than their
individual component types. See Section 8 for a discussion of when to use
bundled vs individual field types.

### Custom Variant Maps

Applications can define their own variant tables at compile time using the
`IOTDATA_VARIANT_MAPS` and `IOTDATA_VARIANT_MAPS_COUNT` defines. This completely
replaces the default variant table.

```c
/* Define custom variants */
const iotdata_variant_def_t my_variants[] = {
    [0] = {
        .name = "soil_sensor",
        .num_pres_bytes = 1,
        .fields = {
            { IOTDATA_FIELD_BATTERY,     "battery"    },
            { IOTDATA_FIELD_LINK,        "link"       },
            { IOTDATA_FIELD_TEMPERATURE, "soil_temp"  },
            { IOTDATA_FIELD_HUMIDITY,    "soil_moist" },
            { IOTDATA_FIELD_DEPTH,       "soil_depth" },
            { IOTDATA_FIELD_NONE,        NULL         },
        },
    },
};
```

Compile with:

```bash
cc -DIOTDATA_VARIANT_MAPS=my_variants -DIOTDATA_VARIANT_MAPS_COUNT=1 ...
```

Custom variants may use any combination of the available field types and may
place them in any field position. Up to 15 variants can be registered as variant
IDs 0-14; with variant 15 reserved for the mesh protocol (see Appendix G).

### Registered Variants

| Variant | Name            | Pres Bytes | Fields | Notes                        |
| ------- | --------------- | ---------- | ------ | ---------------------------- |
| 0       | weather_station | 2          | 12     | Default (built-in)           |
| 1-14    | (application)   | —          | —      | User-defined via custom maps |
| 15      | MESH PROTOCOL   | —          | —      | Mesh protocol (Appendix G)   |

A receiver encountering an unknown variant SHOULD not process the packet and
flag it as using an unknown variant (see Section 11.4).

## 8. Field Encodings

Each field type has a specified bit layout that is independent of which presence
field it occupies. Fields are always packed MSB-first.

The protocol provides over 20 built-in field types. Some of these exist in both
individual and bundled forms, to aid efficiency for cases where like data (e.g.
temperature, pressure and humidity) are always concurrently measured and
transmitted.

- **Environment** (Section 8.3) is a convenience bundle that packs temperature,
  pressure, and humidity into a single 24-bit field. The same three measurements
  are also available as individual field types: Temperature (8.9), Pressure
  (8.10), and Humidity (8.11). The encodings and quantisation are identical.

- **Wind** (Section 8.12) is a convenience bundle that packs wind speed,
  direction, and gust into a single 22-bit field. The same three measurements
  are also available as individual field types: Wind Speed (8.13), Wind
  Direction (8.14), and Wind Gust (8.15). The encodings and quantisation are
  identical.

- **Rain** (Section 8.16) is a convenience bundle that packs rain rate, and rain
  size into a single 12-bit field. The same two measurements are also available
  as individual field types: Rate Rate (8.17), and Rain Size (8.18). The
  encodings and quantisation are identical.

- **Air Quality** (Section 8.19) is a convenience bundle that packs air quality
  index, air quality pm, and air quality gas into a single multi-bit field. The
  same three measurements are also available as individual field types: Air
  Quality Index (8.20), Air Quality PM (8.21), and Air Quality Gas (8.22). The
  encodings and quantisation are identical.

- **Radiation** (Section 8.23) is a convenience bundle that packs radiation cpm,
  and radiation dose into a single 28-bit field. The same two measurements are
  also available as individual field types: Radiation CPM (8.24), and Radiation
  Dose (8.25). The encodings and quantisation are identical.

A variant definition chooses which form to use. The default weather station
variant uses many of the bundled forms as the sensors generate the entire bundle
of values concurrently. A custom variant might use the individual forms to
include only the specific measurements it needs, or to place them in different
priority positions, or where they are sourced from different sensors at
different times. For example, the commonly used BME280/680 sensor can generate
temperature, pressure and humidity readings concurrently.

Note that at this point, some bundles have no standalone forms, such as the
Solar bundle with Irradiance and Ultraviolet measurements. This may be addressed
in future versions of this protocol.

### 8.1. Battery

6 bits total.

```text
 0   1   2   3   4   5
+---+---+---+---+---+---+
|   Level           |Chg|
|   (5 bits)        |(1)|
+---+---+---+---+---+---+
```

**Level** (5 bits): Battery charge level, quantised from 0-100% to 0-31.

Encode: `q = round(level_pct / 100.0 * 31.0)`

Decode: `level_pct = round(q / 31.0 * 100.0)`

Resolution: ~3.2 percentage points.

**Charging** (1 bit): 1 = charging, 0 = discharging/not charging.

### 8.2. Link

6 bits total.

```text
 0   1   2   3   4   5
+---+---+---+---+---+---+
|   RSSI        | SNR   |
|   (4 bits)    | (2)   |
+---+---+---+---+---+---+
```

**RSSI** (4 bits): Range: -120 to -60 dBm. Resolution: 4 dBm (15 steps).

Encode: `q = (rssi_dbm - (-120)) / 4`

Decode: `rssi_dbm = -120 + q * 4`

**SNR** (2 bits): Range: -20 to +10 dB. Resolution: 10 dB (3 steps: -20, -10, 0,
+10).

Encode: `q = round((snr_db - (-20.0)) / 10.0)`

Decode: `snr_db = -20.0 + q * 10.0`

This field is source-agnostic: while designed for LoRa link metrics, the same
encoding is suitable for 802.11ah or other low-power RF links with comparable
RSSI and SNR ranges.

### 8.3. Environment

24 bits total.

```text
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Temperature  | Pressure      | Humidity      |
|  (9 bits)     | (8 bits)      | (7 bits)      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Temperature** (9 bits): Range: -40.00°C to +80.00°C. Resolution: 0.25°C (480
steps, 9 bits = 512 values).

Encode: `q = round((temp_c - (-40.0)) / 0.25)`

Decode: `temp_c = -40.0 + q * 0.25`

**Pressure** (8 bits): Range: 850 to 1105 hPa. Resolution: 1 hPa (255 steps).

Encode: `q = pressure_hpa - 850`

Decode: `pressure_hpa = q + 850`

**Humidity** (7 bits): Range: 0 to 100%. Resolution: 1% (7 bits = 128 values,
0-100 used).

Encode/Decode: direct (no quantisation needed).

### 8.4. Solar

14 bits total.

```text
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Irradiance    | UV Idx  |
|   (10 bits)     | (4 bits)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Irradiance** (10 bits): Range: 0 to 1023 W/m². Resolution: 1 W/m². Direct
encoding.

**UV Index** (4 bits): Range: 0 to 15. Direct encoding.

### 8.5. Depth

10 bits total.

Range: 0 to 1023 cm. Resolution: 1 cm. Direct encoding.

This is a generic depth field. The variant label determines its semantic meaning
(snow depth, ice thickness, water level, etc.). The wire encoding is identical
regardless of label.

### 8.6. Flags

8 bits total.

```text
 0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
|       Flags (8 bits)          |
+---+---+---+---+---+---+---+---+
```

General-purpose bitmask. Bit assignments are deployment-specific and are not
defined by this protocol. Example uses include: low battery warning, sensor
fault indicators, tamper detection, or configuration acknowledgement flags.

### 8.7. Position

48 bits total.

```text
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              Latitude (24 bits)               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              Longitude (24 bits)              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Latitude** (24 bits): Range: -90.0° to +90.0°.

Encode: `q = round((lat - (-90.0)) / 180.0 * 16777215.0)`

Decode: `lat = q / 16777215.0 * 180.0 + (-90.0)`

Resolution: 180.0 / 16777215 ≈ 0.00001073° ≈ 1.19 metres at the equator.

**Longitude** (24 bits): Range: -180.0° to +180.0°.

Encode: `q = round((lon - (-180.0)) / 360.0 * 16777215.0)`

Decode: `lon = q / 16777215.0 * 360.0 + (-180.0)`

Resolution: 360.0 / 16777215 ≈ 0.00002146° ≈ 2.39 metres at the equator,
reducing with cos(latitude).

This field is source-agnostic. The position may originate from a GNSS receiver,
WiFi geolocation, cell tower triangulation, or static configuration. The
protocol does not indicate the source or its accuracy; see Section 11.2 and 11.3
for discussion.

### 8.8. Datetime

24 bits total.

```text
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Ticks (24 bits)                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Ticks** (24 bits): Time offset from January 1 00:00:00 UTC of the current
year, measured in 5-second ticks.

Encode: `ticks = seconds_from_year_start / 5`

Decode: `seconds = ticks * 5`

Maximum value: 16,777,215 ticks = 83,886,075 seconds ≈ 970.9 days.

Resolution: 5 seconds.

The year is NOT transmitted. The receiver resolves the year using its own clock;
see Section 11.1 for the year resolution algorithm.

This field is source-agnostic. The time may originate from a GNSS receiver, NTP
synchronisation, or a local RTC. The protocol does not indicate the source or
its drift characteristics; see Section 11.3.

### 8.9. Temperature (standalone)

9 bits total.

Range: -40.00°C to +80.00°C. Resolution: 0.25°C (480 steps, 9 bits = 512
values).

Encode: `q = round((temp_c - (-40.0)) / 0.25)`

Decode: `temp_c = -40.0 + q * 0.25`

This is the same encoding as the temperature component of the Environment bundle
(Section 8.3). Use this standalone type in variants that need temperature
without pressure and humidity.

### 8.10. Pressure (standalone)

8 bits total.

Range: 850 to 1105 hPa. Resolution: 1 hPa (255 steps).

Encode: `q = pressure_hpa - 850`

Decode: `pressure_hpa = q + 850`

This is the same encoding as the pressure component of the Environment bundle
(Section 8.3).

### 8.11. Humidity (standalone)

7 bits total.

Range: 0 to 100%. Resolution: 1% (7 bits = 128 values, 0-100 used).

Encode/Decode: direct (no quantisation needed).

This is the same encoding as the humidity component of the Environment bundle
(Section 8.3).

### 8.12. Wind (bundle)

22 bits total.

```text
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Speed      | Direction     |  Gust       |
|  (7 bits)   | (8 bits)      | (7 bits)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

This is a convenience bundle that packs wind speed, direction, and gust speed
into a single field. The component encodings are identical to the standalone
Wind Speed (8.13), Wind Direction (8.14), and Wind Gust (8.15) types.

**Speed** (7 bits): Range: 0 to 63.5 m/s. Resolution: 0.5 m/s.

Encode: `q = round(speed_ms / 0.5)`

Decode: `speed_ms = q * 0.5`

**Direction** (8 bits): Range: 0° to 355° (true bearing). Resolution: ~1.41°
(360/256).

Encode: `q = round(direction_deg / 360.0 * 256.0) & 0xFF`

Decode: `direction_deg = q / 256.0 * 360.0`

**Gust** (7 bits): Range: 0 to 63.5 m/s. Resolution: 0.5 m/s.

Encode/Decode: same as Speed.

### 8.13. Wind Speed (standalone)

7 bits total.

Range: 0 to 63.5 m/s. Resolution: 0.5 m/s.

Encode: `q = round(speed_ms / 0.5)`

Decode: `speed_ms = q * 0.5`

Same encoding as the speed component of the Wind bundle (8.12).

### 8.14. Wind Direction (standalone)

8 bits total.

Range: 0° to 355° (true bearing). Resolution: ~1.41° (360/256).

Encode: `q = round(direction_deg / 360.0 * 256.0) & 0xFF`

Decode: `direction_deg = q / 256.0 * 360.0`

Same encoding as the direction component of the Wind bundle (8.12).

### 8.15. Wind Gust (standalone)

7 bits total.

Range: 0 to 63.5 m/s. Resolution: 0.5 m/s.

Encode: `q = round(gust_ms / 0.5)`

Decode: `gust_ms = q * 0.5`

Same encoding as the gust component of the Wind bundle (8.12).

### 8.16. Rain (bundle)

12 bits total.

```text
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+
|  Rate         | Size  |
|  (8 bits)     | (4)   |
+-+-+-+-+-+-+-+-+-+-+-+-+
```

This is a convenience bundle that packs rain rate and size into a single field.
The component encodings are identical to the standalone Rain Rate (8.17), and
Rain Size (8.18) types.

**Rate** (8 bits):

Range: 0 to 255 mm/hr. Resolution: 1 mm/hr. Direct encoding.

**Size** (4 bits):

Range: 0 to 6.0mm. Resolution: 0.25 mm.

Encode: `q = round(rain_size / 0.25)`

Decode: `rain_size = q * 0.25`

### 8.17. Rain Rate (standalone)

8 bits total.

Range: 0 to 255 mm/hr. Resolution: 1 mm/hr. Direct encoding.

### 8.18. Rain Size (standalone)

4 bits total.

Range: 0 to 6.0mm. Resolution: 0.25 mm.

Encode: `q = round(rain_size / 0.25)`

Decode: `rain_size = q * 0.25`

### 8.19. Air Quality (bundle)

Variable length (minimum 21 bits).

This is a convenience bundle that packs air quality index, particulate matter,
and gas readings into a single field. The component encodings are identical to
the standalone Air Quality Index (8.19), Air Quality PM (8.20), and Air Quality
Gas (8.21) types.

```text
+-----------+-----------+-----------+
| AQ Index  | AQ PM     | AQ Gas    |
| (9 bits)  | (4+ bits) | (8+ bits) |
+-----------+-----------+-----------+
```

The three sub-fields are packed in order: index, PM, and gas. Each sub-field
includes its own presence mask, so absent PM channels and gas slots consume no
bits beyond the mask itself.

Minimum: 9 (index) + 4 (PM mask, no channels) + 8 (gas mask, no slots) = 21
bits. Typical SEN55 full reading: 9 + 36 + 24 = 69 bits.

### 8.20. Air Quality Index (standalone)

9 bits total.

Range: 0 to 500 AQI (Air Quality Index). Resolution: 1 AQI. Direct encoding (9
bits = 512 values, 0-500 used).

### 8.21. Air Quality PM (standalone)

4 to 36 bits total (variable).

```text
 0
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+- - - - -+
|P|P|P|P| ch0   | ch1 ...  (8 bits per present channel)
|1|25|4|10|       |
+-+-+-+-+-+-+-+-+- - - - -+
```

4-bit presence mask followed by 8 bits for each present PM channel. Resolution:
5 µg/m³.

**Presence mask** (4 bits):

- Bit 0: PM1 present
- Bit 1: PM2.5 present
- Bit 2: PM4 present
- Bit 3: PM10 present

**Each channel** (8 bits):

Range: 0 to 1275 µg/m³. Resolution: 5 µg/m³ (255 steps).

Encode: `q = value_ugm3 / 5`

Decode: `value_ugm3 = q * 5`

The 5 µg/m³ resolution matches the ±5 µg/m³ precision of typical
laser-scattering PM sensors (e.g. Sensirion SEN55, Plantower PMS5003).

Typical sensors output all four channels simultaneously; a presence mask of 0xF
(all present) with 4 × 8 = 32 data bits is the common case, giving 36 bits
total.

### 8.22. Air Quality Gas (standalone)

8 to 84 bits total (variable).

```text
 0
 0 1 2 3 4 5 6 7 8 9 ...
+-+-+-+-+-+-+-+-+- - - - - - -+
|V|N|C|C|H|O|R|R| slot0 | slot1 ...
|O|O|O|O|C|3|6|7|       |
|C|X|2| |H| | | |       |
+-+-+-+-+-+-+-+-+- - - - - - -+
```

8-bit presence mask followed by data for each present gas slot. Each slot has a
fixed bit width and resolution determined by its position in the mask.

**Presence mask** (8 bits):

- Bit 0: VOC Index
- Bit 1: NOx Index
- Bit 2: CO₂
- Bit 3: CO
- Bit 4: HCHO (formaldehyde)
- Bit 5: O₃ (ozone)
- Bit 6: Reserved
- Bit 7: Reserved

**Slot encodings**:

| Slot | Gas  | Bits | Resolution  | Range    | Unit |
| ---- | ---- | ---- | ----------- | -------- | ---- |
| 0    | VOC  | 8    | 2 index pts | 0-510    | idx  |
| 1    | NOx  | 8    | 2 index pts | 0-510    | idx  |
| 2    | CO₂  | 10   | 50 ppm      | 0-51,150 | ppm  |
| 3    | CO   | 10   | 1 ppm       | 0-1,023  | ppm  |
| 4    | HCHO | 10   | 5 ppb       | 0-5,115  | ppb  |
| 5    | O₃   | 10   | 1 ppb       | 0-1,023  | ppb  |
| 6    | Rsvd | 10   | —           | —        | —    |
| 7    | Rsvd | 10   | —           | —        | —    |

Encode: `q = value / resolution`

Decode: `value = q * resolution`

VOC and NOx index slots carry Sensirion SGP4x-style algorithm indices (1-500
typical). The 2-point resolution is well within the ±15/±50 index point
device-to-device variation.

CO₂ at 50 ppm resolution covers the full SCD4x range (0-40,000 ppm) and exceeds
its ±40 ppm + 5% accuracy.

HCHO at 5 ppb resolution matches the ~10 ppb accuracy of typical electrochemical
formaldehyde sensors (e.g. Sensirion SEN69C, Dart WZ-S).

A typical Sensirion SEN55 station (VOC + NOx) sends 8 + 8 + 8 = 24 bits. A SEN66
station (VOC + NOx + CO₂) sends 8 + 8 + 8 + 10 = 34 bits.

### 8.23. Radiation (bundle)

28 bits total.

```text
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  CPM                      | Dose                      |
|  (14 bits)                | (14 bits)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

This is a convenience bundle that packs radiation CPM and dose into a single
field. The component encodings are identical to the standalone Radiation CPM
(8.24) and Radiation Dose (8.25) types.

**CPM** (14 bits):

Range: 0 to 16383 counts per minute (CPM). Resolution: 1 CPM. Direct encoding.

This field carries the raw count rate from a Geiger-Müller tube or similar
radiation detector.

**Dose** (14 bits):

Range: 0 to 163.83 µSv/h. Resolution: 0.01 µSv/h (16,383 steps).

Encode: `q = round(dose_usvh / 0.01)`

Decode: `dose_usvh = q * 0.01`

This field carries the computed dose rate. The relationship between CPM and dose
rate is detector-specific and is not defined by this protocol.

### 8.24. Radiation CPM (standalone)

14 bits total.

Range: 0 to 16383 counts per minute (CPM). Resolution: 1 CPM. Direct encoding.

This field carries the raw count rate from a Geiger-Müller tube or similar
radiation detector.

### 8.25. Radiation Dose (standalone)

14 bits total.

Range: 0 to 163.83 µSv/h. Resolution: 0.01 µSv/h (16,383 steps).

Encode: `q = round(dose_usvh / 0.01)`

Decode: `dose_usvh = q * 0.01`

This field carries the computed dose rate. The relationship between CPM and dose
rate is detector-specific and is not defined by this protocol.

### 8.26. Clouds

4 bits total.

Range: 0 to 8 okta. Resolution: 1 okta. Direct encoding (4 bits = 16 values, 0-8
used).

Clouds measures cloud cover in okta (eighths of sky covered), following the
standard meteorological convention where 0 = clear sky and 8 = fully overcast.

### 8.27. Image

Variable length. Minimum 2 bytes (length + control), maximum 256 bytes (length +
control + 254 bytes of pixel data).

```text
 0               1
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-  ...  -+
|  Length (8)    |  Control (8)  | Pixel Data   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-  ...  -+
```

This is the only variable-length data field in the protocol. The length byte at
the start tells the decoder how many additional bytes follow, allowing the field
to be skipped without understanding its contents.

**Length** (8 bits): Number of bytes that follow the length byte, including the
control byte and all pixel data. Range: 1-255.

A length of 1 indicates a control byte only with no pixel data. This is not
useful in practice but is legal.

The total field size in bytes is `1 + Length`. The total field size in bits is
`(1 + Length) × 8`.

**Control** (8 bits): Describes the pixel format, image dimensions, compression
method, and flags. The decoder reads this byte to determine how to interpret all
subsequent bytes.

```text
 0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| Format| Size  | Comp  | Flags |
| (2)   | (2)   | (2)   | (2)   |
+---+---+---+---+---+---+---+---+
```

**Format** (bits 7-6): Pixel depth.

| Value | Name    | Bits/pixel | Description                       |
| ----- | ------- | ---------- | --------------------------------- |
| 0     | BILEVEL | 1          | Black and white (1-bit per pixel) |
| 1     | GREY4   | 2          | 4-level greyscale                 |
| 2     | GREY16  | 4          | 16-level greyscale                |
| 3     |         | —          | Reserved                          |

For BILEVEL, each pixel is a single bit: 0 = black, 1 = white. Pixels are packed
MSB-first within each byte, left-to-right across each row, rows top-to-bottom.

For GREY4, each pixel is 2 bits: 0 = black, 1 = dark grey, 2 = light grey, 3 =
white. Pixels are packed MSB-first, four pixels per byte.

For GREY16, each pixel is 4 bits (one nibble): 0 = black, 15 = white. Pixels are
packed high-nibble-first, two pixels per byte.

**Size** (bits 5-4): Image dimensions (width × height).

| Value | Dimensions | Pixels | Raw bytes (1bpp) | Raw bytes (4bpp) |
| ----- | ---------- | ------ | ---------------- | ---------------- |
| 0     | 24 × 18    | 432    | 54               | 216              |
| 1     | 32 × 24    | 768    | 96               | 384              |
| 2     | 48 × 36    | 1,728  | 216              | 864              |
| 3     | 64 × 48    | 3,072  | 384              | 1,536            |

All sizes use a 4:3 aspect ratio. The size tier determines both width and
height; non-standard dimensions are not supported.

**Compression** (bits 3-2): Compression method applied to pixel data.

| Value | Name       | Description                          |
| ----- | ---------- | ------------------------------------ |
| 0     | RAW        | Uncompressed pixel data              |
| 1     | RLE        | Run-length encoding (Section 8.27.1) |
| 2     | HEATSHRINK | Heatshrink LZSS (Section 8.27.2)     |
| 3     |            | Reserved                             |

**Flags** (bits 1-0):

| Bit | Name     | Description                                       |
| --- | -------- | ------------------------------------------------- |
| 1   | FRAGMENT | This image is a fragment; more fragments follow   |
| 0   | INVERT   | Display with inverted polarity (0=white, 1=black) |

The FRAGMENT flag enables multi-packet image transmission for cases where the
pixel data exceeds the available payload. Fragments share the same control byte;
the receiver reassembles using the packet sequence number and station_id. For
v1, single-frame images (FRAGMENT = 0) are the expected case.

The INVERT flag indicates that the pixel sense is reversed. This is useful for
difference-frame images where motion pixels are naturally encoded as 1 (white on
black background). The flag allows the display layer to render with the correct
visual polarity without the encoder needing to invert the pixel data.

#### Design Philosophy

The Image field defines a container for a rectangular pixel grid. It does not
specify what the pixels represent. The sensor implementation decides what is
most informative — a full-frame downscale, a cropped region-of-interest around
detected motion, a background-subtracted difference mask, a depth map, or any
other rectangular image. The field carries the result; the semantics are a
property of the sensor and variant, not the encoding.

#### Variable-Length Decoding

Unlike all other data fields in the protocol, Image has a variable bit width.
The decoder handles this as follows:

1. The presence bit for the Image slot is set.
2. The decoder reads the first byte (Length).
3. The decoder consumes `Length` additional bytes.
4. Decoding continues at the next field's bit offset.

Implementations that do not support Image can skip the field by reading the
length byte and advancing by `Length` bytes, without interpreting the control
byte or pixel data. This preserves forward compatibility: a decoder compiled
without Image support can still decode all other fields in the packet.

#### 8.27.1. RLE Compression

When Compression = RLE, the pixel data is encoded as a sequence of run-length
pairs. Each pair is a single byte:

```text
 0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
|Val|       Run Length (7)       |
+---+---+---+---+---+---+---+---+
```

For BILEVEL format, **Val** (bit 7) is the pixel value (0 or 1) and **Run
Length** (bits 6-0) is the number of consecutive pixels with that value, minus 1
(range 1-128 pixels per run).

For GREY4 and GREY16 formats, the encoding switches to a byte-pair scheme: the
first byte is a raw pixel value (2 or 4 bits, zero-padded to 8 bits) and the
second byte is the run count minus 1. This produces 2 bytes per run but handles
the wider pixel values cleanly.

Runs that exceed 128 pixels (BILEVEL) or 256 pixels (greyscale) are split into
consecutive run entries with the same value.

The decoder reconstructs the pixel grid left-to-right, top-to-bottom, consuming
runs until width × height pixels have been produced.

RLE is particularly effective for BILEVEL images with large uniform regions,
such as background-subtracted motion frames, where compression ratios of 2:1 to
6:1 are typical.

#### 8.27.2. Heatshrink Compression

When Compression = HEATSHRINK, the pixel data (in its raw packed form) has been
compressed using the heatshrink LZSS algorithm.

The heatshrink parameters are fixed by this protocol and MUST NOT be varied
per-packet:

- Window size: 8 (256-byte window)
- Lookahead size: 4 (16-byte lookahead)

These parameters are chosen for minimal RAM usage at the decoder (approximately
256 bytes for decompression state) while still providing useful compression. The
decoder does not need to be told the parameters; they are implicit in the field
type.

Heatshrink is most useful for GREY4 and GREY16 formats where pixel data has more
entropy than BILEVEL and simple RLE is less effective.

#### 8.27.3. Payload Budget

The length byte (8 bits) limits the field value to 255 bytes after the length
byte itself: 1 control byte plus up to 254 bytes of pixel data.

The following table shows which format/size combinations fit within 254 bytes
without compression:

| Size    | BILEVEL (1bpp) | GREY4 (2bpp) | GREY16 (4bpp) |
| ------- | :------------: | :----------: | :-----------: |
| 24 × 18 |     54 B ✓     |   108 B ✓    |    216 B ✓    |
| 32 × 24 |     96 B ✓     |   192 B ✓    |    384 B ✗    |
| 48 × 36 |    216 B ✓     |   432 B ✗    |    864 B ✗    |
| 64 × 48 |    384 B ✗     |   768 B ✗    |   1,536 B ✗   |

Combinations marked ✗ require compression to fit. In practice, BILEVEL at 32 ×
24 (96 bytes raw, typically 40-60 bytes with RLE) is the recommended default for
single-frame LoRa transmission. It provides sufficient resolution to distinguish
human silhouettes, vehicles, and animals while leaving substantial room for
other iotdata fields in the same packet.

The LoRa payload limit (222 bytes at SF7/125kHz, 115 bytes at SF9, 51 bytes at
SF10) further constrains the practical combinations. For higher spreading
factors, 24 × 18 BILEVEL with RLE is the safest choice.

#### 8.27.4. Recommended Practices

- **Default choice:** BILEVEL format, 32 × 24 size, RLE compression. This
  produces 40-60 byte thumbnails for typical motion frames, fits comfortably in
  a single LoRa packet at any spreading factor, and requires trivial
  encode/decode logic.

- **ROI cropping:** If the sensor detects motion in a small region of the camera
  frame, cropping to that region before downscaling preserves more detail than
  downscaling the entire frame. The Image field does not carry crop coordinates;
  these are a property of the sensor's processing pipeline, not the transport
  encoding.

- **Difference frames:** For background-subtracted motion images, set the INVERT
  flag if the natural encoding is white-on-black (motion pixels = 1). The
  resulting BILEVEL image compresses exceptionally well with RLE due to large
  background regions.

- **Greyscale use:** GREY16 at 24 × 18 with heatshrink (216 bytes raw, typically
  100-150 bytes compressed) provides a richer visual at the cost of decode
  complexity. Use when the MCU has sufficient resources and the additional
  visual detail is valuable.

- **Multi-frame spanning:** The FRAGMENT flag enables splitting a large
  thumbnail across multiple packets. The gateway reassembles fragments using
  {station_id, sequence} ordering. This adds complexity and fragility (any lost
  fragment invalidates the image) and is not recommended for v1 deployments.

#### 8.27.5. JSON Representation

In the canonical JSON output, the Image field is represented as a structured
object under its variant label (e.g. `"image"`, `"thumbnail"`, `"motion_image"`,
depending on the variant map definition):

```json
{
  "image": {
    "format": "bilevel",
    "size": "32x24",
    "compression": "rle",
    "fragment": false,
    "invert": false,
    "pixels": "base64-encoded-pixel-data"
  }
}
```

The gateway performs decompression before base64-encoding the `pixels` field, so
downstream consumers receive uniform raw pixel data regardless of the
compression method used on the wire.

- `format`: One of `"bilevel"`, `"grey4"`, `"grey16"`.
- `size`: One of `"24x18"`, `"32x24"`, `"48x36"`, `"64x48"`.
- `compression`: One of `"raw"`, `"rle"`, `"heatshrink"`.
- `fragment`: Boolean.
- `invert`: Boolean.
- `pixels`: Base64-encoded decompressed pixel data.

The `compression` field records the wire method for diagnostics but is not
needed for rendering.

## 9. TLV Data

The TLV (Type-Length-Value) section provides an extensible mechanism for
diagnostic data, firmware metadata, user-defined payloads, and future sensor
metadata. It is present only when the TLV bit (bit 6 of Presence Byte 0) is set.
By preference, it should not be used for sensor data per se: such data should
have a designated field type.

The TLV section begins immediately after the last data field, at whatever bit
offset that field ended. There is no alignment padding.

### 9.1. TLV Header

Each TLV entry begins with a 16-bit header:

```text
 0                               1
 0   1   2   3   4   5   6   7   8   9   10  11  12  13  14  15
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
|Fmt|       Type (6)        |Mor|           Length (8)          |
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
```

**Format** (1 bit): 0 = raw bytes. 1 = packed 6-bit string.

**Type** (6 bits): Application-defined type identifier, range 0-63. See Section
9.4 for types.

**More** (1 bit): 1 = another TLV entry follows this one. 0 = this is the last
TLV entry.

**Length** (8 bits): For raw format: number of data bytes (0-255). For string
format: number of characters (0-255).

### 9.2. Raw Format

When Format = 0, the data section is `Length` bytes (Length × 8 bits), packed
MSB-first with no alignment.

Total TLV entry size: 16 + (Length × 8) bits.

### 9.3. Packed String Format

When Format = 1, each character is encoded as 6 bits using the character table
in Appendix A. This saves 25% compared to 8-bit ASCII for the supported
character set (alphanumeric plus space).

Total TLV entry size: 16 + (Length × 6) bits.

Characters outside the 6-bit table MUST NOT be transmitted. An encoder MUST
reject strings containing unencodable characters.

### 9.4. TLV Types

| Type      | Name        | Format | Description                                        |
| --------- | ----------- | ------ | -------------------------------------------------- |
| 0x01-0x0F | (reserved)  | —      | Reserved for globally designated TLVs              |
| 0x01      | VERSION     | string | Firmware and hardware version identification       |
| 0x02      | STATUS      | raw    | Uptime, lifetime uptime, restart count and reason  |
| 0x03      | HEALTH      | raw    | CPU temperature, supply voltage, heap, active time |
| 0x04      | CONFIG      | string | Configuration key-value pairs                      |
| 0x05      | DIAGNOSTIC  | string | Free-form diagnostic message                       |
| 0x06      | USERDATA    | string | User interaction event                             |
| 0x08-0x0F | (reserved)  | —      | Reserved for future globally designated TLVs       |
| 0x10-0x1F | (reserved)  | —      | Reserved for future quality/metadata TLVs          |
| 0x20-     | (available) | —      | Available for proprietary TLVs                     |

Types 0x01-0x0F are reserved for globally designated types, as specified in, and
extended by, this document. They have encoding functions provided in the
reference implementation. Types 0x10-0x1F are reserved for sensor metadata (see
Section 11.3 and Section 15) and may have future reference implementation
support. Types 0x20 onwards are available for application use.

### 9.5. Global TLV Types

The following TLV types are globally designated and have fixed semantics across
all variants and deployments. Implementations SHOULD use these types for their
intended purpose to aid interoperability between sensors, gateways, and
downstream consumers.

All global TLV types are optional. A sensor includes them when the information
is available and the payload budget permits. The recommended transmission
strategy varies by type:

- **VERSION**: Once at boot (first packet after restart).
- **STATUS**: Every Nth packet (e.g. every 10th), or periodically.
- **HEALTH**: Less frequently (e.g. every 50th), or when significantly changed.
- **CONFIG**: Once at boot, or after configuration changes.
- **DIAGNOSTIC**: When a notable condition occurs.
- **USERDATA**: When a user interaction event occurs.

#### 9.5.1. Version (0x01)

Variable length, string format.

Identifies the firmware and hardware versions running on the device. This is
essential for fleet management: knowing which devices are running which firmware
version after an OTA campaign, or identifying hardware revisions with known
issues.

The content uses the same space-delimited key-value convention as Config
(Section 9.5.4), encoded with the 6-bit packed character set (Appendix A):

```text
KEY1 VALUE1 KEY2 VALUE2 ...
```

Recommended keys:

| Key | Description                                        |
| --- | -------------------------------------------------- |
| FW  | Firmware version (build number or encoded version) |
| HW  | Hardware revision                                  |
| BL  | Bootloader version                                 |
| ID  | Device model or type identifier                    |
| SN  | Serial number or unique identifier                 |

Examples:

- `FW 142 HW 3` — firmware build 142, hardware revision 3
- `FW 20401 HW 2 BL 5` — firmware 2.4.1 (encoded as 20401), hardware rev 2,
  bootloader 5
- `ID SNOWV2 FW 38 HW 1` — device model SNOWV2, firmware 38
- `FW 12 HW 1 SN A04F` — with serial number

The key namespace is the same as Config: application-defined, short uppercase
identifiers. The keys listed above are recommendations, not requirements. A
minimal implementation may send only `FW` and `HW`.

Since version information is static within a boot cycle, this TLV is typically
sent only in the first packet after a restart. The gateway or upstream system
can cache it per station_id.

Since the 6-bit character set does not include dots or hyphens, semantic version
strings such as `2.4.1` cannot be encoded directly. Recommended alternatives:

- Concatenated digits: `20401` for 2.4.1 (convention: MMPPP where
  MM=major×100+minor, PPP=patch).
- Plain build number: `142` (monotonically increasing).
- Separate keys: `FWMAJ 2 FWMIN 4 FWPAT 1` (verbose but explicit).

The build number approach is simplest and sufficient for most deployments.

**JSON representation:**

```json
{
  "type": 1,
  "format": "version",
  "data": {
    "FW": "142",
    "HW": "3"
  }
}
```

The gateway parses the space-delimited tokens into key-value pairs, identical to
the Config JSON representation.

#### 9.5.2. Status (0x02)

9 bytes, raw format.

Reports device boot lifecycle: how long since last restart, how long the device
has been alive in total across all boots, how many times it has restarted, and
why the most recent restart occurred.

```text
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Session Uptime (24 bits)             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Lifetime Uptime (24 bits)            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Restarts (16 bits)    | Reason (8)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Session Uptime** (3 bytes, uint24, big-endian): Time since the most recent
boot, measured in 5-second ticks. This matches the resolution and encoding of
the Datetime field (Section 8.8).

Encode: `ticks = uptime_seconds / 5`

Decode: `seconds = ticks * 5`

Maximum: 16,777,215 ticks = 83,886,075 seconds ≈ 970.9 days.

**Lifetime Uptime** (3 bytes, uint24, big-endian): Total accumulated uptime
across all boots since first commissioning, measured in 5-second ticks. Same
encoding as session uptime.

This value requires non-volatile storage (NVS, EEPROM, or flash). The device
persists the accumulated total periodically (e.g. every hour or at shutdown) and
adds the current session uptime when encoding the TLV.

Devices that do not track lifetime uptime MUST transmit 0x000000. The receiver
interprets this as "not tracked" rather than "zero uptime".

**Restarts** (2 bytes, uint16, big-endian): Total number of device starts since
first commissioning, including the current boot. Wraps at 65535. A value of 1
indicates the device has never restarted since first power-on.

**Reason** (1 byte, uint8): Reason for the most recent restart. Bit 7 determines
the interpretation:

- **Bit 7 clear (0x00-0x7F):** Globally defined reason codes, specified by this
  protocol. All implementations MUST use these values for the corresponding
  conditions.

- **Bit 7 set (0x80-0xFF):** Vendor-specific or device-specific reason codes.
  The interpretation depends on the device type and firmware. Receivers that do
  not recognise a vendor-specific code SHOULD display it as a numeric value.

Globally defined reason codes:

| Value     | Name       | Description                                  |
| --------- | ---------- | -------------------------------------------- |
| 0x00      | UNKNOWN    | Reason not available or not determined       |
| 0x01      | POWER_ON   | Cold boot (initial power application)        |
| 0x02      | SOFTWARE   | Intentional software-initiated reset         |
| 0x03      | WATCHDOG   | Watchdog timer expiry                        |
| 0x04      | BROWNOUT   | Supply voltage dropped below threshold       |
| 0x05      | PANIC      | Unrecoverable software fault or exception    |
| 0x06      | DEEPSLEEP  | Wake from deep sleep (normal operation)      |
| 0x07      | EXTERNAL   | External reset pin or button                 |
| 0x08      | OTA        | Reset following over-the-air firmware update |
| 0x09-0x7F | (reserved) | Reserved for future globally defined reasons |

Most microcontrollers expose the reset reason register at boot. For example,
ESP32 provides `esp_reset_reason()` and STM32 provides `__HAL_RCC_GET_FLAG()`.
The encoder maps the platform-specific value to the nearest globally defined
code where possible, or to a vendor-specific code (0x80+) for platform-specific
conditions that have no global equivalent.

The DEEPSLEEP reason (0x06) is expected in normal operation for battery-powered
sensors that sleep between transmission cycles. A high restart count with
DEEPSLEEP reason is healthy; a high restart count with WATCHDOG or PANIC reason
indicates a fault.

**JSON representation:**

```json
{
  "type": 2,
  "format": "status",
  "data": {
    "session_uptime": 86400,
    "lifetime_uptime": 1209600,
    "restarts": 12,
    "reason": "watchdog"
  }
}
```

The gateway destructures the 9-byte raw data into named fields. Uptime values
are converted to seconds (ticks × 5) for the JSON output. A lifetime_uptime of 0
is omitted from the JSON or represented as `null` to indicate "not tracked". The
`reason` field is a lowercase string using the name column from the reason table
for globally defined codes (0x00-0x7F), or the numeric value for vendor-specific
codes (e.g. `"reason": 131`).

#### 9.5.3. Health (0x03)

7 bytes, raw format.

Reports runtime hardware state: thermal, electrical, memory, and duty cycle
metrics. These change during operation and are useful for detecting overheating,
power supply issues, memory leaks, and validating power budgets.

```text
 0               1               2
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| CPU Temp (8)  |      Supply Voltage (16)      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Free Heap (16)        | Active (16)   :
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
:  Active cont. |
+-+-+-+-+-+-+-+-+
```

**CPU Temperature** (1 byte, int8, signed): Internal die temperature in degrees
Celsius. Range: -40 to +85°C. Resolution: 1°C.

Most MCUs have an internal temperature sensor: ESP32 provides
`temperatureRead()`, STM32 provides an internal ADC channel. The reading
reflects die temperature, which is typically 5-15°C above ambient depending on
workload and packaging.

Devices without an internal temperature sensor MUST transmit 0x7F (127). This
value is outside the normal operating range and the receiver interprets it as
"not available".

**Supply Voltage** (2 bytes, uint16, big-endian): Raw supply rail voltage in
millivolts. Range: 0-65535 mV.

This is distinct from the Battery field (Section 8.1) which reports a percentage
level. Supply voltage provides absolute electrical data: solar panel output
voltage, regulator headroom, voltage sag under transmit load, or direct battery
voltage before any regulation.

For devices powered via a regulated 3.3V rail, this may be a fixed value and is
less informative. For solar-powered devices with a wide input range, this is a
key diagnostic.

**Free Heap** (2 bytes, uint16, big-endian): Remaining free heap memory in
bytes. Range: 0-65535.

ESP32 provides `esp_get_free_heap_size()`. For devices with more than 65535
bytes free, report 65535 (capped). A steadily decreasing free heap over time
indicates a memory leak.

Devices without dynamic memory allocation or without a mechanism to query free
heap MUST transmit 0xFFFF (65535). Since this is also the cap value, the
receiver treats it as "healthy or not tracked".

**Session Active** (2 bytes, uint16, big-endian): Accumulated time spent in
active state (not in deep sleep) since the most recent boot, measured in
5-second ticks.

Maximum: 65535 ticks = 327675 seconds ≈ 91.0 hours.

The firmware increments this counter each time it wakes from sleep, accumulating
the duration of each active period. Comparing session active to session uptime
(from Status, Section 9.5.2) yields the duty cycle:

`duty_cycle = session_active / session_uptime`

A sensor with 86400s session uptime but 200 active ticks (1000s) has a duty
cycle of ~1.2%, confirming that power budgets are being met.

For devices that do not sleep (always-on gateways, relay nodes), session active
equals session uptime and this field provides no additional information. Such
devices may omit the Health TLV or set session active to 0x0000 to indicate "not
tracked".

**JSON representation:**

```json
{
  "type": 3,
  "format": "health",
  "data": {
    "cpu_temp": 34,
    "supply_mv": 3842,
    "free_heap": 42816,
    "session_active": 1050
  }
}
```

The gateway destructures the 7-byte raw data into named fields. The `cpu_temp`
is signed degrees Celsius. The `supply_mv` is millivolts. The `free_heap` is
bytes. The `session_active` is converted to seconds (ticks × 5).

A `cpu_temp` of 127 is omitted from the JSON or represented as `null` to
indicate "not available".

#### 9.5.4. Config (0x04)

Variable length, string format.

Reports current device configuration as space-delimited key-value pairs, encoded
using the 6-bit packed character set (Appendix A).

The content is a sequence of alternating tokens separated by single spaces:

```text
KEY1 VALUE1 KEY2 VALUE2 ...
```

Odd-position tokens (1st, 3rd, 5th, ...) are keys. Even-position tokens (2nd,
4th, 6th, ...) are values. The total token count MUST be even (every key has a
corresponding value).

Keys and values MUST NOT contain spaces. Keys and values may use any length and
any mix of characters available in the 6-bit character set. Short uppercase
identifiers are recommended for keys to minimise wire size, but this is a
convention, not a requirement.

Examples:

- `TX 30 SF 7 PW 14 CH 23` — radio configuration
- `INT 10 BAT LOW` — 10-second interval, battery threshold LOW
- `MODE NORMAL THRESH 50` — operating mode and threshold
- `FW 142 HW 3` — firmware version 142, hardware revision 3

The key namespace is application-defined and not standardised by this protocol.
Different sensor types may use different keys. The receiver presents the pairs
as-is; it does not need to understand the key semantics.

In the rare case where configuration values contain characters outside the 6-bit
set, raw format (Format = 0) MAY be used with 8-bit ASCII bytes following the
same space-delimited convention. This should be avoided where possible.

**JSON representation:**

```json
{
  "type": 4,
  "format": "config",
  "data": {
    "TX": "30",
    "SF": "7",
    "PW": "14",
    "CH": "23"
  }
}
```

The gateway parses the space-delimited tokens into alternating key-value pairs
and presents them as a JSON object. Both keys and values are strings.

#### 9.5.5. Diagnostic (0x05)

Variable length, string format.

A free-form diagnostic message from the device. This is the device's mechanism
for reporting conditions that do not map to any structured field: error
messages, warning strings, state transitions, or any other human-readable
diagnostic information.

The message is encoded using the 6-bit packed character set (Appendix A). This
covers uppercase alphanumeric characters, digits, and space — sufficient for
diagnostic messages.

Examples:

- `SENSOR FAULT I2C`
- `LOW SIGNAL`
- `SD FULL`
- `LORA TX FAIL 3`
- `BME280 CRC ERR`

There is no structure imposed on the message content. The protocol does not
define severity levels, error codes, or categories. Conventions such as
prefixing with a subsystem name (`I2C`, `LORA`, `SD`) are recommended but not
required.

In the rare case where a diagnostic message contains characters outside the
6-bit set, raw format (Format = 0) MAY be used with 8-bit ASCII bytes. This
should be avoided where possible as it increases the wire size by 33%.

Multiple diagnostic messages may be sent by chaining TLV entries using the More
bit. Each entry carries one message.

**JSON representation:**

```json
{ "type": 5, "format": "string", "data": "SENSOR FAULT I2C" }
```

The `format` field reflects the wire encoding (`"string"` or `"raw"` in the
exceptional case). The `data` field is always a decoded text string regardless
of wire format.

#### 9.5.6. Userdata (0x06)

Variable length, string format.

Reports a user-initiated event or interaction, encoded using the 6-bit packed
character set (Appendix A).

This covers any event that originates from physical user interaction with the
device rather than from automated sensor readings: button presses, switch
changes, mode selections, tamper detection, or manual triggers.

The content is a free-form string describing the event. Examples:

- `BTN A` — button A pressed
- `BTN B LONG` — button B long-press
- `MODE 2` — user selected operating mode 2
- `TAMPER` — enclosure tamper switch triggered
- `ARM` — user armed the device
- `CAL START` — user initiated calibration
- `DOOR OPEN` — door sensor triggered

No structure is imposed on the message content. The sensor firmware defines the
event vocabulary appropriate to its hardware and application.

In the rare case where an event description contains characters outside the
6-bit set, raw format (Format = 0) MAY be used with 8-bit ASCII bytes. This
should be avoided where possible.

**JSON representation:**

```json
{ "type": 6, "format": "string", "data": "BTN A" }
```

The `format` field reflects the wire encoding. The `data` field is a decoded
text string.

## 10. Canonical JSON Representation

Gateways and servers typically convert binary packets to JSON for storage,
forwarding, and human inspection. The reference implementation provides
bidirectional conversion (`iotdata_decode_to_json` and
`iotdata_encode_from_json`) with the following canonical mapping.

The JSON field names are derived from the variant's field labels, so the same
binary encoding may produce different JSON keys depending on variant. For
example, a default weather station variant produces `"wind"` as a bundled JSON
object, while a custom variant using individual wind fields produces separate
`"wind_speed"`, `"wind_direction"`, and `"wind_gust"` keys. Similarly, the
`"depth"` field type may produce `"snow_depth"`, `"soil_depth"`, or any other
label depending on the variant definition.

### Example JSON (Variant 0, weather_station)

```json
{
  "variant": 0,
  "station": 42,
  "sequence": 1234,
  "packed_bits": 120,
  "packed_bytes": 15,
  "battery": {
    "level": 84,
    "charging": false
  },
  "link": {
    "rssi": -96,
    "snr": 10.0
  }
  "environment": {
    "temperature": 21.5,
    "pressure": 1013,
    "humidity": 45
  },
  "wind": {
    "speed": 5.0,
    "direction": 180,
    "gust": 8.5
  },
  "rain": {
    "rate": 3,
    "size": 2.5,
  },
  "solar": {
    "irradiance": 850,
    "ultraviolet": 7
  },
}
```

### TLV in JSON

TLV entries are represented as an array under `"data"`. Each entry contains
`"type"`, `"format"`, and `"data"` fields.

The `"type"` field is the numeric TLV type identifier. The `"format"` field
indicates how the `"data"` field should be interpreted.

#### Globally Defined Types

TLV types that have a defined JSON representation (Section 9.5) are destructured
by the gateway into structured objects or decoded strings. The `"format"` field
reflects the structured type rather than the wire encoding:

| Type | Format      | `data` contains                                 |
| ---- | ----------- | ----------------------------------------------- |
| 0x01 | `"version"` | Structured object: firmware/hardware key-values |
| 0x02 | `"status"`  | Structured object: uptimes, restarts, reason    |
| 0x03 | `"health"`  | Structured object: temp, voltage, heap, active  |
| 0x04 | `"config"`  | Structured object: key-value pairs              |
| 0x05 | `"string"`  | Decoded diagnostic text string                  |
| 0x06 | `"string"`  | Decoded userdata text string                    |

Example with all global types:

```json
{
  "data": [
    {
      "type": 1,
      "format": "version",
      "data": {
        "FW": "142",
        "HW": "3"
      }
    },
    {
      "type": 2,
      "format": "status",
      "data": {
        "session_uptime": 86400,
        "lifetime_uptime": 1209600,
        "restarts": 12,
        "reason": "watchdog"
      }
    },
    {
      "type": 3,
      "format": "health",
      "data": {
        "cpu_temp": 34,
        "supply_mv": 3842,
        "free_heap": 42816,
        "session_active": 1050
      }
    },
    {
      "type": 4,
      "format": "config",
      "data": {
        "TX": "30",
        "SF": "7",
        "PW": "14"
      }
    },
    { "type": 5, "format": "string", "data": "LOW SIGNAL" },
    { "type": 6, "format": "string", "data": "BTN A" }
  ]
}
```

Note that a single packet would not typically contain all of these. A normal
transmission might include only sensor data fields with no TLV entries at all,
or one or two TLV entries such as Status and a Diagnostic message. Packets may
also contain repeated entries, for example, multiple Diagnostic or Userdata
TLVs.

#### Unrecognised and Proprietary Types

TLV types that do not have a defined JSON representation — including proprietary
types (0x20+), reserved types, and any type the gateway does not recognise —
fall back to a generic encoding based on the wire format bit:

| Wire format | `"format"` | `"data"` contains          |
| ----------- | ---------- | -------------------------- |
| raw (0)     | `"raw"`    | Base64-encoded byte string |
| string (1)  | `"string"` | Decoded text string        |

Examples:

```json
{
  "data": [
    { "type": 32, "format": "raw", "data": "A0b4901=" },
    { "type": 33, "format": "string", "data": "HELLO WORLD" }
  ]
}
```

This ensures that all TLV entries are representable in JSON even if the gateway
has no knowledge of the type's semantics. The raw Base64 or decoded string is
passed through for downstream consumers to interpret.

#### Format Field Summary

The `"format"` field serves as a discriminator for how to parse the `"data"`
field. The complete set of values:

| Value       | `data` type     | Source                                  |
| ----------- | --------------- | --------------------------------------- |
| `"raw"`     | string (Base64) | Fallback for unrecognised raw TLV types |
| `"string"`  | string (text)   | String-format TLVs (wire or defined)    |
| `"version"` | object          | Version TLV (0x01)                      |
| `"status"`  | object          | Status TLV (0x02)                       |
| `"health"`  | object          | Health TLV (0x03)                       |
| `"config"`  | object          | Config TLV (0x04)                       |

Note that `"string"` appears both as the defined format for Diagnostic (0x05)
and Userdata (0x06), and as the fallback for unrecognised string-format TLVs.
This is intentional — the representation is identical in both cases (a plain
text string), so no distinction is needed.

### Round-Trip Guarantee

The JSON representation MUST support lossless round-trip conversion: encoding a
packet to binary, decoding to JSON, re-encoding from JSON, and comparing the
resulting binary MUST produce an identical byte sequence. The reference
implementation test suite verifies this property.

## 11. Receiver Considerations

[moved]

### 11.1. Datetime Year Resolution

[moved]

### 11.2. Position Source Ambiguity

[moved]

### 11.3. Sensor Metadata and Interoperability

[moved]

### 11.4. Unknown Variants

A receiver encountering a variant number that it does not have a table entry for
SHOULD:

1. Fall back to variant 0's field mapping for decoding.
2. Flag the packet as using an unknown variant in its output (e.g. a warning in
   the print output or a field in the JSON).
3. NOT reject the packet, since the field encodings are universal and the data
   is likely still meaningful.

In the reference implementation, `iotdata_get_variant()` returns variant 0's
table as a fallback for any unknown variant number.

### 11.5. Quantisation Error Budgets

[moved]

#### Boundary Conditions

[moved]

## 11.6. Error Handling and Malformed Packets

The protocol is designed for environments where packet corruption is handled at
the link layer (LoRa CRC, LoRaWAN MIC, cellular integrity checks). However,
decoders may encounter malformed packets due to firmware bugs, version
mismatches, partial reception on links without CRC, or deliberate fuzzing. This
section defines the expected decoder behaviour.

### General Principle

A decoder that encounters any condition it cannot resolve MUST discard the
entire packet. Partial decoding — where some fields are extracted and others are
silently skipped or defaulted — is NOT RECOMMENDED, as it can produce internally
inconsistent records (e.g. a wind direction without a wind speed, or a position
from a different transmission cycle than the temperature).

A decoder MAY log or count discarded packets for diagnostic purposes. The
discard reason SHOULD be made available to the operator.

### Specific Conditions

The following conditions MUST result in packet discard:

1. **Packet too short.** A packet shorter than 5 bytes (32-bit header + 1
   presence byte) is not a valid iotdata packet.

2. **Unknown variant.** A variant ID that does not appear in the decoder's
   variant table. See also Section 11.4. The decoder cannot determine field
   widths or ordering without a variant definition, so no fields can be
   extracted.

3. **Truncated fields.** The presence bits indicate a field is present, but the
   remaining packet data is insufficient to contain it. This typically indicates
   corruption or a version mismatch where the receiver's field table does not
   match the transmitter's.

4. **Truncated TLV.** The TLV bit (bit 6 of Presence Byte 0) is set, but the
   remaining data after the last data field is insufficient to contain a valid
   TLV header (16 bits), or a TLV entry's length field extends past the end of
   the packet.

5. **Extension byte overflow.** A presence byte chain exceeds the decoder's
   maximum supported depth (4 bytes in the reference implementation). A decoder
   SHOULD discard rather than attempt to process an unexpectedly deep presence
   chain, as it may indicate corruption of the extension bits.

### Conditions That SHOULD NOT Cause Discard

The following conditions are anomalous but not fatal. A decoder SHOULD process
the packet and MAY flag the anomaly:

1. **Quantised value at range boundary.** A decoded value at exactly the minimum
   or maximum of its defined range is valid. It may represent a clamped
   out-of-range input (see Section 11.5), but the decoder cannot distinguish
   this from a genuine boundary reading.

2. **Out-of-range quantised value.** A raw quantised value that exceeds the
   number of defined steps (e.g. humidity = 120 in a 7-bit field with range
   0–100) indicates corruption or a version mismatch. The decoder SHOULD clamp
   the value to the defined range and MAY flag the anomaly. Discarding is also
   acceptable.

3. **Sequence number discontinuity.** A gap in the sequence number indicates
   lost packets, not a malformed packet. The receiver SHOULD track and report
   gaps but MUST NOT discard the current packet.

4. **Unknown TLV type.** A TLV entry with an unrecognised type code is not an
   error. The decoder MUST skip the entry using its length field and continue
   processing subsequent TLV entries and SHOULD preserve the entry in its
   generic form (Section 10) for downstream consumers.

5. **Trailing bytes.** If all presence-indicated fields and TLV entries have
   been decoded and bytes remain in the packet, the decoder SHOULD ignore the
   trailing data. This allows future protocol extensions to append data without
   breaking existing decoders.

### Implementation Guidance

Decoders MUST validate buffer bounds before every field read. The bit-packing
functions in the reference implementation accept a `max_bits` parameter and
return an error if a read would exceed it. Implementations that omit bounds
checking risk buffer overflows from crafted or corrupted packets.

A decoder operating on untrusted input (e.g. a gateway receiving packets from
unknown stations) SHOULD treat all packets as potentially malformed and MUST NOT
assume that a valid header implies a valid payload.

## 12. Packet Size Reference

[flag: the 'tens of bytes' size table is cited heavily by TARGETING (Sec.3, App.I)]

The following table shows exact bit and byte counts for common packet
configurations, using variant 0 (weather_station).

| Scenario                          | Fields                                  | Bits | Bytes |
| --------------------------------- | --------------------------------------- | ---- | ----- |
| Heartbeat (no data)               | header + pres0                          | 40   | 5     |
| Minimal (battery only)            | + battery                               | 46   | 6     |
| Battery + environment             | + battery, environment                  | 70   | 9     |
| Typical pres0 (bat+env+wind+rain) | + battery, environment, wind, rain_rate | 104  | 13    |
| Full pres0 (all 6 fields)         | + battery, env, wind, rain, solar, link | 124  | 16    |
| Full station (all 12 fields)      | + all 12 field types (pres0 + pres1)    | 253  | 32    |

For comparison, the equivalent data in JSON would typically be 200-600 bytes,
and in a packed C struct with byte alignment would be 40-60 bytes.

## 13. Implementation Notes

[moved]

### 13.1. Reference Implementation

[moved]

### 13.2. Encoder Strategy

[moved]

### 13.3. Compile-Time Options

[moved]

#### Test targets

[moved]

### 13.4. Build Size and Stack Usage

[moved]

#### Build summary for x86-64, aarch64 and esp32-c3 systems

[moved]

#### Build output for x86-64 (-O6 and -Os)

[moved]

#### Build output for esp32-c3 (-Os)

[moved]

#### Stack usage for x86-64 (-Os)

[moved]

### 13.5. Variant Table Extension

[moved]

## 14. Security Considerations

[moved]

## 15. Future Work

[moved]

## 16. Versioning and Forward Compatibility

### 16.1. Protocol Version

This document defines version 1 of the IoT Sensor Telemetry Protocol. The
protocol does not carry an explicit version field in the packet header. Version
identification relies on the combination of variant ID and the field table known
to the receiver.

This is a deliberate design choice. A version field would cost 2–4 bits in every
packet — significant when the minimum useful packet is 46 bits. The trade-off is
that version negotiation and graceful version coexistence are not supported at
the wire level.

### 16.2. Compatibility Model

The protocol's compatibility properties differ by component:

**Within a variant definition (fully compatible).** Adding or removing optional
fields within an existing variant does not break compatibility. A transmitter
that begins including a new field (e.g. adding air quality to a weather station
that previously omitted it) is handled transparently by the presence bit
mechanism. Receivers that understand the variant's field table will decode the
new field; the packet is self-describing within the scope of the variant.

**New variant definitions (forward compatible).** A transmitter using a new
variant ID (e.g. variant 3 for a soil sensor) produces packets that existing
receivers cannot decode, because the receiver does not have the field table for
variant 3. The receiver MUST discard such packets (Section 11.4, Section 11.6).
This is the intended behaviour — variants are deployment-specific, and receivers
are expected to be configured with the variant tables relevant to their
deployment.

**Field encoding changes (incompatible).** Any change to a field type's bit
width, quantisation formula, or semantic meaning is a breaking change. A
receiver using the old encoding will silently produce incorrect values. There is
no mechanism to detect this at the wire level. Such changes MUST be accompanied
by a new variant ID, ensuring that old receivers discard the packet rather than
misinterpret it.

**Header changes (incompatible).** Any change to the header layout (variant
field width, station ID width, sequence width, or total header length) breaks
all existing encoders and decoders. Such changes are not contemplated for
version 1 and would constitute a new protocol version, distinguishable only by
out-of-band means (e.g. separate radio channel, different LoRaWAN port, or
application-layer framing).

### 16.3. Receiver Requirements

A receiver MUST be configured — at compile time or runtime — with the set of
variant definitions it is expected to decode. A receiver MUST reject packets
with variant IDs not in its configured set (Section 11.4).

A receiver SHOULD NOT attempt heuristic detection of unknown field layouts.
Because fields are bit-packed with no delimiters or self-describing type tags,
misalignment by even one bit corrupts all subsequent fields in the packet.

### 16.4. Upgrading Deployments

[flag: operational upgrade guidance - could be TARGETING]

When a deployment upgrades field definitions or introduces new variants, the
following procedure is RECOMMENDED:

1. **Update receivers first.** Gateways and servers are updated with the new
   variant tables before any transmitter firmware is changed. This ensures that
   new-format packets are understood upon arrival.

2. **Update transmitters.** Sensors are updated via OTA or physical access. The
   transition period — where some sensors use the old variant and others use the
   new — is handled naturally, since each packet carries its variant ID and
   receivers can decode both.

3. **Retire old variants.** Once all transmitters have been updated, old variant
   definitions may be removed from receiver configurations. This is optional;
   retaining them costs only the memory for the field table.

For breaking changes (field encoding modifications), the old and new encodings
MUST use different variant IDs. This allows both to coexist during the
transition.

### 16.5. Mesh Protocol Versioning

The mesh protocol (Appendix G) uses a separate versioning strategy. Mesh control
packets are identified by variant ID 15 and dispatched by the ctrl_type field.
Reserved ctrl_type values (0x7–0xF) MUST be silently discarded by nodes that do
not recognise them, allowing incremental deployment of new mesh packet types.
See Appendix G, Section J.7 for details.

### 16.6. Future Version Considerations

[moved]

## Appendix A. 6-Bit Character Table

The packed string format (TLV Format = 1) encodes each character as 6 bits using
the following table:

| Value | Char  | Value | Char | Value | Char | Value | Char   |
| ----- | ----- | ----- | ---- | ----- | ---- | ----- | ------ |
| 0     | space | 16    | p    | 32    | 5    | 48    | L      |
| 1     | a     | 17    | q    | 33    | 6    | 49    | M      |
| 2     | b     | 18    | r    | 34    | 7    | 50    | N      |
| 3     | c     | 19    | s    | 35    | 8    | 51    | O      |
| 4     | d     | 20    | t    | 36    | 9    | 52    | P      |
| 5     | e     | 21    | u    | 37    | A    | 53    | Q      |
| 6     | f     | 22    | v    | 38    | B    | 54    | R      |
| 7     | g     | 23    | w    | 39    | C    | 55    | S      |
| 8     | h     | 24    | x    | 40    | D    | 56    | T      |
| 9     | i     | 25    | y    | 41    | E    | 57    | U      |
| 10    | j     | 26    | z    | 42    | F    | 58    | V      |
| 11    | k     | 27    | 0    | 43    | G    | 59    | W      |
| 12    | l     | 28    | 1    | 44    | H    | 60    | X      |
| 13    | m     | 29    | 2    | 45    | I    | 61    | Y      |
| 14    | n     | 30    | 3    | 46    | J    | 62    | Z      |
| 15    | o     | 31    | 4    | 47    | K    | 63    | (rsvd) |

Value 63 is reserved for a future escape mechanism to extend the character set.

The corresponding encode/decode functions in the reference implementation:

```c
static inline int char_to_6bit(char c) {
    if (c == ' ')              return 0;
    if (c >= 'a' && c <= 'z')  return 1 + (c - 'a');
    if (c >= '0' && c <= '9')  return 27 + (c - '0');
    if (c >= 'A' && c <= 'Z')  return 37 + (c - 'A');
    return -1;  /* unencodable */
}

static inline char sixbit_to_char(uint8_t val) {
    if (val == 0)              return ' ';
    if (val >= 1  && val <= 26) return 'a' + (val - 1);
    if (val >= 27 && val <= 36) return '0' + (val - 27);
    if (val >= 37 && val <= 62) return 'A' + (val - 37);
    return '?';
}
```

## Appendix B. Quantisation Worked Examples

### B.1. Battery Level

Input: 75%

```text
q = round(75 / 100.0 * 31.0) = round(23.25) = 23
Decoded: round(23 / 31.0 * 100.0) = round(74.19) = 74%
Error: 1 percentage point
```

### B.2. Temperature

Input: -15.25°C

```text
q = round((-15.25 - (-40.0)) / 0.25) = round(24.75 / 0.25) = round(99.0) = 99
Decoded: -40.0 + 99 * 0.25 = -40.0 + 24.75 = -15.25°C
Error: 0.00°C (exact)
```

### B.3. Position (59.334591°N, 18.063240°E)

Latitude:

```text
q = round((59.334591 - (-90.0)) / 180.0 * 16777215)
  = round(149.334591 / 180.0 * 16777215)
  = round(0.829636617 * 16777215)
  = round(13918991.6) = 13918992

Decoded: 13918992 / 16777215.0 * 180.0 + (-90.0)
       = 0.829636653 * 180.0 - 90.0
       = 149.334597 - 90.0 = 59.334597°

Error: 0.000006° ≈ 0.67 m
```

Longitude:

```text
q = round((18.063240 - (-180.0)) / 360.0 * 16777215)
  = round(198.063240 / 360.0 * 16777215)
  = round(0.550175667 * 16777215)
  = round(9230415.2) = 9230415

Decoded: 9230415 / 16777215.0 * 360.0 + (-180.0)
       = 0.550175631 * 360.0 - 180.0
       = 198.063227 - 180.0 = 18.063227°

Error: 0.000013° ≈ 0.72 m (at 59°N, cos correction)
```

### B.4. Datetime

Input: Day 5, 12:00:00 (432,000 + 43,200 = 475,200 seconds from year start)

```text
ticks = 475200 / 5 = 95040
Decoded: 95040 * 5 = 475200 seconds
Error: 0 seconds (exact, since input is a multiple of 5)
```

Input: Day 5, 12:00:03 (475,203 seconds — not a multiple of 5)

```text
ticks = 475203 / 5 = 95040 (integer division, truncated)
Decoded: 95040 * 5 = 475200 seconds
Error: 3 seconds (truncation towards zero)
```

Note: the encoder uses integer division (truncation), not rounding, for the
datetime field. This means the decoded time is always ≤ the actual time, with a
maximum error of 4 seconds.

## Appendix C. Complete Encoder Example

The following example from the reference implementation test suite demonstrates
encoding a full weather station telemetry packet:

```c
#define IOTDATA_VARIANT_MAPS_DEFAULT
#include "iotdata.h"

/* Encode a full weather station packet (variant 0) */
void encode_full_packet(uint8_t *buf, size_t buf_size, size_t *out_len)
{
    iotdata_encoder_t enc;

    iotdata_encode_begin(&enc, buf, buf_size, 0, 42, 50000);

    /* Pres0 fields — most common, smallest packet when only these */
    iotdata_encode_battery(&enc, 95, true);
    iotdata_encode_link(&enc, -76, 10.0f);
    iotdata_encode_environment(&enc, -2.75f, 1005, 95);
    iotdata_encode_wind(&enc, 12.0f, 270, 18.5f);
    iotdata_encode_rain(&enc, 3, 15); // x10 units
    iotdata_encode_solar(&enc, 450, 7);

    /* Pres1 fields — trigger extension byte */
    iotdata_encode_cloud(&enc, 6);
    iotdata_encode_air_quality_index(&enc, 75);
    iotdata_encode_radiation(&enc, 100, 0.50f);
    iotdata_encode_position(&enc, 59.334591, 18.063240);
    iotdata_encode_datetime(&enc, 3251120);
    iotdata_encode_flags(&enc, 0x42);

    iotdata_encode_end(&enc, out_len);
    /* Result: 32 bytes for all 12 fields */
}
```

Decoding on the receiver side:

```c
/* Decode and inspect */
iotdata_decoder_t dec;
iotdata_decode(buf, len, &dec);

printf("Station %u: %.2f°C, %u hPa, wind %.1f m/s @ %u°\n",
       dec.station, dec.temperature, dec.pressure,
       dec.wind_speed, dec.wind_direction);

/* Or decode to JSON for forwarding */
char *json;
iotdata_decode_to_json(buf, len, &json);
/* ...forward json to MQTT, database, etc... */
free(json);
```

## Appendix D. Transmission Medium Considerations

[moved]

### D.1. Design Principle: One Frame, One Transmission

[moved]

### D.2. LoRa (Raw PHY)

[moved]

### D.3. LoRaWAN

[moved]

### D.4. Sigfox

[moved]

### D.5. IEEE 802.11ah (Wi-Fi HaLow)

[moved]

### D.6. Cellular (NB-IoT, LTE-M, SMS)

[moved]

### D.7. Medium Selection Summary

[moved]

## Appendix E. System Implementation Considerations

[moved]

### E.1. Microcontroller Class Taxonomy

[moved]

### E.2. Memory Footprint

[moved]

### E.3. Encoder Architecture: Store-Then-Pack

[moved]

### E.4. Encoder Alternative: Pack-As-You-Go

[moved]

### E.5. Compile-Time Field Stripping (#ifdef)

[moved]

### E.6. Floating Point Considerations

[moved]

### E.7. Dependencies and Portability

[moved]

### E.8. Stack vs Heap Allocation

[moved]

### E.9. Endianness

[moved]

### E.10. Real-Time Considerations

[moved]

### E.11. Platform-Specific Notes

[moved]

### E.12. Class 1 Hand-Rolled Encoder Example

[moved]

## Appendix F. Example Weather Station Output

The `test-example` target generates pseudo sensor data simulating a weather
station to illustrate quantisation effects and ancillary (dump, print and JSON)
functionality.

```text
╔══════════════════════════════════════════════════╗
║  iotdata weather station simulator               ║
║  Station 42 — variant 0 (weather_station)        ║
║  30s reports / 5min full reports with position   ║
║  Press Ctrl-C to stop                            ║
╚══════════════════════════════════════════════════╝

────────────────────────────────────────────────────────────────────────────────
** Packet #1  [17:29:08]  *** 5-minute report (with position/datetime) ***
────────────────────────────────────────────────────────────────────────────────

** Sensor values:

    battery:      85.2%
    link:          -85 dBm   SNR 4.8 dB
    temperature: +14.75 °C
    pressure:     1013 hPa
    humidity:       55 %
    wind:          4.1 m/s @ 172°  (gust 8.7 m/s)
    rain:            3 mm/hr, 0.5 mm/d
    solar:         393 W/m²  UV 3
    clouds:          4 okta
    air quality:    41 AQI
    radation:       22 CPM,     0.10 µSv/h
    position:    59.334588, 18.063240
    datetime:    3518948 s from year start
    flags:       0x01

** Binary (32 bytes):

    00 2A 00 01 BF 7E D2 26 DD 1B 71 0F 44 40 C5 89
    34 14 80 2C 00 56 A3 18 84 66 C2 78 55 E9 68 08

** Diagnostic dump:

      Offset     Len  Field                            Raw  Decoded                       Range
      ------     ---  -----                            ---  -------                       -----
           0       4  variant                            0  0                             0-14 (15=rsvd)
           4      12  station                           42  42                            0-4095
          16      16  sequence                           1  1                             0-65535
          32       8  presence[0]                      191  0xbf                          ext|tlv|6 fields
          40       8  presence[1]                      126  0x7e                          ext|7 fields
          48       5  battery_level                     26  84%                           0..100%%, 5b quant
          53       1  battery_charging                   0  discharging                   0/1
          54       4  link_rssi                          8  -88 dBm                       -120..-60, 4dBm
          58       2  link_snr                           2  0 dB                          -20..+10, 10dB
          60       9  temperature                      219  14.75 C                       -40..+80C, 0.25C
          69       8  pressure                         163  1013 hPa                      850..1105 hPa
          77       7  humidity                          55  55%                           0..100%%
          84       7  wind_speed                         8  4.0 m/s                       0..63.5, 0.5m/s
          91       8  wind_direction                   122  172 deg                       0..355, ~1.4deg
          99       7  wind_gust                         17  8.5 m/s                       0..63.5, 0.5m/s
         106       8  rain_rate                          3  3 mm/hr                       0..255 mm/hr
         114       4  rain_size                          1  0.4 mm/d                      0..6.3 mm/d
         118      10  solar_irradiance                 393  393 W/m2                      0..1023 W/m2
         128       4  solar_ultraviolet                  3  3                             0..15
         132       4  clouds                             4  4 okta                        0..8 okta
         136       9  air_quality                       41  41 AQI                        0..500 AQI
         145      14  radiation_cpm                     22  22 CPM                        0..65535 CPM
         159      14  radiation_dose                    10  0.10 uSv/h                    0..163.83, 0.01
         173      24  latitude                    13918992  59.334592                     -90..+90
         197      24  longitude                    9230415  18.063230                     -180..+180
         221      24  datetime                      703789  day 40 17:29:05 (3518945s)    5s res
         245       8  flags                              1  0x01                          8-bit bitmask

Total: 253 bits (32 bytes)

** Decoded:

Station 42 seq=1 var=0 (weather_station) [253 bits, 32 bytes]
  battery:             84% (discharging)
  link:                -88 dBm RSSI, 0 dB SNR
  environment:         14.75 C, 1013 hPa, 55%
  wind:                4.0 m/s, 172 deg, gust 8.5 m/s
  rain:                3 mm/hr, 0.4 mm/d
  solar:               393 W/m2, UV 3
  clouds:              4 okta
  air_quality:         41 AQI
  radiation:           22 CPM, 0.10 uSv/h
  position:            59.334592, 18.063230
  datetime:            day 40 17:29:05 (3518945s)
  flags:               0x01

** JSON:

{"variant":0,"station":42,"sequence":1,"packed_bits":253,"packed_bytes":32,"battery":{"level":84,"charging":false},"link":{"rssi":-88,"snr":0},"environment":{"temperature":14.75,"pressure":1013,"humidity":55},"wind":{"speed":4,"direction":172,"gust":8.5},"rain":{"rate":3,"size":4},"solar":{"irradiance":393,"ultraviolet":3},"clouds":4,"air_quality":41,"radiation":{"cpm":22,"dose":0.099999994039535522},"position":{"latitude":59.334592183506032,"longitude":18.06323039908591},"datetime":3518945,"flags":1}

────────────────────────────────────────────────────────────────────────────────
** Packet #2  [17:29:38]  30-second report
────────────────────────────────────────────────────────────────────────────────

** Sensor values:

    battery:      84.9%
    link:          -85 dBm   SNR 5.5 dB
    temperature: +14.48 °C
    pressure:     1013 hPa
    humidity:       55 %
    wind:          3.6 m/s @ 171°  (gust 7.2 m/s)
    rain:            5 mm/hr, 0.0 mm/d
    solar:         390 W/m²  UV 3

** Binary (16 bytes):

    00 2A 00 02 3F D2 36 D5 1B 70 EF 43 81 41 86 30

** Diagnostic dump:

      Offset     Len  Field                            Raw  Decoded                       Range
      ------     ---  -----                            ---  -------                       -----
           0       4  variant                            0  0                             0-14 (15=rsvd)
           4      12  station                           42  42                            0-4095
          16      16  sequence                           2  2                             0-65535
          32       8  presence[0]                       63  0x3f                          ext|tlv|6 fields
          40       5  battery_level                     26  84%                           0..100%%, 5b quant
          45       1  battery_charging                   0  discharging                   0/1
          46       4  link_rssi                          8  -88 dBm                       -120..-60, 4dBm
          50       2  link_snr                           3  10 dB                         -20..+10, 10dB
          52       9  temperature                      218  14.50 C                       -40..+80C, 0.25C
          61       8  pressure                         163  1013 hPa                      850..1105 hPa
          69       7  humidity                          55  55%                           0..100%%
          76       7  wind_speed                         7  3.5 m/s                       0..63.5, 0.5m/s
          83       8  wind_direction                   122  172 deg                       0..355, ~1.4deg
          91       7  wind_gust                         14  7.0 m/s                       0..63.5, 0.5m/s
          98       8  rain_rate                          5  5 mm/hr                       0..255 mm/hr
         106       4  rain_size                          0  0.0 mm/d                      0..6.3 mm/d
         110      10  solar_irradiance                 390  390 W/m2                      0..1023 W/m2
         120       4  solar_ultraviolet                  3  3                             0..15

Total: 124 bits (16 bytes)

** Decoded:

Station 42 seq=2 var=0 (weather_station) [124 bits, 16 bytes]
  battery:             84% (discharging)
  link:                -88 dBm RSSI, 10 dB SNR
  environment:         14.50 C, 1013 hPa, 55%
  wind:                3.5 m/s, 172 deg, gust 7.0 m/s
  rain:                5 mm/hr, 0.0 mm/d
  solar:               390 W/m2, UV 3

** JSON:
{"variant":0,"station":42,"sequence":2,"packed_bits":124,"packed_bytes":16,"battery":{"level":84,"charging":false},"link":{"rssi":-88,"snr":10},"environment":{"temperature":14.5,"pressure":1013,"humidity":55},"wind":{"speed":3.5,"direction":172,"gust":7},"rain":{"rate":5,"size":0},"solar":{"irradiance":390,"ultraviolet":3}}
```

## Appendix G. Mesh Protocol

[moved]

### Overview

[moved]

### G.1. Use Cases and System Roles

[moved]

#### G.1.1. The Problem

[moved]

#### G.1.2. The Solution

[moved]

#### G.1.3. System Roles

[moved]

#### G.1.4. Role Capabilities

[moved]

### G.2. Design Principles

[moved]

#### G.2.1. Seamless Operation

[moved]

#### G.2.2. Protocol Integration

[moved]

#### G.2.3. Opaque Forwarding

[moved]

#### G.2.4. Multiple Gateway Support

[moved]

#### G.2.5. Gradient-Based Routing

[moved]

### G.3. Protocol Flows

#### G.3.1. Topology Discovery

Topology is built through periodic beacon propagation from gateways outward.

```text
Gateway (cost=0)
    │
    │  BEACON (gateway_id=G, generation=N, cost=0)
    │
    ▼
Relay A hears beacon, adopts Gateway as parent, sets cost=1
    │
    │  BEACON (gateway_id=G, generation=N, cost=1)   [after random 1–5s jitter]
    │
    ▼
Relay B hears Relay A's rebroadcast, adopts Relay A as parent, sets cost=2
    │
    │  BEACON (gateway_id=G, generation=N, cost=2)   [after random 1–5s jitter]
    │
    ▼
...continues outward until no new nodes hear the beacon
```

Gateways transmit beacons at a regular interval (of which there is no
recommended default, as this should be a function of the periodicity and density
of sensor network, but 60 seconds is a reasonable figure). Each beacon carries a
generation counter that increments per round. Relays compare incoming beacons
against their current state:

- Newer generation (modular comparison within half the 12-bit range): update
  parent if cost is equal or better.
- Same generation, lower cost: adopt the new sender as parent.
- Same generation, equal or higher cost: suppress — do not rebroadcast.

The random rebroadcast jitter (1–5 seconds) prevents synchronised retransmission
from nodes that hear the same beacon simultaneously, reducing collisions in
dense areas.

#### G.3.2. Sensor Data Forwarding

Sensor data flows inward from sensors toward gateways, relayed transparently by
relays.

```text
Sensor S transmits raw iotdata packet (variant=V, station=S, seq=N)
    │
    │  [raw packet, no mesh awareness]
    │
    ├──────────────────────┐
    ▼                      ▼
Gateway (hears directly)   Relay A (hears sensor)
    │                      │
    │ process normally     │ wrap in FORWARD, send to parent
    │                      │
    │                      ▼
    │                  Gateway (receives FORWARD)
    │                      │
    │                      │ unwrap inner packet
    │                      │ dedup: {S, N} already seen? → discard
    │                      │ otherwise process normally
    ▼                      ▼
    [sensor data processed once]
```

When a relay hears a raw sensor packet (any variant 0–14), it waits a short
random backoff (200–1000ms). If during that backoff it hears another relay
forward the same packet (identified by matching origin station and sequence), it
suppresses its own forward. This Trickle-style suppression reduces redundant
airtime in areas where multiple relays overlap.

If no suppression occurs, the relay wraps the raw sensor packet in a FORWARD
control message (variant 15, ctrl_type 0x1) addressed to its parent and
transmits. The parent, if another relay, repeats the process — unwrap, dedup,
re-wrap with its own header, forward to its parent — until the packet reaches a
gateway.

#### G.3.3. Relay-by-Relay Acknowledgement

Each FORWARD is acknowledged by the receiving parent to confirm delivery.

```text
Relay A                          Relay B (A's parent)
  │                             │
  │──── FORWARD (seq=X) ───────>│
  │                             │
  │<──── ACK (fwd_station=A, ──>│
  │           fwd_seq=X)        |
  │                             |
  [clear retry timer]           [forward inner packet upstream]
```

If no ACK is received within a timeout (recommended 500ms for high frequency
sensor networks, up to 15-30 seconds for low frequency networks), the sender
retries up to a configurable number of attempts (recommended: 3). After
exhausting retries, the sender marks its parent as unreliable, promotes its
backup parent (if available), and retransmits the FORWARD to the new parent. If
no backup parent is available, the node broadcasts a ROUTE_ERROR and enters an
orphaned state, listening for beacons to reattach to the tree.

#### G.3.4. Fast Failover

When a relay loses all upstream paths, it broadcasts a ROUTE_ERROR so downstream
nodes can immediately reroute rather than waiting for beacon timeout.

```text
Relay B (was Relay C's parent)    Relay C (child of B)
  │                            │
  [B loses its parent]         │
  │                            │
  │──── ROUTE_ERROR ──────────>│
  │     (reason=parent_lost)   │
  │                            │
                               [C immediately seeks alternative parent from neighbour table]
```

This converts a multi-minute outage (waiting for 3 missed beacon rounds × 60s =
180s) into sub-second failover in the best case.

#### G.3.5. Network Monitoring

Relays periodically send NEIGHBOUR_REPORT messages upstream to the gateway,
providing a snapshot of their local topology view. These reports are forwarded
like any other data (wrapped in FORWARD by upstream relays). The gateway
aggregates reports from all relays to build a complete network topology graph,
enabling operators to visualise the mesh, identify weak links, and plan node
placement.

THE WRAPPING IS NOT OPTIONAL, and is what makes the graph complete. A report sent
bare reaches only a gateway already within direct radio range — which is precisely
the case where the gateway could have observed that relay's beacons for itself. The
reports that carry information nothing else can supply are the ones from relays two
or more hops out, and those arrive only if intermediate relays carry them. Wrapped
in a FORWARD the report inherits duplicate suppression, the cost gradient, hop
acknowledgement and retry, and relays carry each other's reports without needing to
recognise the type at all.

The report is a STANDING RECORD of a slowly-changing thing, not telemetry: what
changes a neighbour table is a node failing, a season's foliage, or weather. Its
cadence should be measured in minutes rather than seconds — and at depth its cost
is paid more than once, since every relay on the path carries it.

#### G.3.6. Reachability Testing (not implemented)

An operator may want to ask whether a node is reachable *now*, and how long a
round trip takes, without waiting for its next scheduled transmission. That is a
cheaper question than a trace and can be answered when a trace cannot: a trace
record grows with every hop and may arrive truncated, whereas a reachability
probe is a small fixed frame that either returns or does not.

This was once reserved as a pair of control types, PING and PONG, with the
gateway's probe **routed downstream** toward a named target. That is why it was
never built: downstream routing does not exist in this mesh, which floods
outward and follows a cost gradient inward. The feature needed a routing project
rather than a message format.

It belongs under DIAG as a kind, beside SURVEY and TRACE. The request then rides
the broadcast flood outward and the answer rides a FORWARD inward, exactly as
those two already do, and nothing new is needed to carry it. No control type or
kind id is reserved for it in advance -- see G.8.

#### G.3.7. Reception Survey

The mesh can be asked which of its nodes hear a given station, and how well. An
operator issues a DIAG SURVEY request naming one or more stations and a lifetime;
every node that takes the directive watches for DIRECT receptions of those
stations, and reports what it heard in periodic batches.

This answers a question the rest of the protocol cannot. FORWARD tells the gateway
that a packet arrived and which relay delivered it; it says nothing about the
relays that also heard it, the margin any of them had, or whether a station sits
one weak link away from having no coverage at all. Walking a site with a mobile
station under an active survey produces a reception map; leaving a survey running
against a deployed sensor shows its margin changing over a season.

Only the FIRST hop is measured. A node records a reception only when it hears the
station directly — a bare packet, not one arriving inside a FORWARD. An RSSI taken
from a relayed copy describes the relay-to-relay link, which NEIGHBOUR_REPORT
already covers, and would otherwise be indistinguishable from the measurement
being asked for.

#### G.3.8. Path Tracing

A FORWARD names only the relay that sent it. Each hop unwraps, re-wraps under its
own station and discards the previous one, so by the time a frame reaches the
gateway the route it took has been thrown away. A TRACE records it: every hop
appends what it is, what gradient cost it advertises, and what it heard the hop
before it at. What arrives is the path, with the margin on each link.

That answers a question neither of the other diagnostics can. NEIGHBOUR_REPORT
says which relays can hear each other; SURVEY says which relays hear a given
station directly. Neither says which of those links a frame actually used, and on
a mesh with alternatives those are different facts. For a roaming instrument it
is the question that matters — not "who can hear me" but "how does what I send
get home, and how much margin does it have on the way".

**Two messages, not one.** A REQUEST asks a named station to start recording and
carries no hops of its own; a RECORD is the frame that accumulates them. The op
distinguishing RECORD from RESPONSE is load-bearing rather than cosmetic: a
RECORD is still collecting and relays append to it, a RESPONSE is a finished
record being carried home and must be left alone. Without the distinction a
relay cannot tell them apart, and would corrupt the measurement it was asked to
deliver.

**One leg per frame.** An upward trace records the path inward to a gateway; a
downward trace records the path outward to a station. Carrying both legs in one
frame does not fit, so the direction is stated and only that leg is recorded. A
downward record is flooded rather than forwarded, because FORWARD only moves
toward lower cost — which also means the path it records is a branch of the
flood, not a route, and should be read as such.

**Truncation is stated, not silent.** A hop with no room for another record sets
a flag and passes on what it has. Room is checked *before* appending: appending
first and then finding the frame too large would drop the whole trace, and the
frame documenting the longest path is the one most worth keeping.

**ALL or ONE selects the transport.** Under ALL the first hop re-originates the
record under its own station and sequence, so that two relays which both heard
the subject become two frames that duplicate suppression treats as distinct, and
the fan of routes survives. Under ONE the first hop simply forwards, the
siblings collide on one `{origin, sequence}`, and exactly one path arrives — the
route real traffic would have taken. Both are useful, and they answer different
questions.

### G.4. Packet Structures

#### G.4.1. Standard iotdata Header (all packets, all variants)

| Byte | Bits        | Field                                                               |
| ---- | ----------- | ------------------------------------------------------------------- |
| 0    | [7:4]       | variant_id (4 bits: 0–14 = sensor data, 15 = mesh control)          |
| 0–1  | [3:0]+[7:0] | station_id (12 bits: 0–4095)                                        |
| 2–3  | [15:0]      | sequence (16 bits, big-endian)                                      |
| 4    | [7:0]       | presence bitmap (variants 0–14) \| ctrl_type + payload (variant 15) |

The variant and station_id are packed into a 4+12 bit structure:

```c
byte[0] = (variant << 4) | (station_id >> 8)
byte[1] = station_id & 0xFF
```

This packing primitive recurs throughout the mesh protocol wherever a 4-bit
field is paired with a 12-bit station_id or generation counter.

#### G.4.2. Variant 15 Common Header

All mesh control packets share this structure:

| Byte | Bits  | Field                     | Notes                                                                       |
| ---- | ----- | ------------------------- | --------------------------------------------------------------------------- |
| 0–1  | 4+12  | `0xF` \| `sender_station` | The mesh node transmitting this packet                                      |
| 2–3  | 16    | `sender_seq`              | Mesh sequence counter (separate from any sensor data sequence if dual-role) |
| 4    | [7:4] | `ctrl_type`               | Mesh packet type (0x0–0xF)                                                  |
| 4    | [3:0] | type-specific             | Upper nibble of first payload field                                         |

The remaining 4 bits of Byte 4 and the whole bytes of Byte 5 onward are
control-type-specific. Fields pack as a bitstream from byte 4, MSB-first, with
no padding except where explicitly noted.

#### G.4.3. BEACON (ctrl_type 0x0)

Originated by gateways, rebroadcast by relays. Flows outward from gateway.

| Byte | Bits | Field                         | Range   | Notes                                          |
| ---- | ---- | ----------------------------- | ------- | ---------------------------------------------- |
| 0–1  | 4+12 | `0xF` \| `sender_station`     | 0–4095  | Who (re)broadcast this copy                    |
| 2–3  | 16   | `sender_seq`                  | 0–65535 |                                                |
| 4–5  | 4+12 | `ctrl=0x0` \| `gateway_id`    | 0–4095  | Originating gateway                            |
| 6    | 8    | `cost`                        | 0–255   | 0 at gateway, +1 per relay                     |
| 7    | 4+4  | `flags` \| `generation[11:8]` |         | flags: b0 = accepting forwards, b1–b3 reserved |
| 8    | 8    | `generation[7:0]`             | 0–4095  | Beacon round counter                           |
| 9–10 | 16   | `interval_s`                  | 0–65535 | Originating gateway's beacon interval, seconds |

**Total: 11 bytes.**

Byte packing detail:

```c
buf[4]  = (0x0 << 4) | (gateway_id >> 8)
buf[5]  = gateway_id & 0xFF
buf[6]  = cost
buf[7]  = (flags << 4) | ((generation >> 8) & 0x0F)
buf[8]  = generation & 0xFF
buf[9]  = interval_s >> 8
buf[10] = interval_s & 0xFF
```

`interval_s` is the cadence of the gateway that ORIGINATED the beacon, carried
unchanged by every relay that rebroadcasts it. A relay derives its parent-loss
timeout from this value rather than from a locally configured interval, so a
gateway that beacons slowly does not cause relays compiled with a faster default
to declare it lost. It is mandatory: a relay cannot safely assume a default for a
field whose absence it cannot distinguish from a short frame.

Generation uses wraparound comparison: beacon A is newer than B if
`(A - B) mod 4096` is in the range 1–2047. At a 60-second beacon interval,
generation wraps every ~68 hours.

#### G.4.4. FORWARD (ctrl_type 0x1)

Wraps a raw sensor packet for relay toward the gateway.

| Byte | Bits | Field                     | Range   | Notes                                          |
| ---- | ---- | ------------------------- | ------- | ---------------------------------------------- |
| 0–1  | 4+12 | `0xF` \| `sender_station` | 0–4095  | This relay                                     |
| 2–3  | 16   | `sender_seq`              | 0–65535 |                                                |
| 4    | 4+4  | `ctrl=0x1` \| `ttl[7:4]`  |         |                                                |
| 5    | 4+4  | `ttl[3:0]` \| `0`         | 0–255   | 4-bit pad aligns inner packet to byte boundary |
| 6+   | 8×N  | `inner_packet`            |         | Raw iotdata bytes, opaque                      |

**Total: 6 + N bytes.**

Byte packing detail:

```c
buf[4] = (0x1 << 4) | (ttl >> 4)
buf[5] = (ttl & 0x0F) << 4           /* lower nibble is zero pad */
memcpy(&buf[6], inner_packet, N)     /* byte-aligned, no shifting */
```

The 4-bit pad at byte 5 lower nibble ensures the inner packet starts at a byte
boundary (offset 6). This is a deliberate trade-off: the pad may cause up to 11
bits of wasted space in the worst case (as the inner packet may already have up
to 7 bits wasted in the final byte alignment), but avoids requiring every relay
to bit-shift the entire opaque payload. For relay hot-path performance (just a
memcpy), this is the right choice. The pad nibble is reserved for future use
(e.g. priority, retry count).

Inner packet length is derived from the radio layer: `N = rx_packet_len - 6`.

For duplicate suppression, the relay reads bytes 6–9 of the radio frame (the
inner packet's iotdata header) to extract the originating sensor's station_id
and sequence:

```c
origin_station = ((buf[6] & 0x0F) << 8) | buf[7]
origin_sequence = (buf[8] << 8) | buf[9]
```

No FORWARD nesting occurs. Each relay creates a fresh FORWARD with its own
sender_station and sender_seq. The inner_packet bytes are always the original
sensor transmission, regardless of how many relays have occurred.

#### G.4.5. ACK (ctrl_type 0x2)

Relay-by-relay acknowledgement of a received FORWARD.

| Byte | Bits | Field                       | Range   | Notes                               |
| ---- | ---- | --------------------------- | ------- | ----------------------------------- |
| 0–1  | 4+12 | `0xF` \| `sender_station`   | 0–4095  | Parent sending the ACK              |
| 2–3  | 16   | `sender_seq`                | 0–65535 |                                     |
| 4–5  | 4+12 | `ctrl=0x2` \| `fwd_station` | 0–4095  | Child whose FORWARD is being ACKed  |
| 6–7  | 16   | `fwd_seq`                   | 0–65535 | Child's sender_seq from the FORWARD |

**Total: 8 bytes.**

#### G.4.6. ROUTE_ERROR (ctrl_type 0x3)

Broadcast by a relay that has lost all upstream paths.

| Byte | Bits | Field                     | Range   | Notes                                   |
| ---- | ---- | ------------------------- | ------- | --------------------------------------- |
| 0–1  | 4+12 | `0xF` \| `sender_station` | 0–4095  | Orphaned node                           |
| 2–3  | 16   | `sender_seq`              | 0–65535 |                                         |
| 4    | 4+4  | `ctrl=0x3` \| `reason`    | 0–15    | 0=parent_lost, 1=overloaded, 2=shutdown |

**Total: 5 bytes.** The minimum possible mesh packet — just the common header
with a reason code.

Reason codes:

| Value   | Meaning                                       |
| ------- | --------------------------------------------- |
| 0x0     | parent_lost — all upstream links failed       |
| 0x1     | overloaded — too many children, shedding load |
| 0x2     | shutdown — graceful node shutdown             |
| 0x3–0xF | reserved                                      |

#### G.4.7. NEIGHBOUR_REPORT (ctrl_type 0x4)

Periodic topology snapshot sent upstream to the gateway.

**Header:**

| Byte | Bits | Field                                   | Range   | Notes                                   |
| ---- | ---- | --------------------------------------- | ------- | --------------------------------------- |
| 0–1  | 4+12 | `0xF` \| `sender_station`               | 0–4095  | Reporting node                          |
| 2–3  | 16   | `sender_seq`                            | 0–65535 |                                         |
| 4–5  | 4+12 | `ctrl=0x4` \| `parent_id`               | 0–4095  | Current parent (0xFFF if orphaned)      |
| 6    | 8    | `my_cost`                               | 0–255   | Reporting node's cost                   |
| 7    | 6+2  | `num_neighbours` \| `gateway_id[11:10]` | 0–63    | Number of neighbour entries that follow |
| 8    | 8    | `gateway_id[9:2]`                       | 0–4095  | Current active gateway tree             |
| 9    | 2    | `gateway_id[1:0]`                       |         |                                         |

**Neighbour entry (4 bytes each):**

| Offset | Bits | Field        | Range | Notes                                        |
| ------ | ---- | ------------ | ----- | -------------------------------------------- |
| +0     | 8    | `cost`       | 0–255 | Neighbour's advertised cost                  |
| +1     | 8    | `rssi`       | 0–255 | dBm + 160; 0x00 = no reading, 0xFF = saturated |
| +2–3   | 16   | `station_id` | 0–4095 | The heard station                            |

**Total: 9.2 bytes + 4N bytes.**

THE RSSI IS FULL RESOLUTION, 1 dBm steps, and uses the SAME encoding as a DIAG
SURVEY observation (Section G.4.11) so that a measured RSSI means one thing in this
protocol rather than two — including its three states: a reading, a saturated
reading, and none.

Earlier revisions quantised it to 4 bits, in 5 dBm steps from a floor of −120 dBm.
That was adequate for what the field was built for — CHOOSING a route, where nothing
sensible routes over a link weaker than that. It is the wrong instrument for
MEASURING one. A LoRa receiver works down to roughly −137 dBm, so the quantised form
was blind across the last 17 dB of usable range: a link sitting at −128 and slowly
degrading read as a constant −120 until the moment it failed, which is precisely
what a topology record exists to see coming.

Example sizes:

| Neighbours | Total bytes |
| ---------- | ----------- |
| 4          | 26          |
| 8          | 42          |
| 16         | 74          |
| 32         | 138         |
| 63         | 262         |

A sender MUST clamp the entry count to what the frame holds rather than refusing to
send: a relay in a dense spot has more neighbours than a frame can carry, and a
report listing most of them is worth more than no report. At 4 bytes per entry a
255-byte frame holds 61 neighbours, and the 222 bytes available at SF7/125kHz hold
53 — both comfortably beyond what a real deployment produces.

#### G.4.8. DIAG (ctrl_type 0x5)

A generic envelope for the protocol's own diagnostics. The `kind` names the
subject; the `op` says what this frame does about it. Both directions use the same
ctrl_type.

**Envelope:**

| Byte | Bits | Field                     | Range   | Notes                             |
| ---- | ---- | ------------------------- | ------- | --------------------------------- |
| 0–1  | 4+12 | `0xF` \| `sender_station` | 0–4095  | Originator of this frame          |
| 2–3  | 16   | `sender_seq`              | 0–65535 |                                   |
| 4    | 4+4  | `ctrl=0x5` \| `kind`      | 0–15    | Which diagnostic                  |
| 5–6  | 4+12 | `op` \| `directive_id`    | 0–4095  | What this frame does about it     |

**Total: 7 bytes + op-specific body.**

An op rather than a kind per direction: pairing request and response as separate
kinds would halve the kind space and leave it ragged the first time a diagnostic
has no response. CANCEL_ALL, which carries no subject-specific body at all, could
not be expressed by a REQ/RSP kind split.

**Ops (upper nibble of byte 5):**

| op      | Name       | Direction | Meaning                                       |
| ------- | ---------- | --------- | --------------------------------------------- |
| 0x0     | REQUEST    | outward   | Install a directive                           |
| 0x1     | RESPONSE   | inward    | Results for a directive                       |
| 0x2     | ERROR      | inward    | The directive could not be taken              |
| 0x3     | CANCEL     | outward   | Retire one directive                          |
| 0x4     | CANCEL_ALL | outward   | Retire every directive of that kind           |
| 0x5     | RECORD     | either    | In flight and still collecting (TRACE)        |
| 0x6–0xF | reserved   | —         | —                                             |

**Kinds (lower nibble of byte 4):**

| kind    | Name     | Meaning                              |
| ------- | -------- | ------------------------------------ |
| 0x0     | SURVEY   | Who hears a station, and how well    |
| 0x1     | TRACE    | The path a frame took, hop by hop    |
| 0x2–0xF | reserved | —                                    |

**ERROR body (op 0x2):**

| Byte | Bits | Field    | Range | Notes             |
| ---- | ---- | -------- | ----- | ----------------- |
| 7    | 8    | `reason` | 0–255 | See below         |

**Total: 8 bytes.**

| reason    | Name        | Meaning                               |
| --------- | ----------- | ------------------------------------- |
| 0x0       | UNSUPPORTED | Kind not implemented on this node     |
| 0x1       | FULL        | No room for another directive         |
| 0x2       | MALFORMED   | Body did not parse                    |
| 0x3–0xFF  | reserved    | —                                     |

An ERROR is sent rather than staying silent because, for a SURVEY, silence is
itself a legitimate result. A node that cannot take the directive must say so, or
its silence will be read as an absence of coverage that does not exist.

**CANCEL and CANCEL_ALL bodies (ops 0x3, 0x4):** none. **Total: 7 bytes.**
CANCEL names one directive by `directive_id`; CANCEL_ALL retires every directive
of that kind and sets `directive_id` to 0, because not having to know what is
outstanding is the point of it.

#### G.4.9. DIAG SURVEY (kind 0x0)

**REQUEST body (op 0x0):**

| Byte | Bits | Field        | Range   | Notes                                    |
| ---- | ---- | ------------ | ------- | ---------------------------------------- |
| 7–8  | 16   | `duration_m` | 0–65535 | Directive lifetime, MINUTES              |
| 9–10 | 16   | `batch_s`    | 0–65535 | Reporting window, SECONDS                |
| 11+  | 16   | `station`    | 0–4095  | A station to observe (2 bytes each)      |

**Total: 11 + 2N bytes.**

THERE IS NO COUNT FIELD, here or in the response. The records are fixed-size and the
frame length is known, so a count would be a second statement of the same fact — and
one that could disagree with the frame it described, with nothing to say which was
right. A receiver reads station entries until the body runs out.

What that buys, beyond the byte: a body whose length is not a whole number of
records is STRUCTURALLY invalid and MUST be refused, which is a check a count could
never provide. It rests on frame integrity, which the radio's CRC already supplies
and which a count never protected anyway — a corruption that survived the CRC would
have corrupted the count too.

The lifetime is a DURATION, not a packet count. "The next N packets from station X"
never completes if X is dead, and the purpose of a timebox is that a forgotten
directive stops by itself. Minutes, so a directive can outlive a working day;
the batch window is in seconds, so a walked survey can report every few seconds
while a background watch reports every half hour.

One lifetime covers the whole directive rather than one per station: a survey is a
single operation, and per-station expiry would oblige a node to run N timers for it.

**RESPONSE body (op 0x1):**

| Byte | Bits | Field    | Range | Notes                                     |
| ---- | ---- | -------- | ----- | ----------------------------------------- |
| 7+   | —    | `record` | —     | Records, read until the body runs out     |

**Observation record (6 bytes each):**

| Offset | Bits | Field      | Range    | Notes                                        |
| ------ | ---- | ---------- | -------- | -------------------------------------------- |
| +0–1   | 16   | `station`  | 0–4095   | The station that was heard                   |
| +2–3   | 16   | `sequence` | 0–65535  | Its sequence — the key its payload joins on   |
| +4     | 8    | `rssi`     | 1–254    | dBm + 160; 0x00 = no reading, 0xFF = saturated |
| +5     | 8    | `snr_q4`   | −127–127 | SNR in quarter-dB, signed; −128 = not available |

**Total: 7 + 6N bytes**, plus 12 for an extension record if one is present.

**Extension records.** `station` is `0x000` — `IOTDATA_MESH_STATION_RESERVED`, already
"do not assign to nodes" — in a record that is not an observation at all. A reader
meeting one consumes that record's own length instead of six and carries on. This is
the door through which the response grows without a flag, a count or a new op.

One is defined: the OBSERVER'S POSITION, 12 bytes, appearing at most once because
there is one observer.

| Offset | Bits | Field       | Range        | Notes                                   |
| ------ | ---- | ----------- | ------------ | --------------------------------------- |
| +0–1   | 16   | `0x000`     | —            | Reserved: this record is not an observation |
| +2–5   | 32   | `latitude`  | ±900000000  | Degrees × 1e-7, signed                    |
| +6–9   | 32   | `longitude` | ±1800000000 | Degrees × 1e-7, signed                    |
| +10–11 | 16   | `altitude`  | ±32767      | Metres, signed                          |

A node that knows where it stands — a relay at a surveyed location, or a node with a
GNSS of its own — turns a column of RSSI numbers into a map. It is OPTIONAL because
most relays do not know, and it costs nothing when omitted.

An EMPTY BODY is a valid and REQUIRED response: it states that the directive is live
on this node and that nothing was heard in the window. Without it, a lost REQUEST
and a genuine coverage hole are indistinguishable, and a survey that invents
coverage holes is worse than no survey.

The payload itself is NOT carried. Whichever node forwards the observed packet
delivers it to the gateway by the ordinary path; the gateway joins an observation
to that payload on `(station, sequence)`. Repeating the payload once per observer
would multiply airtime for data already in hand.

This entry does NOT use the 4-bit `rssi_q4` of G.4.7. That quantisation floors at
−120 dBm, which is adequate for choosing a route and useless for a survey: −120 to
−137 dBm is exactly the edge being measured, and all of it would read as zero. SNR
earns its own byte because below the noise floor RSSI flattens out while SNR keeps
falling, making it the better discriminator at the edge.

Not every radio reports SNR: a UART LoRa module typically appends RSSI and nothing
else. Such a node MUST send `snr_q4 = -128` rather than 0, which would be a
plausible-looking reading rather than an absent one.

THE RSSI BYTE HAS THREE STATES, and only one of them is a number. A receiver may
give a reading, have none to give, or give one that ran off the top of its scale
when the sender was very close. Rendering the last two AS readings invents
measurements, and a survey exists to be believed, so both ends of the byte are
reserved to say so: `0x00` is "no reading available" and `0xFF` is "saturated —
at least as strong as this receiver can report". Neither can collide with a real
value, because no LoRa receiver reads −160 dBm and none reads +95. A real reading
MUST be clamped into `0x01`–`0xFE` rather than onto a sentinel.

A consumer MUST NOT treat `0x00` or `0xFF` as a magnitude. A saturated reception
is a real one and should be counted as such; an absent reading should be omitted
from any aggregate rather than folded in as a value.

Example sizes (without a position record; add 12 with one):

| Observations | Total bytes |
| ------------ | ----------- |
| 0            | 7           |
| 1            | 13          |
| 4            | 31          |
| 8            | 55          |
| 16           | 103         |
| 32           | 199         |

A node MUST emit a batch when the frame is full as well as when the window
expires; at the smallest payloads (SF12) roughly seven observations fill a frame,
so a busy window would otherwise silently discard observations.

#### G.4.10. DIAG TRACE (kind 0x1)

**Body (after the 7-byte DIAG envelope):**

| Byte | Bits | Field                        | Notes                                      |
| ---- | ---- | ---------------------------- | ------------------------------------------ |
| 7–8  | 4+12 | `flags` \| `subject_station` | The station being traced                   |
| 9+   | 32×N | hop records, 4 bytes each    | To the end of the frame                     |

**Flags (upper nibble of byte 7):**

| Bit  | Name               | Meaning when set                                    |
| ---- | ------------------ | --------------------------------------------------- |
| 0    | `DOWN`             | Recording the outward leg; clear means inward       |
| 1    | `TRUNCATED`        | A hop had no room: the path is incomplete           |
| 2    | `ONE`              | One path only; clear means keep the fan             |
| 3    | `RESPONSE_REQUIRED`| Return a RESPONSE; clear means consume where it lands |

**Hop record (4 bytes, identical to a NEIGHBOUR_REPORT entry):**

| Byte | Field      | Notes                                                        |
| ---- | ---------- | ------------------------------------------------------------ |
| 0    | `cost`     | That hop's advertised gradient cost; `0xFF` = none (a sensor) |
| 1    | `rssi`     | What that hop heard the PREVIOUS hop at, dBm + 160            |
| 2–3  | `station`  | That hop                                                      |

`subject_station` is not the same as hop 0: on a downward trace hop 0 is the
gateway. **There is no hop count** — records are fixed size against a known frame
length, so the count comes out of the length and a body that is not a whole
number of records is structurally refusable, which a count field could never be.

`RESPONSE_REQUIRED` is opt-in so that zero is the quiet state: a truncated or
zeroed flags nibble produces silence rather than an unexpected transmission.
Without it the terminating node consumes the record where it lands — a gateway
publishes it, a station writes it to its own diagnostics — and answers nobody.

**Total: 9 bytes + 4 per hop.**

#### G.4.11. Reserved (ctrl_type 0x6–0xF)

Reserved for future use. Relays receiving an unrecognised ctrl_type should
silently discard the packet.

#### D.11 Packet Summary

| ctrl    | Name             | Direction                     | Bytes    | Version |
| ------- | ---------------- | ----------------------------- | -------- | ------- |
| 0x0     | BEACON           | outward (gateway → relays)    | 11       | v1      |
| 0x1     | FORWARD          | inward (relays → gateway)     | 6 + N    | v1      |
| 0x2     | ACK              | single relay (parent → child) | 8        | v1      |
| 0x3     | ROUTE_ERROR      | broadcast                     | 5        | v1      |
| 0x4     | NEIGHBOUR_REPORT | inward (relays → gateway)     | 9.2 + 4N | v1      |
| 0x5     | DIAG             | both (see op)                 | 7 + N    | v1      |
| 0x6–0xF | reserved         | —                             | —        | —       |

### G.5. Node Operation and Requirements

#### G.5.1. Relay Node State

A relay maintains the following state in RAM. Total memory footprint is under
512 bytes for typical configurations.

**Routing state:**

- `parent` — station_id, cost, RSSI, last beacon time (8 bytes)
- `backup_parent` — same structure (8 bytes)
- `my_cost` — current relay count to gateway (1 byte)
- `my_gateway` — gateway_id of the tree this node belongs to (2 bytes)
- `beacon_generation` — most recently processed generation (2 bytes)

**Neighbour table (up to 63 entries):**

- Per entry: station_id, cost, RSSI, last_heard timestamp (8 bytes each)
- Typical: 8–16 entries = 64–128 bytes
- Entries expire after a configurable timeout (recommended: 5× beacon interval)

**Duplicate suppression ring (32–64 entries):**

- Per entry: origin_station_id (12 bits) + origin_sequence (16 bits) = 4 bytes
  packed
- Ring of 64 entries = 256 bytes
- FIFO: oldest entry evicted when ring is full

**Forward retry queue (4–8 entries):**

- Per entry: pending FORWARD packet buffer, retry count, timestamp of last
  attempt, parent at time of send
- Entries cleared on ACK receipt or after max retries

#### G.5.2. Relay Node Main Loop

```text
initialise:
    listen for beacons to join a tree
    set status = orphaned

on receive packet:
    if variant == 15:
        switch (ctrl_type):
            BEACON:     process_beacon()
            FORWARD:    unwrap, dedup, re-wrap, forward to parent
            ACK:        match against forward retry queue, clear entry
            ROUTE_ERROR: if sender is my parent, trigger parent reselection
            other:      discard
    else:
        // raw sensor packet (variant 0–14)
        schedule_forward(packet)   // backoff, dedup, wrap, send to parent

periodic timers:
    beacon rebroadcast   — on beacon receipt, after 1–5s random jitter
    forward retry        — check pending queue, retransmit if ACK timeout
    parent timeout       — if no beacon for 3 rounds, orphan and reselect
    neighbour report     — send report upstream every N minutes
    own sensor readings  — if dual-role, encode and transmit own data
```

#### G.5.3. Gateway Additions

An existing iotdata gateway requires three additions to support mesh:

**Beacon origination:** Every N seconds (60 default), transmit a BEACON with
cost=0 and an incrementing generation counter. The gateway_id is the gateway's
own station_id.

**FORWARD handling:** On receiving a variant 15 packet with ctrl_type 0x1,
extract the inner packet starting at byte 6 and process it through the normal
iotdata receive path (decode, store, display). Send an ACK back to the FORWARD's
sender.

**Duplicate suppression:** Maintain a ring buffer of recently-seen {station_id,
sequence} pairs. Check every incoming sensor packet (whether received directly
or unwrapped from a FORWARD) against this ring. Discard duplicates, keeping the
first arrival.

The existing iotdata decode path for variants 0–14 is completely untouched.

#### G.5.4. Duplicate Suppression

Duplicate suppression is critical because the same sensor packet may arrive at a
gateway via multiple paths: directly, via one relay, or via different relay
chains. Without dedup, every measurement would be recorded multiple times.

The dedup key is {origin_station_id, origin_sequence}, extracted from the
iotdata header of the original sensor packet. Both relays and gateways maintain
dedup rings:

- **At the relays:** prevents forwarding the same sensor packet twice (e.g. two
  relays both hear the same sensor and both forward upstream — the upstream
  relay deduplicates).
- **At the gateway:** prevents processing the same data twice when it arrives
  both directly and via relay.

A ring buffer of 64 entries is sufficient for most deployments. With 16 sensors
transmitting every 5–15 seconds, the ring covers approximately 5–20 minutes of
history. The ring is FIFO — the oldest entry is evicted when the buffer is full.

#### G.5.5. Parent Selection and Failover

A relay selects its parent using the following priority:

1. Lowest cost (fewest relays to gateway)
2. If equal cost, highest RSSI (strongest signal)
3. If equal cost and RSSI, prefer existing parent (stability)

The backup parent is the second-best candidate by the same criteria.

**Failover triggers:**

- FORWARD ACK timeout after max retries — parent is unreachable.
- ROUTE_ERROR received from parent — parent has lost its own uplink.
- Beacon timeout — no beacon from parent's tree for 3 consecutive rounds.

On failover, the node promotes its backup parent, recalculates cost (new
parent's cost + 1), and continues forwarding. If no backup is available, the
node broadcasts a ROUTE_ERROR with reason=parent_lost and enters orphaned state,
listening for beacons from any tree.

#### G.5.6. Beacon Rebroadcast Rules

A relay rebroadcasts a received beacon only if:

1. The beacon's generation is newer than the last processed generation for this
   gateway_id (modular comparison: newer if difference mod 4096 is in range
   1–2047), OR
2. The beacon has the same generation but offers a strictly lower cost than the
   current best seen for this generation.

If the beacon does not meet either condition, it is suppressed. This prevents
beacon storms in dense deployments where many nodes hear the same beacon
simultaneously. The random rebroadcast jitter (1–5 seconds) further reduces
collision probability.

#### G.5.7. Forward Suppression (Trickle)

When a relay hears a raw sensor packet that it intends to forward, it waits a
random backoff period (200–1000ms) before transmitting the FORWARD. During this
backoff, if the node hears another relay transmit a FORWARD containing the same
inner packet (identified by matching origin station and sequence in the inner
header), it cancels its own forward.

This Trickle-style suppression (inspired by RFC 6206) significantly reduces
redundant airtime in areas where multiple relay nodes have overlapping coverage.
In the worst case (no other relay forwards), it adds 200–1000ms latency to the
first relay. In dense areas, it eliminates duplicate transmissions entirely.

#### G.5.8. Reception Survey Operation

A node supporting DIAG SURVEY maintains a bounded table of active directives,
keyed by `directive_id`.

**Taking a directive.** On a SURVEY REQUEST:

- If `directive_id` is already in the table, the node rebroadcasts the frame (see
  below) but does NOT restart its window, reset its batch, or re-arm its expiry.
- If it is not, and there is room, the node installs it, starts its window, and
  rebroadcasts.
- If there is no room, the node replies ERROR/FULL and does not rebroadcast.

The idempotence is what makes re-issue safe. An operator who believes some nodes
missed a directive simply sends it again: nodes that missed it install it, nodes
that already hold it merely pass it on, and nobody's window moves. No targeting
information is needed, and a retry cannot corrupt the survey already in progress.

**Propagation.** A SURVEY REQUEST is flooded. A node rebroadcasts the frame with
its own `sender_station` and `sender_seq`, preserving `kind`, `op` and
`directive_id`. To terminate the flood, a node MUST NOT rebroadcast the same
`directive_id` more often than once per rebroadcast-suppression interval; with
that rule every node transmits at most once and the flood dies out naturally,
without a hop count. An operator re-issuing a directive SHOULD space re-issues
further apart than that interval, or intermediate nodes will decline to pass the
re-issue on.

**Observation.** While a directive is live, a node records a reception when, and
only when, the received frame's OWN sender identity is a named station. That rule
is what makes the measurement the right one: a relayed copy carries the forwarding
relay's identity in its header, not the originator's, so it cannot be mistaken for
a direct reception of the station that sent it. A node hearing both the original
transmission and a relayed copy therefore records only the original. The same rule
makes surveying a RELAY work without a special case, since a relay's beacons and
forwards are its own frames.

Each observation records the station, its sequence, and the RSSI and SNR of that
reception. A node MUST NOT record a downstream frame under this rule: a downstream
frame's header station is its TARGET, not its sender.

**Reporting.** A node emits a SURVEY RESPONSE when its batch window expires or
when a further entry would not fit the frame, whichever comes first, and emits it
whether or not it has anything to report. Responses are carried INSIDE a FORWARD
like ordinary data, so they inherit duplicate suppression, relay-by-relay
acknowledgement and retry; an observation happens once and unacknowledged loss
would appear as a coverage hole that is not real.

A response is originated by the OBSERVING node, under its own station identity and
its own sequence. This matters for duplicate suppression: several nodes reporting
the same observed `(station, sequence)` are emitting genuinely different frames,
and attributing them to the observed station instead would cause the gateway to
discard all but one of exactly the reports the survey exists to collect.

**Expiry.** A directive is retired when its lifetime elapses, or on CANCEL naming
its id, or on CANCEL_ALL. A node SHOULD emit a final response covering any
outstanding observations as it retires the directive.

**Gateways.** A gateway takes directives and records direct receptions under the
same rules. Its own observations need no transmission, but they are the same
measurement and are reported alongside those arriving from relays.

**Airtime.** A survey adds one uplink per observing node per batch window, on top
of the ordinary forwarding of the observed traffic. In a duty-cycle-limited band
this is significant, and it is the reason directives are timeboxed rather than
standing: the batch window should be set as long as the survey can tolerate, and
`count = 0` responses make a long window safe to interpret.

### G.6. Deployment Considerations

[moved]

#### G.6.1. Hardware

[moved]

#### G.6.2. Range and Relay Budgets

[moved]

#### G.6.3. Latency Budget

[moved]

#### G.6.4. Airtime and Duty Cycle

[moved]

#### G.6.5. Recommended Maximum Configuration Per Deployment

[moved]

### G.7. Example Deployments

[moved]

#### G.7.1. Moderate Farm (Mixed Arable and Livestock)

[moved]

#### G.7.2. Forest Research Station

[moved]

#### G.7.3. Considerations for Moving Sensors

[moved]

### G.8. Protocol Version History

| Version      | Description                                                                                                                                                         |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| v1           | Initial mesh protocol. BEACON, FORWARD, ACK, ROUTE_ERROR, NEIGHBOUR_REPORT. Gradient-based routing with single parent selection and relay-by-relay acknowledgement. |
| v2           | Adds DIAG (ctrl_type 0x5), a generic diagnostics envelope, and its first kind, SURVEY: which nodes hear a given station, and at what margin. Backward compatible — a node that does not implement it discards an unrecognised ctrl_type. |
| v2           | Adds DIAG kind TRACE: the path a frame actually took, hop by hop, with the margin on each link. Removes the PING/PONG control type reservations (0x6, 0x7) unused since v1 — reachability, if wanted, becomes a DIAG kind rather than a control type, because as a control type it required downstream routing that does not exist. No id is reserved for it: these two spent long enough reserved to be documented under three different numbers in three different places. |

### G.9. Reserved Identifiers

| Identifier   | Value     | Meaning                           |
| ------------ | --------- | --------------------------------- |
| variant_id   | 0x0F (15) | Mesh control packet               |
| ctrl_type    | 0x0–0x7   | Defined mesh packet types         |
| ctrl_type    | 0x8–0xF   | Reserved for future use           |
| diag kind    | 0x0       | SURVEY                            |
| diag kind    | 0x1–0xF   | Reserved for future use           |
| diag op      | 0x0–0x4   | Defined diagnostic operations     |
| diag op      | 0x5–0xF   | Reserved for future use           |
| diag reason  | 0x3–0xFF  | Reserved for future use           |
| parent_id    | 0xFFF     | Orphaned (no parent)              |
| station_id   | 0x000     | Reserved (do not assign to nodes) |
| reason codes | 0x3–0xF   | Reserved for future use           |

### G.10. Future Considerations

[moved]

#### G.10.1. Cross-Gateway Duplicate Suppression

[moved]

#### G.10.2. Potential Additional Control Packet Types

[moved]

#### G.10.3. Extended Neighbour Metrics

[moved]

#### G.10.4. Security Considerations

[moved]

#### G.10.5. Power Management for Relay Nodes

[moved]

#### G.10.6. Network Capacity Planning

[moved]

#### G.10.7. Interoperability and Versioning

[moved]

## Appendix H. System Architecture Considerations

[moved]

### H.1. Transmission Scheduling

[moved]

#### Interval Selection

[moved]

#### Jitter

[moved]

#### Adaptive Intervals

[moved]

### H.2. Gateway Architecture

[moved]

#### Receive Path

[moved]

#### State Management

[moved]

### H.3. Operational Monitoring

[moved]

#### Per-Station Metrics

[moved]

#### System-Wide Metrics

[moved]

#### Alerting

[moved]

### H.4. Time Synchronisation

[moved]

### H.5. Data Pipeline Considerations

[moved]

### H.6. Multi-Gateway Deployments

[moved]

## Appendix I. Comparison with Alternative Encodings and Embedded Libraries

[moved]

### I.1. Test Payload

[moved]

### I.2. Generic Serialisation Formats

[moved]

#### I.2.1. iotdata (this protocol)

[moved]

#### I.2.2. JSON (compact, no whitespace)

[moved]

#### I.2.3. CBOR (Concise Binary Object Representation, RFC 8949)

[moved]

#### I.2.4. Protocol Buffers (Protobuf, varint encoding)

[moved]

#### I.2.5. MessagePack

[moved]

#### I.2.6. Raw C struct (packed)

[moved]

### I.3. IoT-Specific Encodings

[moved]

#### I.3.1. CayenneLPP

[moved]

#### I.3.2. Nanopb (Protocol Buffers for Embedded C)

[moved]

#### I.3.3. Bitproto

[moved]

#### I.3.4. TinyCBOR and QCBOR

[moved]

### I.4. Encoding Summary

[moved]

### I.5. Analysis

[moved]

### I.6. Embedded Library Design Comparison

[moved]

#### I.6.1. Reference Libraries

[moved]

#### I.6.2. Design Principle Comparison

[moved]

#### I.6.3. Positioning

[moved]

### I.7. Impact on LoRa Airtime and Battery Life

[moved]

## Appendix J. Known Limitations and Open Issues

[moved]

### J.1. Pressure Range

[moved]

### J.2. Wind Speed Range and Resolution

[moved]

### J.3. Linear Quantisation vs. Real-World Distributions

[moved]

### J.4. Datetime Resolution vs. Bit Allocation

[moved]

### J.5. Header Bit Allocation

[moved]

### J.6. Presence Byte TLV Bit Placement

[moved]

### J.7. Rain Drop Size Semantics

[moved]

### J.8. UV Index Range

[moved]

### J.9. Information-Theoretic Efficiency

[moved]

### J.10. Sensor-Specific Field Semantics

[moved]

### J.11. Irradiance Range

[moved]

### J.12. Image Field Practicality

[moved]

### J.13. Absence of Test Vectors

[moved]

### J.14. Formal Decode Specification

[moved]

### J.15. Bundle vs. Standalone Asymmetry

[moved]

### J.16. Cloud Cover Resolution

[moved]

### J.17. Additional Sensor Field Types

[moved]

#### J.17.1. Soil Moisture and Conductivity

[moved]

#### J.17.2. Water Quality: pH

[moved]

#### J.17.3. Water Quality: Electrical Conductivity

[moved]

#### J.17.4. Water Quality: Dissolved Oxygen

[moved]

#### J.17.5. Water Quality: Turbidity

[moved]

#### J.17.6. Water Quality: ORP (Oxidation-Reduction Potential)

[moved]

#### J.17.7. Water Flow / Discharge

[moved]

#### J.17.8. Leaf Wetness / Surface Moisture

[moved]

#### J.17.9. Thermal Rate of Change and Fire Detection

[moved]

### J.18. Silent Decode of Corrupted Payloads

[moved]

### J.19. No Schema Tooling or Code Generation

[moved]

### J.20. C-Only Reference Implementation

[moved]

### J.21. No LoRaWAN Ecosystem Integration

[moved]

### J.22. No Variant Table Discovery or Advertisement

[moved]

### J.23. Relationship to ASN.1 Packed Encoding Rules (UPER)

[moved]

### J.24. Relationship to SenML and LwM2M

[moved]

### J.25. Energy and Battery Life Impact

[moved]

### J.26. Delta and Differential Encoding

[moved]

### J.27. Variant Map Transmission and Global Field Type Identifiers

[moved]

### J.28. Mesh Protocol Comparison

[moved]

