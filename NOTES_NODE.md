# NOTES — The Node Protocol: Management, Control and Observability

> Working note, 2026-09-26. Specifies the NODE protocol: the system TLV types, what each one is
> for, how they are encoded, and the one operating structure they all share. Sits alongside
> [NOTES_DOWNSTREAM](NOTES_DOWNSTREAM.md), which carries anything travelling down, and
> [NOTES_CONTENT](NOTES_CONTENT.md), which specifies the CONTENT type in full.

---

## 1. Where this sits

The iotdata protocol is **compact telemetry**. A frame carries a station, a sequence and a variant,
and then presence-coded fields whose meaning the variant fixes at compile time. That is the whole
base protocol, and it is deliberately small: a sensor that only ever reports a temperature needs
nothing else.

Alongside the fields, **every iotdata frame can carry TLVs**. The TLV mechanism is part of the base
protocol and is always available -- it is not an extension and nothing opts into it.

**Anything with a station id is a node.** An endpoint is a node, a relay is a node, a gateway is a
node. Some nodes additionally run a mesh. The word carries no implication about power, topology or
role; it means only "a thing this protocol can address".

**The node protocol is optional, exactly as the mesh is.** A conformant iotdata implementation may
carry no system TLVs whatsoever and still be entirely correct. But a node that adopts the node
protocol adopts it *as specified*: the point of writing it down is that a manager, a gateway, or a
console written by somebody else can talk to it without being told anything first.

Its purpose is **management, control, observability and interoperability** -- a standardised way to
package and deliver what a node is, how it is doing, what it can be told, and what it has recorded.
As with the base protocol and the mesh, the spec comes with a **library implementation**, so the
conformant behaviour is something a node inherits rather than reimplements.

### Two principles, applied everywhere

**(a) System or proprietary, by the upper bit.** Every space this protocol defines is split in half
by its top bit: the lower half is defined here, the upper half belongs to whoever is building the
device and is guaranteed never to be claimed. The rule is the same whatever the field's width:

```
TLV type     6 bits    0x00-0x1F system      0x20-0x3F proprietary
kvr key      8 bits    0x00-0x7F system      0x80-0xFF proprietary
bitmask      N bits    lower N/2 system      upper N/2 proprietary
```

The bitmask row is the newest statement of it and it was already latent: `RECEIVE_TYPES` is a u32
whose bit N means "I accept TLV type N", and the type field is 6 bits -- so types 0-31 are exactly
the system half and a u32 spans precisely them. Widening it to u64 would put the proprietary types
in the top half without changing a rule.

**(b) A mandatory minimum, then best practice.** What a node MUST do is small and testable. Beyond
it, this note says what works well and why, and a node is free to disagree.

## 2a. The types, and what kind of thing each one is

The axis that predicts everything else is **role**. Once the types are sorted by what kind of thing
they are, their encoding and their obligations follow, and two long-standing oddities stop being
oddities at all.

| type | name | role | structure | key means | direction |
|---|---|---|---|---|---|
| 0x00 | RECEIVE | advertisement | kvr | field | up |
| 0x01 | VERSION | report, static | kvr | field | up |
| 0x02 | VARIANT | report, static | kvr | **index** | up |
| 0x03 | CONTROL | command + inventory | kvr | field = command | both |
| 0x04 | STATUS | report, dynamic | kvr | field (some repeated) | up |
| 0x05 | CONFIG | report + **mutable** | kvr | field | both |
| 0x06 | DIAGNOSTICS | report, bulk | kvr | field | up |
| 0x07 | CONTENT | **transfer** | packed | -- | both |
| 0x08 | DISCRIMINATOR | **modifier** | opaque | -- | both |
| 0x09 | SETTINGS | report + **mutable** | kvr | field | both |
| 0x3F | PARTIAL | **framing** | opaque | -- | both |

**PARTIAL is not a report and never was.** It is framing -- the same kind of thing as the TLV
header's MORE bit -- and it shares a number space with VERSION and STATUS only by accident of where
it was declared. It belongs in the base protocol as `IOTDATA_TLV_TYPE_PARTIAL`, and it takes
**`IOTDATA_TLV_TYPE_MAX` (0x3F)**: all ones, which this protocol already uses as its sentinel
convention -- `IOTDATA_SEQUENCE_DOWN` is all ones, and so is `IOTDATA_STATION_BROADCAST`. Nominally
that is inside the proprietary half, and the exception is deliberate: a sentinel is not a type
anybody allocates, and an implementer reaching for all-ones was never going to get a usable number.
The one consequence is that `iotdata_tlv_type_is_system(0x3F)` is false, so anything gating on that
must know -- harmless in `iotdata_node_receive_accepts()`, which asks whether a node accepts a type,
and framing is not something a node accepts or refuses.

**DISCRIMINATOR is a modifier, and used to be called TIMESTAMP.** Its one load-bearing property is
that including it changes the frame's hash, which is what lets an operator send the same downstream
command twice against content-addressed dedup (NOTES_DOWNSTREAM §7). It is not interpreted by
anybody. The old name described a recommended payload rather than the mechanism, and invited
gateways to try to read it.

### Encoding: RAW and kvr, and nothing else -- but that is the NODE layer's rule

**The base protocol puts no structure inside a TLV.** Its two formats say only how the bytes are
packed: `FMT_RAW` is opaque binary, `FMT_STRING` is 6-bit packed characters. The TLV mechanism is
available to any implementation to carry anything at all, and nothing in the base requires a payload
to be key-value shaped.

**kvr and kvs are OFFERED conventions** for structuring those bytes -- available to anyone, adopted
automatically by no one. This protocol standardises on them, which is a decision of this layer:

Node TLVs are **`FMT_RAW` carrying a kvr**. There are no other node encodings, and the exceptions
are only those types that carry no keys at all -- CONTENT, which is packed positional
(NOTES_CONTENT §5), and DISCRIMINATOR and PARTIAL, which are opaque.

