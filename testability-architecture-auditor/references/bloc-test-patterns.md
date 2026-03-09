# Bloc Test Patterns Reference

This reference defines what bloc-testable architecture looks like from an auditor's perspective. Use it to calibrate findings in Step 5 (Bloc/Cubit testability).

---

## blocTest Anatomy

```dart
blocTest<LoginBloc, LoginState>(
  'emits [loading, success] when credentials are valid',
  build: () => LoginBloc(repository: FakeAuthRepository()),
  seed: () => LoginInitial(),            // optional initial state to emit before act
  act: (bloc) => bloc.add(LoginSubmitted(email: 'a@b.com', password: '123')),
  wait: const Duration(milliseconds: 100), // optional: wait for async operations
  expect: () => [
    LoginLoading(),
    LoginSuccess(user: fakeUser),
  ],
  verify: (bloc) {
    verify(() => fakeRepo.signIn('a@b.com', '123')).called(1);
  },
);
```

**For the `blocTest` to work, the Bloc must:**
1. Receive `FakeAuthRepository` via constructor — no hardcoded `AuthRepositoryImpl`
2. Emit `LoginLoading` then `LoginSuccess` synchronously (or within `wait`) — no `BuildContext` call that would throw in a headless test
3. Have no `Navigator.push` or `showDialog` inside the event handler — those crash in a unit test environment

---

## Navigation Side Effect Pattern

**Incorrect (blocks blocTest):**

```dart
// VIOLATION — BuildContext in Bloc blocks blocTest
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  Future<void> _onLogin(LoginEvent event, Emitter emit) async {
    emit(AuthLoading());
    final user = await _repo.signIn(event.email, event.password);
    emit(AuthSuccess(user));
    Navigator.of(event.context).pushReplacementNamed('/home'); // HIGH — crashes in blocTest
  }
}
```

**Correct (fully testable):**

```dart
// CORRECT — Bloc emits navigation state; BlocListener in widget handles nav
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  Future<void> _onLogin(LoginEvent event, Emitter emit) async {
    emit(AuthLoading());
    final user = await _repo.signIn(event.email, event.password);
    emit(AuthSuccess(user)); // BlocListener navigates on this state
  }
}

// In the widget:
BlocListener<AuthBloc, AuthState>(
  listener: (context, state) {
    if (state is AuthSuccess) Navigator.pushReplacementNamed(context, '/home');
  },
);
```

The `blocTest` can now assert `[AuthLoading(), AuthSuccess(user)]` cleanly.

---

## Side-Effect-Only Events (verify-only blocTest)

Some Bloc events exist only to trigger a side effect (analytics, push notification) without changing state. Test them with `verify` instead of `expect`:

```dart
blocTest<AnalyticsBloc, AnalyticsState>(
  'calls trackEvent for PageViewed',
  build: () => AnalyticsBloc(tracker: mockTracker),
  act: (bloc) => bloc.add(PageViewed(name: 'home')),
  expect: () => [],  // no state change
  verify: (_) => verify(() => mockTracker.track('home')).called(1),
);
```

**Audit signal:** A Bloc that only has side-effect events with no `expect` is testable — do not flag it as untestable because `expect` is empty.

---

## Async Event Testing Patterns

| Async pattern | How to test | Audit note if missing |
|---|---|---|
| `await emit.forEach(stream, ...)` | Use real or fake `StreamController` injected via ctor | HIGH if stream is a global/static singleton |
| `await repo.fetch()` with delay | Set `wait:` in `blocTest` to cover async gap | MEDIUM if event has no test at all |
| `Timer.periodic` in constructor | Inject `Duration` and provide fake timer or skip | MEDIUM — constructor timer makes bloc hard to close cleanly |
| `Future.delayed` inside event | Replace with injectable delay fn or `Clock` | MEDIUM |

---

## MockBloc in Widget Tests

When writing `testWidgets` for a screen that consumes a Bloc, use a `MockBloc` to control state:

```dart
// mocktail
class MockLoginBloc extends MockBloc<LoginEvent, LoginState> implements LoginBloc {}

testWidgets('shows loading indicator when state is LoginLoading', (tester) async {
  final bloc = MockLoginBloc();
  when(() => bloc.state).thenReturn(LoginLoading());

  await tester.pumpWidget(
    BlocProvider<LoginBloc>.value(
      value: bloc,
      child: const MaterialApp(home: LoginScreen()),
    ),
  );

  expect(find.byType(CircularProgressIndicator), findsOneWidget);
});
```

**Audit signal:** If `MockBloc` is absent from all widget tests, the tests either use a real Bloc (requires real DI setup) or skip testing state-driven UI — both are coverage gaps.

---

## BlocListener Side Effect Testability

`BlocListener` callbacks (navigation, snackbars, dialogs) are triggered by state changes. To test them in `testWidgets`:

```dart
testWidgets('navigates to home on AuthSuccess', (tester) async {
  final bloc = MockAuthBloc();
  whenListen(bloc, Stream.fromIterable([AuthSuccess(fakeUser)]),
      initialState: AuthLoading());

  await tester.pumpWidget(
    BlocProvider<AuthBloc>.value(
      value: bloc,
      child: MaterialApp(
        home: LoginScreen(),
        routes: {'/home': (_) => const HomeScreen()},
      ),
    ),
  );
  await tester.pumpAndSettle();

  expect(find.byType(HomeScreen), findsOneWidget);
});
```

**Audit signal:** If no widget test exercises a `BlocListener` side effect, navigation and dialog logic is completely untested regardless of `blocTest` coverage.

---

## Event/State Sequence Coverage Thresholds

| Coverage level | Signal | Audit classification |
|---|---|---|
| 0 `blocTest` calls for a Bloc with 3+ events | No bloc tests exist | HIGH — entire state machine untested |
| `blocTest` only for happy path | Error/empty states not tested | MEDIUM |
| `blocTest` for happy + error paths | Core coverage present | Acceptable for Level 3 |
| `blocTest` for all event/state combinations including edge cases | Full coverage | Level 4 signal |

---

## Severity Table — Bloc Test Blockers

| Finding | Severity | Rationale |
|---|---|---|
| `BuildContext` param in Bloc event handler | HIGH | `blocTest` crashes |
| `Navigator.push` called inside Bloc handler | HIGH | No navigation in headless test |
| `showDialog` / `ScaffoldMessenger` inside Cubit | HIGH | UI call in headless test crashes |
| Concrete `*Impl` in Bloc constructor | HIGH | Cannot inject fake in blocTest |
| `DateTime.now()` in Bloc for time logic | MEDIUM | Non-deterministic state diffs |
| `Timer.periodic` in Bloc constructor | MEDIUM | Bloc cannot be closed cleanly in test |
| Zero `blocTest` in project with 5+ Blocs | HIGH | Entire state machine untested |
| `blocTest` only for happy path | MEDIUM | Error/empty states unverified |
