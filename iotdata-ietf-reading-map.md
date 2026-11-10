# Position Map for iotdata — constrained IoT, the IETF and beyond

A curated, structured map of the standards and technologies iotdata has to be
understood against. It is still IETF-heavy, because the IETF is where a BoF happens,
but it is organised around the problem space rather than around IETF/IAB working
groups, and it takes in the IEEE, 3GPP, ETSI, Bluetooth SIG, CSA, LoRa Alliance and
others where they own the relevant ground. The aim is to arrive with a whole
worldview, not an IETF-shaped one. Each entry is tagged:

- **[ADOPT]** — existing, mature work you can use directly rather than reinvent.
- **[POSITION]** — work iotdata overlaps with or is an alternative to; you must be
  able to articulate the difference in a BoF.
- **[VOCAB]** — read primarily for the terminology, framing and landscape knowledge
  you are expected to speak fluently.
- **[PROCESS]** — how the IETF and the BoF→WG path actually work.

## How to use this

1. **Structure.** Part I builds the worldview: the dimensions for classifying
   constrained devices and links (§1), the technologies classified in those terms
   (§2), and where IP fits and how devices are used (§3). Part II places iotdata in
   that world: what it competes with and what it adopts (§4–8). *Positioning &
   differences* at the end gathers the argument.
2. **Status changes.** Drafts (`draft-ietf-…`) expire every 6 months and revise
   constantly. IETF draft numbers here reflect their state in late 2025, except
   draft-ietf-iotops-7228bis, which was re-checked in November 2026; before citing
   any draft, pull the current revision from its working-group page on
   `datatracker.ietf.org`. RFCs (numbered, no `draft-` prefix) are stable. Non-IETF
   specifications (§2) are noted as of November 2026; industry alliances revise
   roughly yearly, so check each body's site before citing a version.
3. **Four framings that matter for the BoF** (carried over from the design
   discussion; the full case is in *Positioning & differences* at the end):
   - **Bottom-up, not top-down.** The IETF's constrained-device work largely adapts
     IPv6 and the web stack down to these links; iotdata is designed up from the
     link budget and meets IP at an edge gateway. Present it as a different design
     point (7228bis integration class I2), not as the IETF getting it wrong.
   - **Heterogeneity, not homogeneity.** The deployed world has not converged on IPv6
     everywhere: Zigbee, BLE, Z-Wave, LoRaWAN and Sigfox sit alongside Thread and
     Wi-SUN, and even Matter bridges to non-IP devices. The realistic architecture
     interfaces and adapts at boundaries — which is what the IETF's own 6lo and SCHC
     work already does. Say so as an observation, politely, not as a rebuttal.
   - Do **not** pitch iotdata as "a better CBOR." Pitch it in the
     constrained-telemetry / LPWAN space — i.e. against **SCHC and SenML** — where
     the comparison is apples-to-apples and the differentiation (sub-byte packing +
     presence flags + domain-typed quantisation) is real. CayenneLPP is not a
     technical peer; mention it only as evidence that no IETF format serves this space.
   - The **SUIT information model (RFC 9124) is encoding-agnostic.** The cleanest
     firmware story — technically and politically — is "an iotdata serialisation of
     the SUIT information model," which is *complementary* to CBOR/SUIT rather than
     competing.
4. **Critical path** (read these before writing a BoF proposal): RFC 5434 → RFC 7228
   (+ 7228bis) + RFC 8376 → RFC 8428 (SenML) + RFC 8724 + RFC 9011 (SCHC) → RFC 8949
   + RFC 8610 (CBOR/CDDL) → RFC 9019 + RFC 9124 + draft-ietf-suit-manifest → RFC 9052
   (COSE). Everything else is depth around these.

---

## 0. Process & BoF readiness — read first

The BoF is a sales pitch to a room that has seen many formats die. These set the rules of that room.

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 5434** | Considerations for Having a Successful Birds-of-a-Feather (BoF) Session | [PROCESS] | The single most directly relevant document to your stated goal. What a BoF is for, the difference between a "form a WG" BoF and a "discuss a problem" BoF, and the failure modes. Read it twice. |
| **Tao of the IETF** | (living web doc at `ietf.org/about/participate/tao`, formerly RFC 6722) | [PROCESS] | Orientation for newcomers: how WGs, ADs, areas, IESG, mailing lists, and consensus actually operate. |
| **RFC 2026** | The Internet Standards Process — Revision 3 (BCP 9) | [PROCESS] | The standards-track machinery (Proposed/Internet Standard, maturity levels) your work would eventually move through. |
| **RFC 7282** | On Consensus and Humming in the IETF | [PROCESS] | What "rough consensus" means and how decisions get made — directly shapes how you should frame and defend iotdata. |
| **RFC 2418** | IETF Working Group Guidelines and Procedures (BCP 25) | [PROCESS] | How a WG is chartered and run — the thing a successful BoF leads to. |
| **RFC 8179** | Intellectual Property Rights in IETF Technology (BCP 79) | [PROCESS] | IPR disclosure rules. Relevant given you have an existing implementation and may hold IP. |
| **RFC 3935** | A Mission Statement for the IETF | [VOCAB] | "Running code and rough consensus," the interoperability-first ethos. Useful for understanding why principle #8 (no-interop) is a hard sell as-is. |