```
kvr:  [key:u8][vlen:u8][value ...] *     repeated, self-delimiting, order is key order
```

Two other mechanisms exist in the library and the node protocol uses neither:

- **kvs**, a space-delimited `name value` text form inside `FMT_STRING`. Nothing outside its own
  unit test uses it. Kept for now against a future need, or removal.
- **`IOTDATA_TLV_FMT_STRING`**, a 6-bit character packing worth 25% on text. VERSION's strings are
  its natural customer, but the alphabet is `a-z A-Z 0-9 space` -- **it has no punctuation**, so
  `0.9.9-rc1` cannot be carried. Parked for that reason, which is not the one usually assumed.

### The key byte has two meanings, and that is fine

In almost every type, a kvr key **names a field**. In VARIANT it **is an index**: key 3 means
variant 3, not "field 3". Both are legitimate, and the distinction only matters when reading the
request rule below -- asking VARIANT for key 3 means "variant 3's definition", not "field 3's
value". Same mechanism, different noun.

## 2b. How every type operates

### Request and response: key selection

One rule, and it holds across every role:

> **An unqualified request means the full key set. A qualified request means those keys.**

What *happens* to the selected keys is the role's business, and only that part varies:

| role | unqualified | qualified |
|---|---|---|
| report | every key and its value | those keys and their values |
| mutable | every key and its value | assign, then return the result |
| command | every key that exists (the inventory) | actuate those |

That is what makes the protocol discoverable without documentation: point a manager at an unknown
node, ask each type for everything, and it tells you what it has.

### Two generic CONTROL keys carry it, and nothing is derived

The rule needs a wire form, and there is exactly one for every type:

```
REQUEST   value:  [ subject:u8 ][ subject-specific selection ... ]
CONTROL   value:  [ subject:u8 ][ action:u8 ][ subject-specific arguments ... ]
```

**A subject is a superset of a TLV type.** Most subjects are types, but setting a station id is
about no type at all, and neither is blocking a station -- so the byte addresses a wider space:

```
0x00..0x3F   the TLV types, verbatim, keeping their own system/proprietary split
0x40..0x7F   system subjects that are NOT types -- NODE, MESH
0x80..0xFF   proprietary subjects
```

That gives the mesh an honest home. It was never a TLV type -- its state is reported inside STATUS's
upper range -- so its *mutation* had nowhere to sit except bolted onto a report type. As a subject
it needs neither.

An empty selection is the unqualified request. A non-empty one is subject-specific, because each
subject owns its inner protocol: VERSION reads it as a list of keys, STATUS reads it as a scope byte
followed by optional keys, and a proprietary subject reads it however its vendor likes.

**An action id is scoped by its subject, exactly as a key and an event id are.** DIAGNOSTICS defines
*enable*, *clear*, *dump*; NODE defines *reboot*, *reset*, *set-station*; MESH defines
*filters-update*, *peers-clear*. Nothing collides because nothing shares a namespace.

This replaces a scheme where a control key was **derived** from the type as `type << 3`, reserving
eight keys per type. That scheme had three problems, and the third is the one that matters:

- it spent half the system control range (0x00-0x3F) on eight types, mostly on empty slots
- it collided: RECEIVE is type 0x00, so its derived request key was 0x00, which is REBOOT
- **it could not address the proprietary half at all.** Type 0x2A derives 0x150, off the end of a
  byte -- so a vendor's own types were unrequestable, which quietly contradicted rule (a)

And it gave the qualified request nowhere to live. Under the old scheme `VERSION_REQUEST` took no
value, so "tell me just your serial" was unsayable; the selection rule above was aspirational rather
than encodable.

So the CONTROL type has exactly **two keys**, and everything else is vocabulary. Rebooting is
`control(NODE, reboot)` rather than a key of its own: three bytes instead of one, for a command sent
once a year, in exchange for never accumulating flat keys again.

### Three types carry three different things

| type | carries | examples |
|---|---|---|
| **SETTINGS** | values that configure the **protocol** | station id, reporting schedule, receive schedule |
| **CONFIG** | values that configure the **device** | sample interval, thresholds, calibration |
| **CONTROL** | **verbs** | reboot, reset, dump, update a table |

The line between the first two is not "node protocol versus the rest" -- a station id is *base*
protocol and still belongs in SETTINGS. What unites SETTINGS is that it configures **the protocol's
own behaviour**; what unites CONFIG is that it configures **the thing the device is for**.

**A write is not a verb**, which is why SETTINGS is a type and not a set of control actions.
`set-station` and `set-trigger` were actions that assigned a value, and an action has no defined
answer -- so a rejected or clamped write looked exactly like a successful one. As keys of a mutable
type they inherit the rule that already governs CONFIG: assign, then return what is now true.
Reading matters as much: under the action form, seeing a node's whole schedule was one round trip
per subject; now it is one request.

The argument for keeping the reporting schedule out of CONFIG is extensibility, not tidiness. A
**proprietary** TLV type would need a proprietary CONFIG key to carry its schedule -- allocated by
the vendor, unknown to the library, so the library's machinery could not manage it. As a repeated
SETTINGS key with the subject **inside the value**, one implementation serves both halves of the
space with no coordination at all.

That shape has now been the right answer three times -- STATUS's table entries, VARIANT's
definitions, and the reporting schedule -- and it is worth stating as an idiom: **when a thing is
per-something-unbounded, the key repeats and the discriminator goes in the value.**

The consequence is worth stating plainly: **CONFIG's system key range goes empty.** Everything in it
today is protocol configuration and leaves. CONFIG becomes a type whose contents are entirely the
implementation's, and a node with nothing application-shaped to configure answers `request(CONFIG)`
with an empty kvr -- which is an answer, not a refusal (§2b). "A sensor that does not support
CONFIG" is not a thing; "a sensor with no config keys" is.

