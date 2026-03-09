# Rebuild Patterns Reference — Audit Guide

Use this file to evaluate whether the audited project minimizes unnecessary widget rebuilds. Unnecessary rebuilds are the most common Flutter performance issue and the easiest to fix once identified.

---

## Frame Budget Context

| Target FPS | Budget per frame |
|---|---|
| 60 fps | 16 ms |
| 90 fps | 11 ms |
| 120 fps | 8 ms |

Flutter splits this budget across: build → layout → paint → composite. Unnecessary rebuilds consume build time before layout and paint even begin. A single full-screen rebuild on a complex tree can consume the entire 16ms budget by itself.

---

## BlocBuilder — Correct vs Incorrect

### Without `buildWhen` — rebuilds on EVERY state emission

```dart
// ❌ Rebuilds on every ProductState emission, including
//    intermediate loading states that don't affect this widget
BlocBuilder<ProductBloc, ProductState>(
  builder: (context, state) => ProductCard(name: state.name),
)
```

### With `buildWhen` — targeted rebuild

```dart
// ✅ Only rebuilds when the name field actually changes
BlocBuilder<ProductBloc, ProductState>(
  buildWhen: (previous, current) => previous.name != current.name,
  builder: (context, state) => ProductCard(name: state.name),
)
```

---

## BlocSelector — Surgical Sub-Field Rebuild

When only one field of the state is consumed, `BlocSelector` avoids rebuilding when other fields change:

```dart
// ❌ Full rebuild whenever any ProductState field changes
BlocBuilder<ProductBloc, ProductState>(
  builder: (context, state) => Badge(count: state.cartCount),
)

// ✅ Rebuilds ONLY when cartCount changes
BlocSelector<ProductBloc, ProductState, int>(
  selector: (state) => state.maybeWhen(
    loaded: (products) => products.where((p) => p.inCart).length,
    orElse: () => 0,
  ),
  builder: (context, count) => Badge(count: count),
)
```

**Audit signal:** If `BlocBuilder` extracts only one field from state and the state has 3+ fields, it should be `BlocSelector`.

---

## Provider / Riverpod — Granular Selects

```dart
// ❌ Rebuilds when ANY field of CartState changes
final cartState = context.watch<CartNotifier>().state;
Text('${cartState.itemCount} items')

// ✅ Rebuilds ONLY when itemCount changes
final itemCount = context.select<CartNotifier, int>(
  (notifier) => notifier.state.itemCount,
);
Text('$itemCount items')
```

---

## setState — Scope Violations

### Full-screen rebuild (violation)

```dart
class ProductScreenState extends State<ProductScreen> {
  bool _isLoading = false;
  List<Product> _products = [];

  Future<void> _load() async {
    setState(() => _isLoading = true);       // ← rebuilds entire Scaffold
    _products = await repo.fetchProducts();
    setState(() => _isLoading = false);      // ← rebuilds entire Scaffold again
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: _isLoading
          ? const CircularProgressIndicator()
          : ProductList(products: _products),
    );
  }
}
```

### Scoped rebuild (correct)

```dart
// ✅ Extract the changing part into a small StatefulWidget
//    or use ValueNotifier + ValueListenableBuilder
final _isLoading = ValueNotifier<bool>(false);

ValueListenableBuilder<bool>(
  valueListenable: _isLoading,
  builder: (context, loading, child) =>
      loading ? const CircularProgressIndicator() : child!,
  child: ProductList(products: _products),  // child is not rebuilt
)
```

---

## AnimationController — Rebuild Scope

### Driving setState (violation)

```dart
// ❌ The entire StatefulWidget rebuilds 60x/second
class _MyWidgetState extends State<MyWidget>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this, duration: const Duration(seconds: 1))
      ..addListener(() => setState(() {}));  // ← full rebuild per frame
  }
}
```

### AnimatedBuilder with isolated leaf (correct)

```dart
// ✅ Only the FadeTransition widget updates — zero setState
FadeTransition(
  opacity: _controller,   // Directly uses animation, no setState needed
  child: const MyContent(),
)

// Or isolate the animated portion:
AnimatedBuilder(
  animation: _controller,
  child: const ExpensiveStaticWidget(),  // ← built once, reused
  builder: (context, child) => Transform.scale(
    scale: _controller.value,
    child: child,          // ← expensive child is NOT rebuilt
  ),
)
```

**Key:** Pass static subtrees as the `child` parameter to `AnimatedBuilder`. The `child` is built once and passed back to `builder` on each frame without rebuilding.

---

## StreamBuilder — Initialization Issue

```dart
// ❌ Causes two builds: ConnectionState.waiting (frame 1)
//    then ConnectionState.active with data (frame 2)
StreamBuilder<List<Product>>(
  stream: productStream,
  builder: (context, snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return const CircularProgressIndicator();
    }
    return ProductList(products: snapshot.data ?? []);
  },
)

// ✅ Skips the waiting frame — renders data immediately
StreamBuilder<List<Product>>(
  stream: productStream,
  initialData: const [],  // ← no waiting state rendered
  builder: (context, snapshot) =>
      ProductList(products: snapshot.data!),
)
```

---

## Rebuild Isolation — RepaintBoundary

When a frequently-rebuilding widget lives alongside static content, wrap it in `RepaintBoundary` to isolate its repaint from the rest of the tree:

```dart
// ✅ The scrolling list does not trigger repaint of the static header
Column(
  children: [
    const AppHeader(),           // static — repainted only when changed
    RepaintBoundary(
      child: LiveFeedList(),     // frequently updates — isolated repaint
    ),
  ],
)
```

**When NOT to add `RepaintBoundary`:**
- On every widget (each boundary allocates a compositing layer)
- On static widgets that rarely rebuild (adds overhead with no benefit)
- Inside `ListView.builder` items unless each item has heavy internal animation

---

## Severity Reference

| Pattern | Severity | Impact |
|---|---|---|
| `BlocBuilder` without `buildWhen` on full-screen widget | HIGH | Full-screen rebuild on every state emission |
| `context.watch<X>()` consuming one field of large state | MEDIUM | Rebuilds on all state fields, not just the consumed one |
| `setState` wrapping async logic on a Scaffold-owning widget | HIGH | Full-screen rebuild per async step |
| `AnimationController` → `setState` at parent level | HIGH | Full-widget rebuild at 60–120fps |
| `StreamBuilder` without `initialData` | LOW | One extra build frame on initialization |
| `BlocBuilder` where `BlocSelector` would suffice | MEDIUM | Rebuilds on irrelevant state fields |
| No `RepaintBoundary` around animated widget in static tree | MEDIUM | Animated widget triggers repaint of static siblings |
