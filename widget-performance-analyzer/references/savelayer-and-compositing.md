# saveLayer() and Compositing Reference — Audit Guide

Use this file to identify implicit `saveLayer()` calls and compositing layer misuse in the audited project. This is the most commonly overlooked Flutter performance issue and the leading cause of unexplained jank in production apps.

---

## What is saveLayer()?

`saveLayer()` is a Skia (and Impeller) GPU operation that:
1. Flushes the current render pipeline
2. Allocates an **offscreen texture** (framebuffer)
3. Renders the widget subtree into that texture
4. Composites the texture back onto the main framebuffer

This process is expensive in both GPU time and memory. It can easily cost 2–5ms per frame — a significant fraction of the 16ms budget.

**The problem:** `saveLayer()` is called implicitly by several common Flutter widgets. Developers often add them unaware of the GPU cost.

---

## Implicit saveLayer() Triggers — Ranked by Severity

### 1. `Opacity` with a continuously-changing value — HIGH

```dart
// ❌ Calls saveLayer() on EVERY animation frame (60-120x/second)
AnimatedBuilder(
  animation: _fadeController,
  builder: (context, child) => Opacity(
    opacity: _fadeController.value,  // changes every frame
    child: child,
  ),
)

// ✅ FadeTransition operates at the compositing layer — zero saveLayer()
FadeTransition(
  opacity: _fadeController,
  child: const MyWidget(),
)
```

**Exception:** `Opacity(opacity: 0.0)` and `Opacity(opacity: 1.0)` are short-circuited by Flutter — no `saveLayer()` is triggered at exactly these values.

**Exception:** `AnimatedOpacity` for state-driven (non-frame-driven) opacity changes is acceptable — it only triggers `saveLayer()` during the transition, not continuously.

---

### 2. `BackdropFilter` — HIGH

`BackdropFilter` **always** triggers `saveLayer()` regardless of how it is used. It must blur/sample what is behind it, which requires an offscreen buffer.

```dart
// ❌ saveLayer() on every repaint
BackdropFilter(
  filter: ImageFilter.blur(sigmaX: 10, sigmaY: 10),
  child: Container(color: Colors.white.withOpacity(0.2)),
)
```

**Audit rule:** Flag any `BackdropFilter` inside a `ListView.builder` item or inside a widget that rebuilds frequently. Accept it only in static overlays (modals, drawers) that are shown infrequently.

---

### 3. `ShaderMask` / `ColorFiltered` — MEDIUM

Both apply a shader or color filter over a subtree by compositing into an offscreen buffer.

```dart
// ❌ saveLayer() on every rebuild of the parent
ShaderMask(
  shaderCallback: (bounds) => gradient.createShader(bounds),
  child: const Icon(Icons.star),
)
```

Accept only on static widgets. Flag if inside a rebuild-heavy context.

---

### 4. `BoxDecoration` with `boxShadow` + `borderRadius` — MEDIUM

This combination forces `saveLayer()` because Skia must composite the shadow under the rounded clip:

```dart
// ❌ saveLayer() on every repaint of this widget
Container(
  decoration: BoxDecoration(
    borderRadius: BorderRadius.circular(12),
    boxShadow: [BoxShadow(blurRadius: 8, color: Colors.black26)],
  ),
  child: ...,
)
```

**Fix:** Use `borderRadius` alone (no shadow) or `PhysicalModel` which has a more optimized shadow path on some platforms.

**Audit rule:** Flag this combination on list items — every visible item triggers `saveLayer()` on every scroll frame.

---

### 5. `ClipRRect` / `ClipOval` / `ClipPath` — MEDIUM

Clipping promotes the clipped widget to its own compositing layer:

```dart
// ❌ Compositing layer per list item — significant raster cost in lists
ClipRRect(
  borderRadius: BorderRadius.circular(8),
  child: Image.network(url),
)

// ✅ No compositor layer needed
Container(
  decoration: BoxDecoration(
    borderRadius: BorderRadius.circular(8),
    image: DecorationImage(image: NetworkImage(url), fit: BoxFit.cover),
  ),
)
```

**Audit rule:** Flag `ClipRRect` inside `ListView.builder` item widgets. Accept it in static, non-scrolling layouts.

---

## RepaintBoundary — Isolation Strategy

`RepaintBoundary` tells Flutter: "when this widget repaints, do not repaint its siblings or ancestors."

Each `RepaintBoundary` creates a compositing layer. Too few = unnecessarily large repaint regions. Too many = excessive layer overhead.

### When to recommend RepaintBoundary

```dart
// ✅ Spinner inside a static screen — without RepaintBoundary,
//    the entire screen repaints at 60fps during loading
Column(
  children: [
    const StaticHeader(),
    RepaintBoundary(
      child: const LoadingSpinner(),  // isolated — only this repaints
    ),
    const StaticFooter(),
  ],
)

// ✅ Lottie/Rive animation alongside static content
RepaintBoundary(
  child: LottieAnimation(),
)

// ✅ Live-updating widget (timer, price ticker, live score)
RepaintBoundary(
  child: PriceTicker(price: currentPrice),
)
```

### When NOT to recommend RepaintBoundary

```dart
// ❌ On every list item — each layer has memory cost
ListView.builder(
  itemBuilder: (context, index) => RepaintBoundary(  // overkill
    child: ProductCard(product: products[index]),
  ),
)

// ❌ On a widget that rarely repaints (wasted layer)
RepaintBoundary(
  child: const AppLogo(),  // static — never repaints
)
```

**Audit signal:** If `RepaintBoundary` appears nowhere in the codebase AND the app has continuous animations (loaders, lottie, video), flag it as a missing optimization opportunity.

---

## FadeTransition vs Opacity vs AnimatedOpacity

| Widget | Mechanism | saveLayer()? | When to use |
|---|---|---|---|
| `FadeTransition` | Compositing layer opacity | ❌ Never | `AnimationController`-driven fades |
| `AnimatedOpacity` | Implicit animation | Only during transition | State-driven fades (e.g. show/hide on button press) |
| `Opacity` with static value 0.0 or 1.0 | Short-circuited | ❌ Never | Conditional visibility |
| `Opacity` with dynamic non-0/1 value | `saveLayer()` | ✅ Every repaint | **Avoid** — use `FadeTransition` instead |

---

## Severity Summary

| Pattern | Context | Severity |
|---|---|---|
| `Opacity(opacity: animValue)` with continuous animation | Anywhere | HIGH |
| `BackdropFilter` | Inside `ListView.builder` items | HIGH |
| `BackdropFilter` | Static modal / drawer | LOW (acceptable) |
| `BoxDecoration` with `boxShadow` + `borderRadius` | List items | HIGH |
| `ClipRRect` | Inside `ListView.builder` | MEDIUM |
| `ShaderMask` / `ColorFiltered` | Inside frequently-rebuilt widget | MEDIUM |
| No `RepaintBoundary` around continuous animation | Alongside static content | MEDIUM |
| Excessive `RepaintBoundary` on static widgets | Anywhere | LOW |
