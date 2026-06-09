---
title: 'Two Phases, Two APIs'
description: 'How enforcing the boot/run boundary makes applications predictable, testable, and honest about their own state'
published: true
tags: 'architecture, softwareengineering, programming'
id: 3852903
date: '2026-06-09T03:01:24Z'
---

Most applications have no enforced lifecycle boundaries. Services get resolved at arbitrary times, configs mutate mid-request, init logic bleeds into request handling. The result is an application that is difficult to reason about because its state is never truly settled.

### The Two Phases

**Boot**: The period before the application handles any work. The sole purpose of the boot phase is construction, registering service definitions, merging configuration, wiring dependencies, and running initialization logic. When boot ends, the service layer is immutable, nothing new can be registered, nothing can be reconfigured.

**Run**: The application transforms inputs into outputs using the fully constructed service layer. No reconfiguration, no new definitions, no structural mutations, only pure transformation.

The hard boundary between boot and run is not a convention, it is a constraint.

### Why the Boundary Matters

- **Predictability**: if the service layer is immutable after boot you always know what you are working with during a request
- **Testability**: an immutable service graph is easy to inspect and reproduce in tests
- **Debuggability**: boot-time errors fail loud and early, they do not surface as mysterious runtime behavior
- **Security**: a service layer that cannot be mutated at runtime prevents the injection of rogue providers or configuration overrides, shrinking the attack surface

Boot is boot, run is run, this should be enforced, not suggested.

### The Two APIs

The package [`georgeff/kernel`](https://github.com/MikeGeorgeff/kernel) enforces this at the type level.

When correctly composed, kernel definitions are registered before boot is called, and the fully initialized container is available after.

```php
$kernel = new Kernel();

$kernel->addDefinition(MyClass::class, fn() => new MyClass());
$kernel->boot();
$container = $kernel->getContainer();

$container->get(MyClass::class);
```

When `addDefinition` is called after boot, a `KernelException` will be thrown because the service layer is immutable.

```php
$kernel = new Kernel();

$kernel->boot();
$kernel->addDefinition(MyClass::class, fn() => new MyClass());
```

If the container is accessed before boot, a `KernelException` will be thrown because the container has not yet been initialized.

```php
$kernel = new Kernel();

$container = $kernel->getContainer();

$kernel->boot();
```

Run begins the moment `boot()` returns, after the kernel is a resolved, immutable service graph. The handoff from construction to transformation is complete.

### Modules as Boot-Time Citizens

In the `georgeff/kernel` package, modules enforce the phase separation by design. `ModuleInterface::register()` is a boot-time method, whose purpose is to register service definitions, tags, and decorators. `BootableModuleInterface::boot()` is called at the tail end of the boot cycle, providing access to the built container for initialization work. New definitions cannot be registered at this stage since the service graph is already sealed. Neither has any role in run, making it structurally impossible to blur the line.

### Phases as Architecture

The boot/run distinction is not a pattern unique to this kernel, it is a general principle that most applications implement accidentally and inconsistently. Making it explicit, enforcing it with hard constraints, and designing your application around it produces software that is honest about its own state.

---

**Related**: [Natural Aristoi](https://dev.to/georgeff/natural-aristoi-h6j) — the philosophy behind the design decisions in this package.
