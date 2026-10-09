# NOTES - TBD (unclassified chunks)

> Chunks whose targeting/specification placement is unresolved - lifted here verbatim for manual triage.

## 1. Current State (Reference Baseline)

These fields are already defined in the iotdata protocol specification. No work
is needed except where issues are flagged.

| Field            | Bits     | Range                              | Resolution     | Notes                                            |
| ---------------- | -------- | ---------------------------------- | -------------- | ------------------------------------------------ |
| Battery          | 6        | 0–100%, charging flag              | ~3.2%          |                                                  |
| Link             | 6        | RSSI -120–-60 dBm, SNR -20–+10 dB  | 4 dBm / 10 dB  |                                                  |
| Temperature      | 9        | -40 to +80°C                       | 0.25°C         | Standalone + in Environment bundle               |
| Pressure (baro)  | 8        | 850–1105 hPa                       | 1 hPa          | **J.1: range too narrow, needs fix**             |
| Humidity         | 7        | 0–100%                             | 1%             | Standalone + in Environment bundle               |
| Environment      | 24       | Temp+Press+Humid bundle            | as above       |                                                  |
| Wind Speed       | 7        | 0–63.5 m/s                         | 0.5 m/s        | **J.2: max may be too low**                      |
| Wind Direction   | 8        | 0–355°                             | ~1.41°         |                                                  |
| Wind Gust        | 7        | 0–63.5 m/s                         | 0.5 m/s        |                                                  |
| Wind             | 22       | Speed+Dir+Gust bundle              | as above       |                                                  |
| Rain Rate        | 8        | 0–255 mm/hr                        | 1 mm/hr        |                                                  |
| Rain Size        | 4        | 0–6.0 mm                           | 0.25 mm        | **J.7: semantics undefined**                     |
| Rain             | 12       | Rate+Size bundle                   | as above       |                                                  |
| Solar Irradiance | 10       | 0–1023 W/m²                        | 1 W/m²         | **J.11: may need 11 bits; J.15: no standalone**  |
| UV Index         | 4        | 0–15                               | 1              | **J.8: max 15 may be low; J.15: no standalone**  |
| Solar            | 14       | Irradiance+UV bundle               | as above       |                                                  |
| Clouds           | 4        | 0–8 okta                           | 1 okta         |                                                  |
| AQ Index         | 9        | 0–500 AQI                          | 1              |                                                  |
| AQ PM            | 4–36     | 0–1275 µg/m³ per channel           | 5 µg/m³        | 4 channels: PM1, PM2.5, PM4, PM10                |
| AQ Gas           | 8–84     | Variable per gas                   | Variable       | VOC, NOx, CO₂, CO, HCHO, O₃ + 2 reserved         |
| Air Quality      | 21+      | Index+PM+Gas bundle                | as above       |                                                  |
| Radiation CPM    | 14       | 0–16383 CPM                        | 1 CPM          |                                                  |
| Radiation Dose   | 14       | 0–163.83 µSv/h                     | 0.01 µSv/h     |                                                  |
| Radiation        | 28       | CPM+Dose bundle                    | as above       |                                                  |
| Depth            | 10       | 0–1023 cm                          | 1 cm           | Generic: snow, water level, ice, soil depth      |
| Position         | 48       | Lat/lon ±90°/±180°                 | ~1.2 m         |                                                  |
| Datetime         | 24       | Year-relative, 5s ticks            | 5 s            |                                                  |
| Flags            | 8        | 8-bit bitmask                      | —              | Deployment-defined                               |
| Image            | Variable | Pixel data + control               | —              | Experimental                                     |

Known issues carried forward: J.1 (barometric pressure range), J.2 (wind speed
max), J.7 (rain size semantics), J.8 (UV Index max), J.11 (irradiance bits),
J.15 (no standalone irradiance/UV).

---

