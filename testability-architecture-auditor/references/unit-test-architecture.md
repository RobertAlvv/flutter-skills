# Unit Test Architecture Reference

This reference defines what a unit-testable Flutter architecture looks like from an auditor's perspective. Use it to calibrate findings in Step 1 (DI patterns), Step 3 (business logic placement), and Step 4 (repository abstraction).

---

## Why Architecture Determines Unit Testability

A unit test exercises a single class in isolation, replacing all collaborators with test doubles (fakes or mocks). This is only possible when the class:

1. Declares collaborators as constructor parameters typed to **abstract interfaces**
2. Contains **no side effects** (no network, no platform calls, no file I/O)
3. Has **no hidden dependencies** (no static singletons, no service locator access inside methods)

If any of these three conditions is violated, the class either cannot be tested in isolation or requires significant test scaffolding that exposes the coupling.

---

## Abstract Interface Coverage Table

| Class type | Requires abstract interface? | Reason |
|---|---|---|
| Repository | YES — always | Performs I/O; data layer must be fakeable |
| DataSource / API client | YES — always | Network call; must not reach real server in unit tests |
| Local storage (SharedPreferences, Hive) | YES | Platform channel call; requires device or complex mock |
| UseCase | NO (unless shared by multiple consumers) | Pure logic delegating to injected repository; testable via fake repository |
| Domain entity | NO | Plain Dart class; no side effects |
| Formatter / value calculator | NO | Pure function; no collaborators to replace |
| Clock / DateTime provider | YES | `DateTime.now()` is non-deterministic; inject `Clock` abstraction |
| Analytics service | YES | External I/O; must be silenced in tests |

---

## Fake vs Mock Decision Table

| Situation | Use Fake | Use Mock (mockito/mocktail) |
|---|---|---|
| Repository with simple CRUD behavior | ✅ Fake — implement in-memory list | — |
| Repository where you need to verify exact call arguments | — | ✅ Mock — `verify(_repo.save(any))` |
| Repository that should throw in specific test cases | ✅ Fake — `throw Exception()` in condition | ✅ Mock — `when(_repo.save(any)).thenThrow(...)` |
| Service locator dependency inside the class under test | ❌ Neither — refactor to inject via constructor first | — |
| `DateTime` / clock source | ✅ Fake `Clock` with fixed time | — |

---

## mockito vs mocktail Comparison (for audit calibration)

| Criterion | mockito | mocktail |
|---|---|---|
| Code generation required | YES — `@GenerateMocks` + `build_runner` | NO |
| Null-safe support | YES (v5+) | YES |
| Stub API | `when(_repo.x()).thenReturn(y)` | `when(() => _repo.x()).thenReturn(y)` |
| Verify API | `verify(_repo.x()).called(1)` | `verify(() => _repo.x()).called(1)` |
| `pubspec.yaml` dev dep | `mockito`, `build_runner` | `mocktail` |

**Audit signal:** If neither `mockito` nor `mocktail` is present in `pubspec.yaml` dev_dependencies, and no manual `Fake*` or `Mock*` classes exist in `test/`, the project has no mock infrastructure. Every Bloc that depends on a repository cannot be unit-tested without real I/O.

---

## UseCase Testability Contract

A UseCase is unit-testable when it satisfies all of:

- Constructor receives only abstract-typed collaborators
- No Flutter imports (`package:flutter/`)
- No `BuildContext` reference anywhere in the class
- No direct instantiation of repositories or services inside method bodies
- All returned types are plain Dart (no `Widget`, no `BuildContext`)

```dart
// Testable UseCase
class PlaceOrderUseCase {
  final OrderRepository _orders;
  final InventoryRepository _inventory;
  final Clock _clock;

  PlaceOrderUseCase(this._orders, this._inventory, this._clock);

  Future<OrderResult> execute(Cart cart) async {
    final available = await _inventory.checkAvailability(cart.items);
    if (!available) return OrderResult.outOfStock();
    final order = Order(cart: cart, placedAt: _clock.now());
    return _orders.save(order);
  }
}
```

**Test for this UseCase requires only:**
- `FakeOrderRepository` (in-memory list)
- `FakeInventoryRepository` (returns `true` or `false`)
- `FakeClock` (fixed `DateTime`)

No `flutter_test`, no widget pump, no platform setup.

---

## Severity Table — Unit Test Blockers

| Finding | Severity | Rationale |
|---|---|---|
| Bloc constructor takes `*Impl` not `*Repository` | HIGH | Cannot inject fake; blocTest fails |
| Repository/DataSource has no abstract interface | HIGH | No type to inject a double against |
| Service locator called in Bloc method body | HIGH | Hidden dep; test cannot replace it |
| `Dio()` constructed inside a UseCase | HIGH | Network call leaks into domain |
| No mockito/mocktail, no Fake classes | MEDIUM | Test infra missing but fixable without restructure |
| `DateTime.now()` in UseCase without clock injection | MEDIUM | Non-deterministic output, tests may pass/fail by time |
| Analytics service hardcoded (not injected) | LOW | Tests produce noise analytics; not a correctness issue |
