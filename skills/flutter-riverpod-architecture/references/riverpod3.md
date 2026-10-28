# Riverpod 3 Versions and APIs

## Dependencies and Generation

Target runtime 3.x and select compatible stable versions using the project's Flutter/Dart SDK, existing constraints, and lockfile. Annotation, generator, and lint packages evolve independently and resolve through their actual dependency relationships.

- Runtime dependencies: flutter_riverpod, riverpod_annotation.
- Development dependencies: riverpod_generator, build_runner; add riverpod_lint as appropriate for the project.
- For new dependencies, select current compatible stable versions within the target major. Preserve existing lockfiles and treat upgrades as explicit dependency changes.
- Keep pubspec.lock for applications and run generation, analysis, and relevant tests after upgrades.
- Configure riverpod_lint according to the resolved version's documentation. Versions using analysis server plugins belong under the top-level plugins section of analysis_options.yaml.

References: [Setup and dependencies](https://riverpod.dev/docs/introduction/getting_started), [Generator](https://pub.dev/packages/riverpod_generator), [Lint](https://pub.dev/packages/riverpod_lint).

## Core APIs

| Scenario | Syntax and semantics |
| --- | --- |
| Function provider | `Ref ref`; widgets use WidgetRef |
| Class provider | `class X extends _$X`; build parameters define families and annotations define lifetimes |
| AsyncValue | `.value` reads the nullable known value; `.requireValue` requires a value; render data, loading, and failure explicitly |
| State notifications | Updates are filtered by `==` by default; express changes through immutable updates and appropriate value equality |
| Lifecycle | Capture the operation-start Ref and check its mounted state on completion |
| Observer | Callbacks receive ProviderObserverContext; a final class can extend ProviderObserver |

Rethrown provider-read errors may be wrapped in ProviderException, with the original error in `exception`. AsyncValue.error and Observer expose errors according to their own contracts. Classify errors at the relevant boundary and preserve original stack traces.

An AsyncValue can contain previous data together with loading/error flags during refresh. Choose between retaining data and showing full loading/error views according to product needs. Use state indicators such as hasValue for nullable data.

References: [v3 behavior](https://riverpod.dev/docs/3.0_migration), [AsyncValue](https://pub.dev/documentation/riverpod/latest/riverpod/AsyncValue-class.html), [Observer](https://riverpod.dev/docs/concepts2/observers).

## Automatic Retry

Provider computation failures trigger retry. The current default permits up to 10 retries, with exponential backoff from 200ms to 6.4s, and skips Error and ProviderException.

A retry policy has the signature `Duration? retry(int retryCount, Object error)`: return a delay to continue or null to stop. Configure it globally on ProviderScope / ProviderContainer or locally with `@Riverpod(retry: policy)`. Distinguish transient network failures, authentication failures, and validation failures.

Define retry and idempotency policies separately for user commands. Failure-query tests can use `retry: (_, _) => null` for deterministic call counts and timing.

References: [Retry](https://riverpod.dev/docs/concepts2/retry), [Default implementation](https://pub.dev/documentation/riverpod/latest/riverpod/ProviderContainer/defaultRetry.html).

## Lifecycles and Caching

`@riverpod` enables automatic disposal by default. `@Riverpod(keepAlive: true)` retains state without listeners; dependency changes and invalidation still dispose the previous computation. Families with many parameter combinations generally use automatic disposal and parameters with stable `==/hashCode`, such as IDs, records, or immutable value objects.

- Keep state alive through UI watch, strong listen, or a temporary keepAlive link as appropriate.
- Close a `ref.keepAlive()` link according to an explicit cache-expiration policy.
- Use onDispose to clean up owned requests, timers, and subscriptions. The provider creating a shared client owns its cleanup.
- Offscreen Flutter consumers pause according to TickerMode, and subscriptions support pause/resume. Give continuously running work an explicit business owner.
- Use `weak: true` listeners for observation and strong listeners when keeping state alive.

A Notifier instance can survive rebuilds. Start async commands with `final operationRef = ref` and check `operationRef.mounted` after awaiting so results belong to the initiating lifecycle. Handle out-of-order requests within the same lifecycle through cancellation, request sequences, or serialization.

References: [Automatic disposal](https://riverpod.dev/docs/concepts2/auto_dispose), [Family](https://riverpod.dev/docs/concepts2/family), [Refs](https://riverpod.dev/docs/concepts2/refs), [Cancellation and debounce](https://riverpod.dev/docs/how_to/cancel).

## Optional Extensions

- The generator supports generic providers and multiple or named family parameters.
- Where supported by the resolved version, FutureOr provider initialization can compose async dependencies with `.requireValue`. Conventional async composition can use `await ref.watch(dep.future)`.
- Use Ref.isPaused, onManualInvalidation, and family-wide overrideWith2 according to the project's resolved API version.
- Testing utilities include ProviderContainer.test, WidgetTester.container(), and overrideWithBuild; see [Testing](testing.md).

Mutation models command idle/pending/error/success states separately. Import `flutter_riverpod/experimental/mutation.dart`, call `mutation.run(ref, (tsx) async { ... })`, and obtain dependencies through `tsx.get(provider)` to keep them alive during the operation. Use separate keys or instances for entity- or page-specific operations.

Offline persistence restores state through `experimental/persist.dart`, Storage, and Notifier.persist. The generator can assist with `@JsonPersist()`. Include user, tenant, and family boundaries in keys and design serialization, expiration, sign-out cleanup, and data protection.

Mutation and persistence are experimental APIs. Check their interfaces against the resolved version and cover their behavior with tests. Data layers own database migrations, offline write queues, and conflict resolution.

References: [Runtime changelog](https://pub.dev/packages/riverpod/changelog), [Ref API](https://pub.dev/documentation/riverpod/latest/riverpod/Ref-class.html), [Mutation](https://riverpod.dev/docs/concepts2/mutations), [Persistence](https://riverpod.dev/docs/concepts2/offline).
