# NOTES — Content: Advertisement, Request and Transfer

> Working note, 2026-09-17. Specifies the CONTENT mechanism (`IOTDATA_NODE_TLV_CONTENT`), which
> exists as a type but has no defined behaviour. Builds on
> [NOTES_DOWNSTREAM](NOTES_DOWNSTREAM.md), whose flood-and-hold rules carry every frame described
> here. To be folded into §5 and a new appendix.

---

## 1. Premise: symmetrical by design, downward by implementation

Content is **bidirectional and fully symmetrical**. A station can offer content upward; a gateway,
a relay or anything else can offer it downward, and either side can request. Nothing in the
encoding distinguishes the directions — a request is a request and a transfer is a transfer.

**Only the downward case is being implemented.** The upward case is the same machinery pointed the
other way, and is deliberately left unbuilt until something needs it.

There are two separable aspects, and conflating them is the mistake to avoid:

1. **The manifest** — advertising, offering or requesting a *transfer*, with the metadata that lets
   the other end decide whether it wants it: naming, size, encoding, integrity, expiry. Close in
   spirit to HTTP's content negotiation headers.
2. **The transfer** — moving bytes for an agreed **content-id**.

## 2. The content-id is the only thread, and provisioning is out of band

Every advertisable or requestable thing has a **content-id**: an opaque integer that ties the
manifest, the transfer, any caching and any signature stream together. The protocol carries nothing
else about identity.

**How a content-id comes to exist, and how it is withdrawn, is NOT the protocol's business.** A
gateway is handed content over MQTT; a relay might be given a file; either could have it baked into
a flash image. The protocol only ever advertises, requests and transfers. That boundary is what
lets the same module serve three very different nodes (§9).

**Signatures are not built in, but they are not optional either.** A signature is just another
content-id carrying another stream (§7), so the mechanism is reused rather than special -- and
every manifest must name one, leaving the decision of whether to FETCH it with the endpoint that
pays the airtime. An OTA is therefore always two content-ids: the object and its block hashes.

## 3. The manifest

**One manifest per MANIFEST TLV, and the content-id comes from the header.** An OTA exists in
flavours (platform, encoding), each flavour is its own content-id, and the receiving node picks --
so a frame offering several flavours carries several MANIFEST TLVs. The framing already allows a
repeated type: TLVs decode into a flat list, not a per-type slot, and **the protocol puts no limit
on how many a frame carries** -- the frame's own space is the only bound. `IOTDATA_TLV_MAX` (8 in
this library) is a DECODER-side RAM dimensioning choice, not a wire rule; a sender packing many
small TLVs into one frame is leaning on the receiver's dimensioning, so staying well inside it is a
practical courtesy rather than a constraint.

That one decision collapses the encoding. With exactly one manifest per TLV there is no repeating
group to delimit, no entry key and no nesting, so the manifest is **bitpacked positional** like the
transfer rather than a kvr:

```
MANIFEST                                       bytes
  header   : sub-kind:4 | tag:4 | id:24          4    id = the content-id; tag = generation (§5)
  size     : u24                                 3    of the ON-AIR object, before any decoding
  hash     : u64                                 8    CRC-64/XZ of the whole on-air object
  sig-id   : u24                                 3    the block-hash stream (§7); MANDATORY,
                                                      0 only in a signature stream's own manifest
  expiry   : u16                                 2    minutes from receipt; 0 = revoked,
                                                      0xFFFF = indefinite
  encoding : u8                                  1    content-type:4 | compression:4
  name     : NUL-terminated                      n+1  free text; a lone NUL is no name
                                        = 21 + name + 1
```

**Nothing is flagged, counted or conditional.** The one optional field has a natural absent value
instead: a bare NUL is no name.

**`sig-id` is mandatory** (§7 -- always provided, optional to use), which leaves `0` exactly one
legitimate meaning: **this IS a signature stream**, and naming one for it would recurse forever.
That is the base case rather than an exception, and it needs no flag because a `sig-id` of 0 is
*already* malformed by construction -- the allocation rule below makes the top 12 bits `0x000`
unassignable -- so the value was never available to mean anything else.

**The name is NUL-terminated rather than length-prefixed**, which costs exactly the same byte and
buys two things. A decoder can hand the name out as a C string in place, with no copy and no
terminator to append. And there is no length field that can disagree with the TLV's own length --
the same reason the CHUNKS runs walk themselves rather than carrying a count (§5).

22 bytes for a manifest with no name, 54 with a 32-byte one, against ~53 for the kvr shape this
replaces even before the name.

**`size` is u24, and u16 is not an option** even though the chunk index is one. They count
different things: the index counts chunks, so u16 buys `65536 x 32 = 2 MiB`, while `size` counts
BYTES, so u16 would buy 64 KiB -- below the realistic working range and below most of §11's airtime
table. The only route to two bytes is to count chunks and carry the final short chunk's length
separately, which is `u16 + u8` = the same three bytes with an extra field and a padding question
for the gross hash to answer. So the binding ceiling is the index's **2 MiB** (§4), `size` has
headroom it will never use, and the field stays byte-aligned at the content-id's width.

**Encoding is one byte of two nibbles, AND the name carries text.** The nibbles answer "can I
decode this at all", which every node including an 8-bit one must decide without parsing a string;
the name is for the human and for OTA platform/semver matching. This is exactly why HTTP separates
`Content-Type` from `Content-Encoding` from the entity -- and the two nibbles are precisely that
pair.

```
 7   6   5   4 | 3   2   1   0
 content-type  | compression
 ^ high bit of each nibble = proprietary
```

