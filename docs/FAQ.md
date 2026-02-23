# rosgraph — Frequently Asked Questions

> **Parent:** [ROSGRAPH.md](ROSGRAPH.md) (technical proposal)

Organized by who's asking. Find your perspective, jump to the
questions that matter to you.

---

## Table of Contents

0. [General](#0-general)
1. [New ROS Developer](#1-new-ros-developer)
2. [Engineering Lead / System Integrator / DevOps](#2-engineering-lead--system-integrator--devops)
3. [MoveIt / nav2 / Popular Module User](#3-moveit--nav2--popular-module-user)
4. [AI-Assisted Developer](#4-ai-assisted-developer)
5. [Package Maintainer / ROS Governance](#5-package-maintainer--ros-governance)
6. [Educator / University Researcher](#6-educator--university-researcher)
7. [Embedded / Resource-Constrained Developer](#7-embedded--resource-constrained-developer)
8. [The Skeptic](#8-the-skeptic)
9. [Safety-Critical Engineer](#9-safety-critical-engineer)

---

## 0. General

### What problem does rosgraph solve?

When you connect ROS 2 nodes together, mistakes are invisible. If one
node sends a `Twist` message but another node expects a
`TwistStamped`, nothing warns you — the subscriber just never receives
data. If you misspell a topic name in a launch file, the node launches
fine but sits there doing nothing. You end up staring at
`ros2 topic list` wondering why nothing is connected.

rosgraph catches these wiring mistakes before you even launch your
system. You describe what each node publishes, subscribes to, and what
settings it needs in a short YAML file. Then `rosgraph lint` checks
that everything fits together — like a spell checker, but for your
ROS graph.

See [ROSGRAPH.md §1, "The Problem,
Concretely"](ROSGRAPH.md#the-problem-concretely) for four real-world
examples.

### How much do I need to learn?

One file per node (`interface.yaml`, about 15 lines) and three
commands:

```bash
rosgraph generate .   # creates starter code from your YAML
rosgraph lint .       # checks for wiring mistakes
rosgraph monitor      # watches the running system for problems
```

Your editor will autocomplete the YAML fields for you — no need to
memorize the format. See the [Quick
Start](ROSGRAPH.md#quick-start-what-it-looks-like) for a complete
example.

### What's the overhead?

Per node: one `interface.yaml` file (~15-30 lines). Most of it is
information you're already specifying in code (topic names, message
types, QoS settings, parameter names) — `interface.yaml` centralizes
it.

What you get back:
- No pub/sub boilerplate (generated)
- No parameter declaration boilerplate (generated via
  `generate_parameter_library`)
- Pre-launch graph validation
- Runtime graph monitoring
- Auto-generated API documentation

The net line-count change is typically negative for nodes with
parameters.

### What about my launch files and parameter configs?

`system.yaml` (Layer 2) overlaps heavily with both — all three
describe which nodes run, with what parameters, and with what
remappings. The long-term direction is convergence: `system.yaml`
becomes the graph spec, the parameter config, *and* the launch
description in one file. `rosgraph generate` emits a runnable launch
file from the same spec that `rosgraph lint` validates — no drift
between what you analyze and what you run.

For projects with multiple deployment configurations (sim, real, test),
each gets its own `system.yaml`, replacing both the per-config launch
file and the per-config parameter YAML. See [ROSGRAPH.md
§3.2](ROSGRAPH.md#32-schema-layers).

### Won't the spec just drift from reality like NoDL?

NoDL died because it was a pure description format — no code
generation. Maintaining a spec that doesn't produce anything is
thankless work.

`interface.yaml` generates code. If you change the spec, the generated
code changes. If you change the code without changing the spec,
`rosgraph monitor` flags the discrepancy at runtime. The two-way
binding (codegen + runtime monitoring) is what prevents the drift
that killed NoDL.

The honest limitation: business logic is hand-written. If a developer
adds an undeclared publisher inside a callback, `rosgraph lint` won't
catch it at build time. `rosgraph monitor` catches it at runtime as
`UnexpectedTopic`. See [ROSGRAPH.md
§12](ROSGRAPH.md#12-scope--limitations).

---

## 1. New ROS Developer

### What does rosgraph do for me?

- **Writes the repetitive code.** Creating publishers, subscribers,
  and declaring parameters — `rosgraph generate` handles this from
  your YAML file. You write only the interesting part (what your node
  actually *does*).
- **Catches mistakes early.** Mismatched message types, misspelled
  topic names, incompatible connection settings — found in seconds,
  not after a 30-second launch-debug-relaunch cycle.
- **Keeps settings in one place.** Parameter names, types, and default
  values live in `interface.yaml` instead of scattered across your
  code, launch files, and README.

### Will error messages make sense?

Yes — this is a design priority. Each error tells you:

- **Where:** which file and line has the problem
- **What:** a plain description of what's wrong
- **How to fix it:** a suggested correction, auto-applied when safe

No cryptic stack traces. No silent failures. See [ROSGRAPH.md
§10.3](ROSGRAPH.md#103-static-analysis-architecture) for the error
design.

---

## 2. Engineering Lead / System Integrator / DevOps

### How does this scale to hundreds of packages?

- **Lint performance target:** 100 packages in under 5 seconds
  (Design Principle 7). Analysis is single-pass over the graph model
  with parallel per-package processing and content caching.
- **Multi-workspace analysis:** Installed `interface.yaml` files in
  underlays serve as cached facts. Only your workspace is analyzed,
  not the entire underlay. See [ROSGRAPH.md
  §3.12](ROSGRAPH.md#312-multi-workspace-analysis).
- **Differential analysis:** `--new-only` reports only issues
  introduced since the base branch. No noise from existing code.

### I compose nodes from multiple vendors. How does rosgraph help?

`system.yaml` (Layer 2 schema, [ROSGRAPH.md
§3.2](ROSGRAPH.md#32-schema-layers)) declares the intended system
composition — which nodes, which namespaces, which parameter overrides,
which remappings. `rosgraph lint` validates the composed graph:

- **Type mismatches** across package boundaries
- **QoS incompatibilities** between a vendor's publisher and your
  subscriber
- **Disconnected subgraphs** — nodes that should be connected but
  aren't due to a namespace or remapping error

If a vendor doesn't ship `interface.yaml`, use `rosgraph discover`
([ROSGRAPH.md
§3.10](ROSGRAPH.md#310-rosgraph-discover--runtime-to-spec-generation))
to generate one from a running instance of the vendor's node.

### How does rosgraph fit into CI?

rosgraph is CI-first by design (Design Principle 8):

```yaml
# GitHub Actions example
- name: Lint graph
  run: rosgraph lint . --output-format sarif --new-only --base main

- name: Check breaking changes
  run: rosgraph breaking --base main

- name: Run contract tests
  run: rosgraph test
```

Output formats: `text`, `json`, `sarif` (GitHub Security tab),
`github` (Actions annotations), `junit` (test reports). See
[ROSGRAPH.md §3.11](ROSGRAPH.md#311-configuration).

---

## 3. MoveIt / nav2 / Popular Module User

### Does rosgraph work with nav2's plugin system?

Yes, via the mixin system ([ROSGRAPH.md
§3.2](ROSGRAPH.md#32-schema-layers)). Plugins that inject interfaces
into a host node are declared as mixins:

```yaml
# nodes/follow_path/interface.yaml
node:
  name: follow_path
  package: nav2_controller

mixins:
  - ref: dwb_core/dwb_local_planner
  - ref: nav2_costmap_2d/costmap
```

The host's effective interface = its own declaration + all mixin
interfaces merged. Mixins are Phase 2 (G15). Phase 1 works for nodes
without plugins.

### What about `generate_parameter_library` compatibility?

Full compatibility is a non-negotiable design principle ([ROSGRAPH.md
§2, DP9](ROSGRAPH.md#2-design-principles)). The `parameters:` section
of `interface.yaml` IS the `generate_parameter_library` format. A
standalone gen_param_lib YAML file works as-is when placed in
`interface.yaml`. rosgraph delegates to gen_param_lib at build time.
See [ROSGRAPH.md §9.2](ROSGRAPH.md#92-tool-assessments).

---

## 4. AI-Assisted Developer

### How does rosgraph work with AI coding tools?

`interface.yaml` is a machine-readable contract — exactly what LLMs
are good at consuming and generating. The `InterfaceDescriptor` IR
([ROSGRAPH.md §3.3](ROSGRAPH.md#33-the-interfacedescriptor-ir)) is a
JSON blob containing a node's complete API: topics, types, QoS,
parameters, lifecycle state. An AI agent reads this to understand what
a node does, generate implementation code, write tests, or suggest
fixes — without parsing C++ or Python source.

See [ROSGRAPH.md §3.13](ROSGRAPH.md#313-ai--tooling-integration) for
the full AI integration design.

### Can I use `rosgraph generate` as an agent tool?

Yes. An AI agent writing a ROS node can:
1. Generate `interface.yaml` from a natural language description
2. Run `rosgraph generate .` as a tool call to get type-safe
   scaffolding
3. Write only the business logic into the generated skeleton
4. Run `rosgraph lint .` to verify the graph is correct

This avoids the common failure mode of LLMs hallucinating ROS
boilerplate (wrong QoS defaults, missing component registration,
incorrect parameter declaration).

---

## 5. Package Maintainer / ROS Governance

### Do I have to adopt rosgraph to be compatible with it?

No. Packages without `interface.yaml` are skipped, not errored (Design
Principle 6). Downstream users can run `rosgraph discover` against your
running node to generate a spec for their own use. Your package doesn't
need to ship `interface.yaml` for others to benefit — though shipping
one is much better, since discovered specs require human review and may
miss QoS details.

### What's the adoption path toward `ros_core`?

Deliberately incremental ([ROSGRAPH.md §4, "Adoption
Path"](ROSGRAPH.md#adoption-path)):

1. **`ros-tooling` organization** — institutional backing, CI
   infrastructure, release process.
2. **REP for `interface.yaml` schema** — formalizes the declaration
   format as a community standard, independent of the rosgraph tool.
3. **docs.ros.org tutorial integration** — if "write your first node"
   uses `interface.yaml`, every new ROS developer learns it from day
   one.
4. **`ros_core` proposal** — after demonstrated adoption across
   multiple distros.

### Why not extend existing tools instead?

Each existing tool covers one capability but none covers the full
scope. The gap analysis ([ROSGRAPH.md
§9.3](ROSGRAPH.md#93-gap-analysis)) shows five major gaps: graph diff,
graph linting, QoS static analysis, behavioral properties, and CI graph
validation. No single existing tool can be extended to fill all five.

rosgraph builds on existing work where possible:
- `generate_parameter_library` for parameters (used as-is)
- `rosgraph_monitor_msgs` for runtime message definitions (adopted)
- cake's design decisions for code generation (validated)
- HAROS's metamodel for the graph model (adapted)

---

## 6. Educator / University Researcher

### Can I use rosgraph for teaching ROS 2?

Yes. The Quick Start
([ROSGRAPH.md §1](ROSGRAPH.md#quick-start-what-it-looks-like))
shows a complete workflow in 3 commands. For teaching,
`interface.yaml` forces students to think about their node's API
before writing implementation code — topics, types, QoS, parameters.
This is better pedagogy than copy-pasting publisher boilerplate and
tweaking it.

### How does rosgraph relate to HAROS?

HAROS ([ROSGRAPH.md §10.6](ROSGRAPH.md#106-ros-domain-prior-art-haros))
was the prior art for graph analysis in ROS — built at the University
of Minho (2016–2021). rosgraph borrows HAROS's metamodel and HPL
property language concepts, but differs fundamentally:

- **HAROS extracted interfaces from source code.** rosgraph uses
  explicit declarations (`interface.yaml`).
- **HAROS was ROS 1 only.** rosgraph is built for ROS 2 concepts:
  QoS, lifecycle, components, actions, DDS discovery.
- **HAROS died because extraction broke.** catkin → ament, rospack →
  colcon, XML launch → Python launch. Declaration-based tools don't
  break when the build system changes.

---

## 7. Embedded / Resource-Constrained Developer

### Does rosgraph add runtime overhead to my nodes?

The generated code uses a composition pattern (has-a `Node`, not is-a
`Node`). This adds one pointer indirection — single nanoseconds. The
generated pub/sub wrappers are thin forwarding calls. No virtual
dispatch is added beyond what the ROS client library already uses.

Parameter validation (via `generate_parameter_library`) runs at
parameter-set time, not in the hot path. See [ROSGRAPH.md §3.4,
"Design decisions"](ROSGRAPH.md#34-rosgraph-generate--code-generation).

### Does `rosgraph monitor` run on the robot?

Yes, but it's optional. `rosgraph monitor` is a separate process — it
doesn't instrument or modify your nodes. If your platform can't spare
the resources, don't run it. You still get full value from build-time
tools (`rosgraph generate`, `rosgraph lint`).

Runtime targets ([ROSGRAPH.md
§3.14](ROSGRAPH.md#314-scale--fleet-considerations)):
- Memory: < 50MB resident
- CPU: < 5% of one core at steady-state (5s scrape interval)
- No additional DDS traffic beyond standard discovery

---

## 8. The Skeptic

### This proposal has 51 features. Is this realistic?

Phase 1 ([ROSGRAPH.md §4](ROSGRAPH.md#4-phasing)) is the commitment:
~12 features covering core schema, basic code generation, and
highest-value lint and monitor rules. Later phases are contingent on
adoption.

The tool builds on existing work — cake for code generation,
`generate_parameter_library` for parameters, `graph-monitor` message
definitions for runtime. Phase 1 is stabilizing and unifying existing
pieces, not building from scratch.

### When should I NOT use rosgraph?

- **Quick prototyping** — single throwaway node, not worth the file.
- **Single-node packages** — minimal lint value, though codegen may
  still save boilerplate.
- **Highly dynamic interfaces** — nodes that create/destroy publishers
  at runtime based on conditions can't be fully declared.

See [ROSGRAPH.md §12, "When Not to Use
rosgraph"](ROSGRAPH.md#when-not-to-use-rosgraph).

---

## 9. Safety-Critical Engineer

### Does rosgraph help with certification?

rosgraph is not a safety tool — it's a development and verification
tool that produces artifacts useful in safety cases. See [ROSGRAPH.md
§11](ROSGRAPH.md#11-safety--certification).

Key artifacts:

| rosgraph artifact | Evidence type |
|---|---|
| `interface.yaml` | Software architecture description |
| `rosgraph lint` SARIF output | Static analysis results |
| `rosgraph monitor` logs | Runtime verification evidence |
| `rosgraph test` results | Interface conformance evidence |
| `rosgraph breaking` output | Change impact analysis |

### What about behavioral properties?

Phase 1-2 covers structural properties: type matches, QoS
compatibility, graph connectivity. Behavioral analysis (Phase 3+) adds
temporal and causal properties, inspired by HAROS HPL:

```
globally: /emergency_stop causes /motor_disable within 100ms
globally: /heartbeat absent_for 500ms causes /safe_stop
```

See [ROSGRAPH.md §11.4](ROSGRAPH.md#114-behavioral-properties-future).
