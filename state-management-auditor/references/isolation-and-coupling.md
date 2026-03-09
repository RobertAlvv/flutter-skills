# State Isolation and Coupling Reference — Audit Guide

Use this file to evaluate whether state classes are properly isolated from the Flutter UI layer and whether architectural coupling exists between features through shared state.

---

## The Cardinal Rule

**State classes must contain zero Flutter UI imports.**

A Bloc, Cubit, `StateNotifier`, or `ChangeNotifier` that can only be unit-tested with `WidgetTester` (because it imports `BuildContext`, `Widget`, or any class from `package:flutter/`) is architecturally coupled to the UI layer and will resist refactoring.

---

## Import Coupling — Detection and Severity

### State class importing Flutter UI — HIGH severity

```dart
// ❌ Bloc importing Flutter — cannot be tested without flutter_test
import 'package:flutter/material.dart';            // ← violation
import 'package:flutter_bloc/flutter_bloc.dart';    // ← OK (state management only)

class NavigationBloc extends Bloc<NavigationEvent, NavigationState> {
  NavigationBloc(this._context) : super(NavigationState.initial()) {
    // ...
  }
  final BuildContext _context;   // ← architectural violation
}
```

**Audit grep:**
```bash
grep -rn "import 'package:flutter/material\|import 'package:flutter/widgets\|import 'package:flutter/cupertino" lib/ --include="*_bloc.dart" --include="*_cubit.dart" --include="*_notifier.dart" --include="*_state.dart"
```

### BuildContext stored or passed to state class — HIGH severity

```dart
// ❌ BuildContext passed to Cubit constructor or event
class UserCubit extends Cubit<UserState> {
  UserCubit(this.context) : super(const UserState());
  final BuildContext context;  // ← never do this

  Future<void> logout() async {
    await _auth.signOut();
    Navigator.of(context).pushReplacementNamed('/login');  // ← navigation from state class
  }
}
```

---

## Navigation From State Classes — Correct Patterns

Navigation, dialogs, and snackbars must be triggered from the UI layer in response to state changes, **never** from within the state class.

### ❌ Wrong — navigation inside Bloc event handler

```dart
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  on<LogoutRequested>((event, emit) async {
    await _auth.signOut();
    navigatorKey.globalKey.currentState?.pushReplacementNamed('/login'); // ❌
    emit(const AuthState.unauthenticated());
  });
}
```

### ✅ Correct — BlocListener drives navigation

```dart
// State class only emits state — it knows nothing about navigation
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  on<LogoutRequested>((event, emit) async {
    await _auth.signOut();
    emit(const AuthState.unauthenticated());  // ✅ state only
  });
}

// UI reacts to the state change and navigates
BlocListener<AuthBloc, AuthState>(
  listenWhen: (previous, current) => current.status == AuthStatus.unauthenticated,
  listener: (context, state) {
    Navigator.of(context).pushReplacementNamed('/login');  // ✅ UI handles navigation
  },
  child: ...,
)
```

---

## UI Calling Business Logic — Detection

### Repository calls inside widgets — HIGH severity

```dart
// ❌ Widget performing data fetching — bypasses state management entirely
class ProductScreenState extends State<ProductScreen> {
  @override
  void initState() {
    super.initState();
    ProductRepository().fetchProducts().then((products) {   // ❌ direct repo call
      setState(() => _products = products);
    });
  }
}

// ✅ Widget delegates to Bloc
class ProductScreenState extends State<ProductScreen> {
  @override
  void initState() {
    super.initState();
    context.read<ProductBloc>().add(const ProductFetchRequested());  // ✅
  }
}
```

**Audit grep:**
```bash
grep -rn "Repository()\|Service()\|dio\.get\|http\.get\|FirebaseFirestore" lib/ --include="*.dart" | grep -v "_bloc\|_cubit\|_repository\|_service\|_datasource\|\.g\.dart"
```

### Business logic decisions inside `build()` — MEDIUM severity

```dart
// ❌ Conditional business logic inside build — runs on every rebuild
Widget build(BuildContext context) {
  final state = context.watch<CartBloc>().state;
  if (state.items.length > 10) {
    analyticsService.logEvent('large_cart');  // ❌ side effect in build()
  }
  return CartSummary(items: state.items);
}

// ✅ Side effects belong in BlocListener
BlocListener<CartBloc, CartState>(
  listenWhen: (prev, cur) => prev.items.length <= 10 && cur.items.length > 10,
  listener: (context, state) {
    analyticsService.logEvent('large_cart');  // ✅ runs once, not on every rebuild
  },
  child: BlocBuilder<CartBloc, CartState>(
    builder: (context, state) => CartSummary(items: state.items),
  ),
)
```

---

## Feature-to-Feature Coupling Through Shared State

### Direct Bloc access across feature boundaries — MEDIUM severity

```dart
// ❌ Feature B reads Feature A's Bloc directly — tight coupling
class CheckoutScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final cartItems = context.read<CartBloc>().state.items;  // ❌ checkout depends on cart Bloc
    final userAddress = context.read<UserProfileBloc>().state.address; // ❌ cross-feature coupling
    // ...
  }
}
```

This creates a hidden dependency: `CheckoutScreen` can only function when both `CartBloc` and `UserProfileBloc` are in its ancestor tree. Adding a test or a new flow that uses `CheckoutScreen` must also provide both blocs.

### Correct approach — pass data as constructor parameters

```dart
// ✅ CheckoutScreen receives its data — no cross-feature state coupling
class CheckoutScreen extends StatelessWidget {
  const CheckoutScreen({
    required this.cartItems,
    required this.deliveryAddress,
    super.key,
  });

  final List<CartItem> cartItems;
  final Address deliveryAddress;
}
```

Or use a dedicated `CheckoutBloc` that receives the necessary data via its event/constructor, decoupled from the source blocs.

---

## Domain Layer Isolation

When a proper domain layer exists (`usecase/`, `domain/`), it must not be bypassed:

```dart
// ❌ Screen calling repository directly — bypasses domain layer and its validation/rules
class OrderScreenState extends State<OrderScreen> {
  void _placeOrder() {
    context.read<OrderBloc>().add(
      OrderPlaced(items: _items, total: _total),  // ✅ via Bloc
    );
    orderRepository.create(_items);   // ❌ also calling repository directly from screen
  }
}

// ❌ Bloc calling repository directly when a use case exists
class OrderBloc extends Bloc<OrderEvent, OrderState> {
  on<OrderPlaced>((event, emit) async {
    await _orderRepository.create(event.items);  // ❌ bypasses PlaceOrderUseCase
  });
}

// ✅ Bloc uses use case — repository only called through domain layer
class OrderBloc extends Bloc<OrderEvent, OrderState> {
  on<OrderPlaced>((event, emit) async {
    await _placeOrderUseCase(PlaceOrderParams(items: event.items));  // ✅
  });
}
```

---

## Severity Reference

| Pattern | Severity |
|---|---|
| State class imports `package:flutter/material.dart` | HIGH |
| `BuildContext` stored in Bloc/Cubit/Notifier | HIGH |
| Navigation (`Navigator.push`) called from within state class | HIGH |
| Repository/service call inside widget `build()` or `initState()` | HIGH |
| Side effects (analytics, logging) inside widget `build()` | MEDIUM |
| Bloc reading another Bloc directly without `BlocListener` or event | MEDIUM |
| Screen accessing two unrelated feature Blocs directly | MEDIUM |
| Bloc bypassing domain use case to call repository directly (when use case exists) | MEDIUM |
| Minor business condition evaluated in widget (e.g. `if (items.isEmpty)`) | LOW |