| content-type (what it is) | | compression (how it is wrapped) | |
|---|---|---|---|
| 0 | binary | 0 | none |
| 1 | bsdiff | 1 | heatshrink |
| 2 | tarball | 2 | xz |
| 3 | zipfile | 3 | gzip |
| 4-7 | (spare, standard) | 4-7 | (spare, standard) |
| 8-15 | proprietary | 8-15 | proprietary |

**The per-nibble proprietary bit earns its place**, because an unknown value means two different
things and the receiver should act differently: an unknown STANDARD value means "my firmware is
older than this manifest" and a newer build might handle it, while an unknown PROPRIETARY value
means "this was never meant for me". One bit distinguishes a version problem from a scope problem.

**Three bits is tighter than it looks.** A plausible compression list -- none, heatshrink, xz, gzip,
lz4, zstd -- already fills six of the eight standard slots, and content-types will fill just as
fast. The escape is that this is a kvr field: if the nibbles run out, a wider key can be added
later and old readers skip it, so the risk is bounded rather than structural.

**A compressed content-type must include something the small devices can actually decode.** An
xz/LZMA2 decoder needs its dictionary in RAM -- 64 KiB at the small end -- which is affordable on
an ESP32-C3 with ~200 KB free and impossible on an 8-bit part. Hence heatshrink in the standard
list, whose window is 256 bytes of state. Without it, "compressed" would silently mean "not for
small devices", which would undercut the entire point of advertising flavours and letting the
endpoint choose.

**No capability negotiation is needed anywhere.** The advertiser publishes every flavour it has and
the endpoint picks one it can handle; nothing has to announce which codecs it supports.

### The name, and where the spec stops

The name is **optional** -- content identified by its id needs none, and a calibration blob
legitimately has no use for one. **32 bytes is a recommendation, not a limit**: nothing breaks at
200, you simply fit fewer entries per advertisement frame (an entry is ~30 bytes without it, so 32
keeps four per frame and 64 drops it to three).

**What the name MEANS is not in this spec, and that is the boundary the whole design rests on.** The
protocol provides the tools to package, deliver and structure content -- the content-type and
compression nibbles, the sizes, the hashes, the ranges. What the content is FOR is the
application's, and a vendor's naming is precisely the thing a free-text field exists to let them do
their own way.

As **recommended practice** only, an OTA name of the shape `<app>-<platform>-<semver>` --
`sensor-esp32c3-0.9.9` -- works well, because an endpoint already reports exactly those three things
in `IOTDATA_NODE_TLV_VERSION`, so "is this for me?" is a comparison against values it already knows
about itself.

**A patch carries its own base, inside the patch.** There is no `base` field: a bsdiff or any other
patch format already states what it applies to, so duplicating it on the wire would be a second
source of truth for something the payload must get right regardless. The cost is named rather than
hidden -- an endpoint cannot tell from the manifest alone whether a patch matches its current build,
so either the name says so by convention (`sensor-esp32c3-0.9.8-to-0.9.9` takes the same shape as
the whole-image name) or the endpoint fetches and finds out. For the common case, where an
advertiser offers a patch and a whole image side by side as two flavours, the name is what an
endpoint compares and nothing is lost.

**Platform and version are deliberately NOT structured fields.** Making them real fields would buy
exact matching at the price of the protocol owning a platform registry and a version format, which
is a great deal of spec surface for something a string comparison handles -- and the constraint
would leak into every non-OTA use of content, where neither concept means anything.

### Packed positional, and what that costs

The codebase has both conventions and uses them for opposite reasons: STATUS, CONTROL and RECEIVE
are **kvr** (low volume, likely to grow, optional fields free when absent), while TABLE rows are
**packed positional** (repeated, fixed, high volume). One-manifest-per-TLV puts the manifest on the
packed side with the transfer, because the thing kvr is good at -- a variable set of optional
fields in one container -- stopped being the shape of the problem once the container held exactly
one entry.

**What kvr bought, and how much survives.** A kvr lets a field be added later and skipped by old
readers. Positional packing keeps a weaker form of that for nothing: the TLV carries its own
length, so a field **appended** after the name is simply never read by an old decoder, which stops
when it has consumed what it knows. Two rules make it work -- new fields go at the end, never
inserted among the existing ones, and the escape is one-way, since an old encoder cannot emit what
it does not know about. That is enough for a format whose shape is now fixed by the frame budget
rather than by negotiation.

**The content-id is u24** (§4 fixes the width), and it is the header's, not a field of its own.

### Several flavours are several TLVs, and need no fragmentation

Nine flavours at ~56 bytes with a 32-byte name is ~500 bytes, which does not fit one 233-byte
frame, so a full offer spans frames however it is encoded. It needs **no fragmentation mechanism**:
each frame carries as many COMPLETE manifests as fit -- four with 32-byte names -- and every frame
is independently useful, because each MANIFEST TLV is a whole statement about one content-id and the
receiver accumulates them by id. An advertiser with nine
flavours sends three frames. Inventing fragmentation for this would buy nothing.

### Who allocates a content-id

Not mandated by the protocol -- the id is opaque on the wire -- but **recommended as
`(station << 12) | counter`**, giving every station 4096 ids of its own. Any station that originates
content owns its id space, gateway, relay or endpoint alike, which is what lets the upward direction
work with no second mechanism: a sensor offering a calibration blob mints ids exactly as a gateway
does.

It also comes with a free validity check: station 0 is unassignable and 0xFFF is broadcast, so a
content-id whose top 12 bits are `0x000` or `0xFFF` is malformed by construction.

