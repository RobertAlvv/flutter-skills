# Layout Cost Reference — Audit Guide

Use this file to identify high-cost layout widgets and patterns in the audited project. Layout issues are often invisible in DevTools frame charts because they accumulate gradually — a single `IntrinsicHeight` inside a 50-item list adds 50× the layout cost compared to one outside a list.

---

## Flutter Layout Protocol — Two Key Rules

1. **Constraints flow down** — parents tell children their maximum/minimum size
2. **Sizes flow up** — children report their actual size back to parents

Most layout widgets resolve in **O(1)** — a single pass through the tree. Violations of this protocol create multi-pass layouts that scale badly with tree size.

---

## Multi-Pass Layout Widgets — The Expensive Ones

### IntrinsicHeight and IntrinsicWidth — HIGH severity in scroll contexts

`IntrinsicHeight` / `IntrinsicWidth` perform a **second complete layout pass** over their entire subtree to measure intrinsic dimensions before the real layout pass. This is O(n) where n is the subtree depth — and compounds in scroll views:

```dart
// ❌ Inside ListView.builder: EVERY visible item does 2 layout passes per frame
ListView.builder(
  itemBuilder: (context, index) => IntrinsicHeight(  // ← 2x layout cost per item
    child: Row(
      crossAxisAlignment: CrossAxisAlignment.stretch,
      children: [
        LeftColumn(),
        const VerticalDivider(),
        RightColumn(),
      ],
    ),
  ),
)

// ✅ Use fixed height or ConstrainedBox instead
ListView.builder(
  itemBuilder: (context, index) => SizedBox(
    height: 80,
    child: Row(
      children: [
        LeftColumn(),
        const VerticalDivider(),
        RightColumn(),
      ],
    ),
  ),
)
```

**Flutter documentation states:** `IntrinsicHeight` is "relatively expensive" and should be avoided in `ListView` items. In the worst case it is O(n²) due to the recursive intrinsic measurement protocol.

---

### Wrap — MEDIUM severity in scroll contexts

`Wrap` requires two passes: one to measure children, one to flow them into rows/columns. It is acceptable for static layouts but costly inside `ListView.builder`:

```dart
// ❌ Two layout passes per visible list item
ListView.builder(
  itemBuilder: (context, index) => Wrap(
    children: tags.map((t) => Chip(label: Text(t))).toList(),
  ),
)

// ✅ If tag count is bounded and small, this is acceptable at ≤5 tags
// For large or dynamic tag sets, pre-calculate layout outside build()
```

**Audit rule:** Flag `Wrap` inside `ListView.builder` when the child count per item could exceed 5.

---

## Container vs Specific Widgets

`Container` is a convenience widget that composes multiple behaviors. It adds an extra layer in the render tree compared to using the specific widget directly:

| You need | Use instead of Container |
|---|---|
| Only padding | `Padding` |
| Only a background color | `ColoredBox` |
| Only a background + border/radius | `DecoratedBox` |
| Only a fixed size | `SizedBox` |
| Only alignment | `Align` |
| Multiple properties | `Container` is fine |

```dart
// ❌ Container with only color — unnecessary render object overhead
Container(
  color: Colors.blue,
  child: const Text('Hello'),
)

// ✅ ColoredBox is a leaf render object — more efficient
const ColoredBox(
  color: Colors.blue,
  child: Text('Hello'),
)
```

**Audit rule:** `Container` with only one property set is a low-severity finding. Flag only when it appears repeatedly inside `ListView.builder` items or in hot animation paths.

---

## Offstage and Visibility — Hidden But Still Laid Out

```dart
// ❌ Offstage widgets STILL participate in layout — full layout cost, zero pixels drawn
Offstage(
  offstage: true,
  child: const ExpensiveWidget(),  // laid out on every frame
)

// ❌ Visibility(visible: false) without maintainSize: false still occupies space
Visibility(
  visible: _showPanel,
  child: const PanelWidget(),
)

// ✅ Conditional rendering — widget is removed from the tree entirely
if (_showPanel) const PanelWidget(),

// ✅ Or IndexedStack if you need to preserve state
IndexedStack(
  index: _showPanel ? 0 : 1,
  children: [
    const PanelWidget(),
    const SizedBox.shrink(),
  ],
)
```

**Audit signal:** `grep -rn "Offstage\\|Visibility" lib/ --include="*.dart"` — check each use to confirm `offstage: true` is not permanently set or toggled frequently.

---

## Excessive Nesting — Layout Overhead Accumulation

Each layout widget creates a render object in Flutter's render tree. While individual render objects are cheap, deep chains of single-child layout widgets accumulate traversal cost:

```dart
// ❌ 5 render objects for what could be 1
Container(
  padding: const EdgeInsets.all(16),
  child: Align(
    alignment: Alignment.center,
    child: Padding(
      padding: const EdgeInsets.symmetric(horizontal: 8),
      child: Center(
        child: SizedBox(width: 200, child: myWidget),
      ),
    ),
  ),
)

// ✅ 1 render object
Align(
  alignment: Alignment.center,
  child: Padding(
    padding: const EdgeInsets.fromLTRB(24, 16, 24, 16),
    child: SizedBox(width: 200, child: myWidget),
  ),
)
```

**Audit rule:** Flag chains of 4+ single-child layout wrappers (`Padding`, `Align`, `Center`, `SizedBox`, `Container`, `ConstrainedBox`) that could be collapsed into fewer widgets.

---

## MediaQuery and Theme — Dependency Over-Registration

Every call to `MediaQuery.of(context)` or `Theme.of(context)` registers the widget as a **dependent of that InheritedWidget**. When the keyboard opens/closes, `MediaQueryData` changes, triggering a rebuild of every widget that called `MediaQuery.of(context)`:

```dart
// ❌ Widget rebuilds whenever any MediaQueryData field changes
//    (including keyboard insets, padding, display features)
Widget build(BuildContext context) {
  final size = MediaQuery.of(context).size;
  return SizedBox(width: size.width * 0.5, child: ...);
}

// ✅ Only depends on screen size changes (not keyboard, not padding)
Widget build(BuildContext context) {
  final width = MediaQuery.sizeOf(context).width;  // Flutter 3.10+
  return SizedBox(width: width * 0.5, child: ...);
}
```

**Available targeted accessors (Flutter 3.10+):**
- `MediaQuery.sizeOf(context)` — only rebuilds on size change
- `MediaQuery.paddingOf(context)` — only rebuilds on padding change
- `MediaQuery.viewInsetsOf(context)` — only rebuilds on keyboard inset change

**Audit signal:** `grep -rn "MediaQuery.of(context)" lib/ --include="*.dart"` in deeply nested widgets — recommend targeted accessors where only one field is needed.

---

## Severity Summary

| Pattern | Context | Severity |
|---|---|---|
| `IntrinsicHeight` / `IntrinsicWidth` | Inside `ListView.builder` items | HIGH |
| `IntrinsicHeight` / `IntrinsicWidth` | In static, non-scrolling layout | LOW |
| `Wrap` with many children | Inside `ListView.builder` items | MEDIUM |
| `Offstage(offstage: true)` on heavy widget | Permanent or frequently toggled | MEDIUM |
| `Container` with single property | In hot scroll path | LOW |
| 4+ nested single-child layout wrappers | Anywhere | LOW |
| `MediaQuery.of(context)` for single field | In deeply nested or frequently-rebuilt widget | LOW |
