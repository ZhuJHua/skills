# Testing and Validation

## Toolchain

Use the project's Flutter/FVM SDK and existing CI, lint, and generated-file conventions:

```sh
flutter pub get
dart run build_runner build
dart format <changed-dart-files>
flutter analyze
flutter test
```

Use `dart run build_runner watch` during development. If dependency tooling fails to compile internally, select a compatible combination based on the SDK, package constraints, and changelogs.

## Test Boundaries

- Test domain rules directly; test service/repository orchestration, mapping, and consistency.
- Create an independent ProviderContainer.test for each test with automatic teardown.
- Override repository/client boundaries and keep the real controller as the test subject.
- Keep autoDispose state alive with strong container.listen subscriptions; close them and use container.pump to advance disposal.
- Control timing with provider.future, Completer, container.pump, and WidgetTester.pump. In testWidgets, pump as needed before awaiting delivered async results.
- Give failure-query tests deterministic retry policies.

## Companion Unit Tests

These tests accompany the [implementation patterns](patterns.md). Replace the package name and source path for the target project:

```dart
import 'dart:async';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:your_app/tasks.dart';

class ControlledRepository implements TaskRepository {
  final pending = Completer<void>();
  var adds = 0;
  final values = <Task>[];

  @override
  Future<void> add(String title) async {
    adds++;
    await pending.future;
    values.add(Task(title));
  }

  @override
  Future<List<Task>> fetchAll() async => List.unmodifiable(values);
}

void main() {
  test('duplicate submissions write once and refresh the loaded list on success', () async {
    final repository = ControlledRepository();
    final container = ProviderContainer.test(
      retry: (_, _) => null,
      overrides: [taskRepositoryProvider.overrideWithValue(repository)],
    );
    final command = container.listen(addTaskControllerProvider, (_, _) {});
    final query = container.listen(tasksProvider, (_, _) {});
    addTearDown(command.close);
    addTearDown(query.close);
    expect(command.read().isLoading, false);
    expect(await container.read(tasksProvider.future), isEmpty);

    final controller = container.read(addTaskControllerProvider.notifier);
    final first = controller.submit('first');
    expect(command.read().isLoading, true);
    expect(await controller.submit('second'), false);
    expect(repository.adds, 1);
    repository.pending.complete();
    expect(await first, true);
    expect((await container.read(tasksProvider.future)).single.title, 'first');
    expect(command.read().hasError, false);
  });

  test('request completion is handled safely after page disposal', () async {
    final repository = ControlledRepository();
    final container = ProviderContainer.test(
      overrides: [taskRepositoryProvider.overrideWithValue(repository)],
    );
    final command = container.listen(addTaskControllerProvider, (_, _) {});
    final result = container
        .read(addTaskControllerProvider.notifier)
        .submit('late');
    command.close();
    await container.pump();
    repository.pending.complete();
    expect(await result, false);
  });
}
```

Add tests for failure states, retries, stale results after rebuilding, account changes, out-of-order results, and cache-invalidation scope as relevant to the change.

## Widgets and Overrides

Inject fakes through a root ProviderScope's overrides and mount the required environment, such as MaterialApp. Access the actual scope through `tester.container()`.

Verify observable behavior: data, empty states, failure recovery, submission button state, messages, refreshes, and input/routing lifecycles.

Use `overrideWithBuild` to replace Notifier initialization and Future/Stream provider `overrideWithValue` to replace query results. Choose the override level according to the tested boundary and use ordinary lists for overrides.

References: [Testing guide](https://riverpod.dev/docs/how_to/testing), [Container.test](https://pub.dev/documentation/riverpod/latest/riverpod/ProviderContainer/ProviderContainer.test.html), [Lint](https://pub.dev/packages/riverpod_lint).
