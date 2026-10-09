# NOTES — Directionality, Authority, and the Downstream Path

> Working note, 2026-09-13. Captures a design rationale that was arrived at by reasoning
> through the running code, was NOT fully contemplated when the mesh protocol was drafted,
> and is presently absent from the specification. To be folded into §5, Appendix G and
> Appendix H when the specs are next revisited. See "What has to change" at the end.

---

## 1. The premise: iotdata is a one-directional protocol

The primary use case is **one-way telemetry from highly constrained devices** — 8-bit MCUs
upward — over battery- and airtime-constrained links. Everything in the header follows from
that, and most of the apparent asymmetries elsewhere are consequences of it rather than
oversights.

The protocol is **fire and forget**. There is no acknowledgement, no session, no
retransmission and no flow control at the protocol level. A packet is emitted and the
transmitter's obligation ends.

**`(station, sequence)` is the authority.** The transmitting station names its own packets;
the sequence counter is what gives receivers deduplication and loss determination. The
station is the *author* of its own identity space, and in the base protocol it is the only
authority that exists.

## 2. The header has a receiver, not a sender/receiver pair

The 32-bit header is `variant(4) | station(12) | sequence(16)`. There is deliberately **no
addressing pair**:

- Upstream, `station` is the **origin** — the author of the packet.
- Downstream, `station` is the **target** — the intended recipient.

The field changes meaning with direction, and the direction is inferred from the sequence
sentinel (§3). This is not sloppiness. A sender/receiver pair would cost a further 12 bits on
every telemetry packet in order to serve a case that is a small minority of traffic, on
devices where the header is already a significant fraction of a short frame.

The consequence to hold onto: **the protocol can say who a frame is for, or who it is from,
but never both.**

## 3. No direction bit — a sentinel sequence instead

There is no up/down bit in the header, and there should not be one. One bit of every header,
forever, spent on a distinction that is irrelevant to ~95% of traffic, is the wrong trade for
this protocol's target deployments.

Instead, `sequence == 65535` (`IOTDATA_SEQUENCE_DOWN`) marks a downward frame, and assignable
sensor sequences cap at 65534 (`IOTDATA_SEQUENCE_ASSIGNABLE_MAX`). One value of a 16-bit space
encodes the direction; nothing else is spent.

**Specification gap.** §5 currently states only that the sequence is "monotonically
increasing… wrapping from 65535 to 0", with no reservation noted. The reference
implementation has reserved 65535 since the down path existed. The spec is wrong as written
and must be corrected.

## 4. Why downstream cannot borrow upstream's identity model

This is the substance of the note, and the thing that was not contemplated.

**Upstream, identity is an EVENT** — "this transmission, by this station". An event needs an
authority to name it, and the sensor is the natural and sole authority: it authors its own
sequence, and nobody else can author into that space. `(origin, sequence)` is therefore a
sound global key.

**Downstream, identity is a VALUE** — an instruction. And there is no author authority
available:

- Any gateway may originate a down frame for any target. Two gateways choosing sequence
  numbers independently would collide.
- The frame carries **no originator field** with which to disambiguate them (§2 — the header
  has a receiver, not a pair). Namespacing the sequence by originator is not possible without
  spending header bits the protocol has declined to spend.

More importantly, **an authority is not wanted here.** Two gateways issuing the same
instruction to the same target are issuing *the same instruction*, and it should be delivered
once. A sequence-based key would propagate both. A content-based key correctly collapses
them.

**Therefore downstream deduplication is content-oriented: the key is `(target station,
content)` — sentinel plus content.** This is content-addressing, and it is chosen for the
same reason content-addressing is chosen anywhere: the thing being identified is a value, not
an event, and values are their own names.

A useful property falls out of it: **content keys converge without coordination.** Two nodes
that have agreed on nothing, hanging off different gateways, independently derive the same
key for the same instruction. Downstream dedup state stops being per-node and becomes a
shared namespace.

## 5. Consequences that follow

**5.1 Add-only. There is no supersede.** A new `(station, content)` is a new entry on the
list, not a replacement for an existing one. "Newer" is not expressible, because ranking two
instructions requires an authority and there isn't one. Note the cost: supersede was
previously a release valve (a target reconfigured repeatedly reused one slot), so add-only
makes **eviction policy load-bearing** rather than a corner case — the same fairness set the
upward path needs (oldest, oldest-for-that-station, per-station cap).

