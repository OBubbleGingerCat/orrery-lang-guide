# Event benchmark

A synthetic model of event-driven work, which the GPGPU-Sim model (clocks
only) never exercises. It comes in two shapes: lanes that depend on each
other at every stage, and a mesh network on chip that is closer to a real
model.

## Lanes

```text
Top (clocks)
└── lanes: [Lane; lane_count]
    ├── source: Source        every `every` clock edges: emit a token
    ├── stages: [Stage; stage_count]
    │     on a token from its own lane or the next one:
    │     state = mix(state, token, `work` rounds); pass its own lane's token on
    └── sink: Sink            total += token
```

Stage *k* of lane *i* takes the tokens at position *k* of its own lane and
of lane *i* + 1, so the lanes depend on each other. Each hop between
stages is either one delta cycle or a declared minimum delay of 1 ns
([Declared delays](../guide/time#declared-delays)). Lanes can run on clocks out
of phase, so that each timepoint has only a few lanes active; and a
variant ends each lane in stages that the clock triggers too.

## Mesh

A `width` × `height` mesh of routers. Each router's clock injects
packets, a share of them to a hotspot so the load is uneven; packets go X
then Y, one per output per cycle from per-output queues, over links that
are events with a physical delay (2 ns by default). A variant adds a
monitor that sums every router's counters on the clock and stops the
simulation once a goal number of packets has been delivered, which
exercises [stopping](../guide/running#stopping) in an event-driven model.

## Gated units

Units on gated clocks ([Clock gates](../guide/time#clock-gates)): each
mixes its state at every run and, after `busy` runs, closes its gate for
1 to `idle` edges, as many as its state says. One configuration writes
the gate at every run (one run, one idle edge); the other keeps a unit
busy for 20 runs and idle for up to 80 edges. The program prints the
runs, the gated edges and a checksum.

## What it shows

Every configuration prints a checksum, which every number of workers must
agree on.