### Protocol configuration MUST persist

> A node MUST persist its protocol configuration across a power cycle -- at minimum its station id
> and its per-type triggers -- whether or not it persists any application configuration of its own.

The floor is operational rather than aesthetic: **a node whose station id cannot be changed and
retained cannot be commissioned, or re-commissioned, in the field.** And a manager forced to
re-apply triggers after every restart is fighting the device rather than configuring it.

The cost is small enough to mandate. A station id is two bytes and a trigger record is eight, so ten
system types plus a few of a vendor's own is on the order of a hundred bytes of non-volatile
storage.

**RTC-retained memory is not persistence.** It survives a watchdog, a software restart and a deep
sleep, and it does not survive a power cycle -- which is exactly the case commissioning cares about.

This also disposes of most of the persistence-discovery problem. Protocol configuration always
persists, so there is nothing to declare about it; the only open question is whether *application*
configuration does, which is a static property of the device and therefore belongs in
`VERSION_CAPABILITIES` (§3.2) rather than in a response flag. A manager detects the restart it needs
to react to from STATUS's `restarts` and `uptime`, so no new mechanism is required anywhere.

### The response is the truth

A response is never an acknowledgement. It is **what is now true**, whatever the request asked for.
A write that was refused, clamped, or partially applied returns the values the node actually holds,
and a manager that wants to know what happened compares them with what it sent.

This is one principle, not three special cases: an immutable CONFIG key, a value clamped to range,
and a trigger the node cannot honour all behave identically and all read identically.

> **PARKED.** What the response does *not* carry is a reason. A manager cannot currently distinguish
> "immutable" from "clamped" from "unsupported". A per-value feedback or error code would fix it and
> there is no elegant shape for one yet -- anything per-key risks doubling the size of every
> response. Deliberately left open rather than solved badly.

### The four triggers

Beyond being answerable on request, a type may be emitted **spontaneously**. There are four
triggers, they OR together, and they are configured at the **gross type level only** -- there is no
subscribing to "STATUS.reason changed" or "CONFIG.period_status written".

| trigger | fires | parameter |
|---|---|---|
| **at-startup** | once, at boot | none |
| **on-period** | on a cadence | the periodicity |
| **on-change** | a value that was X is now Y | a **minimum** period, as a flap guard |
| **on-event** | something happened, with no previous value | an **event selector** |

**on-change and on-event are different things** and conflating them causes trouble. On-change suits
values that are discrete and stable: VERSION after an OTA, CONFIG after a write, RECEIVE after a
schedule change, VARIANT after a switch. On-event suits things with no previous value at all: a
diagnostics record was written, a fault occurred, a threshold was crossed.

**An event id is scoped by its type, exactly as a key is.** DIAGNOSTICS defines its own events --
*recorder full*, *an error-level record was written* -- CONFIG defines *write refused*, VARIANT
defines *switched*. None of them collide because none of them share a namespace, and the upper half
of the selector is proprietary like everything else.

**Coarse to configure, precise to implement.** Because the subscription is per-type, on-change for
STATUS would fire every second on `uptime` alone unless the emitter is sensible. So the exclusion is
an obligation on the node rather than a configuration:

> A node MUST NOT count a monotonically-varying value -- an uptime, a lifetime, a counter -- as a
> change for the purposes of on-change.

### The REPORT entry

One subject's complete schedule is **9 bytes**, a repeated SETTINGS key (§3.11):

```
SETTINGS / REPORT
  subject : u8      a TLV type, or a proprietary one
  flags   : u16     bit0 at-startup | bit1 on-period | bit2 on-change | bit3 on-event
                    bits 0-7 system, bits 8-15 proprietary (rule (a))
  period  : u16     seconds; the on-period cadence
  change  : u16     seconds; the MINIMUM interval between on-change emissions
  events  : u16     bitmask, scoped by the subject; bits 0-7 system, 8-15 proprietary
```

The flags field is u16 rather than u8 for a specific reason: under rule (a) a u8 yields exactly four
system bits, and the four triggers defined here would fill it on the day it shipped, leaving no room
for a fifth ever. One byte buys eight system triggers and eight proprietary ones.

### Conformance: every obligation is a MUST, and they differ by antecedent

The question is not how strong an obligation is, it is **what makes it bind**. There is no
SHOULD-shaped middle ground and no type a node may decline to implement.

| kind | binds when | examples |
|---|---|---|
| **unconditional** | always | every type is answerable on request; unknown TLVs are ignored |
| **lifecycle** | at a moment in the node's life | VERSION at startup |
| **conditional on state** | its precondition is true *right now* | PARTIAL if a report spans; DISCRIMINATOR if a frame must be distinct; RECEIVE if the node sleeps and wants downlink |
| **configured** | when told | the four triggers; the obligation is to honour the configuration, not to have one |

The third row is what removes the word "optional" from the vocabulary. There is no node that "does
not implement PARTIAL" -- only a node that never spans a report. The obligation fires from a fact
about what the node is doing at that instant, not from a decision it made at design time.

### The two floors

**Unknown TLVs are ignored, never fatal.** This is the rule that makes everything else affordable,
and it applies to types and to keys alike:

> A node MUST ignore a TLV type it does not implement and MUST NOT treat the frame as malformed.
> The same applies to an unrecognised key inside a type it does implement.

Without it the proprietary half is unusable -- a vendor's 0x2x type has to pass harmlessly through a
gateway that has never heard of it -- and no system type could ever be added. The library already
makes this structurally true: it performs no type validation and hands every decoded TLV up as data.

**An empty answer is an answer.** This is what makes "every type, always answerable" cost nothing.
A node with no content, no mesh and no diagnostics still answers those requests; it answers with
nothing, and that is information rather than an error. The codebase has said so for a while, in
`iotdata_status_pack`:

