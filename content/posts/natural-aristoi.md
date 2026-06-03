---
title: Natural Aristoi
description: 'Frameworks should earn their place in your application through demonstrated merit — not convention, not network effect, not being the path of least resistance.'
published: true
tags: 'architecture, softwareengineering, philosophy, opensource'
---

Frameworks should earn their place in your application through demonstrated merit -- not convention, not network effect, not being the path of least resistance.

### The Concept

Natural Aristoi is a concept championed by Thomas Jefferson describing a governing class defined by virtue and talent, not birth or privilege. Jefferson believed in small government, states' rights over federal, power closest to the people it governs -- local, visible, accountable. Distrust of centralized authority that accumulates complexity, imposes its will and becomes impossible to remove once entrenched.

Small codebase. Components over monolith. Architecture closest to the problem it solves -- visible, intentional, accountable in its behavior. Distrust of the monolithic framework that grows, mandates, and makes decisions on your behalf whether you asked it or not.

### The Problem with Full Frameworks

Frameworks force an all-or-nothing decision. Opinions are baked into every layer. This leads to abstractions that you did not ask for that hide what is actually happening. When your needs diverge, you're fighting the framework instead of engineering the solution.

Ultimately engineering comes in second, the framework's conventions come first.

### The Composability Alternative

Compose only what your application needs -- nothing more, nothing less. Each component earns its place by doing precisely one thing. Shared standards serve as the contract layer -- components speak to each other through interfaces, not implementations. In PHP, PSR standards serve this exact role -- defining interfaces that components agree on without mandating how any of them are implemented. The result is small, purposeful, replaceable pieces that assemble into something cohesive.

The engineer decides the architecture, not the framework.

### Principles

- **Define hard boundaries** -- boot is boot, run is run, this should be enforced, not suggested.
- **Immutability as a constraint** -- the service layer freezes at boot, making predictability a guarantee.
- **Explicit over implicit** -- dependencies are declared, not located; registration is separated from resolution.
- **Pure transformation** -- the system transforms inputs, it does not reconfigure itself.
- **Restraint as a signal** -- 165 lines for a DI container, because that's all it needs to be.

### Why this Matters

Applications built this way are predictable, testable, and honest about their own state. Implementation changes do not cause a snowball effect, they are centralized and isolated. Good software earns its place, flawed software is easily removed.

Engineers who understand their stack make better decisions.