Tooling note: there is no RFC that tells you how to *write* a draft — use the IETF author resources (xml2rfc / kramdown-rfc, the datatracker submission flow). Budget time to learn the format; a well-formed I-D is itself a credibility signal.

---

# Part I — Worldview

The IETF divides this space along its own working-group lines: wide-area links (LPWAN) apart from short-range ones (LoWPAN/6lo), routing apart from terminology apart from operations. Part I divides it by what you need to reason about instead: how to classify devices and links (§1), what technologies exist and how they classify (§2), and where IP fits and how devices are actually used (§3).

## 1. Dimensions — classifying constrained devices and networks (7228++)

The most useful idea in this space is classification along independent dimensions rather than by technology. RFC 7228 started it; 7228bis extends it to twelve tables. Most of the dimensions are physical (memory, energy, link), not IP-specific — only the I-classes are. Use them as the vocabulary for §2 and §3, and to state precisely what iotdata targets.

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 7228** | Terminology for Constrained-Node Networks | [VOCAB] | Defines Class 0/1/2 devices and constrained-network vocabulary that *every* doc below assumes. Your sensors edge toward Class 1. Your ESP32-C3 (~400 KiB SRAM) is beyond 7228's Class 2 — it is **C4** in 7228bis's table (beware the name clash with 7228bis's own class "C3"). Use these terms in your spec. |
| **draft-ietf-iotops-7228bis** | Terminology for Constrained-Node Networks (revision) | [VOCAB] | The update to 7228 in the IOTOPS WG; at -10 (Jul 2026) and with the IESG, so soon the reference. Keeps 7228's classes and adds device classes up to C19, firmware upgradability (F0–F9), isolation, secret shielding, timekeeping, and network classes: link MTU (S0–S19), Internet integration (I0–I9), PHY bit rate (B0–B3). Its weakest part is the M-group/J-group split ("J" for "Jeep"), which adds nothing the C-numbers don't already carry — use the C-classes. |
| **RFC 7102** | Terms Used in Routing for Low-Power and Lossy Networks | [VOCAB] | The ROLL glossary: LLN, LBR, sleepy/non-sleepy node, upstream/downstream, and the traffic patterns MP2P/P2MP/P2P. 7228 borrows from it; you'll need it to describe the mesh layer against RPL (§6). |

**Dimensions at a glance.** The first ten exist in 7228/7228bis. The rest are *proposed* additions — properties that RFC 8376 and the technology survey (§2) describe in prose, one technology at a time, and that become a uniform matrix once they are dimensions.

| Dimension | Classes / values | Source |
| --- | --- | --- |
| Device memory (RAM / flash) | C0–C4 microcontroller; C10–C19 general-purpose | 7228, 7228bis |
| Firmware upgradability | F0 (none) … F9 (hitless) | 7228bis |
| Isolation | Is0 (none) … Is8 (secure enclave) | 7228bis |
| Secret shielding | Sh0, Sh1, Sh9 | 7228bis |
| Energy limitation | E0 event, E1 period, E2 lifetime, E9 mains | 7228 |
| Power strategy for communication | P0 normally-off, P1 low-power, P9 always-on | 7228 |
| Timekeeping | T0 … T10; permanency TP0, TP1, TP9 | 7228bis |
| Link MTU | S0 (3–12 B) … S19 (≥ 64 KiB) | 7228bis |
| Internet integration | I0 none, I1 device-to-cloud, I2 via edge gateway, I9 full | 7228bis |
| PHY bit rate | B0 < 10 bit/s, B1 to 1 kbit/s, B2 to 1 Mbit/s, B3 above | 7228bis |
| Topology *(proposed)* | Star; star-of-stars; relay / multi-hop; mesh | 8376, 7102 |
| Range *(proposed)* | Personal/home; local/campus; wide-area (km+) — the axis the IETF splits 6lo from LPWAN on | 8376, §2 |
| Spectrum *(proposed)* | Licensed; licence-exempt sub-GHz; licence-exempt 2.4 GHz | 8376, §2 |
| Regulatory limits *(proposed)* | Duty cycle (e.g. 1%); messages per day; dwell time | 8376 |
| Link asymmetry & downlink model *(proposed)* | Uplink only; downlink window after uplink; scheduled downlink; always listening | 8376 (LoRaWAN classes, Sigfox), mioty classes |
| Reachability & latency *(proposed)* | Always reachable … reachable only when the device chooses | 7228 §2.2, 8376 |
| Traffic pattern *(proposed)* | MP2P, P2MP, P2P; periodic, event, bulk | 7102, §3 |
| IP-nativeness *(proposed)* | IP to the device; IP terminated at a gateway; no IP | §3; complements the I-classes |

---

## 2. Technologies — the landscape in those terms (8376++)

RFC 8376 surveys four wide-area technologies as of 2018, and the IETF treats short-range links separately (6lo). A defensible view needs both, from whichever body owns them, described in one frame: classify each by the §1 dimensions. Where the IETF has an adaptation (6lo, SCHC profiles), it is listed as the bridge (§2.4).

### 2.1 Surveys

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 8376** | Low-Power Wide Area Network (LPWAN) Overview | [VOCAB] | The scene-setter for your exact deployment class — LoRa/NB-IoT/Sigfox/Wi-SUN characteristics, duty cycle, star topology, asymmetry, plus the generic Dev/RGW/NGW/AAA/App architecture. Frame iotdata's motivation in these terms; note its 2018 scope (four technologies, wide-area only). |
| **RFC 8352** | Energy-Efficient Features of Internet of Things Protocols | [VOCAB] | Power-saving mechanisms per link (802.11 PSM, BLE connection events, 802.15.4 TSCH/CSL/RIT, DECT ULE paging) and above (6LoWPAN, RPL, CoAP), with per-operation energy figures. Each link's mechanism is a column in the §2 matrix; cite it when arguing bytes-on-air → battery life. |

### 2.2 Wide-area (LPWAN)

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **LoRaWAN** | LoRa Alliance: LoRaWAN L2 specification (1.0.x / 1.1) + RP002 Regional Parameters | [VOCAB] | Your primary deployment link. Star-of-stars; Class A/B/C downlink models; frame size and duty cycle set per region and data rate by RP002 — the constraints iotdata is designed to. |
| **LoRaWAN Relay** | LoRa Alliance TS011-1.0.0 | [POSITION] | The LoRa Alliance's own multi-hop answer: a relay end-device forwards for devices out of gateway range. A standards-body comparison point for iotdata's mesh on LoRa. |
| **Sigfox** | Sigfox 0G (now operated by UnaBiz) | [VOCAB] | Ultra-narrowband operator network: ~12-byte uplink payloads, a handful of downlinks per day. The extreme case of uplink-favoured asymmetry and regulatory message limits. |
| **NB-IoT / LTE-M** | 3GPP Release 13 onward; 5G RedCap / eRedCap (Rel-17/18) | [VOCAB] | Licensed, operator-managed cellular IoT: PSM and eDRX sleep, IP or non-IP data delivery. The cellular end of the space; eRedCap is where 3GPP is heading. |
| **mioty** | ETSI TS 103 357 (TS-UNB, Low Throughput Networks) | [VOCAB] | Telegram splitting for interference robustness; v2.1.1 (2024) adds Class B/C downlink. An ETSI-standardised LPWAN, unlike LoRaWAN's alliance specification. |
| **Wi-SUN FAN** | Wi-SUN Alliance FAN 1.1 (over IEEE 802.15.4g) | [VOCAB] | IPv6 mesh (6LoWPAN + RPL) at field-area scale for utilities and cities — evidence that IP-to-device meshes work outdoors, with mains-powered routers. |
| **DECT NR+** | ETSI DECT-2020 NR (TS 103 636 series) | [VOCAB] | Licence-exempt, non-cellular, self-healing mesh aimed at dense professional IoT; commercially available from late 2025. A new mesh-native entrant in the space iotdata's mesh targets. |
| **Wi-Fi HaLow** | IEEE 802.11ah (Wi-Fi Alliance HaLow certification) | [VOCAB] | Sub-1 GHz Wi-Fi with ~1 km range and much higher rates than LPWAN; IP-native. The "just use Wi-Fi" answer at the top of the range and bit-rate scales. |

### 2.3 Local-area & mesh

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **IEEE 802.15.4** | IEEE 802.15.4-2024, Low-Rate Wireless Networks | [VOCAB] | The PHY/MAC under Thread, Zigbee, Wi-SUN and 6LoWPAN; 127-byte frames (class S1). |
| **Thread** | Thread Group, Thread 1.4 | [POSITION] | IPv6 mesh (6LoWPAN) over 802.15.4 and the transport under Matter. The strongest evidence for IP to the device; have an answer for why that doesn't carry over to LPWAN (B1 bit rates, duty cycle, battery-only nodes). |
| **Matter** | CSA, Matter 1.5 | [POSITION] | Application-layer data model over IPv6 (Thread, Wi-Fi, Ethernet). A non-IETF ecosystem that chose IP *and* defined its own data model — expect it as a counter-example. |
| **Zigbee** | CSA, Zigbee 4.0 and Suzi (Nov 2025) | [VOCAB] | Non-IP mesh on 802.15.4; 4.0 adds a sub-GHz long-range mesh ("Suzi"). Non-IP mesh moving into longer range. |
| **Bluetooth LE / Mesh** | Bluetooth SIG: Core Specification 6.x; Mesh Protocol 1.1 | [VOCAB] | Ubiquitous on phones; managed-flood, non-IP mesh; the onboarding path for other ecosystems (e.g. Zigbee Direct). |
| **Z-Wave** | Z-Wave Alliance; ITU-T G.9959; Z-Wave Long Range | [VOCAB] | Sub-GHz home mesh; Long Range adds a star topology reaching miles. |
| **Meshtastic** | (open source; not a standard) | [POSITION] | Community LoRa mesh using managed flooding. The de facto LoRa mesh people will compare iotdata's mesh layer to. |

### 2.4 IETF bridges — IP adaptation for those links

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 7668** | IPv6 over Bluetooth Low Energy | [VOCAB] | 6lo adaptation for BLE — how the IETF brings IP to a non-IP link. |
| **RFC 9159** | IPv6 Mesh over Bluetooth Low Energy Using IPSP | [VOCAB] | Extends RFC 7668 to multi-hop BLE mesh. |
| **RFC 8105** | Transmission of IPv6 Packets over DECT Ultra Low Energy | [VOCAB] | 6lo adaptation for DECT ULE. |
| **RFC 7428** | Transmission of IPv6 Packets over ITU-T G.9959 Networks | [VOCAB] | 6lo adaptation for Z-Wave's PHY/MAC. |
| **RFC 9391** | SCHC over NB-IoT | [VOCAB] | SCHC profile for 3GPP NB-IoT (the LoRaWAN profile, RFC 9011, is in §5). |
| **RFC 9442** | SCHC over Sigfox LPWAN | [VOCAB] | SCHC profile for Sigfox. |

---

## 3. IP fit & usage — where IP belongs and how devices are used

The analysis layer: given the dimensions and the technologies, where is IP carried to the device, where is it adapted, and where is it terminated — and how are devices actually used? This is where iotdata's argument lives; *Positioning & differences* draws on it.

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 8376 §4** | LPWAN Overview — Gap Analysis | [POSITION] | The IETF's own catalogue of why IPv6 machinery doesn't fit LPWAN: header size, MTU, fragmentation, neighbour discovery, DNS. The strongest evidence for the bottom-up argument. |
| **7228bis I-classes** | Classes of Internet Integration (I0–I9) | [VOCAB] | Integration level as a dimension; **I2** (device-to-cloud via an edge gateway, mediated) is iotdata's design point. |
| **RFC 7452** | Architectural Considerations in Smart Object Networking (IAB) | [VOCAB] | Technically thin, but it is the IAB's on-record position — generic IP protocols, avoid application-layer gateways and silos — so it is the argument you will face. Its four communication patterns are a crude usage model. |
| **RFC 8477** | Report from the IoT Semantic Interoperability (IOTSI) Workshop 2016 (IAB) | [VOCAB] | Separates information, data and interaction models, and catalogues why cross-ecosystem semantic interop fails. Use its terms when describing what the variant table is. |
| **RFC 9171** | Bundle Protocol Version 7 (DTN) | [POSITION] | Precedent: the IETF standardised a non-IP-native architecture where IP's assumptions break (challenged networks, as RFC 7228 defines them). |
| **Thread + Matter** | (see §2.3) | [POSITION] | The honest counter-example: IP to the device working at scale in the home. Explain the dimensional difference rather than dismissing it. |

**Usage models** *(proposed dimensions)* — how a device is actually used, which matters as much as what link it is on:

- **Initiative:** push (device-initiated uplink); pull (polled); subscribe/observe.
- **Cadence:** periodic telemetry; event or alarm; on demand.
- **Downlink:** none; command/configuration; bulk (firmware).
- **Relay:** direct to gateway; multi-hop mesh; store-and-forward.
- **State:** stateless datagrams; sessions and per-peer state (CoAP exchanges, DTLS).
- **Integration point:** device; edge gateway; cloud (cf. the I-classes and RFC 7452's patterns).

---

# Part II — Where iotdata sits

iotdata spans two layers: a payload encoding (with its field registry / data model) and a lightweight transmission and meshing layer. The work it will be measured against spans both, so read §4–6 together. They are split by the problem each solves, from the top of the stack down: what goes in the payload (§4), how it is compressed and chunked over a constrained link (§5), and how it is carried, discovered and routed (§6). Expect "why not CBOR/SenML + SCHC + CoAP?" — that question cuts across all three. Security (§7) and firmware (§8) are distinct elements you adopt rather than compete on.

## 4. Payload encoding & data models — what iotdata competes with on the wire (study hardest)

SenML is the apples-to-apples format to beat; CayenneLPP is listed only because the room will raise it. CBOR and CDDL are the substrate reviewers will benchmark the encoding and any schema language against; aligning iotdata's data model with CBOR's where possible lets CDDL/COSE/SUIT sit on top of it. SDF is the IETF's protocol-independent way to describe the *meaning* of fields, separate from any encoding.

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 8428** | Sensor Measurement Lists (SenML) | [POSITION] | The IETF's *own* lightweight sensor-telemetry format (JSON/CBOR/XML/EXI). This — not CBOR — is the apples-to-apples format to beat. Your own spec already shows the 6–7× size win; lead with that comparison. |
| **CayenneLPP** | (not an IETF document; myDevices/TTN de-facto LoRaWAN payload format) | [POSITION] | Not a technical peer: a vendor's minimal type-tagged encoding that loses semantics to generic types such as "analog input". Its only relevance is that LoRaWAN deployments fall back on it — evidence that no IETF format serves this space. Expect it to come up; don't benchmark against it. |
| **RFC 8949** | Concise Binary Object Representation (CBOR) — STD 94 | [POSITION] | The format you're benchmarked against. A full Internet Standard. **Read §4.2 (Deterministic Encoding) closely** — anything COSE signs or SUIT verifies needs one canonical byte form; iotdata must define a deterministic encoding or signing/verification won't compose. |
| **RFC 8610** | Concise Data Definition Language (CDDL) | [POSITION] / [ADOPT] | The IETF schema language for CBOR/JSON data. If iotdata has a schema, expect to provide a CDDL mapping; you can also *use* CDDL to specify your own structures. |
| **RFC 9165** | Additional Control Operators for CDDL | [ADOPT] | Extensions (`.plus`, `.feature`, etc.) useful if you express iotdata structures in CDDL. |
| **RFC 8742** | CBOR Sequences | [ADOPT] | Concatenated CBOR items without an enclosing array — relevant if you frame multi-record telemetry. |
| **draft-ietf-cbor-packed** | Packed CBOR | [POSITION] | Compresses CBOR by factoring repeated items into a shared dictionary/table — conceptually adjacent to your registry/variant-table idea. Know it: a reviewer may ask "why not Packed CBOR + a profile?" |
| **RFC 9880** | Semantic Definition Format (SDF) for Data and Interactions of Things | [POSITION] / [ADOPT] | Standards Track (Jan 2026), ASDF WG. A JSON language for Thing models — properties, actions, events, units (SenML units), ranges — with wire formats left to protocol bindings. Expect "can your field registry be expressed as SDF?"; describing the variant table as an SDF model with an iotdata binding is a cheap, credible answer. |

## 5. Compression & fragmentation over LPWAN — your nearest neighbour and chunked delivery (study hardest)

SCHC is your nearest neighbour in the IETF. It solves a *different* problem (generic IPv6/UDP header compression + fragmentation for IP-over-LPWAN) but with the *same core idea* — a static context shared by both ends, so the "schema" lives off the wire. You must be able to say crisply how iotdata differs (application-layer, semantic, domain-typed, quantised sensor fields vs SCHC's rule-based generic field compression) **and** where you'd reuse it (its fragmentation/reassembly is a ready-made, standardised version of your firmware chunk-creep). CoAP block-wise transfer is the other reference model for chunked, resumable delivery, so it sits here too.

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 8724** | SCHC: Generic Framework for Static Context Header Compression and Fragmentation | [POSITION] / [ADOPT] | The framework. Compression via shared static context (≈ your variant table) + optional fragmentation/reassembly (≈ your chunk transport). The most important single comparison for your BoF. |
| **RFC 9011** | SCHC over LoRaWAN | [POSITION] / [ADOPT] | The LoRaWAN profile: parameterisation, the No-ACK / ACK-Always / ACK-on-Error fragmentation modes, Reassembly Check Sequence. This is the standardised answer to your selective-repeat firmware creep — adopt or consciously diverge. |
| **RFC 9441** | Compound ACK for SCHC | [ADOPT] | Bitmap-style acknowledgement of multiple fragment windows in one message — essentially your "missing-chunk bitmap" NACK, already specified. |
| **RFC 8824** | Application of SCHC for the Constrained Application Protocol (CoAP) | [POSITION] | Shows SCHC compressing an application protocol's headers — the conceptual closest to iotdata applying static context at the application layer. |
| **RFC 6282** | Compression Format for IPv6 over 6LoWPAN | [POSITION] | The 6LoWPAN header-compression predecessor to SCHC — useful lineage for the "compress against shared context" idea. |
| **RFC 7959** | Block-Wise Transfers in CoAP | [POSITION] / [ADOPT] | Receiver-driven, numbered, resumable block transfer — the reference model for your chunked delivery (same shape as your firmware creep). |
| **RFC 9177** | CoAP Block-Wise Transfer Options Supporting Robust Transmission | [ADOPT] | Q-Block options for lossy links — more loss-tolerant block transfer, relevant to LoRa. |

## 6. Transport, discovery & mesh — context for your second layer

Your lightweight-transmission/meshing layer is the more novel, more IETF-palatable half of iotdata. These define the surrounding IETF constrained-networking world it would relate to or compete within; the non-IETF meshes it will also be compared to (LoRaWAN Relay, Meshtastic, Thread, Zigbee, BLE Mesh, DECT NR+) are in §2. You don't need to adopt CoAP, but you must know where it sits, because "why not CoAP + SCHC?" is a question you'll be asked.

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 7252** | The Constrained Application Protocol (CoAP) | [POSITION] | The constrained REST protocol — the default "how IoT devices talk" the IETF will compare your transport to. |
| **RFC 7641** | Observing Resources in CoAP | [POSITION] | Server-push/notification over CoAP — comparison point for your "firmware available" advertisement. |
| **RFC 8323** | CoAP over TCP, TLS, and WebSockets | [VOCAB] | For completeness on CoAP transports. |
| **RFC 8132** | PATCH and FETCH Methods for CoAP | [VOCAB] | Partial updates over CoAP — conceptually near delta/partial transfer. |
| **RFC 6690** | Constrained RESTful Environments (CoRE) Link Format | [POSITION] | Resource discovery vocabulary; comparison for how devices advertise capabilities. |
| **RFC 9176** | CoRE Resource Directory | [POSITION] | Registry of constrained resources — comparison for any directory/registry role in your system. |
| **RFC 4944** | Transmission of IPv6 Packets over IEEE 802.15.4 (6LoWPAN) | [VOCAB] | Foundational adaptation-layer thinking for IP over constrained links. |
| **RFC 6775** | Neighbor Discovery Optimization for 6LoWPAN | [VOCAB] | Constrained-network ND; mesh-formation context. |
| **RFC 8505** | Registration Extensions for 6LoWPAN Neighbor Discovery | [VOCAB] | Updated registration model; relevant if your mesh does node registration. |
| **RFC 6550** | RPL: IPv6 Routing Protocol for Low-Power and Lossy Networks | [POSITION] | The IETF mesh-routing standard — the comparison point for your meshing layer. Bring a clear "how iotdata's mesh differs from RPL" answer. |
| **RFC 6551** | Routing Metrics Used for Path Calculation in LLNs | [VOCAB] | The metric vocabulary RPL uses; useful if your mesh makes routing/forwarding decisions. |

(For later RPL revisions and ongoing mesh work, watch the **ROLL** WG datatracker page rather than pinning specific draft numbers. RFC 7102 in §1 is the ROLL glossary.)

---

## 7. Security envelope — adopt, do not invent

Telemetry can delegate integrity to the medium (your principle #7), but firmware and config cannot — they need real cryptographic integrity/authenticity. This is the IETF constrained-crypto stack; SUIT itself is built on it.

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 9052** | CBOR Object Signing and Encryption (COSE): Structures and Process | [ADOPT] | The signing/encryption envelope. This is the layer your firmware TLV needs and the layer SUIT manifests use. Replaces the obsoleted RFC 8152. |
| **RFC 9053** | COSE: Initial Algorithms | [ADOPT] | The algorithm registrations that go with 9052. |
| **RFC 9459** | COSE: AES-CTR and AES-CBC | [ADOPT] | Additional symmetric algorithms, referenced by SUIT firmware-encryption. |
| **RFC 8392** | CBOR Web Token (CWT) | [ADOPT] | Compact claims/tokens — relevant for device identity in your campaign metadata. |
| **RFC 8613** | Object Security for Constrained RESTful Environments (OSCORE) | [ADOPT] | Application-layer object security over CoAP, end-to-end through proxies — the constrained alternative to DTLS. |
| **RFC 9528** | Ephemeral Diffie-Hellman Over COSE (EDHOC) | [ADOPT] | Lightweight authenticated key exchange that establishes an OSCORE context — very small over the wire, designed for LPWAN. Pairs with OSCORE. |
| **RFC 9200** | Authentication and Authorization for Constrained Environments (ACE) using OAuth 2.0 | [ADOPT] | The constrained authorisation framework, if iotdata ever needs an authz model for who may push firmware/config. |

---

## 8. Firmware update — adopt

This is the part you said you'd adopt outright. 9019 and 9124 are stable; the manifest and its extensions are active drafts — **check the SUIT WG datatracker page for current revisions** before relying on them.

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 8240** | Report from the IoT Software Update (IOTSU) Workshop 2016 | [VOCAB] | The problem-statement origin of the SUIT effort: the requirements and open issues (signing, delta updates, end-of-support, authorisation) behind everything below. Its device-capability side lives on in 7228bis's F-classes (§1). |
| **RFC 9019** | A Firmware Update Architecture for Internet of Things | [ADOPT] | The reference architecture: author/distributor/device roles, manifest+image flow, A/B slots, status tracking. Stable. |
| **RFC 9124** | A Manifest Information Model for Firmware Updates in IoT Devices | [ADOPT] | The encoding-agnostic information model — *what* metadata an update must carry. **This is your bridge:** serialise this model in iotdata. Stable. |
| **draft-ietf-suit-manifest** | CBOR-based Serialization Format for the SUIT Manifest | [ADOPT] / [POSITION] | The CBOR wire format of the manifest. Read it as the reference design; iotdata becomes a parallel serialisation of the same model. (Still a draft in late 2025 — verify current state.) |
| **draft-ietf-suit-update-management** | Update Management Extensions for SUIT Manifests | [ADOPT] | Directly relevant to your campaign layer — staging, scheduling, conditions on when/whether to apply. |
| **draft-ietf-suit-firmware-encryption** | Encrypted Payloads in SUIT Manifests | [ADOPT] | If you ever need confidential firmware (AES-KW / ES-DH content-key distribution). |
| **draft-ietf-suit-trust-domains** | SUIT Manifest Extensions for Multiple Trust Domains | [ADOPT] | Dependency/delegation model when more than one authority signs — relevant if vendors/operators have split authority. |
| **draft-ietf-suit-mti** | Mandatory-to-Implement Algorithms for SUIT | [ADOPT] | The crypto-suite baseline for interop — tells you which algorithms to implement. |
| **RFC 4108** | Using CMS to Protect Firmware Packages | [VOCAB] | The pre-SUIT firmware-signing approach SUIT learned from. Historical context, occasionally cited. |
| **RFC 8520** | Manufacturer Usage Description (MUD) | [ADOPT] | Device-capability/behaviour description; SUIT can carry a MUD file. Optional, but in the SUIT WG's scope. |

Note: **LwM2M** (the OMA firmware-update object / state machine discussed earlier) is *not* an IETF document — it's Open Mobile Alliance, layered on CoAP. Use it as a reference for the device-side update state machine, but cite it as OMA, not as an RFC.

---

## Bodies and groups to watch (live status > any number above)

**IETF working groups**

- **schc** — SCHC, fragmentation, LPWAN profiles (successor to the concluded **lpwan** WG). *Your nearest neighbour.*
- **suit** — firmware manifest + extensions. *Your adopt-from for firmware.*
- **cbor** — CBOR, CDDL, Packed CBOR. *Your encoding substrate.*
- **asdf** — SDF semantic data models. *Where a model of your field registry would live.*
- **cose** — signing/encryption. *Your security envelope.*
- **core** — CoAP, block-wise, resource directory. *Your transport ecosystem.*
- **6lo** / **roll** — adaptation and mesh routing. *Your meshing context.*
- **iotops** — constrained-IoT operations, 7228bis. *Your terminology baseline, and the natural home for a 7228++.*
- **ace** — constrained authorisation.

**Outside the IETF**

- **LoRa Alliance** — LoRaWAN, regional parameters, relay. *Your primary link.*
- **IEEE 802.15 / 802.11** — 802.15.4 PHY/MAC; 802.11ah (HaLow).
- **3GPP** — NB-IoT, LTE-M, RedCap/eRedCap.
- **ETSI** — DECT NR+, mioty (Low Throughput Networks).
- **Connectivity Standards Alliance** — Zigbee, Matter.
- **Thread Group**, **Bluetooth SIG**, **Wi-SUN Alliance**, **Wi-Fi Alliance**, **Z-Wave Alliance**, **mioty Alliance**.
- **OMA SpecWorks** — LwM2M.

---

## Positioning & differences — the case in one place

Everything above that bears on "how is iotdata different, and why should the IETF care?", gathered into one argument. The individual entries stay where they are; this is what they add up to.

### The thesis: designed bottom-up, not fitted top-down

The IETF's constrained-device work largely adapts IPv6 and the web stack down to environments they were not designed for: 6LoWPAN and SCHC compress IPv6/UDP headers, CoAP is REST for constrained nodes, and CBOR and SenML carry JSON-era data models in binary. The protocols remain complex and inside the IPv6 worldview. iotdata starts from the link budget — bits, airtime, duty cycle, energy — and works upward, meeting IP at an edge gateway.

The IETF's own documents make much of this case:

- **RFC 8376 gap analysis:** a 40-byte IPv6 header against LPWAN payloads of ~10–100 bytes; a 1280-byte minimum MTU; 6LoWPAN header compression at 11–19 bytes in practice, which can exceed the whole payload; neighbour discovery taking hours or days at LPWAN message rates.
- **7228bis PHY bit-rate classes:** header compression is "indispensable" at B0 (< 10 bit/s) and "vital" at B1 (10–1000 bit/s). LoRa spans B1–B2.
- **RFC 8352:** the radio dominates energy, so bytes on air translate directly into battery life.

The deployed world also argues for heterogeneity rather than homogeneity:

- **Non-IP links have strong adoption** in both local and wide area: Zigbee, BLE (including BLE Mesh) and Z-Wave in homes and buildings; LoRaWAN and Sigfox outdoors; cellular IoT offers non-IP data delivery alongside IP.
- **Even IP-native ecosystems bridge.** Thread and Matter carry IPv6 to the device, yet Matter defines bridges to expose Zigbee and Z-Wave devices rather than requiring them to be replaced.
- **The IETF's own response has been adaptation.** 6lo defines IPv6 per link (BLE, DECT ULE, ITU-T G.9959) and SCHC defines a profile per LPWAN (LoRaWAN, Sigfox, NB-IoT) — interfacing to heterogeneity, link by link.

The realistic architecture, then, is many links with well-defined interfaces and adaptation at the boundaries. iotdata is one such interface: designed for the constrained side, meeting IP at the gateway.

How to make the case without losing the room:

- **Observe, don't rebut.** Don't argue that "IPv6 everywhere" is wrong. Note that the deployed world hasn't converged on it, that the IETF's own 6lo and SCHC work is adaptation to that fact, and offer iotdata as a well-defined interface at that boundary. Acknowledge Thread/Matter's success in the home as part of the same picture.

- **Present a design point, not a correction.** The IETF's identity is IP interoperability (RFC 3935), and RFC 7452 argues for generic protocols and against application-layer gateways and vertical stacks. "You fitted IPv6 from the top down" can land as "you got it wrong"; "this is a different point in the design space, with these trade-offs" does not.
- **Claim a recognised class.** In 7228bis terms, iotdata targets microcontroller-class devices (sensors near C1; ESP32-C3 = C4) on S1–S2 MTU / B1–B2 bit-rate links, with **I2** Internet integration: device-to-cloud via an edge gateway, mediated access. "Not IP to the device" is a named class, not heresy.
- **Arrive with the whole map.** Part I's dimensions and technology survey take in the IETF/IAB work without being centred on it. Arguing from that matrix shows where IP to the device works (Thread, Wi-SUN, HaLow — B2+ links, mains-powered routers) and where it strains (B1 LPWAN, duty-cycled, battery-only), so the bottom-up case comes out as a finding rather than a pitch.
- **Cite precedent.** The IETF standardised the DTN Bundle Protocol (RFC 9171) because IP's assumptions break on challenged networks, a category RFC 7228 itself defines.
- **Attack complexity, not header bytes.** SCHC can compress IPv6/UDP down to a rule ID plus a small residue, so a byte-count argument at the header layer will not survive scrutiny. The stronger ground is (a) the layered complexity carried down to the device — IP semantics, rule contexts, CoAP state, multi-layer tuning — and (b) the payload layer, where SenML is verbose by design.

### Against each neighbour

| Neighbour | What it is | How iotdata differs | Relationship |
| --- | --- | --- | --- |
| **SenML** (RFC 8428) | The IETF's lightweight sensor-telemetry format (JSON/CBOR/XML/EXI) | Sub-byte packing, presence flags, domain-typed quantisation; 6–7× smaller in iotdata's own comparison | The apples-to-apples comparison — lead with it |
| **SCHC** (RFC 8724, 9011) | Generic, rule-based IPv6/UDP header compression and fragmentation; static context shared by both ends | Same core idea (schema off the wire), applied at the application layer to semantic, domain-typed, quantised sensor fields rather than generic header fields | Nearest neighbour. Reuse its fragmentation modes (9011) and Compound ACK (9441) for firmware chunk delivery, or diverge consciously |
| **CBOR** (RFC 8949) | General-purpose binary encoding; Internet Standard | Domain-optimised, not general-purpose — never pitch "a better CBOR" | Align the data model where possible so CDDL/COSE/SUIT sit on top; define a deterministic encoding so signing composes |
| **Packed CBOR** (draft) | CBOR with repeated items factored into a shared table | Adjacent to the registry/variant-table idea | Needs a crisp answer to "why not Packed CBOR + a profile?" |
| **CDDL** (RFC 8610) / **SDF** (RFC 9880) | Schema language / protocol-independent semantic model language | iotdata is a wire encoding; these describe structure and meaning | Provide a CDDL mapping; describe the registry as an SDF model with an iotdata binding |
| **CoAP** (RFC 7252, 7959, 7641) | Constrained REST, block-wise transfer, Observe | To articulate: "why not CoAP + SCHC?" | Block-wise is the reference model for chunked delivery; Observe is the comparison for "firmware available" |
| **RPL** (RFC 6550) | The IETF mesh-routing protocol for LLNs | To articulate: how iotdata's mesh differs | The comparison point for the meshing layer |
| **LoRaWAN Relay** / **Meshtastic** | The LoRa Alliance's standard relay; the de facto community LoRa mesh | To articulate: how iotdata's LoRa mesh differs from both | The comparisons for multi-hop on LoRa specifically |
| **Thread + Matter** | IPv6 mesh to the device plus an application data model, at home scale | Different dimensions: B2 links, mains-powered routers, home range vs B1, duty-cycled, wide-area | The counter-example to explain, not dismiss |
| **SUIT** (RFC 9019, 9124, manifest draft) | Firmware update architecture, encoding-agnostic information model, CBOR manifest | iotdata serialises the same information model | Complementary: "an iotdata serialisation of the SUIT information model" |
| **COSE** (RFC 9052) | Signing and encryption envelope | Not a competitor | Adopt for firmware and config; telemetry delegates integrity to the medium (principle #7) |
| **CayenneLPP** | A vendor's minimal type-tagged LoRaWAN payload encoding; not a standard | Not a technical peer; loses semantics to generic types such as "analog input" | Relevant only as what deployments use today — evidence that no IETF format serves this space |
| **LwM2M** (OMA) | OMA device-management and firmware-update objects over CoAP | Not IETF work | Reference for the device-side update state machine; cite as OMA |

### Questions you will be asked

| Question | Answer, or where it comes from |
| --- | --- |
| Why not CBOR/SenML + SCHC + a profile? | *The* BoF question: the thesis above, the SenML size comparison, and the SCHC difference |
| Why not Packed CBOR + a profile? | §4 — needs a crisp answer |
| Why not CoAP + SCHC? | §6 — needs a crisp answer |
| How does your mesh differ from RPL — and from LoRaWAN Relay and Meshtastic? | §6 and §2 — needs a crisp answer |
| Thread and Matter run IPv6 to the device — why not here? | §2.3 and §3: the dimensional difference (bit rate, duty cycle, power, range) |
| Can the field registry be expressed in CDDL or SDF? | §4 — yes; offer both |
| How is it signed and verified? | A deterministic encoding (cf. CBOR §4.2) plus COSE (§7) |
| Why a gateway rather than end-to-end IP? | I2 as the design target; RFC 7452's gateway concerns are the counterpoint to address |
| Why no interoperability (principle #8)? | The hard sell under RFC 3935 — needs an answer |

### Trade-offs to state openly

- No end-to-end IP to the device and no generic gateway (RFC 7452's concern), in exchange for bits on air, airtime and simplicity.
- A shared static registry that both ends must hold and version — the same burden SCHC's rule contexts carry.
- Telemetry integrity delegated to the link (principle #7); firmware and config carry COSE.
- No interoperability as a principle (#8), which is a hard sell as-is (RFC 3935).

---

## One-paragraph BoF positioning draft (to refine as you read)

> iotdata is an application-layer telemetry encoding for Class 1–2 devices on LPWANs,
> using a shared static field registry (variant table) to achieve sub-byte,
> presence-gated, quantised encoding of sensor data — typically several times smaller
> than SenML while preserving domain semantics that generic formats lose.
> Where the IETF constrained stack adapts IPv6 down to these links, iotdata is
> designed up from the link budget for a world of heterogeneous links, and meets IP
> at an edge gateway. It
> is complementary to that stack: it carries firmware/lifecycle data
> as a serialisation of the SUIT (RFC 9124) information model, signs it with COSE, and
> can reuse SCHC-style fragmentation for over-the-air delivery. It is positioned not
> as a CBOR replacement but as a domain-optimised peer to SCHC and SenML for the
> bandwidth- and energy-constrained sensor-telemetry case.

*Tighten this until it survives the "why not CBOR/SCHC/SenML + a profile?" question — that question is the BoF.*
