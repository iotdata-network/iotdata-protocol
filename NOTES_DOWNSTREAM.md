# NOTES — The Downstream Path: Flooding, Holding and Delivery

> Working note, 2026-09-17. Settles the downstream mechanism that
> [NOTES_DIRECTIONALITY](NOTES_DIRECTIONALITY.md) reasoned toward but left unspecified. That note
> established WHY downstream identity has to be content-addressed; this one specifies WHAT a
> gateway or relay does with a down frame, end to end. To be folded into §5 and Appendix H.

---

## 1. Premise: down was not designed for, and that is fine

The protocol is one-way telemetry from constrained devices. The framing carries station, sequence
and variant, and nothing else: the sequence exists for loss detection, and in a mesh it doubles as
the deduplication key. The mesh is an optional drop-in above that, giving an upstream routed graph
with or without acknowledgement.

Downstream is the rare beast, and it is retrofitted onto that framing by one convention:

> **`sequence == IOTDATA_SEQUENCE_DOWN` (all ones) inverts the meaning of the station field.**
> The station is who the frame is FOR, not who it is FROM.

Everything below follows from that single bit of reinterpretation, and from its consequence: a down
frame has **no identity of its own**. Its sequence is invariant, so two downs to the same station
are distinguishable only by their content.

In a point-to-point deployment this is nearly all that is needed — a station screens arriving
frames and accepts the ones addressed to it. The marker is kept universal even there, so that one
station implementation works in both topologies without knowing which it is in.

**Where this lives.** The downstream path described here is **core protocol**, not an extension: it
is the framing convention plus the flood-and-hold rules that every node needs before anything can be
addressed to it at all. What rides on it -- CONTENT's advertisement, request and transfer TLVs and
their machinery -- belongs to the **node extension**, in the same way the mesh is an additive layer
above the base rather than part of it. The test is whether a node can be a correct participant
without it: a station must understand a down frame addressed to it, so down is core; a station that
never takes content is still a correct station, so content is not.

## 2. The dedup key is the whole frame

Because the sequence is invariant, the deduplication key is **the hash of the entire frame** —
which is exactly `(station, sequence=1s, variant, content)`, and gets broadcast right for free
because the broadcast station id is in those same bytes.

The consequence, and it is a feature before it is a problem: **two identical downs are one down.**
Re-sending the same command inside the dedup lifetime is a no-op. For idempotent, state-relative
commands ("clear diagnostics up to record N") that is what you want.

For a command that is *meant* to happen twice — a reboot — it is an obstruction, and the protocol
answers it by letting the sender make the frame unique on purpose: see §7.

## 3. The flood, and what stops it

Down is **flooded, not routed**. It enters the mesh anywhere and is redistributed by every gateway
and relay that hears it. There is no hop count — the header has no room for one — and none is
needed, because deduplication terminates the flood by itself:

```
A hears a down, it is new to A     -> A rebroadcasts once, remembers the hash
B hears A's copy, new to B         -> B rebroadcasts once, remembers the hash
A hears B's copy, a dup for A      -> A throws it, silently
```

Propagation is therefore bounded by the number of nodes rather than by a distance, and each node
puts each distinct frame on the air **exactly once**. That single rebroadcast is also the first
delivery attempt: it reaches every listening relay, and any station that happens to be awake.

## 4. Holding: the station table

A node holds downs in a table of **stations**, each with a small number of **slots**. Broadcast is
not a special case in that table — it is a pseudo-station whose id is `IOTDATA_STATION_BROADCAST`,
holding slots like any other.

```
station entry:  station id | last-down time | receive-window state | slots[] | broadcast-sent[]
slot:           frame hash | arrival time    | buffer (or none)
broadcast-sent: broadcast hash | time it was given to THIS station
```

A slot with no buffer is pure dedup memory: we remember having seen the frame without keeping it.
That is the common case, and it is what bounds the memory cost of a wide flood.