The alternative considered was a content-addressed id (the truncated hash of the object), which
would let two gateways agree on an id for the same bytes and share a cache. Rejected for now
because 24 bits gives roughly a 1% birthday risk at ~600 concurrently live objects, and the benefit
only appears where two gateways serve one fleet -- where duplicated caching is wasteful rather than
wrong. The failure mode would at least be bounded: a size or hash mismatch in the manifest, so
wasted airtime rather than a wrong flash.

**Expiry is an offset, never an absolute time.** Every node has a monotonic clock and none of them
need a shared time base; an advertisement floods within seconds, so offset-from-receipt is as
accurate as a wall-clock stamp and costs no RTC. Anything wanting to state a real time has
`IOTDATA_NODE_TLV_TIMESTAMP`.

**Minutes, not seconds.** u16 seconds caps at 18 hours, and §11 shows a 512 KiB transfer taking two
days at a 1% duty cycle -- the offer would expire mid-transfer. Minutes give 45 days, which covers
the worst transfer plus a node that only wakes daily, at a resolution far finer than anything an
expiry decides.

**0xFFFF means indefinite**, in keeping with the all-ones convention used for the downstream
sequence and the broadcast station. It is likely to be the COMMON case rather than an edge one: an
OTA should stay available until it is replaced. The consequence is that an indefinite entry never
ages out, so **eviction is its only backstop** -- if an advertiser dies without revoking, the entry
is immortal until a cache runs out of room. That makes the intermediate cache's own capacity bound
load-bearing rather than merely tidy.

**Revocation is expiry = 0.** Re-advertise the content-id with a zero expiry and every holder, cache
and requester drops it. No separate mechanism. A revocation differs from a merely-expired entry in
one way: it must ACTIVELY evict a cached copy, where an expired entry need only stop being
forwarded. It must also **destroy rather than mark** -- a cache that survives a reboot and kept the
revocation only as a live flag comes back serving revoked bytes.

**A manifest entry supersedes what a holder already has for that content-id.** If an entry arrives
for an id a node holds bytes for, and its total size or gross hash differs, the held bytes are
destroyed. That is what makes id reuse safe, since the recommended `(station << 12) | counter` wraps
after 4096; without it a cache serves the old object's chunks under the new id and the requester
fails verification over and over with nothing to tell it why. It also gives "content replaced in
place" for nothing: re-advertise the same id with new bytes and every holder discards and
re-fetches.

**The offset is NOT decremented when forwarded.** A manifest floods, and a flood propagates in
seconds, so adjusting the offset hop by hop would be arithmetic in exchange for nothing.

What keeps that safe is a constraint rather than a calculation: **only the advertiser originates an
advertisement, and everyone else forwards it verbatim.** An advertisement is not requestable -- there
is no "send me your manifest" -- so nothing downstream ever re-states one, and the offset cannot be
restarted by a cache that held the entry for a while. A relay that re-originated an advertisement
from its cache WOULD extend the lifetime indefinitely, so it must not; what a relay answers is range
requests, nothing else.

**Which makes periodic re-origination by the advertiser a requirement, not an optimisation.** An
advertisement's reach in time is the down path's broadcast slot expiry, NOT the manifest's own
expiry: it is a broadcast down frame, so once it ages out of the relays' slot tables nothing
re-delivers it to anybody. An entry with a 45-day or `0xFFFF` expiry is therefore invisible to a node
that joins the network, reboots, or comes back into range after that point -- and having decided an
advertisement is not requestable, that node has no way to ask. So the advertiser re-sends on an
interval comfortably shorter than the broadcast slot expiry. The cost is a few frames a week; the
alternative is a node that can never discover content that is sitting there indefinitely.

**A re-origination must bump the MANIFEST tag**, which is the generation (§5). Being shorter than
the slot expiry puts every refresh inside the dedup lifetime by construction, so a byte-identical
re-advert would be thrown by every relay that still remembers the last one. The generation is what
makes the frame differ.

## 4. The transfer: 32-byte chunks, fixed

An object is divided into **32-byte chunks, system-wide, with no negotiation.** Simplicity first;
the protocol can be revised after testing.

Chunks are addressed by a **u16 index**, which fixes the ceiling:

```
65536 chunks x 32 bytes = 2 MiB maximum object
```

A realistic object for these devices is **up to about 512 KiB** -- a 1 MiB OTA is unlikely on this
class of hardware -- so the ceiling is headroom rather than a constraint. Anything beyond it needs a
protocol revision, not a workaround.

Frame budget, on a 240-byte E22 sub-packet:

```
240 - 5 (header + presence) - 2 (TLV header)   = 233 bytes of value
   - 4 (the 32-bit content header)
   - 3 (one run descriptor)                    = 226
   -> 7 chunks x 32 = 224 bytes of data per frame, 2 spare
```

So a full CHUNKS frame is 238 of 240 (the table in §5), and an eighth chunk is unreachable -- it
would need 256 more bytes, not 2. What the u24 content-id buys is keeping that header at four bytes
rather than five, which is the byte that would otherwise have to come out of the payload -- see §3.

**Why 32 rather than 16**, since both were candidates. A frame's payload must be a whole number of
chunks, and 8, 16 and 32 all pack a frame identically well (224 of 240 bytes; 64 wastes 13% of every
frame, 128 wastes 40%). So between 16 and 32 the frame count is *identical* for every object size,
and so is the re-fetch cost of a bad signature block, which the block size sets rather than the
chunk. What differs is that 32 halves the have-bitmap and doubles the index ceiling. The one thing
16 buys, finer retransmission granularity, is moot: loss is frame-granular, so a lost frame loses
all seven chunks either way.

