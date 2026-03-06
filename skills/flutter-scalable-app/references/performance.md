# Performance Optimization

## The Flutter Performance Mindset

Every frame must complete in **16ms** (60fps) or **8ms** (120fps). The render pipeline: build → layout → paint. Your job is to minimize work in each phase and avoid triggering unnecessary work.

---

## Rebuild Audits — The Checklist

Run this audit on every screen before shipping:

```
□ Every static widget uses const constructor
□ No expensive widgets built inside build() — extract to StatelessWidget
□ BlocSelector used when only a sub-field of state matters
□ Large lists use ListView.builder (lazy) never Column + children
□ Images use CachedNetworkImage with memCacheWidth
□ No synchronous operations blocking the UI thread
□ RepaintBoundary isolates animated areas
□ Keys assigned on dynamic list items
□ No unnecessary ValueKey/UniqueKey on static widgets
```

---

## const — The Cheapest Optimization

```dart
// ❌ Rebuilds every time parent rebuilds
class ProductCard extends StatelessWidget {
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.all(16),          // new object every build
      child: Icon(Icons.chevron_right),     // new object every build
    );
  }
}

// ✅ Zero rebuild cost for static parts
class ProductCard extends StatelessWidget {
  const ProductCard({super.key}); // ← const constructor

  @override
  Widget build(BuildContext context) {
    return const Padding(
      padding: EdgeInsets.all(16),          // reused
      child: Icon(Icons.chevron_right),     // reused
    );
  }
}
```

**Rule:** The Dart analyzer will warn on missing `const` — treat all warnings as errors:
```yaml
# analysis_options.yaml
analyzer:
  errors:
    prefer_const_constructors: warning
    prefer_const_literals_to_create_immutables: warning
```

---

## BlocSelector — Surgical Rebuilds

```dart
// ❌ Entire widget tree rebuilds on every CartState change
BlocBuilder<CartBloc, CartState>(
  builder: (context, state) {
    return Text('${state.items.length} items');
  },
)

// ✅ Rebuilds ONLY when item count changes
BlocSelector<CartBloc, CartState, int>(
  selector: (state) => state.maybeWhen(
    loaded: (cart) => cart.items.length,
    orElse: () => 0,
  ),
  builder: (context, count) => Text('$count items'),
)
```

---

## ListView — Always Use Builder

```dart
// ❌ Builds ALL items at once, even off-screen
ListView(
  children: products.map((p) => ProductCard(product: p)).toList(),
)

// ✅ Builds only visible items + a small buffer
ListView.builder(
  itemCount: products.length,
  itemExtent: 88,             // fixed height = faster layout
  itemBuilder: (context, index) => ProductCard(
    key: ValueKey(products[index].id), // stable key for diff algorithm
    product: products[index],
  ),
)

// For heterogeneous lists with sections
CustomScrollView(
  slivers: [
    SliverAppBar(/* ... */),
    SliverList.builder(
      itemCount: items.length,
      itemBuilder: (ctx, i) => ItemTile(item: items[i]),
    ),
  ],
)
```

---

## Image Loading — Always CachedNetworkImage

```dart
// ❌ Downloads every time, decodes at full resolution, no placeholder
Image.network(url, width: 80, height: 80),

// ✅ Cached disk+memory, decodes at display size, skeleton placeholder
CachedNetworkImage(
  imageUrl: url,
  width: 80,
  height: 80,
  fit: BoxFit.cover,
  memCacheWidth: 160,    // 2x logical pixels for retina
  memCacheHeight: 160,
  placeholder: (_, __) => const _ImageSkeleton(),
  errorWidget: (_, __, ___) => const Icon(Icons.broken_image_outlined),
)

// For hero images (full-width banners):
CachedNetworkImage(
  imageUrl: url,
  memCacheWidth: MediaQuery.sizeOf(context).width.toInt() * 2,
  fit: BoxFit.cover,
)
```

---

## RepaintBoundary — Isolate Animations

```dart
// Wrap animated widgets to isolate their repaint zone
RepaintBoundary(
  child: AnimatedContainer(
    duration: const Duration(milliseconds: 300),
    color: isSelected ? AppColors.primary : AppColors.surface,
    child: child,
  ),
)

// Also useful for complex list items that don't animate
class ComplexProductCard extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return RepaintBoundary(
      child: _ComplexCardContent(/* ... */),
    );
  }
}
```

---

## Heavy Computation — Use compute()

```dart
// ❌ Blocks UI thread — causes jank during large JSON parsing
final products = productJsonList
    .map((j) => ProductModel.fromJson(j))
    .toList();

// ✅ Offload to isolate — UI stays smooth
final products = await compute(
  _parseProducts,
  productJsonList,
);

// Top-level function (required for isolate)
List<ProductModel> _parseProducts(List<Map<String, dynamic>> jsonList) {
  return jsonList.map((j) => ProductModel.fromJson(j)).toList();
}
```

---

## DevTools — Profiling Workflow

```bash
# Run in profile mode (not debug — debug has overhead)
flutter run --profile
```

1. Open DevTools: `flutter pub global activate devtools && flutter pub global run devtools`
2. **Performance tab** → Record 2-3 seconds of scrolling
3. Look for frames over 16ms (shown in red/yellow)
4. **Widget rebuild tracker** → Enable "Track widget rebuilds"
5. Identify widgets with high rebuild counts
6. **Memory tab** → Monitor heap during navigation events

---

## Build Mode Awareness

| Mode | Use for | Performance |
|---|---|---|
| `flutter run` (debug) | Development | Slow — assertions, hot reload |
| `flutter run --profile` | Profiling | Near-production speed |
| `flutter build apk --release` | Shipping | Full AOT compilation |

**Never profile in debug mode** — results are not representative.

---

## Release Build Optimizations

```bash
# Android — split per ABI (smaller downloads)
flutter build apk --split-per-abi --release

# iOS — strip debug symbols
flutter build ipa --release

# Obfuscation (add to android/app/build.gradle)
buildTypes {
  release {
    minifyEnabled true
    shrinkResources true
  }
}
```

---

## Keys — When and How

```dart
// ValueKey — stable identity for list items (required for diff algorithm)
ListView.builder(
  itemBuilder: (ctx, i) => ProductCard(
    key: ValueKey(products[i].id), // ← string/int ID
    product: products[i],
  ),
)

// GlobalKey — when you need to access widget state externally
final _formKey = GlobalKey<FormState>();
Form(
  key: _formKey,
  child: /* ... */,
)
// Later:
if (_formKey.currentState!.validate()) { /* submit */ }

// Avoid UniqueKey() on list items — it creates a new key every build,
// forcing Flutter to destroy and recreate widgets on every rebuild
```

---

## Navigator Stack Management

```dart
// ❌ Deep navigator stacks waste memory and slow back navigation
context.router.push(RouteA());
context.router.push(RouteB());
context.router.push(RouteC()); // stack: splash → login → home → A → B → C

// ✅ After auth, replace the entire stack
context.router.replaceAll([const ShellRoute()]); // stack: shell only

// ✅ After completing a flow, pop to root
context.router.popUntilRoot();
```

---

## Efficient State Initialization

```dart
// ❌ Heavy work in initState blocks first frame
@override
void initState() {
  super.initState();
  // Fetching data here causes a visible empty frame
  _fetchData();
}

// ✅ Dispatch event in BlocProvider.create — happens before build
BlocProvider(
  create: (_) => getIt<ProductBloc>()..add(const ProductStarted()),
  child: const _ProductView(),
)
// BLoC handles loading state, UI shows skeleton immediately
```