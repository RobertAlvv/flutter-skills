# Scroll Performance Reference — Audit Guide

Use this file to evaluate scrolling list implementations in the audited project. Scroll performance is the most user-visible performance dimension — even a single dropped frame during a scroll gesture is noticeable.

---

## The Core Rule: Lazy Loading

Flutter's scroll performance fundamentally depends on **lazy widget construction** — only the widgets currently visible on screen (plus a small buffer) should be built and laid out. Any pattern that builds all items upfront defeats this and causes jank at scale.

---

## ListView Variants — When to Use Each

| Widget | Builds items lazily? | Use when |
|---|---|---|
| `ListView(children: [...])` | ❌ All at once | ≤5 static items that never change |
| `ListView.builder(...)` | ✅ On demand | Any dynamic or large list |
| `ListView.separated(...)` | ✅ On demand | Dynamic list with separator between items |
| `ListView.custom(...)` | ✅ On demand | Custom sliver child delegate needed |

**Audit rule:** Any `ListView(children: [...])` with more than 5 items or dynamic content is a performance violation.

---

## Anti-Pattern 1 — Column + map inside SingleChildScrollView

This is the most common and most harmful scroll anti-pattern. It looks innocent but builds **all items at once**, with no lazy loading whatsoever:

```dart
// ❌ Builds ALL items — identical in behavior to ListView(children:[...])
//    but even harder to spot in code review
SingleChildScrollView(
  child: Column(
    children: products.map((p) => ProductCard(product: p)).toList(),
  ),
)

// ✅ Builds only visible items
ListView.builder(
  itemCount: products.length,
  itemBuilder: (context, index) => ProductCard(product: products[index]),
)
```

**Audit signal:** `grep -rn "SingleChildScrollView" lib/ --include="*.dart"` — open each match and check if the child contains a `Column` with a mapped list.

---

## Anti-Pattern 2 — shrinkWrap: true inside a Scroll View

`shrinkWrap: true` forces the inner `ListView` to compute the height of **all its items** before rendering — this completely defeats lazy loading:

```dart
// ❌ shrinkWrap forces layout of all items upfront
ListView.builder(
  shrinkWrap: true,                              // ← defeats lazy loading
  physics: const NeverScrollableScrollPhysics(),
  itemCount: products.length,
  itemBuilder: (context, index) => ProductCard(product: products[index]),
)

// ✅ Use SliverList inside CustomScrollView for nested scenarios
CustomScrollView(
  slivers: [
    SliverToBoxAdapter(child: const HeaderWidget()),
    SliverList.builder(
      itemCount: products.length,
      itemBuilder: (context, index) => ProductCard(product: products[index]),
    ),
  ],
)
```

**Audit rule:** `shrinkWrap: true` combined with `ListView.builder` is always a violation. Flag it as HIGH severity when the list count is unbounded.

---

## Anti-Pattern 3 — Expensive List Item Widgets

Each `ListView.builder` item is built, laid out, and painted as the user scrolls. Heavy items directly reduce scroll frame rate.

### ClipRRect on list items — MEDIUM severity

```dart
// ❌ Compositing layer promoted for every visible item
ListTile(
  leading: ClipRRect(
    borderRadius: BorderRadius.circular(20),
    child: Image.network(avatarUrl),
  ),
)

// ✅ No compositing layer required
ListTile(
  leading: Container(
    decoration: BoxDecoration(
      shape: BoxShape.circle,
      image: DecorationImage(
        image: NetworkImage(avatarUrl),
        fit: BoxFit.cover,
      ),
    ),
    width: 40,
    height: 40,
  ),
)
```

### Image.network without cacheWidth/cacheHeight — MEDIUM severity

Flutter decodes images at their **native resolution** then scales them to display size. A 2000×2000px avatar displayed at 40×40dp decodes 4 million pixels unnecessarily:

```dart
// ❌ Decodes at full native resolution — wastes memory and decode time
Image.network(
  avatarUrl,
  width: 40,
  height: 40,
)

// ✅ Decodes at display resolution (accounting for device pixel ratio)
Image.network(
  avatarUrl,
  width: 40,
  height: 40,
  cacheWidth: 80,   // 40dp × 2x device pixel ratio
  cacheHeight: 80,
)
```

### BoxDecoration with boxShadow + borderRadius on items — HIGH severity

Triggers `saveLayer()` for every visible item on every scroll frame. See `savelayer-and-compositing.md` for details.

---

## itemExtent and prototypeItem — Scroll Position Optimization

When all items in a list have the same height, Flutter can calculate scroll positions and item indices **mathematically** without measuring each item:

```dart
// Without itemExtent: Flutter measures each item's height
// to determine which items are visible at a given scroll offset
ListView.builder(
  itemCount: products.length,
  itemBuilder: (context, index) => const ProductCard(),
)

// ✅ With itemExtent: O(1) scroll position calculation
ListView.builder(
  itemCount: products.length,
  itemExtent: 72.0,  // known fixed height in logical pixels
  itemBuilder: (context, index) => const ProductCard(),
)

// ✅ prototypeItem: Flutter measures ONE item and uses that height for all
ListView.builder(
  itemCount: products.length,
  prototypeItem: const ProductCard(),
  itemBuilder: (context, index) => ProductCard(product: products[index]),
)
```

**Audit signal:** `ListView.builder` with visually uniform items but no `itemExtent` or `prototypeItem` — flag as LOW/MEDIUM depending on list size.

---

## Nested Scrollables — Constraint Issues

Nested `ListView` inside another `ListView` causes constraint errors or requires workarounds that degrade performance:

```dart
// ❌ Unbounded height constraint — throws RenderBox error or requires shrinkWrap
ListView.builder(
  itemBuilder: (context, index) => Column(
    children: [
      SectionHeader(),
      ListView.builder(        // ← inner list has no bounded height
        itemBuilder: ...,
      ),
    ],
  ),
)

// ✅ Use CustomScrollView with multiple SliverList/SliverGrid
CustomScrollView(
  slivers: [
    SliverToBoxAdapter(child: SectionHeader()),
    SliverList.builder(itemBuilder: ..., itemCount: ...),
    SliverToBoxAdapter(child: AnotherHeader()),
    SliverGrid.builder(itemBuilder: ..., itemCount: ...),
  ],
)
```

---

## Severity Summary

| Pattern | List Size | Severity |
|---|---|---|
| `ListView(children: [...])` with dynamic content | Any | HIGH |
| `Column + map` inside `SingleChildScrollView` | Any dynamic | HIGH |
| `shrinkWrap: true` on `ListView.builder` | Unbounded | HIGH |
| `ClipRRect` inside list items | Visible on scroll | MEDIUM |
| `Image.network` without `cacheWidth`/`cacheHeight` in list | Any | MEDIUM |
| `BoxDecoration` with shadow + borderRadius in list items | Any | HIGH |
| Missing `itemExtent`/`prototypeItem` on uniform-height list | Large (50+) | LOW |
| Nested scrollables without `CustomScrollView` | Any | MEDIUM |
