# IETF Reading Map for iotdata — toward a BoF-ready rearchitecture

A curated, structured set of RFCs and Internet-Drafts to align iotdata with IETF
terminology and standards, organised by intent. Each entry is tagged:

- **[ADOPT]** — existing, mature work you can use directly rather than reinvent.
- **[POSITION]** — work iotdata overlaps with or is an alternative to; you must be
  able to articulate the difference in a BoF.
- **[VOCAB]** — read primarily for the terminology and framing the IETF expects you
  to speak fluently.
- **[PROCESS]** — how the IETF and the BoF→WG path actually work.

## How to use this

1. **Status changes.** Drafts (`draft-ietf-…`) expire every 6 months and revise
   constantly. Numbers here reflect their state in late 2025, except
   draft-ietf-iotops-7228bis, which was re-checked in November 2026; before citing
   any draft, pull the current revision from its working-group page on
   `datatracker.ietf.org`. RFCs (numbered, no `draft-` prefix) are stable.
2. **Two framings that matter for the BoF** (carried over from the design
   discussion):
   - Do **not** pitch iotdata as "a better CBOR." Pitch it in the
     constrained-telemetry / LPWAN space — i.e. against **SCHC, SenML, and
     CayenneLPP** — where the comparison is apples-to-apples and the differentiation
     (sub-byte packing + presence flags + domain-typed quantisation) is real.
   - The **SUIT information model (RFC 9124) is encoding-agnostic.** The cleanest
     firmware story — technically and politically — is "an iotdata serialisation of
     the SUIT information model," which is *complementary* to CBOR/SUIT rather than
     competing.
3. **Critical path** (read these before writing a BoF proposal): RFC 5434 → RFC 7228
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

## 1. Foundational terminology & landscape — the vocabulary

