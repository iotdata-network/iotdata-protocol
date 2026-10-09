# iotdata — Sensor Field Type Architecture

Working reference for the iotdata field type system: architecture, domain
coverage, device constraints, and prioritisation.

---

## 0. Design Rationale & Epistemological Framing

[moved]

### 0.1 Applicable Standards and Frameworks

[moved]

### 0.2 Organising Principles Applied

[moved]

### 0.3 Scope

[moved]

## 1. Current State (Reference Baseline)

[moved]

## 2. Encoding Architecture (Abstract Building Blocks)

[moved]

### 2.1 Three-Layer Field Type Hierarchy

[moved]

### 2.2 Scale Types and Sign Domains

[moved]

#### Scale Types

[moved]

#### Sign Domains

[moved]

#### Why these matter

[moved]

### 2.3 (Reserved)

[moved]

### 2.4 Temporal Interpretation

[moved]

### 2.5 The `_expanded` Convention

[moved]

### 2.6 Presence and Inclusion Semantics

[moved]

### 2.7 Bundles and Co-location

[moved]

### 2.8 Naming Conventions

[moved]

#### Alternative Encoding Variants

[moved]

### 2.9 Derivation Relationships

[moved]

### 2.10 Resolution vs Accuracy Boundary

[moved]

### 2.11 Layer 1 — Generic / Flexible Field Types

[moved]

#### Integer Types

[moved]

#### Floating Point Types

[moved]

#### Opaque / Raw Types

[moved]

### 2.12 Layer 2 — Unit-Based Field Types

[moved]

#### Length / Distance

[moved]

#### Percentage

[moved]

#### Ratio

[moved]

#### Temperature

[moved]

#### Pressure

[moved]

#### Electrical

[moved]

#### Volume / Flow

[moved]

#### Weight / Force

[moved]

#### Speed

[moved]

#### Angle

[moved]

#### Counter

[moved]

#### Sound

[moved]

#### Illuminance

[moved]

#### Concentration (generic)

[moved]

#### Magnetic

[moved]

#### Acceleration

[moved]

#### RPM

[moved]

### 2.13 Open Design Questions (Architecture Level)

[moved]

## 3. Domain Coverage (What We Measure)

[moved]

### 3.1 Atmosphere

[moved]

#### 3.1.1 Weather (Temperature, Pressure, Humidity, Wind, Rain, Clouds)

[moved]

#### 3.1.2 Solar and Light

[moved]

#### 3.1.3 Air Quality (Particulates)

[moved]

#### 3.1.4 Air Quality (Gases)

[moved]

#### 3.1.5 Radiation

[moved]

#### 3.1.6 Lightning

[moved]

### 3.2 Hydrosphere (Water Quality, Flow, Volume)

[moved]

#### 3.2.1 Water Quality

[moved]

#### 3.2.2 Water Flow and Volume

[moved]

#### 3.2.3 Water Level / Stage

[moved]

### 3.3 Lithosphere (Soil)

[moved]

### 3.4 Biosphere

[moved]

#### 3.4.1 Vegetation and Agriculture

[moved]

#### 3.4.2 Acoustic / Wildlife Monitoring

[moved]

#### 3.4.3 Biological / Ecological (Gaps Identified)

[moved]

### 3.5 Built Environment

[moved]

#### 3.5.1 Power and Electrical Monitoring

[moved]

#### 3.5.2 Structural Health (Gaps Identified)

[moved]

#### 3.5.3 Indoor Environment (Gaps Identified)

[moved]

### 3.6 Motion, Vibration, and Mechanical

[moved]

### 3.7 Electromagnetic

[moved]

### 3.8 Rate of Change (Cross-Domain)

[moved]

## 4. Device Coverage (What Sensors Produce)

[moved]

### 4.1 Sensor Chips and Modules

[moved]

#### 4.1.1 Temperature

[moved]

#### 4.1.2 Humidity (typically includes temperature)

[moved]

#### 4.1.3 Barometric Pressure (typically includes temperature)

[moved]

#### 4.1.4 Multi-Sensor Environment (temp + humidity + pressure)

[moved]

#### 4.1.5 Air Quality — Particulate Matter

[moved]

#### 4.1.6 Air Quality — Gas Sensors

[moved]

#### 4.1.7 Soil Sensors

[moved]

#### 4.1.8 Water Quality Sensors

[moved]

#### 4.1.9 Wind

[moved]

#### 4.1.10 Rain

[moved]

#### 4.1.11 Solar / Light / UV

[moved]

#### 4.1.12 Distance / Ranging

[moved]

#### 4.1.13 Weight / Load

[moved]

#### 4.1.14 Acceleration / Motion / Tilt

[moved]

#### 4.1.15 Magnetometer

[moved]

#### 4.1.16 Lightning

[moved]

#### 4.1.17 Sound / Noise

[moved]

#### 4.1.18 Radiation

[moved]

#### 4.1.19 GPS / Position

[moved]

#### 4.1.20 Power Monitoring

[moved]

#### 4.1.21 LoRa / Radio Modules (Link Quality)

[moved]

### 4.2 Sensor Products (Integrated Systems)

[moved]

#### 4.2.1 Weather Station Products

[moved]

#### 4.2.2 Soil / Agriculture Products

[moved]

#### 4.2.3 Water Quality Products

[moved]

#### 4.2.4 Air Quality Products

[moved]

#### 4.2.5 Multi-Purpose IoT / LoRaWAN Sensors

[moved]

#### 4.2.6 Beehive / Livestock / Speciality Agriculture

[moved]

#### 4.2.7 Indoor / Building Sensors

[moved]

### 4.3 Integration Notes

[moved]

## 5. Prioritisation (Cross-Cutting)

[moved]

### Tier 1 — Before v1.0

[moved]

### Tier 2 — v1.x

[moved]

### Tier 3 — Future / Niche

[moved]

### Leverage Analysis (Tier 1)

[moved]