### The receive path, in order

1. **Hash the frame.** If the hash is already known for that station, throw it: no rebroadcast, no
   further processing. This is the flood's terminator.
2. **Find or allocate the station entry.** When the table is full, evict — preferring an entry
   whose slots hold no buffers, then the one whose slots are oldest. **Never evict the broadcast
   entry.** Evicting an entry discards its dedup memory as well as its frames, which narrows flood
   suppression under station pressure; that is an accepted cost, not a bug.
3. **Rebroadcast immediately, always.** (§3.)
4. **Record the hash and arrival time in a slot**, evicting that station's oldest slot if needed.
5. **Decide whether to keep the buffer**, which is the only conditional part:
   - **We have never snooped a receive window from this station** — keep the hash only. We cannot
     deliver to it; some other relay is its witness.
   - **A window is open right now, with room for the frame plus margin** — keep the hash only. The
     rebroadcast in step 3 *was* the delivery.
   - **Otherwise** — keep the buffer, and wait for a window.
   A broadcast is never presumed delivered this way: there is no window for the broadcast
   pseudo-station, so its buffer is always kept.

### The window-snoop path

A **direct** frame (not one wrapped in a mesh FORWARD) carrying a RECEIVE advertisement tells us a
station is listening, and for how long. Wrapped frames are deliberately ignored: their sender is
not our neighbour, so we could not deliver to it anyway.

On snooping one: record or refresh the window, then choose **one** frame to send — the oldest by
arrival time across that station's own slots and the broadcast entry's slots, skipping any
broadcast this station has already been given. Send it if it fits the remaining window with margin.

- a **unicast** frame is then discarded, and its hash retained
- a **broadcast** frame is kept, and `(hash, now)` is recorded in that station's broadcast-sent list

One frame per window, oldest first, unicast and broadcast interleaved strictly by arrival time, so
a burst of unicast cannot starve a broadcast.

### Answering is jittered, and that is what lets dedup suppress

Several relays commonly hold the same frame for the same station, and all of them hear the same
RECEIVE advertisement at the same instant. Without a delay they answer simultaneously, the copies
collide, and the station may get none of them -- the worst outcome available, because the airtime
was spent and nothing was delivered.

So the window send waits a **random delay**, bounded so the frame still fits the window with margin.
What the delay buys is more than collision avoidance:

```
relays A and B both hold frame F for station S, and both snoop S's window at t0
A draws 120 ms, B draws 310 ms
A transmits at t0+120
B hears A's copy         -> F's hash is already in B's dedup memory -> thrown at step 1
B sees S's window is open right now -> that transmission WAS the delivery -> B drops its buffer
```

The second step is not new machinery: it is the receive path's step 5 presumed-delivery rule applied
to a frame we HEARD rather than to one we were handed. Without the jitter, dedup can only discard a
duplicate after both copies have been paid for; with it, only one is ever put on the air.

This is the general case and applies to any frame more than one relay holds. It matters most where
the copies are generated **independently** rather than propagated by the flood -- CONTENT range
responses, where several caches answer one request with byte-identical frames (NOTES_CONTENT §6) --
because there the simultaneity is guaranteed rather than incidental.

**It suppresses only among relays that can hear each other.** Two relays on opposite sides of a
station, out of each other's range, will both deliver however long they wait. The jitter narrows the
duplicate window; it does not close it, and §6's assumption stands unchanged.

### The tick

The tick scans and discards anything older than its threshold: slot buffers, slot hashes, and
broadcast-sent records. There are separate thresholds for unicast and broadcast slots — broadcast
being longer-lived — and the broadcast-sent expiry is what implements the **cyclic resend**: once a
station's record of a broadcast expires, that station becomes eligible to be given it again.

The tick gates itself. Against thresholds measured in hours or days, scanning every loop pass buys
resolution nobody can use, so it costs one comparison per cycle and walks the table on an interval.

