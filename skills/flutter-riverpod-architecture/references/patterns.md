# Implementation Patterns

## Definitions and Exposed State

| Need | Generated definition | Exposed state |
| --- | --- | --- |
| Synchronous dependency or derived query | `@riverpod T name(Ref ref)` | T |
| Async query | Return Future<T> / FutureOr<T> | AsyncValue<T> |
| Continuous subscription | Return Stream<T> | AsyncValue<T> |
| Synchronous mutable state | Extend _$Name with build returning T | T and notifier methods |
| Async state or command | Extend _$Name with build returning FutureOr<T> | AsyncValue<T> and notifier methods |

Import `riverpod_annotation` in each generated source file and add a matching `part 'name.g.dart';`. The generator creates the provider.

## Minimal Read/Write Implementation

This example keeps the model, repository, and providers in `tasks.dart`. Split them according to the [architecture](architecture.md) in a real project. The memory repository simulates an application-lifetime data source; choose the real data source and lifetime for the feature.

```dart
import 'dart:async';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'tasks.g.dart';

class Task {
  const Task(this.title);
  final String title;

  @override
  bool operator ==(Object other) => other is Task && other.title == title;
  @override
  int get hashCode => title.hashCode;
}

abstract class TaskRepository {
  Future<List<Task>> fetchAll();
  Future<void> add(String title);
}

class MemoryTaskRepository implements TaskRepository {
  final _tasks = <Task>[];

  @override
  Future<List<Task>> fetchAll() async => List.unmodifiable(_tasks);

  @override
  Future<void> add(String title) async => _tasks.add(Task(title));
}

@Riverpod(keepAlive: true)
TaskRepository taskRepository(Ref ref) => MemoryTaskRepository();

@riverpod
Future<List<Task>> tasks(Ref ref) =>
    ref.watch(taskRepositoryProvider).fetchAll();

// Named parameters generate a family.
@riverpod
Future<List<Task>> filteredTasks(Ref ref, {required String search}) async {
  final values = await ref.watch(tasksProvider.future);
  return List.unmodifiable(
    values.where((task) => task.title.contains(search)),
  );
}

@riverpod
class AddTaskController extends _$AddTaskController {
  @override
  FutureOr<void> build() {} // Synchronous AsyncData<void> initial state; generates AsyncNotifier.

  // true: successful in the current lifecycle; false: rejected, failed, or stale.
  // The UI displays failures through state.
  Future<bool> submit(String title) async {
    if (state.isLoading) return false;
    final operationRef = ref;
    state = const AsyncLoading(); // Set before awaiting to reject duplicate submissions on this instance.
    try {
      final repository = operationRef.read(taskRepositoryProvider);
      await repository.add(title);
      if (!operationRef.mounted) return false;
      operationRef.invalidate(tasksProvider);
      state = const AsyncData(null);
      return true;
    } catch (error, stackTrace) {
      if (!operationRef.mounted) return false;
      state = AsyncError(error, stackTrace);
      return false;
    }
  }
}
```

Define real entity equality using stable IDs and fields affecting the UI. List.unmodifiable supplies a read-only collection; choose collection value equality according to the project's model strategy.

A command controller uses synchronous `FutureOr<void> build() {}` to generate AsyncNotifier with an immediate AsyncData<void> initial state, allowing the first submission to run immediately.

When using AsyncValue.guard, store the awaited result locally, check the operation-start Ref's mounted state, then assign state. For consistency across pages, let a repository stream or long-lived business owner manage write completion and data updates.

## Widget Consumption

Place `tasks_screen.dart` alongside the file above and use `ProviderScope(child: MaterialApp(home: TasksScreen()))` at the root. Map errors to localized messages in the target application.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'tasks.dart';

class TasksScreen extends ConsumerStatefulWidget {
  const TasksScreen({super.key});

  @override
  ConsumerState<TasksScreen> createState() => _TasksScreenState();
}

class _TasksScreenState extends ConsumerState<TasksScreen> {
  final _title = TextEditingController();

  @override
  void dispose() {
    _title.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final query = ref.watch(tasksProvider);
    final command = ref.watch(addTaskControllerProvider); // Also keeps the controller alive.
    ref.listen(addTaskControllerProvider, (previous, next) {
      if (next.hasError && !identical(previous?.error, next.error)) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Save failed. Please try again.')),
        );
      }
    });

    return Scaffold(
      appBar: AppBar(title: const Text('Tasks')),
      body: Column(
        children: [
          TextField(controller: _title),
          ElevatedButton(
            onPressed: command.isLoading ? null : () async {
              final title = _title.text.trim();
              if (title.isEmpty) return;
              final saved = await ref
                  .read(addTaskControllerProvider.notifier)
                  .submit(title);
              if (!mounted) return; // Check the widget lifecycle separately from the Ref lifecycle.
              // Preserve new text entered while the request was pending.
              if (saved && _title.text.trim() == title) _title.clear();
            },
            child: Text(command.isLoading ? 'Saving...' : 'Add'),
          ),
          Expanded(
            child: query.when(
              loading: () => const Center(child: CircularProgressIndicator()),
              error: (error, stackTrace) => Center(
                child: TextButton(
                  onPressed: () => ref.invalidate(tasksProvider),
                  child: const Text('Loading failed. Tap to retry.'),
                ),
              ),
              data: (values) => values.isEmpty
                  ? const Center(child: Text('No tasks yet'))
                  : ListView(children: [
                      for (final task in values)
                        ListTile(title: Text(task.title)),
                    ]),
            ),
          ),
        ],
      ),
    );
  }
}
```

Use ref.listen inside build for framework-managed lifecycle handling. Use ref.listenManual outside build, such as in initState; its subscription can close early and is cleaned up when the widget is disposed. Trigger success navigation from the submit result when appropriate.

## Extension Decisions

- Class-based family: declare parameters in `build(String id)` and consume with `ref.watch(controllerProvider(id))`.
- Dependency recomputation: use watch in provider build; use read in commands to capture the initiating user and repository.
- Service: accept repositories through constructor parameters and wire them with watch; introduce services for shared business workflows.
- Network requests: register request-token cancellation with onDispose before awaiting; shared clients are cleaned up by their owners.
- Search: drive queries with family parameters and implement debounce/cancellation within the query lifecycle. Choose rejection, serialization, or latest-wins for concurrent commands according to the operation.
- Refresh: invalidate marks cached state stale; refresh reads it immediately. For pull-to-refresh completion, await `ref.refresh(tasksProvider.future)`.
- Performance: use small Consumer widgets or select to subscribe to the required immutable fields.
- Local scope: prefer families for route parameters. Declare dependencies according to scoping and lint rules when overriding providers in nested ProviderScopes.

References: [Generator](https://riverpod.dev/docs/concepts/about_code_generation), [Refs](https://riverpod.dev/docs/concepts2/refs), [DO/DON'T](https://riverpod.dev/docs/root/do_dont), [Scoping](https://riverpod.dev/docs/concepts2/scoping).