Speak this fluently or the room will tune out. The first four are core; the rest are supporting context that overlaps the same ground from other angles (architecture, semantics, energy, mesh).

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 7228** | Terminology for Constrained-Node Networks | [VOCAB] | Defines Class 0/1/2 devices and constrained-network vocabulary that *every* doc below assumes. Your sensors edge toward Class 1. Your ESP32-C3 (~400 KiB SRAM) is beyond 7228's Class 2 — it is **C4** in 7228bis's table (beware the name clash with 7228bis's own class "C3"). Use these terms in your spec. |
| **draft-ietf-iotops-7228bis** | Terminology for Constrained-Node Networks (revision) | [VOCAB] | The update to 7228 in the IOTOPS WG; at -10 (Jul 2026) and with the IESG, so soon the reference. Keeps 7228's classes and adds device classes up to C19, firmware upgradability (F0–F9), isolation, secret shielding, timekeeping, and network classes: link MTU (S0–S19), Internet integration (I0–I9), PHY bit rate (B0–B3). Describe iotdata's target devices and links in these classes. |
| **RFC 8376** | Low-Power Wide Area Network (LPWAN) Overview | [VOCAB] | The scene-setter for your exact deployment class — LoRa/NB-IoT/Sigfox characteristics, duty cycle, star topology, asymmetry. Frame iotdata's motivation in these terms. |
| **RFC 8240** | Report from the IoT Software Update (IOTSU) Workshop 2016 | [VOCAB] | The problem-statement origin of the SUIT effort. Useful background for your firmware-lifecycle framing. |
| **RFC 7452** | Architectural Considerations in Smart Object Networking (IAB) | [VOCAB] | The four IoT communication patterns (device-to-device, -cloud, -gateway, back-end data sharing) and the case for generic protocols over vertical stacks. iotdata's gateway-decode model is the device-to-gateway pattern; expect the room to weigh it against this document's warnings about application-layer gateways. |
| **RFC 8477** | Report from the IoT Semantic Interoperability (IOTSI) Workshop 2016 (IAB) | [VOCAB] | Separates information, data and interaction models, and catalogues why cross-ecosystem semantic interop fails. The companion to RFC 8240 for iotdata's "domain-typed semantics" claim; use its terms when describing what the variant table is. |
| **RFC 8352** | Energy-Efficient Features of Internet of Things Protocols | [VOCAB] | Power-saving mechanisms layer by layer (duty cycling, sleepy-node support, header compression) with per-operation energy figures. Cite it when arguing bytes-on-air → battery life. Covers 802.15.4/BLE/DECT ULE, not LPWANs (that's RFC 8376). |
| **RFC 7102** | Terms Used in Routing for Low-Power and Lossy Networks | [VOCAB] | The ROLL glossary: LLN, LBR, sleepy/non-sleepy node, MP2P/P2MP/P2P, upstream/downstream. 7228 borrows from it; you'll need it to describe the mesh layer against RPL (§2.3). |

---

## 2. Payload encoding and transport — the space iotdata occupies (study hardest)

iotdata spans two layers: a payload encoding (with its field registry / data model) and a lightweight transmission and meshing layer. The work it will be measured against spans both, so read these three subsections together. They are split by the problem each solves, from the top of the stack down: what goes in the payload (2.1), how it is compressed and chunked over a constrained link (2.2), and how it is carried, discovered and routed (2.3). Expect "why not CBOR/SenML + SCHC + CoAP?" — that question cuts across all three.

### 2.1 Payload encoding & data models — what iotdata competes with on the wire

SenML and CayenneLPP are the apples-to-apples formats to beat. CBOR and CDDL are the substrate reviewers will benchmark the encoding and any schema language against; aligning iotdata's data model with CBOR's where possible lets CDDL/COSE/SUIT sit on top of it. SDF is the IETF's protocol-independent way to describe the *meaning* of fields, separate from any encoding.

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
| **RFC 8428** | Sensor Measurement Lists (SenML) | [POSITION] | The IETF's *own* lightweight sensor-telemetry format (JSON/CBOR/XML/EXI). This — not CBOR — is the apples-to-apples format to beat. Your own spec already shows the 6–7× size win; lead with that comparison. |
| **CayenneLPP** | (not an IETF document; myDevices/TTN de-facto LoRaWAN payload format) | [POSITION] | The incumbent you'd actually displace in LoRaWAN deployments. Not standardised, but the room will know it. Your edge: you keep the semantics CayenneLPP loses to its generic "analog input" type. |
| **RFC 8949** | Concise Binary Object Representation (CBOR) — STD 94 | [POSITION] | The format you're benchmarked against. A full Internet Standard. **Read §4.2 (Deterministic Encoding) closely** — anything COSE signs or SUIT verifies needs one canonical byte form; iotdata must define a deterministic encoding or signing/verification won't compose. |
| **RFC 8610** | Concise Data Definition Language (CDDL) | [POSITION] / [ADOPT] | The IETF schema language for CBOR/JSON data. If iotdata has a schema, expect to provide a CDDL mapping; you can also *use* CDDL to specify your own structures. |
| **RFC 9165** | Additional Control Operators for CDDL | [ADOPT] | Extensions (`.plus`, `.feature`, etc.) useful if you express iotdata structures in CDDL. |
| **RFC 8742** | CBOR Sequences | [ADOPT] | Concatenated CBOR items without an enclosing array — relevant if you frame multi-record telemetry. |
| **draft-ietf-cbor-packed** | Packed CBOR | [POSITION] | Compresses CBOR by factoring repeated items into a shared dictionary/table — conceptually adjacent to your registry/variant-table idea. Know it: a reviewer may ask "why not Packed CBOR + a profile?" |
| **RFC 9880** | Semantic Definition Format (SDF) for Data and Interactions of Things | [POSITION] / [ADOPT] | Standards Track (Jan 2026), ASDF WG. A JSON language for Thing models — properties, actions, events, units (SenML units), ranges — with wire formats left to protocol bindings. Expect "can your field registry be expressed as SDF?"; describing the variant table as an SDF model with an iotdata binding is a cheap, credible answer. |

### 2.2 Compression & fragmentation over LPWAN — your nearest neighbour and chunked delivery

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

### 2.3 Transport, discovery & mesh — context for your second layer

Your lightweight-transmission/meshing layer is the more novel, more IETF-palatable half of iotdata. These define the surrounding constrained-networking world it would relate to or compete within. You don't need to adopt CoAP, but you must know where it sits, because "why not CoAP + SCHC?" is a question you'll be asked.

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

## 3. Security envelope — adopt, do not invent

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

## 4. Firmware update — adopt

This is the part you said you'd adopt outright. 9019 and 9124 are stable; the manifest and its extensions are active drafts — **check the SUIT WG datatracker page for current revisions** before relying on them.

| Doc | Title | Tag | Why |
| --- | --- | --- | --- |
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

## Working-group pages to watch (live status > any number above)

- **schc** — SCHC, fragmentation, LPWAN profiles (successor to the concluded **lpwan** WG). *Your nearest neighbour.*
- **suit** — firmware manifest + extensions. *Your adopt-from for firmware.*
- **cbor** — CBOR, CDDL, Packed CBOR. *Your encoding substrate.*
- **asdf** — SDF semantic data models. *Where a model of your field registry would live.*
- **cose** — signing/encryption. *Your security envelope.*
- **core** — CoAP, block-wise, resource directory. *Your transport ecosystem.*
- **6lo** / **roll** — adaptation and mesh routing. *Your meshing context.*
- **iotops** — constrained-IoT operations, 7228bis. *Your terminology baseline.*
- **ace** — constrained authorisation.

---

## One-paragraph BoF positioning draft (to refine as you read)

> iotdata is an application-layer telemetry encoding for Class 1–2 devices on LPWANs,
> using a shared static field registry (variant table) to achieve sub-byte,
> presence-gated, quantised encoding of sensor data — typically several times smaller
> than SenML and CayenneLPP while preserving domain semantics those formats lose. It
> is complementary to the IETF constrained stack: it carries firmware/lifecycle data
> as a serialisation of the SUIT (RFC 9124) information model, signs it with COSE, and
> can reuse SCHC-style fragmentation for over-the-air delivery. It is positioned not
> as a CBOR replacement but as a domain-optimised peer to SCHC and SenML for the
> bandwidth- and energy-constrained sensor-telemetry case.

*Tighten this until it survives the "why not CBOR/SCHC/SenML + a profile?" question — that question is the BoF.*