The **final chunk of an object is short** whenever the size is not a multiple of 32, and so is any
range containing it. The receiver clamps both from the manifest's total size; the wire has no
special case for either.

## 5. The wire formats, and why the tag exists

All three sub-kinds share **one 32-bit header**, so a decoder reads four bytes and switches on a
nibble:

```
header (every sub-kind):   sub-kind:4 | tag:4 | id:24

MANIFEST : header | size:u24 | hash:u64 | sig-id:u24 | expiry:u16 | encoding:u8
                  | name:NUL-terminated                            (§3)
RANGES   : header | maxframes:4 | reserved:4 | [ start:u16, count:u8 ] * N
CHUNKS   : header |                            [ start:u16, count:u8, data ] * M
```

**The `id` is always the content-id**, in all three. **The `tag` means something different in each**,
which is what lets one header serve all of them:

| sub-kind | what the tag is |
|---|---|
| MANIFEST | the **generation** -- see below |
| RANGES | the requester's tag for this attempt |
| CHUNKS | the tag of the request being answered, echoed back |

**Neither carries a range count**, because both are derivable. In RANGES, `N = (len - 5) / 3` since
the descriptor is a fixed three bytes. In CHUNKS the runs are **interleaved**, each one's data being
`count x 32` bytes, so the structure walks itself until the TLV length is exhausted -- a single-pass
parse with nothing to disagree with itself, rather than a count field that could contradict the
payload. That the data runs are then non-contiguous costs nothing: each goes to a different offset
in the object, so a receiver was always going to copy them out one at a time.

**`count` stays u8 in both**, and the reason is asymmetric. A descriptor cannot be smaller than
three bytes whatever `count` is, since `start` needs sixteen bits on its own -- so narrowing it to
four bits only leaves the space idle. Meanwhile in RANGES a count is *not* bounded by one frame
(`maxframes` bounds the response instead), so u8 lets one descriptor ask for a 255-chunk run: a
whole window's worth of contiguous fetching in three bytes.

**`maxframes`**: 0 means 1, so a zeroed or forgotten field is conservative rather than bursty;
1-15 mean themselves. It is a CEILING, not a demand -- a responder sends the least of `maxframes`,
what the ranges call for, and what the requester's advertised window can carry (§10).

**Request order is priority, but a preference rather than a rule.** A responder fills in the order
given and SKIPS what it cannot serve, which is what lets a cache answer with the part it holds
instead of staying silent and forfeiting the point of caching. The requester never has to guess,
because the response echoes the ranges it actually served.

| frame | overhead | data | total of 240 |
|---|---|---|---|
| CHUNKS, one run | 7 | 224 (7 chunks) | 238 |
| CHUNKS, two runs | 10 | 192 (6 chunks) | 202 |
| RANGES, 4 ranges | 17 | -- | 24 |

Note what the second row says: **a fragmented answer carries a whole chunk less than a contiguous
one**, which is a standing incentive for a responder to prefer long runs and for a requester to ask
for contiguous ranges. An eighth chunk is unreachable in any case -- it would need 256 more bytes --
so 238 is the shape of a full data frame.

### The MANIFEST tag is a generation, and it is load-bearing

A manifest has no request to match, so its tag carries the **generation**: incremented by the
advertiser each time it re-originates the same offer.

Without it the periodic re-origination §3 requires **cannot work at all**. A re-advert is
byte-identical to the one before, downstream identity is the hash of the whole frame, and the
re-origination interval must be SHORTER than the broadcast slot expiry -- so every refresh lands
inside the dedup lifetime by construction and is thrown by every relay that still remembers it.
Guaranteed suppression, every time, for the one message whose whole job is to keep arriving.

Bumping the generation makes the frame differ, which is exactly the mechanism
`IOTDATA_NODE_TLV_TIMESTAMP` exists for (NOTES_DOWNSTREAM §7) -- but free here, since the field was
otherwise idle. Four bits wrapping after sixteen generations is ample when generations are days
apart, and it gives a receiver a cheap way to tell a refresh from a changed offer without comparing
size and hash.

### Why the tag exists at all

**Downstream identity is the content hash** (NOTES_DOWNSTREAM §2). Without a tag, a lost response
that is re-requested produces a byte-identical frame, which every relay correctly throws as a
duplicate, and the transfer stalls until the dedup memory forgets it. The tag makes each attempt a
distinct frame. It is also the requester's only means of telling a fresh answer from a cached
replay.

### The tag is four bits, and that is conditional

Four bits wraps after **sixteen** requests. The tag's protection lasts only as long as the dedup
memory, which is bounded by the TTL *and* by the slot count -- and with 16 slots a station's chunk
frames evict each other after about sixteen frames. Those two numbers are the same order, which is
uncomfortably close: in the gap-filling phase at the end of a transfer, where scattered holes are
re-requested, a hole asked for again sixteen requests later can land on a tag whose earlier hash has
not yet been evicted, and that hole is then unfetchable until it is.

So the four-bit tag is taken **together with a short dedup lifetime for CONTENT frames -- of the
order of a minute** (§6). That is independently correct: a chunk frame's flood completes in seconds,
so remembering it for a day wastes dedup slots that are thrashing anyway. With a one-minute window
the tag-width question retires entirely.

> **TO TEST, deliberately.** This is the one decision here resting on an interaction rather than on
> arithmetic. The gap-filling phase must be exercised on real hardware with real loss -- scattered
> holes, repeated re-requests, several relays answering -- and the tag wrap watched for suppressed
> responses. If it bites, the fixes in order of preference are: shorten the content dedup lifetime
> further; widen the tag to eight bits at the cost of the unified header; or give content its own
> dedup table.


