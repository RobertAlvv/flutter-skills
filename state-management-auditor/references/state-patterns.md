# State Management Patterns Reference — Audit Guide

Use this file to evaluate which state management patterns are present, whether they are applied correctly, and what the severity of observed deviations is.

---

## Pattern Decision Guide

### BLoC vs Cubit — When Each Is Appropriate

| Scenario | Recommended | Reason |
|---|---|---|
| Simple toggle, counter, filter | `Cubit` | No need for event objects when state transitions are trivial |
| Multiple event sources triggering the same state change | `Bloc` | Named events make the source of each transition traceable |
| Feature with 5+ distinct user interactions | `Bloc` | Event classes document the API surface |
| Shared state reused across multiple screens | `Cubit` or `Bloc` | Both work; consistency with the rest of the codebase decides |
| Authentication / session management | `Bloc` | Complex event flow (login, logout, token refresh, error) warrants full Bloc |

**Audit signal:** A `Cubit` with more than 8 methods is a signal it has grown into `Bloc` territory without being converted (technical debt).

---

## flutter_bloc — Correct Structure

### Bloc class structure

```dart
// ✅ Events are sealed classes — exhaustive pattern matching enforced
sealed class ProductEvent {}
final class ProductFetchRequested extends ProductEvent {
  const ProductFetchRequested({required this.categoryId});
  final String categoryId;
}
final class ProductRefreshRequested extends ProductEvent {}

// ✅ States are immutable with copyWith
final class ProductState extends Equatable {
  const ProductState({
    this.status = ProductStatus.initial,
    this.products = const [],
    this.errorMessage,
  });

  final ProductStatus status;
  final List<Product> products;
  final String? errorMessage;

  ProductState copyWith({
    ProductStatus? status,
    List<Product>? products,
    String? errorMessage,
  }) => ProductState(
    status: status ?? this.status,
    products: products ?? this.products,
    errorMessage: errorMessage ?? this.errorMessage,
  );

  @override
  List<Object?> get props => [status, products, errorMessage];
}
```

### Common state structure violations

```dart
// ❌ Mutable state — Equatable won't detect changes reliably
class ProductState {
  List<Product> products = [];   // mutable field
  bool isLoading = false;        // mutable field
}

// ❌ Generic event names — no documentation value
class UpdateState extends ProductEvent {}
class ChangeValue extends ProductEvent { final dynamic value; }

// ❌ State with 15+ fields — signals it should be split into sub-states
class AppState {
  final User? user;
  final Theme theme;
  final List<Product> products;
  final CartState cart;
  final List<Order> orders;
  // ... 10 more fields
}
```

---

## Riverpod — Correct Patterns

### Provider types and their use cases

| Provider type | Use for | Autodispose? |
|---|---|---|
| `Provider` | Immutable value, service, repository | Usually not |
| `StateProvider` | Simple primitive state (bool, int, enum) | Optional |
| `StateNotifierProvider` | Complex state with methods | Yes for feature-scoped |
| `FutureProvider` | Single async fetch | Yes |
| `StreamProvider` | Live stream (Firestore, WebSocket) | Yes |
| `NotifierProvider` | Riverpod 2.x replacement for StateNotifier | Yes for feature-scoped |

### Rebuild scope control

```dart
// ❌ Rebuilds on any field change in CartState
final cartState = ref.watch(cartProvider);
Text('${cartState.itemCount} items');

// ✅ Rebuilds only when itemCount changes
final itemCount = ref.watch(cartProvider.select((s) => s.itemCount));
Text('$itemCount items');
```

### Common Riverpod violations

```dart
// ❌ ref.read in build() — misses updates
Widget build(BuildContext context, WidgetRef ref) {
  final user = ref.read(userProvider);   // will not rebuild when user changes
  return Text(user.name);
}

// ❌ Provider declared inside a widget build method
Widget build(BuildContext context, WidgetRef ref) {
  final localProvider = Provider((ref) => MyService());  // recreated on every build
}
```

---

## Provider (flutter_provider) — Common Violations

```dart
// ❌ ChangeNotifier doing business logic inline
class CartNotifier extends ChangeNotifier {
  Future<void> addToCart(Product p) async {
    final response = await http.post(Uri.parse('/cart'), body: ...); // ❌ direct HTTP
    notifyListeners();
  }
}

// ✅ ChangeNotifier delegates to repository
class CartNotifier extends ChangeNotifier {
  CartNotifier(this._repository);
  final CartRepository _repository;

  Future<void> addToCart(Product p) async {
    await _repository.addToCart(p);
    notifyListeners();
  }
}
```

---

## setState — Acceptable vs Violation

| Usage | Acceptable? | Why |
|---|---|---|
| Toggle expansion of an `ExpansionTile` | ✅ Yes | Purely local, no business logic, no external data |
| Show/hide password in a `TextField` | ✅ Yes | Local ephemeral UI state |
| Triggering a `PageController.animateToPage()` | ✅ Yes | UI-only operation |
| Fetching products from an API on screen open | ❌ No | Business logic belongs in Bloc/Cubit |
| Updating shared state that multiple widgets depend on | ❌ No | Use a state management solution |
| Wrapping an `AnimationController.forward()` call | ❌ Not needed | `setState` not required — animation drives itself |

---

## Severity Reference

| Pattern | Severity |
|---|---|
| Bloc/Cubit importing `package:flutter/material.dart` | HIGH |
| `BuildContext` stored as field in a state class | HIGH |
| Business logic (API, repository call) inside widget build or initState | HIGH |
| `setState` on full-screen Scaffold widget with complex state | HIGH |
| Mixed state management solutions across features with no migration plan | MEDIUM |
| `ChangeNotifier` (Provider) used alongside `flutter_bloc` in same feature | MEDIUM |
| Cubit with 10+ methods (should be Bloc) | MEDIUM |
| State class with 15+ fields (should be decomposed) | MEDIUM |
| Generic event names (`UpdateState`, `ChangeValue`) | LOW |
| `ref.read` used in `build()` instead of `ref.watch` | MEDIUM |