> *An EMPTY payload is a legitimate answer and not an error: a plain sensor asked for the mesh group
> is correctly telling you it is in no mesh, and answering with the node group it was not asked for
> would be worse than answering with nothing.*

## 3. The types in detail

### 3.1 RECEIVE (0x00) -- when this node can be reached

A sleeping node is unreachable except in the windows it chooses to open, and nothing else in the
system can know when those are. RECEIVE is how a node states it, and it is the only reason
downstream delivery works at all: a relay holds a frame for a station until it snoops that station's
advertisement (NOTES_DOWNSTREAM §4).

| key | width | meaning |
|---|---|---|
| 0x00 `DURATION` | u16 | milliseconds the receiver stays on |
| 0x01 `TYPES` | u32 | bitmask, bit N = "I accept TLV type N" |
| 0x02 `OFFSET` | u16 | **proposed**: seconds until the window opens |

**Absence of DURATION means the system default**, 5000 ms, rather than "no window" -- a holder has
to decide whether a frame fits before sending it and cannot do that against an unknown. There is no
concept of an always-on station: a permanently listening node advertises a very large duration and
is handled by exactly the same rules.

**OFFSET is new and not yet implemented.** It lets a node say "I will be available in an hour"
rather than only "I am available now", which is what a duty-cycled node with a long sleep actually
wants to express. u16 seconds gives an 18.2-hour ceiling, which is the same trade as CONTENT's
expiry being minutes rather than seconds. Two consequences, both in the down module:

- the trigger changes from *on snoop, send* to *on snoop, send at the stated time*
- `iotdata_node_receive_fits()` must size against the window's own duration rather than against
  what remains of it from now

### 3.2 VERSION (0x01) -- what this node IS

Static properties, fixed at build or at manufacture. VERSION is the **root of discoverability**: it
is the one type with a lifecycle obligation, because everything else being optional is only
tolerable if the node says what it is.

| key | meaning |
|---|---|
| 0x00 `HARDWARE` | `"board/arch"` |
| 0x01 `FIRMWARE` | `"stack[+low]"` -- what runs beneath the application |
| 0x02 `SOFTWARE` | `"app/semver/stamp"` |
| 0x03 `SERIAL` | a stable per-device id, never a removable NIC's address |
| 0x04 `CAPABILITIES` | 16-bit entries, `[key:4][mask:12]`, big-endian |

**Capabilities are static properties; CONTROL is operative actions.** The distinction is normative
and the two must not be derived from one another. "I have mesh capability" is a property and belongs
here; "disable meshing" is an action and belongs in CONTROL. A node can have a capability it exposes
no command for, and a command that corresponds to no declared capability.

The encoding already supports rule (a) twice over: the 4-bit key gives 8 system capability groups
and 8 proprietary, and the 12-bit mask gives 6 system bits and 6 proprietary within each.

**Two capabilities that other parts of the protocol turn on**, both in the FEATURES group:

| | |
|---|---|
| `MESH` | this node runs a mesh, so its STATUS carries the 0x40..0x7F range and its CONTROL accepts the MESH subject |
| `CONFIG_PERSIST` | this node's APPLICATION configuration survives a power cycle |

Protocol configuration always persists (§2b), so there is nothing to declare about it -- but a
device whose own settings evaporate on a power cycle is something a manager must know in advance,
and it is a static property, so it belongs here rather than in a response flag.

> **OPEN: granularity.** Persistence may be per-item rather than per-node -- a station id burned in
> while a threshold is volatile -- and one bit cannot say that. Whether the coarse claim is good
> enough, or whether this folds into the parked per-value feedback question (§2b), is undecided.

### 3.3 VARIANT (0x02) -- which telemetry shapes this node produces

The base protocol's variant maps are compile-time: a gateway knows what variant 3 looks like because
it was built knowing. VARIANT is the **dynamic** answer to the same question, for a decoder that was
not.

### A variant id is LOCAL. A field id is GLOBAL.

This is the whole design, and everything else follows from it. The variant number in the iotdata
header, and the presence map it selects, mean something only **on the node that sent them** -- node
A's variant 3 and node B's variant 3 are unrelated. What is shared across the whole system is the
identity of a **field**: "air temperature in centi-degrees" is the same thing everywhere.

So the VARIANT TLV translates one node's local numbering into global terms, and a decoder that has
never met this node can then read its telemetry.

### The encoding

A kvr with **two keys, each repeated once per variant**:

```
ENTRY  0x00   [ variant:u8 ][ field id:u16 ] *              what a DECODER needs
NAMES  0x01   [ variant:u8 ][ name NUL ] [ name NUL ] *     what a PERSON needs
```

The key used to *be* the variant number, which was neat and spent the whole key space on one thing,
leaving nowhere to put anything else about a variant. Carrying the number in the value costs one
byte and buys the second key.

Ids are positional: index N is the field in presence slot N. Slot geometry is the base protocol's --
`IOTDATA_MAX_DATA_FIELDS` is 27 (6 in the first presence byte, then 7 in each of up to three more)
-- so a fully-populated ENTRY is 55 bytes and a node defining three still fits one frame.

```
0xFFFF    the slot carries nothing (all ones, the sentinel convention)
0x0000..  system field ids
0x8000..  proprietary field ids (rule (a))
```

**NAMES are descriptive and advisory.** The variant's own name comes first, then one per slot in the
same order. A decoder works from the ids alone; nothing may depend on a name being present, spelled
a particular way, or unique. A name that does not fit is dropped from the tail exactly as an empty
slot is, so an absent one reads as "not stated".

**Names are not sent by default.** An unqualified request answers with ENTRY alone, because the ids
are what make telemetry readable and the names are several times their size -- the same trade STATUS
makes by leaving the tables out of its default scope. The selection byte is a bitmask over the key
space, bit N selecting key N, which is VARIANT's own reading of it (§2b: each subject owns its
inner protocol).