## 6. Content is an ordinary down frame

A chunk response floods, dedups and holds exactly like a command, and that is deliberate:

- a response can arrive **after the window that prompted it has closed** — propagation is not
  instant — and holding it is what saves the transfer rather than wasting it
- several relays may have heard the same request and answered it, so those byte-identical responses
  **must** dedup against each other, or every response is amplified N-fold. Dedup alone only
  discards the copies after they have been paid for; what stops them being transmitted at all is
  the down path's **answer jitter** (NOTES_DOWNSTREAM §4), and content is the case that jitter
  exists for -- independently generated answers fire at the same instant, where a flood's copies
  are naturally staggered

The station is in control: it paces its own requests against its battery, its window and what else
it is receiving. That is the system's premise and content does not get an exception.

**One thing content DOES need from the down path: a short dedup lifetime.** A chunk frame's flood
completes in seconds, so the day-long lifetime a command wants is wrong for it in both directions --
it wastes dedup slots that a transfer is thrashing regardless, and it is what makes the four-bit tag
(§5) marginal. CONTENT frames get a lifetime of the order of a minute. That is the cheaper half of
the BULK class and it is required, not deferred.

**Known residual, deferred**: with one shared slot table, a burst of chunk responses can still evict
an undelivered command, and the station cannot see that happening. The fix when it matters is the
other half of the same class bit, used purely as an eviction preference -- bulk evicted before
command. Not built for the first pass.

It has a second symptom worth recognising, because it presents as latency rather than as loss:
slots are delivered oldest-first, one per window, so a command arriving behind a burst of chunk
responses waits behind every one of them. Same bit, same deferral -- but the trigger to build it is
as likely to be a command that took an hour as one that vanished.

## 7. Integrity: a flat block-hash list, not a tree

The signature stream is another content-id whose bytes are **8-byte hashes of fixed 2 KiB blocks**
of the target object, in order: hash *k* lives at offset *8k*.

2 KiB is 64 chunks exactly, so a block boundary is always a chunk boundary and verification needs no
partial-chunk arithmetic.

**The block size is chosen for the cost that is ALWAYS paid, not the one that is almost never
paid**, which is the opposite of the intuition. The link already provides integrity: LoRa CRCs every
frame, so the protocol's problem is lost frames rather than corrupt ones. A 16-bit CRC admits
roughly 1 in 65536 corrupt frames, which at a 10% frame error rate is about a 0.7% chance of one
escape per 512 KiB transfer, and 0.02% for a 20 KiB delta. So the stream wants to be small (paid
every time) and the re-fetch is allowed to be large (paid almost never):

| block | blocks (512 KiB) | stream | overhead, always paid | re-fetch on mismatch |
|---|---|---|---|---|
| 512 B | 1024 | 8 KiB | 1.56% | 2 frames = 2 s |
| 1 KiB | 512 | 4 KiB | 0.78% | 5 frames = 4 s |
| **2 KiB** | **256** | **2 KiB** | **0.39%** | **10 frames = 8 s** |
| 16 KiB | 32 | 256 B | 0.05% | 74 frames = 58 s |

Per-chunk hashing is not a candidate at all: at 32-byte chunks it would be 128 KiB of hashes for a
512 KiB object, a 25% overhead.

At 2 KiB blocks and 32-byte chunks the stream overhead and the have-bitmap overhead come out
identical at 0.39%, so the two costs stay symmetric under further tuning.

**A signature stream is MANDATORY to provide and optional to use.** It is not the advertiser's
discretion to omit -- `sig-id` is a required manifest field (§3) -- because the stream exists
whether or not anyone fetches it, so offering it costs nothing, and it keeps the airtime decision
with the party that pays for it, which is the same principle as a station pacing its own requests.

**The one exception is the base case**: a signature stream's own manifest carries `sig-id = 0`,
since hashing the hashes would recurse without end. Its integrity comes from its manifest's gross
hash instead, which is what makes the recursion terminate cleanly rather than by special pleading. It also lets a node decide per transfer -- a
mains-powered relay can verify everything, a sensor on its last 10% can take the gross hash and
accept the risk of a whole-object re-fetch.


**A flat list, not a Merkle tree.** A tree earns its keep when you cannot fetch arbitrary parts of
the hash set; here the hash set *is* a range-fetchable content stream, so a receiver fetches the 8
bytes it needs for the block it holds, which is cheaper and simpler than a tree path. The list's own
integrity comes from its manifest entry's gross hash, so there is no chicken-and-egg.

**The algorithm is CRC-64/XZ**, for the gross hash and the block hashes alike. Throughput made no
difference to the choice -- 224 bytes per 0.8 s frame is about 280 B/s, so even 100 cycles a byte is
free -- so it rested on three other things. A CRC gives *provable* detection of short bursts, which
is what frame corruption looks like, where a general-purpose hash gives only a probability. A
different polynomial from the link's CRC-16 makes the two detections largely independent, and an
escape from the link layer is exactly what this exists to catch. And it has published test vectors,
so a gateway in C on Linux and a sensor on an 8-bit part can be *proven* to agree -- which matters
when the failure mode is "every block fails verification".

Truncated SHA-256 was rejected for a reason that is not technical and matters more: it would LOOK
like a security control while being 2^32 of collision work, inviting someone to believe the manifest
is authenticated when authenticity deliberately lives inside the payload (§8). A CRC makes the
intent unmistakable.

