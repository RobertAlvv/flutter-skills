# State Scoping and Lifecycle Reference — Audit Guide

Use this file to evaluate whether state is correctly scoped to the features that own it and whether lifecycle (creation and disposal) is properly managed.

---

## The Core Rule: State Lives Where It Is Consumed

State should be provided at the **lowest ancestor** in the widget tree that is shared by all widgets consuming that state. Providing state higher than necessary is the most common scoping violation.

---

## Scoping Decision Guide

| State type | Correct scope | Wrong scope |
|---|---|---|
| Auth session, current user | App root (`MaterialApp` ancestor) | Per-feature |
| Theme, locale | App root | Per-feature |
| Product list for a catalog screen | Route-level (`BlocProvider` at route entry) | App root |
| Cart contents (shared across screens) | App root or `Navigator`-level | Per-screen |
| Form state (text fields, validation) | Screen or form widget level | App root |
| Modal/dialog content | Modal itself | App root |

### Identifying over-scoped state

```dart
// ❌ Feature-specific blocs at app root — live forever, waste memory
MaterialApp(
  home: MultiBlocProvider(
    providers: [
      BlocProvider(create: (_) => AuthBloc()),       // ✅ correct — global
      BlocProvider(create: (_) => ProductListBloc()), // ❌ should be at catalog route
      BlocProvider(create: (_) => CartBloc()),        // depends on product scope
      BlocProvider(create: (_) => CheckoutBloc()),    // ❌ should be at checkout route
    ],
    child: AppRouter(),
  ),
)

// ✅ Feature blocs scoped to their route
GoRoute(
  path: '/catalog',
  builder: (context, state) => BlocProvider(
    create: (_) => ProductListBloc(repository: context.read()),
    child: const CatalogScreen(),
  ),
)
```

**Audit signal:** If `main.dart` or `app.dart` contains more than 3–4 `BlocProvider` entries, investigate whether any of them are feature-specific.

---

## BlocProvider — Lifecycle Guarantee

`BlocProvider` disposes the Bloc automatically when the widget subtree it wraps is removed from the tree. This is the correct lifecycle management approach.

```dart
// ✅ Bloc is created when route is pushed, disposed when route is popped
BlocProvider(
  create: (context) => ProductDetailBloc(
    repository: context.read<ProductRepository>(),
  ),
  child: const ProductDetailScreen(),
)

// ❌ Bloc created manually, never disposed
class _ProductDetailScreenState extends State<ProductDetailScreen> {
  late final ProductDetailBloc _bloc;

  @override
  void initState() {
    super.initState();
    _bloc = ProductDetailBloc(repository: GetIt.I());  // never closed
  }

  @override
  Widget build(BuildContext context) => BlocProvider.value(
    value: _bloc,   // value constructor does NOT dispose
    child: ...,
  );
}
```

**`BlocProvider.value` pitfall:** `BlocProvider.value` passes an existing Bloc instance but does **not** manage its lifecycle. Use it only for providing a Bloc that is owned and disposed elsewhere.

---

## StreamController — Correct Disposal Pattern

```dart
// ❌ StreamController without dispose — memory leak
class _LiveFeedState extends State<LiveFeed> {
  final _controller = StreamController<List<Event>>();

  // No dispose() override — controller never closed
}

// ✅ Proper disposal
class _LiveFeedState extends State<LiveFeed> {
  final _controller = StreamController<List<Event>>();

  @override
  void dispose() {
    _controller.close();  // ← required
    super.dispose();
  }
}
```

---

## ValueNotifier / AnimationController / ScrollController — Disposal Audit

```dart
// ❌ Notifier declared as field but no dispose()
class _CounterState extends State<Counter> {
  final _count = ValueNotifier<int>(0);

  // Missing dispose() — the ValueNotifier is never released
}

// ✅ Correct
class _CounterState extends State<Counter> {
  final _count = ValueNotifier<int>(0);

  @override
  void dispose() {
    _count.dispose();
    super.dispose();
  }
}
```

**Audit grep:** Find `ValueNotifier`, `AnimationController`, `ScrollController`, `TextEditingController` declared in `State` subclasses without a corresponding `dispose()`:

```bash
# Find all State classes — open each and check for dispose()
grep -rn "extends State<" lib/ --include="*.dart" | grep -v "\.g\.dart"
```

---

## StreamSubscription — Cancellation Audit

```dart
// ❌ Subscription never cancelled — rebuilds persist after widget removal
class _NotificationBadgeState extends State<NotificationBadge> {
  late StreamSubscription _sub;

  @override
  void initState() {
    super.initState();
    _sub = notificationStream.listen((event) {
      setState(() => _count = event.count);
    });
    // No dispose() that calls _sub.cancel()
  }
}

// ✅ Proper cancellation
@override
void dispose() {
  _sub.cancel();
  super.dispose();
}
```

---

## Bloc-to-Bloc Communication — Correct Patterns

There are two acceptable patterns for one Bloc responding to another's state:

### Pattern 1 — BlocListener in UI (preferred for UI-triggered flows)

```dart
// AuthBloc state change triggers UserProfileBloc update via UI listener
BlocListener<AuthBloc, AuthState>(
  listenWhen: (previous, current) =>
      previous.status != current.status && current.status == AuthStatus.authenticated,
  listener: (context, state) {
    context.read<UserProfileBloc>().add(UserProfileLoadRequested(userId: state.userId));
  },
  child: ...,
)
```

### Pattern 2 — Bloc subscribing to another Bloc's stream (for background flows)

```dart
// ✅ Bloc subscribes to AuthBloc stream — no BuildContext dependency
class UserProfileBloc extends Bloc<UserProfileEvent, UserProfileState> {
  UserProfileBloc({required AuthBloc authBloc, required UserRepository repository})
      : super(const UserProfileState()) {
    _authSubscription = authBloc.stream.listen((authState) {
      if (authState.status == AuthStatus.authenticated) {
        add(UserProfileLoadRequested(userId: authState.userId));
      }
    });
  }

  late final StreamSubscription _authSubscription;

  @override
  Future<void> close() {
    _authSubscription.cancel();  // ← required
    return super.close();
  }
}
```

**Audit rule:** Flag any Bloc that has a `BuildContext` parameter in its constructor or event handlers — this is the incorrect alternative to the patterns above.

---

## Severity Reference

| Pattern | Severity |
|---|---|
| Feature-specific Bloc at app root (lives for app lifetime) | MEDIUM |
| 8+ feature-specific Blocs in app-root `MultiBlocProvider` | HIGH |
| `StreamController` without `close()` in `dispose()` | HIGH |
| `ValueNotifier` / `AnimationController` without `dispose()` | MEDIUM |
| `StreamSubscription` without `cancel()` | MEDIUM |
| `BlocProvider.value` used as if it managed lifecycle | MEDIUM |
| Bloc instantiated via constructor inside widget without `close()` | HIGH |
| Bloc-to-Bloc via stored `BuildContext` | HIGH |