Variant 15 never appears: it is the mesh variant and carries no telemetry.

**There is no manifest key, and none is needed.** The presence of key N *is* the statement that this
node defines variant N; its absence is the statement that it does not. A bitmask saying the same
thing again could only ever disagree with the keys beside it. What it was really for -- telling a
receiver when it has them ALL across a chunked report -- is what PARTIAL already answers, and
answering that in two layers only creates the chance of the two disagreeing.

**Trailing empty slots are trimmed.** An unused slot in the MIDDLE is `0xFFFF`, because dropping it
would shift every slot after it; a trailing one has nothing after it to shift, and a slot past the
end of a value reads as `0xFFFF` anyway. A definition with two fields in a three-byte presence map
would otherwise spend 40 bytes saying "nothing" twenty times.

**PRODUCES, not decodes.** The report is what this node emits, which is not what its compiled-in
variant table holds: a gateway is built with the maps of every node it must READ, so its table is
full while it originates no telemetry at all, and a relay's only variant is the mesh one. Both
answer with an EMPTY report rather than with somebody else's suite.

Nothing else is carried. A field's type, width, scaling and units are properties of the field id,
held in the registry, not restated per node -- which is the point of the ids being global.

> **INTERIM, and it is the one part not yet real.** There is no registry yet, so the u16 currently
> carries `iotdata_field_type_t` -- the compile-time enum used in the `.fields` entries of the
> variant suite. That is a local number wearing a global number's clothes, and it works only because
> every node in this system is built from the same tree. The wire format does not change when the
> registry arrives; the values in it acquire meaning they do not have today. Implement against the
> enum, and expect to swap the mapping, not the encoding.

### 3.4 CONTROL (0x03) -- what this node can be told

Commands, and the inventory of which commands exist. The key selection rule reads naturally here:
an unqualified CONTROL request returns every command key the node accepts, and a qualified one
actuates those commands.

The whole command set is **two keys** (§2b):

| key | command |
|---|---|
| 0x00 | `REQUEST` -- value: `[subject][selection...]` |
| 0x01 | `CONTROL` -- value: `[subject][action][args...]` |

Everything that used to be a key of its own is now a `(subject, action)` pair:

```
control(NODE,        reboot, [delay_s:u16])
control(NODE,        reset, token, which)
control(NODE,        set-station, station:u16)        -- MUST persist (§2b)
control(<type>,      set-trigger, record:8)           -- MUST persist (§2b)
control(<type>,      get-trigger)
control(DIAGNOSTICS, enable, category)
control(DIAGNOSTICS, clear, [stamp:u32])
control(DIAGNOSTICS, dump)
control(MESH,        filters-update, [station:u16, action:u8] * N)
control(MESH,        filters-clear, scope:u8)
control(MESH,        peers-update | peers-clear | peers-dump)
control(STATUS,      stations-dump)
```

Note what the generic shape buys beyond tidiness: `DIAGNOSTICS_ENABLE` was a flat key with a
one-byte value, so it could say *whether* to record but never *what*. As an action it takes
arguments, and the category comes for free.

**A node MUST answer `REQUEST` for every type it emits.** That is the whole self-consistency rule,
and it is now trivially true rather than a numbering coincidence.

**The inventory answers "what can I be told".** `request(CONTROL)` returns the actions the node
accepts, which under the generic scheme means a list of `(type, action)` pairs rather than a list of
key flags. That is more informative than the old inventory: it says not only *that* a node takes
diagnostics commands but *which*.

### 3.5 STATUS (0x04) -- how this node is DOING

Dynamic state. Two linear ranges and nothing finer:

```
0x00..0x3F  ANY NODE  -- packed linearly from 0x00, extended as needed
0x40..0x7F  THE MESH  -- packed linearly from 0x40, only on a node that runs one
```

There is deliberately no sub-structure -- no scalar range, no table range. Keys are allocated in the
order they are invented and a reader is told what a key is by its keydef, never by arithmetic on its
number.

| range | keys |
|---|---|
| 0x00-0x08 | uptime, lifetime, restarts, reason, temperature, supply, heap free/min, active |
| 0x09-0x0C | stations count + entry, filters count + entry |
| 0x0D-0x0E | **content count + entry** |
| 0x40-0x4F | mesh state, parent, cost, generation, parent rssi, accepting, and the counters |
| 0x50-0x51 | mesh peers count + entry |

**The content table says what this node is holding or in the middle of.** It is the observability
half of CONTENT: the transfer machinery is in the CONTENT type, but "what have you got, and how far
through are you" is a status question and belongs here with every other status question.

```
CONTENT entry, 13 bytes
  [0:2]    u24  content-id
  [3]      u8   state
  [4]      u8   flags -- bit0 PERSISTENT (survives a power cycle)
  [5:6]    u16  chunks held
  [7:8]    u16  chunks total              (0 if no manifest has been seen)
  [9:10]   u16  minutes until expiry      (0xFFFF = indefinite)
  [11:12]  u16  seconds since last progress, saturating

state  0x00 HOLDING    complete and verified; can serve it
       0x01 FETCHING   incomplete, actively requesting
       0x02 CACHING    incomplete, accumulated passively, not requesting
       0x03 FAILED     verification failed, or abandoned
```

FETCHING and CACHING are the distinction NOTES_CONTENT §9 draws between an endpoint's transfer and a
relay's cache -- *"a relay's cache is an endpoint's partial transfer that never completes and is
never consumed"* -- and having it visible is what lets an operator tell a relay quietly filling up
from an endpoint that is actually trying to finish something.

**Seconds-since-progress is what makes a stall visible.** FETCHING says a node is trying; only the
elapsed time says it is getting nowhere, and a transfer that spans days across a duty-cycled link
cannot be diagnosed any other way.