**5.2 Commands MUST be idempotent and state-relative.** "Clear diagnostics up to record N",
not "clear diagnostics". This is a structural requirement, not a style preference: ordering is
not guaranteed, duplicates are expected by construction, and a stale instruction may land
days late. Idempotence is what replaces the acknowledgement this path does not have.

**5.3 Routed up, flooded down — and the asymmetry is forced, not preferred.**

- *Up*: many authors, one destination. A gradient toward a single point is cheap and is
  maintained continuously by beacons for free. Traffic is constant, so the routing state
  earns its keep.
- *Down*: one instruction, many targets, most of them asleep. And the deeper reason —
  **sensors are mesh-unaware.** They never beacon, never announce a parent, never appear in
  any routing state. There is no information in the system from which a downward route could
  be built.

**5.4 The last hop is opportunistic, not routed.** A holder transmits when it *hears* the
target speak. "I just heard it" is the only evidence of a sensor's location the system ever
possesses, and the delivery trigger is exactly that evidence being observed. Better described
as *flood to distribute, opportunistic last hop on evidence* than as plain flooding.

**5.5 Downstream loop-breaking is by dedup memory, not by hop count.** The down frame has no
room for a TTL. What bounds the flood is that an exact `(target, content)` match is ignored
rather than retransmitted, so each node transmits each distinct frame at most once and
propagation is bounded by node count. **This makes dedup memory load-bearing: it must outlive
the flood.** If a node evicts its record and a copy returns from a neighbour, that node treats
it as new and the flood restarts.

**5.6 Which forces the split the upward path already has.** Upstream separates cheap dedup
memory (`fwd_seen` — `(origin, seq, timestamp)`, many entries) from the expensive hold table
(`acktrk` — frame references, few). Downstream currently conflates them: "the slot IS the
dedup state" (`iotdata_node_down.h`). That collapse is elegant and it is sound *only* at one slot
per target. Multi-hold requires the two to separate, or evicting a hold silently erases dedup
memory and 5.5 bites.

**5.7 There is no end-to-end acknowledgement in either direction.** Upstream, the mesh ACK is
hop-local: it means "a better-placed node has taken responsibility", not "it arrived".
Downstream there is none at all, which is why 5.2 is mandatory.

**5.8 Broadcast cannot be held.** No station ever transmits *as* the broadcast id, so no
arrival can ever trigger delivery (5.4). A broadcast down frame is transmitted once and
dropped. This is the known open problem for bulk content distribution, and the reason a
content path needs its own store rather than reusing per-target hold slots.

## 6. The downward direction predates the mesh

Worth separating, because it is easy to conflate. The base protocol already permits a
downward frame: a gateway addressing a node that happens to be in range and awake. That is
the protocol's **asymmetric allowance** — one shot, no holding, no relaying, and it works
without any mesh at all.

The mesh does not create the downward direction. It changes its *character*: from "in range,
one shot" to "flooded, content-addressed, and held for a sleeping target". Everything in §4
and §5 is a property of the mesh-layered case, not of the base protocol.

## 7. What has to change when the specs are revisited

1. **§5 (Header)** — document the reserved sequence value 65535 as the downward sentinel, and
   that assignable sequences therefore cap at 65534. As written the section is incorrect.
2. **§5 (Header)** — state explicitly that `station` denotes the *origin* upstream and the
   *target* downstream, and that the header carries a receiver rather than a sender/receiver
   pair, with the rationale from §2 above.
3. **Appendix G (Mesh Protocol)** — currently has no downstream command mechanism at all. It
   needs the hold-and-deliver model, the content-addressed dedup key, add-only semantics, the
   opportunistic last hop, and the absence of a TTL with dedup memory as the loop bound.
4. **Appendix G** — record that the mesh ACK is hop-local and confers no end-to-end delivery
   guarantee. This is currently easy to misread.
5. **Appendix H (System Architecture)** — the idempotent, state-relative requirement on
   downward commands (5.2), stated as a requirement on deployments and not as advice.
6. **Appendix J (Known Limitations)** — the broadcast-cannot-be-held gap (5.8) and the
   re-flood-after-eviction hazard (5.5).
7. Consider whether the base protocol's one-shot downward allowance (§6) deserves its own
   short section, since it is presently implicit everywhere and stated nowhere.

---

*Origin: reasoning-through session of 2026-09-13, working from `iotdata.h`,
`iotdata_mesh.h`, `iotdata-common/include/iotdata_node_down.h` and the relay's
`relay_forward.h` / `relay_ack.h`.*
