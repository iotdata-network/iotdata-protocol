# NOTES - OPEN ISSUES

> System and protocol issues that need a DECISION, not a patch. Each one is something the code
> cannot catch for itself: a coupling that crosses a node boundary, an assumption held in two
> places that nothing reconciles, or a behaviour that is correct in isolation and wrong in a fleet.
>
> An entry says what the problem is, what the evidence for it is, why nothing catches it today, and
> what the options look like. It does not say what to do -- that is the decision, and it is open.
>
> Resolved entries move out: into the specification if the answer is a rule, into the code if the
> answer is a guard. An entry that has been here a long time is a question nobody has needed to
> answer yet, which is information too.

---

## I.1 The mesh beacon interval is a fleet-wide constant held on one node

**Status: RESOLVED 2026-09-28**, by option 3 below: the root states its cadence in the beacon and a
relay derives its timeouts from what it heard. Kept here rather than deleted because the reasoning
is the record of why the beacon carries a field that looks redundant, and why two config ids are
retired. Raised 2026-09-28 while migrating gateway config to the shared tables.

### What

The root's beacon interval sets the cadence of the whole tree, and every relay's ageing constant is
derived from it -- but the relays cannot see it, and nothing stops the two from disagreeing.

A relay does not beacon on a timer. It rebroadcasts, once, when what it advertises changes:

```c
/* rebroadcast if our advertised beacon changed (first join / new gen / cost) */
if (!was_joined || n->my_cost != old_cost || n->my_generation != old_gen)
    mesh_transmit_beacon(n, now_ms);
```

That is the intended design and it is a good one: ONE CLOCK, at the root. The root beacons every
`MESH_BEACON_INTERVAL_S` and bumps `generation`; the bump propagates down as each relay sees a newer
generation and rebroadcasts once, jittered. N relays beaconing on N intervals would drift against
each other and leave nobody able to say what the tree's refresh rate is.

The consequence is that a peer's freshness is refreshed ONLY by beacons -- `mesh_peer_upsert()` is
reached only from `mesh_on_beacon()` -- so the root's interval bounds how long a relay may wait
before deciding its peers, and its parent, are gone.

### The evidence

The relay's defaults already encode the root's number, in comments, as assumptions:

| constant | default | its own comment |
|---|---|---|
| `IOTDATA_CONFIG_MESH_PEER_TTL_MS` | 300000 | `/* ~5x a 60s beacon */` |
| `IOTDATA_CONFIG_MESH_PARENT_TIMEOUT_MS` | 190000 | `/* ~3 missed 60s beacon rounds */` |
| `IOTDATA_CONFIG_MESH_BEACON_INTERVAL_S` | 60 | the root's, and the "60s" both of the above mean |

Those hold at the defaults. They are also all three SETTABLE, independently, on different machines.
Set the root to `mesh-beacon-interval-s = 600` and leave the relays alone: every relay drops its
parent at 190s, drops its peers at 300s, goes orphan, and re-joins when the next generation arrives
-- a fleet-wide flap every ten minutes, caused by one number on one box, with nothing in any log
saying the two are related.

### Why nothing catches it

The config framework's cross-entry validation (`iotdata_config_update_peek()`) reads the LOCAL
table. These rows are not in one table and never will be: `MESH_BEACON_INTERVAL_S` is in the
gateway's, `MESH_PEER_TTL_MS` and `MESH_PARENT_TIMEOUT_MS` are in each relay's. The relationship is
between machines, so no validator can express it and no single node can refuse the write.

Nor can a relay infer it. It sees beacons arrive, but "slower than I expected" and "my parent is
gone" are the same observation -- which is exactly the condition being mis-detected.

### What the options look like

1. **Say the rule and leave it to the operator.** Document the ratio (`peer_ttl > 3 x beacon
   interval`, say) in Appendix G and stop. Cheapest; relies on whoever changes the number reading
   the right paragraph.
2. **Bound the root's row.** Cap `MESH_BEACON_INTERVAL_S` at something the shipped relay defaults
   survive. Enforceable locally, but it caps the fleet from one node's table and is wrong the moment
   a deployment tunes its relays differently.
3. **Put it in the beacon.** The root already advertises `generation`, `cost` and `flags`; it could
   advertise its interval, and a relay could derive its own timeouts from what it actually hears
   rather than from a constant. This makes the coupling explicit and self-correcting, and it is the
   only option that survives a mixed fleet -- but it is a wire-format change.
4. **Make the relay's timeouts adaptive.** Measure the observed inter-beacon interval and scale the
   timeouts off the measurement. No wire change; harder to reason about, and wrong for the first
   few rounds after a join.

### What was done

**Option 3.** Stating the interval rather than a derived timeout was the deliberate choice: the root
knows its cadence, the relay knows its own tolerance, and the relay is left with something it can
take apart rather than an answer it must accept.