**PERSISTENT is not decoration.** A node holding a half-finished transfer in RAM loses it on the
next power cycle and starts again; one holding it in flash does not. That is the difference between
a transfer that will eventually finish and one that never will, and it is invisible without saying
so. It is also per-content rather than per-node -- a node may stage one object in RAM and another in
its OTA partition -- which is why it lives here and not in VERSION's capabilities.

This entry widens `IOTDATA_NODE_STATUS_ENTRY_MAX` from 11 to 13, since it becomes the widest row.

**STATIONS and FILTERS are not mesh concepts**, which is why they are in the ANY NODE range: any
node that hears traffic has stations it can hear, and any node can refuse one. A gateway with no
mesh has both. Only PEERS -- who a node could route *through* -- is meaningless without a mesh.

**Tables are repeated keys, not a separate kind of TLV.** An entry key carries one fixed-width
record and repeats; its width is an ordinary keydef entry like every other key's. There used to be
three table TLV *types* with per-type row-size machinery; there is now nothing but keys.

**Counts are scalars, and every scalar is emitted before any entry.** A report whose tail is lost
then still says how big each table was, which is the more useful half. A caller that asks only for a
table's entries gets no count, and that is its own choice.

**STATUS has its own floor.** `uptime` and `reason` carry no presence flag: every node has been up
for some length of time and booted for some cause, `unknown` included.

### 3.6 CONFIG (0x05) -- what the DEVICE can be set to

Settable operating parameters of the application, and immutable ones. CONFIG is the only **mutable**
role: a qualified request assigns and then returns the result, per "the response is the truth".

**Its system key range is empty** (§2b). `PERIOD_*` and the `STARTUP` bitmask are gone: both were
protocol configuration, and they are now the trigger record carried by
`control(<subject>, set-trigger)`. What is left is the device's own -- a sample interval, a
threshold, a calibration coefficient -- and those are proprietary keys, because what a device is
*for* is not this protocol's business.

Three things are fixed on the way out, and they are why the old shape could not simply be kept: it
had two shapes for one axis (per-type keys for the period, a bitmask for startup), the `STARTUP`
bitmask carried an implicit "bit N = type N" rule of exactly the sort being removed elsewhere, and
there was no `PERIOD_RECEIVE` or `PERIOD_CONTENT` at all -- so two of the types could not be
scheduled by any means.

A node with nothing application-shaped to configure answers with an empty kvr, which is an answer.

**Rule (a) applies unchanged: 0x00-0x7F is system and 0x80-0xFF is the implementation's.** The
system half is *reserved and empty* -- a device puts its own settings in the proprietary half like
any other proprietary key.

Reserving a half that nothing occupies is deliberate. If an app-level setting ever proves universal
enough to standardise -- a sample interval, say -- the number it wants must not already be in use
across a fleet of devices, or standardising it becomes a breaking change rather than an addition.
128 keys is more than any device here plausibly needs, so the reservation costs nothing and keeps
the option open.

### Where it is kept

Both CONFIG and the protocol configuration of §2b persist through the **datastore**, which is
already a keyed blob store with a platform seam -- NVS on ESP32, a directory of files on Linux --
and already answers `datastore_persistent()`.

**Values are stored as blobs, not as typed entries.** A kvr value is already `[key][vlen][bytes]`,
so a config value *is* a length and some bytes; the type lives in the node's own config table, which
has to know it anyway in order to validate a write. Typing the store as well would put the same
knowledge in two places and force the seam to reconcile NVS's typed getters with a file that has
none.

One blob per key rather than one blob for everything, because that is the grain NVS already works
in: a single write does not rewrite the rest, and wear spreads. Protocol configuration takes a
distinct key prefix from application configuration, so the mandate in §2b can be satisfied
independently of whether the device persists its own settings.

**On a host, there is ONE FILE.** A linux node's configuration is a file a person edits, and the
protocol's own settings live in it beside the device's, routed by name -- a key prefixed `node-` is
a SETTINGS value, anything else is a CONFIG row -- and share the reader, the writer and the
precedence. The alternative, which this replaced, was a second store: an opaque blob in a directory
beside the file, holding values nobody could inspect or edit and which no amount of reading the
configuration would reveal. The layering is the same for both: table defaults, then the operator's
file, then the runtime overlay, then the command line.

**The overlay records what CHANGED, measured against what was loaded** -- not against the build's
default. Every value an operator put in their own file differs from the default too, so measuring
there copies their file into the overlay, and the overlay being read second then shadows the file
they were editing.

### 3.7 DIAGNOSTICS (0x06) -- what this node recorded

The blackbox, delivered. Bulk, append-only, and the natural home of the on-event trigger.

| key | width | meaning |
|---|---|---|
| 0x00 `TYPE` | u8 | `IOTDATA_NODE_DIAG_*` -- what DATA holds |
| 0x01 `DATA` | bytes | the records (blackbox CSV lines) |

Records are **CSV text**, which is why an absent field is an empty field rather than a zero. A dump
spans frames and therefore relies on PARTIAL.

Its event vocabulary is the clearest of any type: *recorder full*, *an error-level record was
written*. A node that emits on those two needs no polling to be useful.

### 3.8 CONTENT (0x07) -- moving bytes

Fully specified in [NOTES_CONTENT](NOTES_CONTENT.md); summarised here only for its place in the
table. CONTENT is the one system type that is **not a kvr**. It is packed positional, because every
byte of overhead costs a chunk of payload, and it carries three sub-kinds behind one 32-bit header:

```
header (every sub-kind):   sub-kind:4 | tag:4 | id:24

MANIFEST : header | size:u24 | hash:u64 | sig-id:u24 | expiry:u16 | encoding:u8 | name:NUL-terminated
RANGES   : header | maxframes:4 | reserved:4 | [ start:u16, count:u8 ] * N
CHUNKS   : header |                            [ start:u16, count:u8, data ] * M
```