## 5. There is no unilateral resend

Every transmission is one of exactly two things:

1. **the opportunistic rebroadcast on arrival**, which reaches listening relays and any station
   awake at that moment, and
2. **the targeted send when a window is known to be open**, for a station we are hearing.

Nothing re-sends on a timer, and nothing retries. There is no acknowledgement anywhere on this
path, so there is no "lost" to detect and no state that a retry could be driven from: a frame
either went on the air or it did not. A command that misses is the **originator's** problem, and
its remedy is to send a new one — which, being content-addressed, must differ (§7).

## 6. What the design assumes, stated plainly

- **Delivery requires a recent witness.** A station is reachable only if some node has snooped a
  receive window from it. A station whose only witness has died gets nothing.
- **Duplicates are possible and are the station's problem.** Two relays may both hold and both
  deliver. The protocol does not prevent it -- the answer jitter (§4) suppresses it between relays
  that can hear each other, and does nothing for two that cannot. A station may keep hashes of recent downs and drop
  repeats, or may simply tolerate them — a very small device can reasonably do nothing. This note
  does not constrain that choice, but a station acting on non-idempotent commands should consider
  that a held copy can arrive *after* the action it asked for, including after a reboot it caused.
- **An originator retry faster than the dedup lifetime does nothing.** The two intervals are
  coupled, and the faster one loses.
- **There is no "always on" station.** A permanently listening node simply advertises a very large
  receive window, and is handled by exactly the same rules.

## 7. The timestamp TLV: making a frame unique on purpose

A system TLV carrying a timestamp, and optionally trailing text. Its timestamp is **not
interpreted by any gateway or relay** — its only load-bearing property is that including it changes
the frame's hash, which is what lets an operator send the same command twice.

It is a specific mechanism for what would otherwise be done with random padding, and it is useful
in its own right for anything that wants to state a time.

## 8. Configurables

All compile-time `#define`s to begin with, expected to become operationally settable through
CONFIG later.

| knob | what it bounds |
|---|---|
| stations | how many distinct targets can be tracked at once |
| slots per station | concurrent downs per target, and dedup depth |
| broadcast-sent slots per station | implicit broadcast repeat period under pressure |
| unicast slot expiry | how long a held command stays worth delivering |
| broadcast slot expiry | the same, longer |
| broadcast-sent expiry | the explicit cyclic resend interval |
| default receive window | assumed duration when a RECEIVE states none |
| window margin | safety room inside a window, beyond the frame's airtime |
| answer jitter | the spread of the window send, and with it how well duplicates suppress |
| scan interval | the tick's resolution |

Two ceilings are not tunable. Times are `uint32` milliseconds on a clock that wraps every 49.7
days, and wrap-safe ordering works over less than half of that, so **no interval may exceed ~24
days**. And those intervals are measured in *uptime*: a broadcast repeat longer than the expected
interval between reboots will rarely fire, so it wants to be well inside it or persisted.

## 9. What has to change

- **RECEIVE**: a station that sleeps states its window duration; absence now means "assume the
  system default" rather than "the node is taking responsibility". The duration is what makes
  presumed delivery (§4 step 5) safe.
- **The down module** is rebuilt around the station table above, replacing one-slot-per-target with
  slots keyed by frame hash, and replacing supersede with coexistence: a new `(station, content)`
  is a new down, never a replacement.
- **Broadcast must be acted on AND flooded.** These are currently the same branch, and a broadcast
  down is consumed at the first hop.
- **The window send is jittered** (§4), and hearing a duplicate of a frame we hold a buffer for,
  while that station's window is open, drops the buffer as presumed delivered.
- **The timestamp TLV** is added to the system range, header and codec only.
- **Reporting**: a station line should coalesce the down module's view of that station with the
  relay's own stations table. Deliberately a reporting-only join; the two tables stay separate
  until there is a reason to unify them.
