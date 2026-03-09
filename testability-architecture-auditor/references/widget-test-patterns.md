# Widget Test Patterns Reference

This reference defines what widget-testable architecture looks like from an auditor's perspective. Use it to calibrate findings in Step 6 (widget test isolation).

---

## What Makes a Widget Widget-Testable

A widget is testable with `testWidgets` when:

1. All its state comes from an injected Bloc/Cubit (via `BlocProvider`) or from constructor parameters — not from a service it instantiates itself
2. It does not call `context.read<Repository>()` in lifecycle methods — only `context.read<Bloc>().add(event)` is acceptable in callbacks
3. Its async behavior is triggered by events, not by `FutureBuilder` calling a real repository
4. It has no hardcoded service locator access inside the widget body

If any of these conditions is violated, the `testWidgets` pump either requires real infrastructure (network, DB) or throws.

---

## Standard Pump Setup Pattern

```dart
Widget buildTestWidget({required LoginBloc bloc}) {
  return MaterialApp(
    home: BlocProvider<LoginBloc>.value(
      value: bloc,
      child: const LoginScreen(),
    ),
  );
}

testWidgets('shows error on LoginFailure', (tester) async {
  final bloc = MockLoginBloc();
  when(() => bloc.state).thenReturn(LoginFailure('Invalid credentials'));

  await tester.pumpWidget(buildTestWidget(bloc: bloc));

  expect(find.text('Invalid credentials'), findsOneWidget);
});
```

**Audit signal:** If a project has 0 `testWidgets` calls in `test/`, or if all `testWidgets` calls require real network/repository setup, widget testing is architecturally blocked.

---

## RepositoryProvider + BlocProvider Nesting

When a screen requires both direct repository access and a Bloc, the correct test pump wraps both:

```dart
await tester.pumpWidget(
  RepositoryProvider<AuthRepository>.value(
    value: fakeAuthRepository,
    child: BlocProvider<ProfileBloc>.value(
      value: MockProfileBloc(),
      child: MaterialApp(home: ProfileScreen()),
    ),
  ),
);
```

**Audit signal:** If `RepositoryProvider` is found in production widget trees but never in test files, the test suite is structurally incomplete — screens that read repositories via `context.read<AuthRepository>()` will throw `ProviderNotFoundException` in tests.

---

## Finder Strategy Audit Signals

| Finder type | When to prefer | Audit concern if over-used |
|---|---|---|
| `find.byType(LoginButton)` | Stable — preferred when widget type is unique | Low risk |
| `find.text('Login')` | Acceptable for display text | Breaks if string is localized |
| `find.byKey(const Key('loginButton'))` | Required when multiple widgets of same type exist | Acceptable |
| `find.byWidget(specificInstance)` | Rare — avoid | Fragile; couples test to widget reference |

**Audit concern:** If all finds use `find.text(...)` with raw English strings and the project has i18n, the test suite will break on locale switch. Note as MEDIUM.

---

## Async Widget Testing Patterns

| Async scenario | Test approach | Audit note if absent |
|---|---|---|
| Bloc emits state after async event | `whenListen(bloc, stream)` + `pumpAndSettle` | MEDIUM — async state transitions untested |
| Widget shows loading spinner then content | Emit `Loading` then `Success` via `whenListen` | MEDIUM |
| Navigation triggered by `BlocListener` | `whenListen` + `pumpAndSettle` + find new route | HIGH if navigation is never tested |
| Form submission validation | `tester.tap(submitButton)` + `tester.pump()` | LOW if form is fully bloc-driven |

---

## initState Anti-Patterns That Block Widget Tests

```dart
// VIOLATION — async call in initState without injection point
class HomeScreen extends StatefulWidget { ... }
class _HomeScreenState extends State<HomeScreen> {
  @override
  void initState() {
    super.initState();
    context.read<ProductBloc>().add(LoadProducts()); // acceptable IF ProductBloc is BlocProvider-injected
  }
}
```

vs.

```dart
// VIOLATION — creates its own repository; testWidgets cannot replace it
class HomeScreen extends StatefulWidget { ... }
class _HomeScreenState extends State<HomeScreen> {
  late final _repo = ProductRepositoryImpl(); // HIGH — hardcoded; no injection point
  @override
  void initState() {
    super.initState();
    _repo.fetchAll().then((data) => setState(() => _products = data));
  }
}
```

The first is testable (pump with `MockProductBloc`). The second is not without modifying the widget class.

---

## Widget Test Coverage Thresholds

| Project scale | Minimum testWidgets count | Audit classification if below |
|---|---|---|
| < 10 screens | 5 | MEDIUM |
| 10–30 screens | 20 | MEDIUM |
| 30+ screens | 40+ | HIGH if < 15 |

These thresholds are baselines for Level 3 architecture. Level 4 expects at least one `testWidgets` per screen for the happy-path state.

---

## Severity Table — Widget Test Blockers

| Finding | Severity | Rationale |
|---|---|---|
| Widget instantiates repository in constructor or initState | HIGH | testWidgets cannot inject fake |
| Widget calls `context.read<Repository>()` in a callback with no RepositoryProvider in tests | HIGH | ProviderNotFoundException at pump |
| No `BlocProvider` or `RepositoryProvider` found in any test file | HIGH | All screens rely on real DI at test time |
| testWidgets count is 0 for a project with 10+ screens | HIGH | Widget layer completely untested |
| Navigation triggered by BlocListener has no widget test | MEDIUM | Navigation correctness unverified |
| find.text() used with raw strings in an i18n project | MEDIUM | Tests break on locale switch |
| No test helper factory for pumping screens | LOW | Test setup is verbose but not a blocker |
