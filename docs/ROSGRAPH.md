# rosgraph — Technical Proposal

> **Status:** Proposal
> **Date:** 2026-02-22
> **Parent:** [MANIFESTO.md](MANIFESTO.md) (direction)

---

## Executive Summary

ROS 2 has no standard schema for declaring node interfaces and no
production-ready tooling for verifying that a running system matches its
declared architecture. The ecosystem is fragmented across single-purpose
tools with overlapping scope and bus factors of one.

Key gaps — no existing tooling:

- **Graph diff** (expected vs. actual)
- **Graph linting** (pre-launch static analysis)
- **CI graph validation**
- **Node API documentation** (hand-written only today)
- **QoS static analysis** (breadcrumb is early-stage/partial)

### The Problem, Concretely

Today in ROS 2:

- Node A publishes `/cmd_vel` as `Twist`. Node B subscribes to
  `/cmd_vel` as `String`. You discover this at runtime — or don't,
  because the subscriber silently receives nothing.
- A publisher uses `BEST_EFFORT` QoS, a subscriber uses `RELIABLE`.
  DDS refuses the connection. A warning is logged but easy to miss in
  a busy console. The subscriber just never gets messages.
- A node crashes mid-deployment. The rest of the system keeps running.
  Nobody knows until a customer reports a failure 20 minutes later.
- You rename a parameter. Three launch files reference the old name.
  `colcon build` succeeds. The system launches. The parameter silently
  takes its default value.

These are real, common bugs in production ROS 2 systems.

### Components

rosgraph is composed of the following components, ordered by priority.
These components may be wrapped by user interfaces (e.g. a CLI), but
are designed as independent, composable libraries.

1. **Node Spec (NoDL)** — a formal, machine-readable schema for
   declaring node interfaces (`interface.yaml`). This is the most core
   part of the project; everything else builds on it.

2. **Code Generation** — `nodl-generator` takes NoDL input and outputs
   code for ROS client libraries (rclcpp, rclpy, rclrs). Must be
   installable as part of a ROS distro (`apt-get install`). Requires a
   plugin/sidechannel architecture so additional client libraries
   (e.g. rcljava) can be supported without modifying the core generator.

3. **Runtime Discovery** — introspect a running system and produce NoDL
   specs from observed nodes. Enables brownfield adoption: point at an
   existing system, generate `interface.yaml` files for every node, then
   iteratively refine them. Unlike runtime monitoring (component 5),
   discovery is a one-time migration tool, not a continuous process.

4. **Node-level Unit Testing** — verify a single node conforms to its
   declared spec in isolation.

5. **Graph Analysis & Comparison** — integration-level verification.
   Static analysis checks the full graph for type mismatches, QoS
   incompatibilities, and missing connections before launch. Runtime
   monitoring continuously diffs the declared graph against the live
   system, flagging drift (crashed nodes, unexpected topics, QoS
   changes) as it happens.

6. **Documentation Generation** — produce API documentation directly
   from NoDL specs.

> **Open question:** implementation language for the generator tooling.

### Key Insights

Three key insights drive the design:

1. **The ROS computation graph is not source code — it is a typed,
   directed graph with QoS-annotated edges.** Analysis tools should
   operate on a graph model, not on ASTs. Source code parsing is a
   loader that feeds the model, not the analysis target.

2. **Verification and analysis are schema conformance problems**
   ("does reality match the spec?"), not traditional program analysis.
   Once you have a machine-readable spec (`interface.yaml`),
   verification falls out naturally — the same pattern as `buf lint`,
   Pact contract tests, and Kubernetes reconciliation.

3. **A declaration without code generation is a non-starter.** NoDL
   proved this. The schema must generate code, documentation, and
   validation to stay in sync with reality. `interface.yaml` is
   simultaneously the source for code generation, the lint target for
   static analysis, the contract for runtime verification, and the
   reference for documentation.

### Example

A minimal `interface.yaml`:

```yaml
schema_version: "1.0"
node:
  name: talker
  package: demo_pkg

publishers:
  - topic: ~/chatter
    type: std_msgs/msg/String
    qos: { reliability: RELIABLE, depth: 10 }

parameters:
  publish_rate:
    type: double
    default_value: 1.0
    description: "Publishing rate in Hz"
    validation:
      bounds<>: [0.1, 100.0]
```

From this single file, the tooling can:
- **Generate** a typed C++/Python node context with publishers and validated parameters — no boilerplate
- **Lint** the full workspace graph for type mismatches and QoS incompatibilities before launch
- **Monitor** the running system and flag drift from the declared spec
- **Discover** a running system's interfaces and produce draft specs for brownfield adoption
- **Document** the node's API automatically

---
