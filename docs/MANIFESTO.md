# ROSGraph — Direction

## Why

Robotics engineers spend too much time on ROS plumbing — writing boilerplate, debugging invisible wiring, and keeping launch files in sync with code — instead of building their application.

The main interfaces of ROS systems (topics, parameters, services, actions) are undocumented by default. As systems grow larger they become harder to reason about, and the lack of well-defined interface contracts blocks automated tooling from helping.

## What

A declarative, observable ROS graph. Engineers declare what their system should be; tooling generates the code and entities as needed, and verifies the running system matches the spec.

## How

1. **Language** — a formal spec to describe node interfaces and system graphs.
2. **Tooling** — translate declarations into working code.
3. **Verification** — compare spec against reality, both at runtime and statically before launch.
