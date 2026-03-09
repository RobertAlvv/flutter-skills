# Rebuild Efficiency Reference — Audit Guide

Use this file to evaluate whether state changes trigger precisely scoped widget rebuilds or unnecessarily large subtree rebuilds. This is the most impactful performance dimension of state management architecture.

---

## Frame Budget Reminder

Flutter targets 16ms/frame at 60fps and 8ms/frame at 120fps. The build phase competes with layout and paint for this budget. A full-screen `Scaffold` rebuild triggered by a minor state change (e.g., a badge counter update) can consume the entire build budget.

---

## BlocBuilder — Rebuild Scope Audit

### Without `buildWhen` — rebuilds on every emission

```dart
// ❌ Rebuilds the entire Scaffold on EVERY ProductState emission
// including intermediate loading states and error states unrelated to the product list
BlocBuilder<ProductBloc, ProductState>(
  builder: (context, state) => Scaffold(
    appBar: ...,
    body: ProductGrid(products: state.products),
  ),
)
```

### With `buildWhen` — targeted rebuild

```dart
// ✅ Only rebuilds when the products list actually changes
BlocBuilder<ProductBloc, ProductState>(
  buildWhen: (previous, current) => previous.products != current.products,
  builder: (context, state) => Scaffold(
    appBar: ...,
    body: ProductGrid(products: state.products),
  ),
)
```

**Audit signal:** `grep -rn "BlocBuilder" lib/ --include="*.dart" | grep -v "buildWhen"` — every match is a potential unnecessary rebuild. Flag HIGH when the subtree is a full screen.

---

## BlocSelector — Sub-Field Rebuild Isolation

When a widget only consumes ONE field of a state with multiple fields, `BlocSelector` prevents rebuilds when other fields change:

```dart
// ❌ Rebuilds whenever ANY ProductState field changes
BlocBuilder<ProductBloc, ProductState>(
  builder: (context, state) => CartBadge(count: state.cartCount),
)

// ✅ Rebuilds ONLY when cartCount changes
BlocSelector<ProductBloc, ProductState, int>(
  selector: (state) => state.cartCount,
  builder: (context, count) => CartBadge(count: count),
)
```

**Audit signal:** `BlocBuilder` that accesses only `state.fieldX` on a state with 5+ fields should be `BlocSelector`.

---

## BlocConsumer — Dual Use Without Scope Control

`BlocConsumer` combines `BlocBuilder` and `BlocListener`. Both fire on every state emission unless constrained:

```dart
// ❌ Both builder and listener fire on every emission
BlocConsumer<CheckoutBloc, CheckoutState>(
  listener: (context, state) {
    if (state.status == CheckoutStatus.success) Navigator.pushNamed(context, '/confirm');
  },
  builder: (context, state) => CheckoutForm(state: state),
)

// ✅ listener fires only on status transitions; builder fires only on form data changes
BlocConsumer<CheckoutBloc, CheckoutState>(
  listenWhen: (previous, current) => previous.status != current.status,
  buildWhen: (previous, current) => previous.formData != current.formData,
  listener: (context, state) {
    if (state.status == CheckoutStatus.success) Navigator.pushNamed(context, '/confirm');
  },
  builder: (context, state) => CheckoutForm(data: state.formData),
)
```

---

## context.watch vs context.select (Provider / Riverpod)

```dart
// ❌ Rebuilds when ANY field of CartState changes (quantity, items, total, discount, ...)
Widget build(BuildContext context) {
  final cart = context.watch<CartNotifier>().state;
  return Badge(count: cart.itemCount);
}

// ✅ Rebuilds ONLY when itemCount changes
Widget build(BuildContext context) {
  final itemCount = context.select<CartNotifier, int>(
    (notifier) => notifier.state.itemCount,
  );
  return Badge(count: itemCount);
}
```

For Riverpod:

```dart
// ❌ Rebuilds on any UserState change
final user = ref.watch(userProvider);
Text(user.displayName);

// ✅ Rebuilds only when displayName changes
final displayName = ref.watch(userProvider.select((u) => u.displayName));
Text(displayName);
```

---

## setState — Scope Violations

The rebuild scope of `setState` is the entire `State` object and all its descendants.

```dart
// ❌ Full-screen rebuild on every counter increment
class ProductScreenState extends State<ProductScreen> {
  int _favoriteCount = 0;
  List<Product> _products = [];  // this data does NOT change on favorite

  void _toggleFavorite(Product p) {
    setState(() => _favoriteCount++);  // ← rebuilds entire screen including ProductGrid
  }
}

// ✅ Extract the mutable part to a leaf StatefulWidget or ValueNotifier
final _favoriteCount = ValueNotifier<int>(0);

ValueListenableBuilder<int>(
  valueListenable: _favoriteCount,
  builder: (context, count, _) => FavoriteButton(count: count),
)
```

**Audit signal:** `setState` inside a `StatefulWidget` that has a `Scaffold` in its `build()` method — the entire screen rebuilds on every call.

---

## Nested State Listeners — Double Rebuild

```dart
// ❌ CartBloc outer rebuild triggers ProductBloc inner rebuild on every emission
BlocBuilder<CartBloc, CartState>(
  builder: (context, cartState) => BlocBuilder<ProductBloc, ProductState>(
    builder: (context, productState) => ProductCard(
      product: productState.product,
      inCart: cartState.contains(productState.product.id),
    ),
  ),
)
```

This pattern rebuilds `ProductCard` on every `CartBloc` AND every `ProductBloc` emission, even when neither change affects the card's output.

**Fix:** Use `BlocSelector` on both, or extract into a widget that takes both values as constructor arguments and controls its own rebuild condition.

---

## StreamBuilder — Unnecessary Extra Frame

```dart
// ❌ Two render frames: first ConnectionState.waiting, then data
StreamBuilder<List<Event>>(
  stream: eventStream,
  builder: (context, snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return const CircularProgressIndicator();
    }
    return EventList(events: snapshot.data ?? []);
  },
)

// ✅ First frame renders data immediately — no waiting frame
StreamBuilder<List<Event>>(
  stream: eventStream,
  initialData: const [],
  builder: (context, snapshot) => EventList(events: snapshot.data!),
)
```

---

## Severity Reference

| Pattern | Subtree Size | Severity |
|---|---|---|
| `BlocBuilder` without `buildWhen` | Full screen (Scaffold) | HIGH |
| `BlocBuilder` without `buildWhen` | Small leaf widget | LOW |
| `context.watch<X>()` consuming one field of large state | Large widget | MEDIUM |
| `setState` in Scaffold-owning `StatefulWidget` for minor update | Any | HIGH |
| `BlocBuilder` where `BlocSelector` would suffice | Any | MEDIUM |
| Nested `BlocBuilder` without scope control | Any | MEDIUM |
| `StreamBuilder` without `initialData` | Any | LOW |
| `BlocConsumer` without `listenWhen`/`buildWhen` | Full screen | HIGH |