Eight bytes is chosen for integrity, not authenticity — see below — and should not be "upgraded" to
32 without recognising that it quadruples the stream.

## 8. The security model, stated as a decision

- **Integrity on the air**: the gross hash, and optionally the block-hash stream.
- **Authenticity inside the payload**: an OTA's own firmware signature lives *within* the package
  and is checked by the OTA code before anything is flashed.
- **Therefore the protocol needs no cryptography at all**, which is a large simplification and the
  reason the split is worth stating explicitly rather than leaving implicit.

The accepted cost: anyone can advertise a content-id and serve garbage, and the integrity check
catches it only *after* the node has spent the airtime and battery. Nothing without authenticity can
avoid that; it is named here so it is a decision rather than a discovery.

## 9. One module, one code path, and a backend per node

`iotdata_node_content.h` is initialised with a **backend**: a small set of accessors over the byte
ranges of a content-id. Everything above that seam is the same code on every node. There are **no
compilation options to enable or disable features**, and none are needed -- the nodes differ in
*what gets called*, not in what exists. A sensor never advertises and never serves a range, but the
advertise and serve paths are `static inline`; if nothing calls them they are never emitted. The
asymmetry lives entirely in the call pattern and the backend.

The one legitimate compile-time knob is the CRC-64 table: 2 KiB table-driven against a bitwise loop,
for a part that cannot spare the 2 KiB. That is a size/speed choice, not a feature.

**Gateway, relay and sensor are the same thing here.** They run identical logic and differ only in
how much of the object they start with and how large a pool they have to snoop from: a gateway is
handed the whole thing, a relay accumulates what passes it, a sensor accumulates what it asked for.
A relay's cache is a sensor's partial transfer that never completes and is never consumed -- the
same content-id keying, the same have-bitmap, the same ageing. So "abandon a transfer" and "evict a
cache entry" are one operation, and no cache-specific machinery is needed anywhere. Because a
content-id is just an integer, a file named for it is a complete store -- which also gives you
content loaded by copying a file, or baked into an image, with no extra mechanism.

### The seam

```c
typedef struct {
    bool     (*have)(uint32_t id, uint16_t chunk);
    int      (*read)(uint32_t id, uint32_t off, void *buf, size_t len);
    int      (*write)(uint32_t id, uint32_t off, const void *buf, size_t len);
    bool     (*create)(uint32_t id, uint32_t total);
    void     (*destroy)(uint32_t id);
    bool     (*complete)(uint32_t id);
    uint32_t (*space)(void);
} content_backend_t;
```

Three of those exist for reasons that are not obvious from "get and set any byte range":

- **`complete()` is what finalisation means, and it is per node.** The module can say "this id is
  whole and verified"; what happens next is not its business. A sensor marks an OTA partition valid
  and sets the boot flag, a gateway closes and fsyncs, a relay does nothing at all. Without this
  hook the application has to re-derive completion, which is exactly the OTA-specific logic that
  has to stay out of the module.
- **`have()` behind the seam is what makes the gateway cheap.** The have-bitmap must survive a
  reboot and the awkward question is where it lives. Behind the seam, that stops being one
  question: a gateway holding whole objects answers *always true* and stores no bitmap at all,
  while a relay or sensor keeps one wherever its own storage allows.
- **`read()` is required for verification, not only for serving.** To check a 2 KiB block the module
  must read back what it wrote, so any backend must be able to read whatever it can write. A
  write-only store cannot participate.

### What the backend owns, and what it does not

The backend owns storage, and those are its own policies with its own configurables (§12): where
the bytes live, the layout, write alignment and granularity, whether it persists at all, whether it
is writable at all (a read-only image-baked store can serve forever and never accumulate), its
capacity, the bitmap if it needs one, and what finalisation means.

The module owns the protocol: ranges, tags, the manifest table, demand tracking, block verification,
and the choice of *which* content to give up -- because demand is a signal only the module sees. So
capacity is the backend's question and priority is the module's: the module asks whether something
will fit, the backend says no, and the module picks the victim by least-recent demand and calls
`destroy()`.

### The three backends, and a fourth for tests

| node | store | starts with | finalisation |
|---|---|---|---|
| gateway | filesystem | whole objects, handed to it over MQTT | close and fsync; `have()` is always true, no bitmap |
| relay | partition | nothing; what passes it | none -- it never consumes what it holds |
| sensor | the partition it will flash from | nothing; what it asked for | mark the OTA slot valid, set the boot flag |

Relay and sensor are both ESP32 flash and are still **not** the same backend: one is an evictable
cache that is never read by its own node, the other is a single transfer that ends in a boot
decision.

A fourth **RAM backend, for the host harness**, is worth building first. It puts the whole module --
ranges, tags, the 4-bit tag wrap flagged in §5, gap filling, block verification, several holders
answering one request -- under host tests with scattered loss simulated and no hardware involved,
which is how everything else in this tree is proven.

**Caching in intermediates is optional and valuable.** A relay snoops chunk frames to fill its own
store, answers requests locally when it can, and snoops manifests to expire or drop what has been
withdrawn. A fleet behind one relay pulls the image over the **backhaul** once -- though the relay
still repeats it to each sensor in turn, which §11 shows is the cost that actually dominates.

## 10. The requester drives, and paces itself

- at most **one request outstanding** — the responder is then entirely stateless
- ask for what **fits the window that is open**, using the same airtime arithmetic the down path
  already uses (`iotdata_node_receive_fits_ms`)
