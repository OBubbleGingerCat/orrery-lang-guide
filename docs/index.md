---
layout: home

hero:
  name: Orrery
  text: Whole-system models in Rust
  tagline: Clocked timing and discrete events in one Rust model, simulated in parallel, with results that never depend on how it runs.
  image:
    light: /logo.svg
    dark: /logo-dark.svg
    alt: Orrery
  actions:
    - theme: brand
      text: Get started
      link: /guide/getting-started
    - theme: alt
      text: The GPGPU-Sim model
      link: /examples/gpgpu-sim

features:
  - title: Deterministic in parallel
    icon:
      light: /icons/deterministic.svg
      dark: /icons/deterministic-dark.svg
      width: 44
      height: 44
    details: The same state, events, end and errors on any number of workers. Orrery works out the model's ordering when it loads it, rejects what is ambiguous, and can check a run against it.
  - title: Models in plain Rust
    icon:
      light: /icons/rust.svg
      dark: /icons/rust-dark.svg
      width: 44
      height: 44
    details: Modules are Rust structs, their state Rust values, their behaviour Rust functions; macros do the rest. Parameters come from a TOML file.
  - title: Clocks and events together
    icon:
      light: /icons/clocks.svg
      dark: /icons/clocks-dark.svg
      width: 44
      height: 44
    details: Any number of clock domains in integer picoseconds, and events with physical delays or delta cycles, in the same model, from a pipeline stage to the network between GPUs.
  - title: Built for speed
    icon:
      light: /icons/speed.svg
      dark: /icons/speed-dark.svg
      width: 44
      height: 44
    details: State is stored in place, task bodies are compiled Rust, and the parts of a model that do not depend on each other run in parallel on many cores.
  - title: Snapshots without stopping
    icon:
      light: /icons/snapshots.svg
      dark: /icons/snapshots-dark.svg
      width: 44
      height: 44
    details: Snapshots are taken while the simulation runs, incrementally, and restore it to reproduce a problem without running everything again.
  - title: Open at the edges
    icon:
      light: /icons/open.svg
      dark: /icons/open-dark.svg
      width: 44
      height: 44
    details: External ports bring in traces, functional models or RTL co-simulation; components and interfaces can be user-defined.
---