- **The wire.** `BEACON` gains `interval_s` (u16 seconds) at bytes 9-10, so 9 bytes becomes 11, and
  the field is MANDATORY: every beacon carries it, and a frame short of 11 bytes is not a beacon.
  It was built optional-by-length first, so a pre-interval root and a new relay could interoperate
  without a flag day. That is the right instinct for a deployed protocol and the wrong one for this
  one, which is still in development with nothing in the field that cannot be reflashed: optional
  buys a compatibility nobody needs and charges for it for ever, in a format where "is the cadence
  stated?" becomes a question every reader keeps having to ask. Mandatory puts the tree's clock in
  every frame and narrows `interval_s = 0` to one honest meaning -- the SENDER does not know it
  either, being a relay that has joined but not yet heard a beacon of its own.
- **The relay's rows became ROUND COUNTS.** `MESH_PEER_TTL_MS` and `MESH_PARENT_TIMEOUT_MS` become
  `MESH_PEER_TTL_ROUNDS` and `MESH_PARENT_MISS_ROUNDS`, keeping their ids (0x042, 0x050). A round
  was always the unit these numbers meant; their old comments said so in words ("~3 missed 60s
  beacon rounds") while their values said milliseconds. Being relative is what makes the
  inconsistency impossible rather than merely unlikely, and the min/max validator between them is
  now checkable locally for the right reason: both sides live on the same node.

  Keeping the ids is a deliberate exception to never-reuse, recorded in `iotdata_node_config_mesh.h`
  beside the rule. Same meaning, different unit and width, so a stale cached name->id map would read
  a live value and be wrong about it -- which is the thing burning an id prevents. It is safe only
  while nothing deployed can be confused by it, which is true today and will not always be.
- **A relay rebroadcasting a beacon carries the interval down**, so a node three hops out ages
  against the same clock as one at the top.
- **The fallback is the build's assumption**, used only until the first beacon arrives -- which is
  also the only window in which a relay has no peers to expire and no parent to lose.

**Changing the interval is safe**, which was not obvious and is the reason the design works: the
root advertises the new value on a beacon that is still due on the OLD schedule, so every relay has
widened its timeouts before the longer gap begins.

**What it would have cost.** A deployment slowing the beacon to save air time -- the obvious thing
to do on a busy 2.4kbps channel, and the only reason anyone would touch the number -- would have
orphaned every relay in the tree once per round, indefinitely, with nothing in any log connecting
the two settings.

## I.2 DIAG is cargo inside FORWARD rather than a peer of it

**Status: OPEN, deliberately parked 2026-10-04.** Raised while designing DIAG kind TRACE. Not to be
taken on before the Sweden deployment: it changes FORWARD's wire format, the duplicate-suppression
key, every relay and the gateway, and that firmware has to be frozen.

### What

FORWARD (ctrl 0x1) is simultaneously two things: the inward transport, and a wrapper announcing
"here is a packet I am carrying". Nothing names the direction a frame should travel, so direction is
implied by WHICH control type carries it -- FORWARD inward, rebroadcast outward. DIAG therefore has
a split personality: bare when it floods outward, wrapped in a FORWARD when it travels inward.

### Why it is worth revisiting

Four symptoms, all already present rather than anticipated:

1. **A payload that travels both ways needs two implementations.** TRACE appends a hop record on
   every hop, and so needs an append site in `forward_on_forward()` for the inward leg and another
   in `survey_rebroadcast()` for the outward one -- the same operation written twice because the two
   directions are different mechanisms.
2. **The transport parses its own cargo.** The dedup key is the ORIGIN's station and sequence, which
   lives in the inner packet, so a relay reads bytes 6-9 of the frame it is carrying to do its own
   routing job. The spec says so outright. A routing header holding origin and sequence itself would
   not need to look inside.
3. **A bare DIAG RESPONSE has no inward path.** `relay_receive_frame_diag()` drops it as "not ours
   to act on", because being carried inward is a property of FORWARD and not of the frame. It is
   harmless today only because every originator wraps its own.
4. **A leaf has to fabricate tree headers.** `app_survey.h` builds its own FORWARD with
   `TTL_DEFAULT` and itself as `sender_station`, despite not being in the mesh and having no cost.
   It works because the gradient check treats a never-heard sender as TTL_INFINITY away -- a node
   outside the tree constructing the tree's transport for itself.

### The shape it wants

A routing header that states direction, ttl, origin station and origin sequence, with sensor data
and DIAG as peer PAYLOADS of it rather than one nested in the other. Then direction is a field,
appending happens once, dedup reads the header it belongs to, and a leaf asks to be carried instead
of describing how.

### What it costs to leave

Every new DIAG kind that travels both ways pays symptom 1 again -- two append sites, two code paths,
two chances to disagree. TRACE is the first; a reachability kind would be the second.