**What it is for.** Content is how anything larger than a frame moves: an OTA image, a calibration
blob, a configuration bundle, a signature stream. The design is deliberately *not* an OTA mechanism
-- it advertises, requests and transfers an opaque object identified by a **content-id**, and what
that object means is the application's business. That boundary is what lets one module serve a
gateway handing out firmware, a relay caching it, and a sensor flashing it, distinguished only by
their storage backend.

The shape in one paragraph: a holder **advertises** a manifest describing a content-id -- its size,
its gross hash, its encoding, an optional name, an expiry, and the content-id of its block-hash
stream. Anyone who wants it **requests** ranges of 32-byte chunks, paced against its own receive
window and battery. Any node that hears the request and holds the bytes may answer with **chunks**;
the answers ride the ordinary downstream path, so they flood, dedup and hold exactly like a command.
A relay that snoops chunks going past fills its own cache and can answer locally next time, which is
what makes a fleet behind one slow link affordable.

**`request(CONTENT)` is usable, and asks a node what it holds.** The answer is a list of manifest
entries -- the content-ids this node has and what they are. That is different from fetching the
content itself, which is RANGES, and different again from an advertisement, which is pushed.

### Two questions, two answers, and they must not be confused

There are exactly two ways to ask a node about content, they are scoped differently, and using the
wrong one gets a misleading answer:

| | `status.content` | `request.content` |
|---|---|---|
| **scope** | everything the node has any relationship with -- in-receipt, cached, on-offer | only what is **on-offer** |
| **shape** | the fixed 13-byte status entry (§3.5) | manifest entries, the same structure a MANIFEST carries |
| **says** | what STATE each content-id is in -- progress, staleness, persistence, expiry | what each content-id IS -- size, hash, encoding, name, signature stream |
| **cost** | a few bytes per id; fine on a cadence | a manifest per id; asked once |

A half-received object appears in `status.content` as FETCHING and does **not** appear in
`request.content` at all, because a node cannot offer what it does not yet have.

### The expiry problem, and why it does not arise here

§3 of NOTES_CONTENT requires that **only the advertiser originates an advertisement** and that the
expiry offset is never decremented on the way -- because an advertisement floods in seconds, so
adjusting it hop by hop would be arithmetic in exchange for nothing, and a cache that re-originated
one would restart the clock and extend the offer indefinitely.

A `request.content` response is not that case. **The holder knows exactly how long it has held the
entry**, because `status.content` obliges it to track exactly that. So:

> An advertisement **states** an expiry from receipt and is never decremented.
> A `request.content` response reports the expiry it has **observed remaining**.

Same field, different semantics, and the difference is legitimate precisely because the reason for
not decrementing -- that a flood takes seconds and elapsed time is immaterial -- is false for a
cache that has been sitting on something for a day. This closes the question both notes were
carrying.

> **DESIGNED, NOT IMPLEMENTED.** Nothing of the above is built. What exists in the header today is a
> pair of leftover kvr keys, `FIRMWARE` (0x00) and `USERDATA` (0x01), which the sub-kind design
> replaces outright and which now actively contradict the spec. They must go.

### 3.9 DISCRIMINATOR (0x08) -- making a frame unique on purpose

Opaque bytes, no keys, not interpreted by anybody. Downstream identity is the hash of the whole
frame, so two identical downs are one down -- which is what you want for an idempotent command and
an obstruction for a reboot. Including a DISCRIMINATOR changes the hash, and that is its entire job.

It is the specific mechanism for what would otherwise be done with random padding. As **recommended
practice** the payload should be a monotonically increasing value or a time, because an operator
reading a log wants to know when a command was issued and nothing is lost by suggesting it -- but
nothing may depend on that, and no gateway or relay may read it.

### 3.11 SETTINGS (0x09) -- what the PROTOCOL is set to

Station id, reporting schedule, receive schedule. A report and a mutable type, so reading and
writing use one mechanism and the response is always what is now true.

| key | value |
|---|---|
| 0x00 `STATION` | u16. **MUST persist** -- this is commissioning |
| 0x01 `REPORT` | repeated: `[subject][flags:u16][period:u16][change:u16][events:u16]` (§2b) |
| 0x02 `RECEIVE` | `[window:u16 ms][interval:u16 s][offset:u16 s]` -- always the triple |

**There is no sentinel, because there is no "not set".** Every settings field always holds a value:
seeded from the build, restored from persistence, or replaced by a write -- exactly as a config row
is. A node cannot be asked to *unset* something, only to set it to something else, and putting a
value back to its default means stating that value.

That is a deliberate trade against the obvious alternative. A "revert to default" operation is one
word on the wire but it means a **different thing on every node it reaches**, because a fleet of
mixed builds does not share one default; a value means the same thing everywhere and is checkable
in the response. It also removes the reader's fallback argument, and with it the second copy of the
answer that each caller was carrying -- copies that agreed only until one of them was forgotten.

The receive triple is still always all three fields: a fixed shape cannot half-arrive.

**A report is DENSE**: one `REPORT` entry for every subject this node can report, including the ones
running at their default. An omitted subject would make the far end guess which default this
particular build holds, and a guess is exactly what cannot be checked from the other end of a radio
link. The entries cost nothing to keep -- the schedule array is inside the persisted block either
way -- so the only price is a longer report on the air, and that price buys the answer.

**Where it persists, and when.** The store is the node state block (`iotdata_node_state.h`), not the
config system. A write **flushes immediately** rather than deferring: `iotdata_state_touch()` is
write-behind, and a power cut between the answer and the flush would leave the node disagreeing
with what it just told a manager. Settings writes are rare, so the flush costs nothing.

**The settings block is the whole answer, or there is no block at all.** A node that keeps one reads
its schedule from there and nowhere else; a node built without one keeps whatever cadence its
firmware states. The two are alternatives and never a merge, which is what stops the schedule
living in two places at once. `ON_PERIOD` clear means *never schedule this* -- a value, carried by
the flag, needing no sentinel to express it.

