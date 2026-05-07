# Maglev Intercept Test Plan

Execution order confirmed by taskweft. Each cycle is a gate for the next; parallel tracks (2, 6, 9, 11) start after Cycle 1 and run alongside the sequential track.

## Cycle 0 — Infrastructure

> Gate for all cycles. ADR: [cycle-0](20260506-maglev-cycle-0-infra.md)

- [ ] `fly status --app multiplayer-fabric-gateway` shows running; dedicated IPv4 assigned
- [ ] CockroachDB volume attached; `fly volumes list` confirms the volume is in the correct region
- [ ] uro app reachable on Fly's 6PN private network
- [ ] Secrets for mTLS certs present (`fly secrets list` shows expected keys for gateway and uro)

## Cycle 1 — Godot Client Gateway Handshake

> Gate for all cycles. ADR: [cycle-1](20260506-maglev-cycle-1-gateway-handshake.md)

- [ ] Godot client establishes the WebTransport/QUIC connection without TLS or handshake error
- [ ] Client receives and logs one datagram (not a stream) from the zone server
- [ ] Client exits cleanly; no orphaned Godot process or open port

## Cycle 2 — Observability (parallel, must finish before Cycle 3)

> Parallel from Cycle 1. ADR: [cycle-2](20260506-maglev-cycle-2-observability.md)

- [ ] VictoriaTraces UI at `http://localhost:10428/select/vmui` shows spans from at least one gateway request
- [ ] VictoriaMetrics at port 8428 returns a non-empty result for a zone-server metric query
- [ ] VictoriaLogs at port 9428 shows at least one log line from uro
- [ ] All four services are Apache 2.0 (no Tempo, no Jaeger)

## Cycle 3 — 16-Client MMOG Load

> Depends on Cycles 1 and 2. ADR: [cycle-3](20260506-maglev-cycle-3-dual-client.md)

- [ ] All 16 clients receive consistent entity state each tick for 60 seconds
- [ ] No client observes a tick gap or duplicate
- [ ] Zone-console shows 16 entities at 20 Hz
- [ ] Gateway process memory stays flat under 16 connections
- [ ] All 14 bots disconnect and reconnect simultaneously; gateway recovers within 5 seconds and all clients resync without a restart
- [ ] Under 100 ms injected latency and 2% packet loss, no client desyncs over 30 seconds

## Cycle 4 — IK Routing and Merge Baseline

> Depends on Cycle 3. ADR: [cycle-4](20260506-maglev-cycle-4-ik-routing.md)

- [ ] Zone server receives and logs at least one IK datagram without parse error at 1 Hz
- [ ] Zone server receives a merged tick containing both input types at 10 Hz
- [ ] No gamepad datagram dropped over 60 seconds at 10 Hz
- [ ] No gateway crash or restart

## Cycle 5 — Full-Rate IK Merge

> Depends on Cycle 4. ADR: [cycle-5](20260506-maglev-cycle-5-ik-merge.md)

- [ ] Zone server processes ticks containing both input types without stalls or skipped events
- [ ] No gamepad datagram is silently dropped over 60 seconds under full IK load
- [ ] Gateway CPU stays within budget under both streams at full rate

## Cycle 6 — Baker Pipeline (parallel)

> Parallel from Cycle 1; must finish before Cycle 7. ADR: [cycle-6](20260506-maglev-cycle-6-baker.md)

- [ ] Both bake jobs exit 0
- [ ] `.caibx` index appears in uro at `/storage/:id/bake` for each asset
- [ ] Zone server can fetch and assemble the train scene from the chunk store before Cycle 7 begins

## Cycle 7 — Physics and VRM Loading

> Depends on Cycles 5 and 6. ADR: [cycle-7](20260506-maglev-cycle-7-physics-vrm.md)

- [ ] Zone server loads the Maglev train scene from the chunk store
- [ ] Core positions agree between both clients each tick for 60 seconds under continuous banking
- [ ] Zone server tick rate holds at 20 Hz with physics active
- [ ] VRM avatars (PCVR and Steam Deck) load from the chunk store without error; humanoid skeleton root present
- [ ] No entity desync over the run

## Cycle 8 — Dynamic Physics and Causal Ordering

> Depends on Cycles 7 and 2. ADR: [cycle-8](20260506-maglev-cycle-8-dynamic-physics-score.md)

- [ ] No entity desync between any of the 16 clients over the 3-minute window
- [ ] Core positions agree within one physics tick across all clients at mission end
- [ ] Zone server tick rate holds at 20 Hz under banking motion, drone AI, and 16-client load
- [ ] Every core-slot `QueueOp` reaches the persona zone; VClock values advance monotonically
- [ ] No `QueueOp` accepted out of causal order
- [ ] `multiplayer-fabric-predictive-bvh` computes at least 2 distinct interest zones across the 16 clients; zone-console confirms all 16 entities visible

## Cycle 9 — CockroachDB Connection (parallel)

> Parallel from Cycle 1. ADR: [cycle-9](20260506-maglev-cycle-9-db-connection.md)

- [ ] Row appears in CockroachDB within one second of the trigger
- [ ] mTLS handshake succeeds; no cert or auth error
- [ ] `gateway_writer` role has DML access; `gateway_admin` is not used at runtime
- [ ] Connection reuse works across multiple sequential writes (`prepare: :unnamed` confirmed)

## Cycle 10 — CockroachDB Score Write

> Depends on Cycles 8 and 9. ADR: [cycle-10](20260506-maglev-cycle-10-db-write.md)

- [ ] Committed score appears in CockroachDB within one tick of mission end
- [ ] `RelReplica.stale` correctly reflects the write on all nodes in the interest band
- [ ] No write occurs from any zone other than the persona authority

## Cycle 11 — Zone-Console TUI (parallel)

> Parallel from Cycle 1. ADR: [cycle-11](20260506-maglev-cycle-11-zone-console.md)

- [ ] zone-console connects without error
- [ ] Tick rate displayed at 15 Hz or above (full 20 Hz target confirmed in Cycle 8)
- [ ] At least one entity visible in the TUI entity list
- [ ] Clean disconnect on exit with no orphaned connection
