---
name: flutter-riverpod-architecture
description: Design, implement, refactor, and review Flutter features with flutter_riverpod 3.x and riverpod_generator, covering feature-first layers, dependency wiring, async reads and writes, lifecycle management, and testing.
---

# Flutter Riverpod Architecture

Organize code by feature with data, domain, application, and presentation layers. Use Riverpod 3 for reactive dependencies, query caching, and interaction state.

## Getting Started

Inspect the project's `AGENTS.md`, `pubspec.yaml`, lockfile, SDK/FVM configuration, and existing feature conventions. Identify state owners, queries, commands, and lifetimes.

- Define providers with `flutter_riverpod` and `riverpod_generator`.
- Consume state with ConsumerWidget / ConsumerStatefulWidget. Manage local input and Flutter controllers in Widget State.
- Target runtime 3.x and select compatible stable versions using the SDK, existing constraints, and lockfile. Resolve annotation and generator packages through their own dependency constraints.
- Read the references relevant to the task, preserve the project's structure, and make incremental changes by feature.

| Task | Reference |
| --- | --- |
| Feature boundaries, directories, layer dependencies, state ownership | [Architecture](references/architecture.md) |
| v3 APIs, retry, lifecycles, caching, extensions | [Versions and APIs](references/riverpod3.md) |
| Generated providers, controllers, widgets, data flow | [Implementation patterns](references/patterns.md) |
| Fakes, overrides, async timing, generation, validation | [Testing and validation](references/testing.md) |

## Core Decisions

1. Keep domain models and rules in pure Dart. Use data for external access and presentation for rendering and interaction state. Introduce application services for shared business orchestration when needed.
2. Use generated function providers for queries and generated Notifier / AsyncNotifier classes for mutable state and user commands. Wire ordinary business objects through providers.
3. Model query data and submission progress separately. Initialize providers with repeatable reads and subscriptions; trigger writes from user events.
4. Use `watch` for reactive dependencies, `read` for one-time access in events, and `listen` for UI reactions such as navigation and messages.
5. Choose automatic disposal, keepAlive, and cleanup policies based on resource ownership. Include the necessary identity, tenant, and query parameters in cache boundaries.
6. Define command concurrency, error contracts, and behavior after leaving the page. Capture the operation-start Ref and check mounted after async gaps before updating state.
7. Publish immutable state and design value equality around v3's `==` notification semantics.
8. Use stable controllers for ordinary features. When adopting Mutation or persistence, account for the API's stability and version constraints.

## Completion Criteria

Generate `.g.dart` files using project conventions and run relevant formatting, analysis, and behavior tests. Cover the success, failure, concurrency, lifecycle, and cache-update scenarios affected by the change.

Report state ownership, data flow, dependency changes, and actual validation results.