**A station change takes effect at once, and restarts the sequence.** `(station, sequence)` is the
whole identity of a packet, so a new id begins a new stream; carrying the counter over would make a
receiver see a station appear mid-sequence and read it as loss. Live rather than at the next boot,
because the reply has to be true -- a response saying "station 0x537" from a node still transmitting
as 0x123 is the one thing the mutable-type contract forbids. The transient it causes is the same one
a reboot causes, and resolves the same way.

**A write arrives as a SETTINGS TLV**, not as a command carrying one -- there is no verb, because
assigning is not an action. The node applies it, persists, and replies with a SETTINGS report. A
value that did not take is simply not in the reply, and that is the entire error channel: a station
of 0 or `IOTDATA_STATION_MAX` is refused because the framing reserves both ends, and a schedule for
a subject that cannot be reported is refused because there is nothing to schedule.

### Pinned values -- the command line as a forcing function

A value stated as a **command-line argument** is immutable for as long as that run lasts. A write
to it is refused from the radio and from the console alike, and the only way out is to restart
without the argument. This applies to CONFIG rows and to SETTINGS equally, because the reason is
the same for both: somebody standing at the machine said what this value is.

**Refused locally too**, which is the part that is a decision rather than a mechanism. The usual
reason to pin is that something about *where* this node is running makes the value not negotiable --
a serial port that is this box's port, a station id an installer assigned, a channel a site licence
fixes. "Not negotiable unless you are sitting at the console" is a weaker claim than the one the
operator made, and a console write that took would be undone silently by the next restart anyway,
since the argument is still there.

**A pin is not a flag on the row.** A row's flags are what the BUILD says and read the same on every
node; a pin is what this INVOCATION says and differs between two boxes running the same binary.
They are reported as the different facts they are.

**Over the air it needs no new machinery**: a pinned value behaves exactly as a read-only one, and
the error channel is the one already there -- the response is what is now true, and a value that did
not take is simply not in it.

### 3.10 PARTIAL (0x3F) -- one report across several frames

Framing, not content. A report that does not fit one frame is split by the sending node, and PARTIAL
is what says so: which span this is, how far through, and whether more follows. Without it a
receiver cannot tell a short frame from a short table.

It belongs to the base protocol rather than to the node protocol, and should be declared as
`IOTDATA_TLV_TYPE_PARTIAL` in the fields layer, at `IOTDATA_TLV_TYPE_MAX` -- all ones, the sentinel
convention this protocol already uses for `IOTDATA_SEQUENCE_DOWN` and `IOTDATA_STATION_BROADCAST`
(§2a).

## 4. State of play

| type | design | implementation |
|---|---|---|
| RECEIVE | settled, except OFFSET | built; OFFSET is a key, its down-path behaviour is not |
| VERSION | settled | built |
| VARIANT | settled; field registry is interim | **built** |
| CONTROL | redesigned around two generic keys | **built** |
| STATUS | settled | **built**, content table included |
| CONFIG | settled; system range **emptied** and reserved | **built** -- framework, persistence, validators; entries defined on simulator and sensor, compiled-in but empty on gateway and relay |
| DIAGNOSTICS | settled | built |
| CONTENT | settled | **not built** |
| DISCRIMINATOR | settled at 0x08 | **built** |
| SETTINGS | settled at 0x09 | **built** -- library, common module, platform, and attached in all four applications |
| PARTIAL | settled at 0x3F | **built**, in the fields layer |

Cross-cutting:

| | design | implementation |
|---|---|---|
| the reporting schedule | settled, as a SETTINGS key | **built** -- carried, persisted, and acted on |
| a node's default station | settled: derived, never 0, never broadcast | **built** on both platforms -- efuse on ESP32, the lowest permanent NIC on linux, hostname as a last resort |
| protocol-config persistence | settled, mandatory | **built** -- a state block, flushed on write |
| the field registry | **not started**; ids are the compile-time enum meanwhile | n/a |

## 5. What has to change

- **CONTENT**: build the three sub-kinds. (Its stale `FIRMWARE`/`USERDATA` keydefs are gone.)
- **CONFIG needs an iteration loop, not a rewrite.** The framework is done and proven on two
  applications; what is missing is COVERAGE. The relay in particular compiles the machinery and
  defines nothing, so it answers CONFIG with an empty report -- correct, and useless. The gateway
  is the harder case and deliberately untouched: it already has a configuration of its own, in its
  own file format, so the work there is migration and not definition.
- **A node must service every action it advertises.** The gateway asserts this
  (`test_mesh_node_control_keys_are_all_handled`); no other node does, and that gap is exactly what
  once let a relay advertise DIAGNOSTICS actions it could not reach. Deferred until after CONTENT,
  because CONTENT adds actions and the test should be written against the finished set.
- **ON_CHANGE and ON_EVENT are carried, not yet fired.** AT_STARTUP and ON_PERIOD drive the
  schedulers; the other two flags persist and report but nothing acts on them.
- **RECEIVE**: add `OFFSET`, and teach the down module that a window can be in the future.
- **The ignore-unknown rule**: assert it in the per-app dispatch, where it would actually break.
- **VERSION at startup**: assert the conformance floor rather than leaving it to configuration.
- **A reportable-type predicate** now decides what can be asked for, so DISCRIMINATOR is correctly
  not requestable. Worth remembering when a type is added: it is a decision about what the type IS,
  not about its number.

## 6. Parked

- **Per-value feedback in a mutable response** (§2b) -- no elegant shape yet.
- **kvs** -- deadweight; keep against a future need, or remove.
- **6-bit STRING packing** -- unused; blocked on punctuation, not case.
- **Spontaneous emission scope** -- whether a periodic STATUS can carry a table, or whether
  spontaneous always means the default scope. The airtime argument favours the latter; a gateway
  wanting its stations table on a cadence is the case against.
