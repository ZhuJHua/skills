# Architecture

## Directories and Responsibilities

Organize features around business capabilities and create the layers each feature needs:

```text
lib/
  app/                         # Root ProviderScope, routing, startup wiring
  core/                        # Shared infrastructure, such as clients and clocks
  features/
    tasks/
      domain/                  # Entities, value objects, business rules
      data/                    # Repositories, data sources, DTOs, mapping
      application/             # Business orchestration when needed
      presentation/
        controllers/
        screens/
        widgets/
test/                          # Mirror the feature structure
```

| Layer | Responsibilities | Dependencies |
| --- | --- | --- |
| domain | Immutable entities, value objects, pure business logic, business failure types | Dart and domain models |
| data | Data access, entity mapping, consistency, data-access error translation | domain, network/database clients |
| application | Workflows spanning repositories, reusable business operations | domain, repositories |
| presentation | Rendering, interactions, command state | Query providers, controllers, business contracts |

Place repository contracts near their implementations in data, or preserve the existing interface/implementation split when the project uses domain ports. Choose abstractions based on business boundaries, reuse, and implementation differences.

Simple models can own JSON serialization. Use DTOs and mappers in data when external schemas differ from domain models or multiple sources are involved.

Keep providers close to the objects or queries they expose. Give ordinary services explicit repository constructor parameters and wire them with `ref.watch` in provider factories. Compose reactive queries directly through providers.

Connect features through public domain, data, or application contracts. Assign shared capabilities to their business owners and keep core focused on common infrastructure.

## Queries, Commands, and Consistency

Queries: data source → repository → query provider → widget. Repositories return entities through Future/Stream; Riverpod supplies reactive dependencies, in-memory caching, and AsyncValue.

Commands: widget event → controller → optional service → repository → data source. Choose an appropriate update path after a successful write:

- Source pushes changes: a repository stream drives query updates.
- Refetch from the server: invalidate affected queries with `ref.invalidate`; use `await ref.refresh(query.future)` when completion matters.
- API returns an authoritative entity: update it in the owning Notifier or coordinate through the repository cache.
- Optimistic updates: preserve rollback data and handle failures and out-of-order results.

Define an owner, invalidation conditions, and synchronization rules for each cache. Implement pagination merging, offline storage, and write synchronization in the corresponding data capabilities.

## State Ownership

| State | Owner |
| --- | --- |
| Input, focus, animation/Tab controllers, temporary page selections | Widget State, with matching initialization and disposal |
| Server lists and details | Query providers parameterized by ID, filters, and relevant identity boundaries |
| Save/delete progress and interaction errors | Command controllers, scoped by entity ID when needed |
| Shared drafts across a multi-step flow | A Notifier with an explicit lifetime |
| Identity/tenant | A session provider; dependent queries watch identity or include it in their keys |

Pass route IDs as family parameters and track row-level command progress per entity.

Give commands that must finish across pages a long-lived service, repository, or queue that owns the business operation and consistency. Page controllers update interaction state within their valid lifetimes.

Capture account, session, and dependencies at command start. Handle cache ownership, old requests, and persisted data when accounts change. Use server idempotency keys, atomic updates, version checks, or transactions for writes across clients.