- keep a **have-bitmap**, which must persist: a transfer spans many wake cycles and probably a
  reboot. At 32-byte chunks a 512 KiB object is 2 KiB of bitmap and a 20 KiB delta is 80 bytes.
  Since the requester asks for contiguous ranges, reception is mostly in order, so a **high-water
  mark plus a short list of holes** holds the same state in tens of bytes and persists trivially --
  worth preferring before the bitmap becomes the thing that forces a filesystem onto a sensor. The
  module never sees any of this directly: it asks the backend's `have()` (§9), so each node
  represents the state however its own storage allows, and the gateway represents nothing
- verify a block as soon as its 64 chunks are present and its hash is known; re-request on mismatch
- the module's tick eventually reports **content available**, and the application asks it what
  arrived and hands the name to whatever consumes it

### The request is an ordinary uplink, and is answered promiscuously

A RANGES request is **not a frame of its own**. It is a TLV piggybacked on a cycle the station was
sending anyway, under the station's own advancing sequence -- one station, one sequence, unchanged
-- so asking costs no extra transmission and no extra wake.

It is answered **promiscuously**: whoever hears it and holds the bytes may answer, gateway, relay or
endpoint alike. That is the whole mechanism by which a cache is useful, and it needs no addressing,
no discovery, and no knowledge anywhere of who holds what.

**A request SHOULD carry a RECEIVE TLV. It must not be required to**, because the request and the
window are decoupled in time and the down path's hold is what decouples them:

```
cycle 21:  sensor sends data + RANGES, and does NOT open a window
           a relay hears it, builds the response, and holds it as an ordinary down
cycle 22:  sensor sends data + RECEIVE, and listens
           the relay's window snoop fires and delivers the held response
```

So a station may ask now and collect whenever suits its power budget, and a station that asks on
every twentieth cycle and listens on the one after is behaving correctly rather than exploiting a
loophole. The responder sizes its answer from the last window it snooped from that station, or from
the system default when it has none -- which makes `maxframes` conservative rather than wrong -- and
the response waits in a slot either way.

**A direct request is itself a witness.** The receive path discards a frame for a station it has
never snooped a window from, on the grounds that it cannot deliver to it (NOTES_DOWNSTREAM §4, step
5). A response built for a request we heard **directly** must be exempt: hearing the uplink is
proof the station is our neighbour, and without the exemption the sequence above would throw the
answer away one cycle before the window opens. A request arriving wrapped in a mesh FORWARD proves
nothing -- its sender is not our neighbour -- and is left to the ordinary rules; the answer still
floods, and whoever is that station's witness delivers it.

## 11. Airtime, which disciplines all of the above

At 2.4 kbps, a 240-byte frame is roughly 0.8 s on air, carrying 224 bytes.

| object | frames | pure airtime | at 1% duty |
|---|---|---|---|
| 512 KiB image | ~2340 | ~31 min | ~2.1 days |
| 200 KiB image | ~915 | ~12 min | ~20 h |
| 75 KiB image | ~343 | ~4.5 min | ~7.5 h |
| 20 KiB bsdiff delta | ~92 | ~74 s | ~2 h |
| 5 KiB calibration blob | ~23 | ~19 s | ~30 min |

Frame counts are identical whether chunks are 16 or 32 bytes, since both fill a frame with 224.

So: encodings are not a nicety, they are the difference between feasible and not; expiry must
exceed the transfer duration with room to spare; and a relay cache is close to a necessity for a
fleet rather than an optimisation.

### The last hop does not amortise, and that is accepted

A relay cache takes the object off the **backhaul** once. It does nothing for the hop to the
sensors: the relay still transmits the whole object once per sensor, and that cost scales with the
fleet.

| | pure airtime | at 1% duty |
|---|---|---|
| 512 KiB to one sensor | 31 min | 2.1 days |
| 512 KiB to ten sensors | 5.2 h | **21 days** |

**Broadcast push was considered and rejected.** Pushing chunks to the broadcast station would serve
a whole fleet in one transmission, and the encoding already allows it, since CHUNKS is an ordinary
down frame. It fails on the receivers rather than on the sender. Endpoints wake, receive and sleep
on their own schedules, so a broadcast reaches only whoever happens to be listening at that instant,
which in a fleet of independently-cycling sensors is almost nobody. Making it work would mean
synchronising the fleet into a common receive window -- a scheduling mechanism and a shared time
base this system deliberately does not have -- and it would buy that by taking away the property the
whole design rests on, which is that the station decides when it listens.

So the cost stands, and the decision is to **state the scale of the system rather than engineer
around it**. This is a low-bandwidth, telemetry-oriented network. Large transfers, OTA included, are
the exception rather than the workload; they are long-winded by nature; and they should be
infrequent. Relay caching is the right answer for the backhaul and for a gateway serving several
relays, and the last hop is simply what it costs.

## 12. Configurables

They fall into two groups, and the split matters: the first group is the protocol and has to agree
across the mesh, the second is one node's private policy and nobody else can see it (§9).

Protocol, shared by everyone:

| knob | what it bounds |
|---|---|
| chunk size | **fixed at 32 bytes, not configurable** (16 was the alternative, see §4) |
| signature block size | 2 KiB, and a multiple of the chunk |
| manifests tracked | how many distinct content-ids a node keeps offers for |
| content-ids tracked | concurrent transfers and cache entries per node |
| max ranges per request | and with it the request frame size |
| content-id width | **fixed at u24**, which is what makes the frame land on 240 exactly |

Backend policy, private to one node:

| knob | what it bounds |
|---|---|
| `DOWN_CACHE_BYTES` | how much content this backend will hold, its only real limit |
| `CONTENT_CACHE_TTL_MS` | this backend's own data lifetime, capped by the manifest expiry |
| write granularity | the flash backends' alignment; the filesystem's is 1 |
| whether it persists, and whether it is writable at all | a read-only baked store serves and never accumulates |

