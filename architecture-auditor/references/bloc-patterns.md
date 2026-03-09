# BLoC / Cubit Reference — State Management Audit Guide

Use this file to evaluate the state management strategy of the audited project. Compare observed patterns against these standards to determine severity of issues.

---

## BLoC vs Cubit — Decision Signals

| Use **Cubit** | Use **BLoC** |
|---|---|
| Simple, linear state (counter, toggle, form step) | Multiple event types with different branching |
| No external stream subscriptions | Firestore / WebSocket real-time subscriptions |
| Stateless business logic | Debounced input (search, typeahead) |
| Sub-state of a single widget | Undo/redo, event history matters |

**Audit signal:** If a Bloc has only one event type, it should have been a Cubit. If a Cubit has `if/else` chains that depend on the type of "call", it should have been a Bloc.

---

## Correct BLoC Structure

A well-implemented Bloc has these characteristics:

- Events and states in separate files (`*_event.dart`, `*_state.dart`)
- States defined with `@freezed` — sealed union with `initial`, `loading`, `loaded`, `error`
- `@injectable` annotation — instantiated via DI, not `SomeBloc()`
- No `BuildContext` references inside the Bloc
- No direct datasource or Firebase calls — only use case calls
- One `on<EventType>(_handler)` per event

```
✅ Correct call chain:
Bloc → UseCase → Repository (interface) → [implemented by] RepositoryImpl → Datasource

❌ Violation:
Bloc → Datasource directly (use case skipped)
Bloc → RepositoryImpl directly (depends on concrete, not interface)
Widget → Repository directly (Bloc skipped)
```

---

## Correct Screen / BlocProvider Split

The screen (`@RoutePage()`) provides the Bloc. The view consumes it. They must be separate:

```
✅ ProductScreen (BlocProvider + getIt<ProductBloc>) 
     └── ProductView (BlocBuilder — reads from context, no DI knowledge)

❌ Anti-pattern:
ProductScreen with Scaffold, AppBar, ListView, BlocBuilder, AND BlocProvider all in one file
```

**Audit signal:** If `@RoutePage()` classes contain `BlocBuilder` directly in `build()`, the screen/view split is absent.

---

## Correct State Consumption Patterns

| Widget | Use for | Audit signal if missing |
|---|---|---|
| `BlocBuilder` | Rendering UI based on state | Using `setState` + manual Bloc stream subscription |
| `BlocListener` | Side effects only (navigation, snackbars) | Navigation called inside `BlocBuilder.builder` |
| `BlocConsumer` | Build + listen together | `BlocBuilder` with navigation logic in `builder` |
| `BlocSelector` | Rebuild only when a sub-field changes | `BlocBuilder` rebuilding entire tree for partial state |

---

## Red Flags — High Severity

- `setState` used inside a widget that also has a `BlocProvider` → mixed state management
- `Bloc` or `Cubit` instantiated with `MyBloc()` instead of `getIt<MyBloc>()` → DI bypassed
- `context.read<MyBloc>()` called inside `initState` or `build()` for triggering events is acceptable; calling it to read state for rendering is a violation
- `StreamSubscription` inside a widget's `State` that could be a `BlocListener`
- Navigation (`context.router.push(...)`) inside `BlocBuilder.builder` → should be in `BlocListener`

---

## Red Flags — Medium Severity

- Single bloc file (`product_bloc.dart`) containing event, state, and bloc classes → missing separation
- States defined as plain classes instead of `@freezed` → no exhaustive pattern matching
- `emit()` called after `await` without checking `isClosed` → potential state emission after disposal
- Bloc with more than ~300 lines → God-bloc, should be split

---

## Stream Subscriptions (Real-Time Data)

When a Bloc subscribes to a Firestore stream, it must:
1. Cancel the `StreamSubscription` in `close()`
2. Use a dedicated event to relay stream data into the Bloc (`on<DataReceived>`)

```
✅ Correct:
  _onStarted → subscribe → stream emits → add(DataReceived(...)) → _onReceived → emit state
  close() → cancel subscription

❌ Violation:
  Widget directly subscribes to a Firestore stream with StreamBuilder
  Bloc exposes a Stream property that the widget listens to directly
```

---

## Mixed State Management — Severity Assessment

| Situation | Severity |
|---|---|
| All features use flutter_bloc consistently | ✅ No issue |
| Majority BLoC, one feature uses Riverpod | MEDIUM |
| Some features BLoC, others `setState` in complex screens | HIGH |
| No state management library — all `setState` | HIGH (Prototype-level) |
| `setState` used only for trivial local UI state (checkbox toggle) | LOW / Acceptable |

---

## Testing Signal

A well-architected state management layer is testable with `bloc_test`:

```
✅ Testable: Bloc that accepts injected use cases (mockable)
❌ Not testable: Bloc that instantiates use cases inline (new GetProductsUseCase())
❌ Not testable: State logic inside widget build() method
```

If the project has `test/features/*/bloc/*_bloc_test.dart` files, state management is likely well-isolated. Their absence is a maintainability signal.