## 13. Encoding shape, and the type space

**One CONTENT type (0x07) with a leading sub-kind nibble** -- MANIFEST / RANGES / CHUNKS -- rather
than three TLV types. That is the mechanism the system TLVs use generally, and it spends none of the
scarce type space. It costs nothing on the wire either, since the nibble shares the 32-bit header
with the tag and the id (§5).

**`request(CONTENT)` asks a node what it holds**, and the answer is a list of manifest entries --
which content-ids it has and what they are. That is a third thing alongside the two below it: an
advertisement is *pushed* by a holder, the bytes are *fetched* with RANGES, and this *enumerates*.

**An advertisement states an expiry; a response reports the remainder.** §3 forbids decrementing the
offset on a flood, because a flood takes seconds and the arithmetic would buy nothing -- but that
reasoning is false for a cache that has held an entry for a day, and such a holder knows exactly how
long it has been, since NODE's `status.content` obliges it to track that anyway. So a
`request(CONTENT)` response carries the expiry it has **observed remaining**, where an
advertisement **states** one from receipt. Same field, different semantics, and the rule that only
the advertiser originates an advertisement is untouched. See NOTES_NODE §3.8.

```c
#define IOTDATA_CONTENT_MANIFEST  0x0   /* one manifest, one content-id                    */
#define IOTDATA_CONTENT_RANGES    0x1   /* which chunks are wanted                         */
#define IOTDATA_CONTENT_CHUNKS    0x2   /* the chunks themselves                           */
```

**The names state what the TLV IS, not what it is doing.** A CHUNKS frame carries chunks whether it
was pushed, pulled, advertised or answered, and a RANGES frame is a set of ranges whoever sent it
and whichever direction it is travelling -- so neither name has to change for the upward case, or
for any future use that moves them for a different reason. Direction itself needs no values of its
own: it is carried by the sequence field, so the upward case reuses the same three kinds unchanged.

**Content is a node extension; the down path it rides on is core protocol.** The framing convention
and the flood-and-hold rules belong to the base, because a station cannot be addressed at all
without them (NOTES_DOWNSTREAM §1). The CONTENT TLVs and everything in this note sit above that, in
the same relationship the mesh has to the base: additive, and a node that never takes content is
still a correct node. The one thing content asks of the core -- the short dedup lifetime of §6 --
is a configurable of the core, not a content mechanism, and so is the answer jitter it relies on.

If a relay ever wants to filter on bulk versus metadata at the type level -- a plausible cache or
airtime policy -- promoting CHUNKS to its own type is a one-line change that leaves the manifest and
request encodings untouched. Un-spending a type slot is not.

> **DONE (2026-09-26), and it affects this space.** `IOTDATA_NODE_TLV_MESH_STATIONS` (0x08),
> `_MESH_PEERS` (0x09) and `_MESH_FILTERS` (0x0A) have been **folded into STATUS** and retired.
> They are now STATUS keys. The key space is **two linear ranges and nothing finer** -- `0x00..0x3F`
> any node, `0x40..0x7F` the mesh -- packed in the order things are invented, with no scalar/table
> sub-ranges and no arithmetic relating one key to another. Stations and filters sit in the ANY NODE
> range because neither is a mesh concept; peers is the only table in the mesh range. A table is
> described by an explicit `{count, entry, width, scope}` record, not derived. Asking for a table is
> `STATUS_REQUEST` with a scope bit, so their three request commands went too, and the per-type
> `row_size`/`is_table` machinery no longer exists.
>
> **Counts are scalars, and every scalar is emitted before any entry**, so a report whose tail is
> lost still says how big each table was. A caller asking only for a table's entries gets no count,
> and that is its own choice.
>
> **0x08-0x0A are free but NOT renumbered.** `TIMESTAMP` stays at 0x0B, so `SYSTEM_COUNT` is still
> 0x0C and there are three holes inside it. Moving TIMESTAMP down would shrink every per-type array
> by a quarter, but `0x08 << 3` is 0x40, which is `CONTROL_MESH_BASE` -- so the renumber needs the
> mesh CONTROL range moved to 0x48 first. Left for the node normalisation pass.

## 14. What has to change

- `iotdata_node_content.h` exists as a commented-out include in every app; it becomes the module
  described here.
- The CONTENT TLV gains the three sub-kinds -- MANIFEST, RANGES, CHUNKS -- and their codecs. The
  manifest is bitpacked positional, one per TLV, repeated per flavour (§3); it is NOT a kvr.
- Four backends behind one seam (§9): RAM for the host harness first, then the gateway's
  filesystem, then a sensor that can complete a transfer, then the relay's cache.
- The down path needs nothing new for the first pass except what it wanted anyway: the answer
  jitter (NOTES_DOWNSTREAM §4), a short dedup lifetime for content frames (§6), and the exemption
  that keeps a response to a directly-heard request (§10).
- **The advertiser re-originates its advertisement periodically** (§3) -- gateway application
  behaviour, not module behaviour, and the only thing that lets a late joiner discover content.
- A manifest entry that differs in size or gross hash **destroys** what a holder has for that id
  (§3).
- The eviction class bit (§6) when bulk starts displacing commands in practice.
- The upward direction, when something needs it.
- The chunk size is the parameter most likely to be revisited after testing. 16 bytes remains the
  alternative and changes nothing structural: only the bitmap size, the index ceiling and the
  chunks-per-block constant.
