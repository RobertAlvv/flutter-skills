# Golden Test Prerequisites Reference

This reference defines the architectural prerequisites for reliable golden tests in Flutter, from an auditor's perspective. Use it to calibrate findings in Step 7 (golden test architecture prerequisites).

---

## What Is a Golden Test

A golden test captures a reference screenshot of a widget and asserts that future renders are pixel-identical:

```dart
testWidgets('OrderCard matches golden', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(home: OrderCard(total: '\$42.00', date: 'Jan 1, 2025')),
  );
  await expectLater(find.byType(OrderCard), matchesGoldenFile('order_card.png'));
});
```

The test passes only if the rendered output is **bit-for-bit identical** to the stored `.png` baseline. Any non-determinism in the widget — a timestamp, a random color, a platform-specific font — will cause the test to fail on a different machine, on CI, or on a later run.

---

## Determinism Requirements

| Non-deterministic input | Impact on golden | Fix | Severity |
|---|---|---|---|
| `DateTime.now()` rendered in widget | Baseline changes every second | Pass date as constructor parameter | HIGH |
| `Random().nextDouble()` for visual value | Baseline pixel-differs every run | Seed Random or pass value as parameter | HIGH |
| `Platform.isIOS` / `Platform.isAndroid` branch affecting layout or colors | Baseline differs between macOS CI and Linux CI | Inject `TargetPlatform` parameter or use platform-agnostic design | MEDIUM |
| `defaultTargetPlatform` check in widget tree | Same as above | Explicit `TargetPlatform` override in test | MEDIUM |
| Animated widget not settled before capture | Captures intermediate frame | Ensure `pumpAndSettle()` is called; or set animation to completed | MEDIUM |
| Custom font not loaded | Text renders in fallback font — pixel differs from device render | Use `loadFonts` fixture or `fontFamily: null` in test theme | MEDIUM |
| `Image.network(url)` with no mock | Image area is blank/broken in golden | Use `Image.asset` in test or provide `FakeHttpClient` | MEDIUM |

---

## Font Loading Setup for Golden Tests

Golden tests on CI fail if the test environment does not load the custom fonts used by the app. The auditor should check for a font-loading helper:

```dart
// Expected in test/helpers/ or test/golden/
Future<void> loadAppFonts() async {
  final fontLoader = FontLoader('Roboto')
    ..addFont(rootBundle.load('assets/fonts/Roboto-Regular.ttf'));
  await fontLoader.load();
}

// Used in golden test setUp:
setUp(() async => loadAppFonts());
```

**Audit signal:** If the app uses custom fonts (check `pubspec.yaml` → `flutter.fonts`) and no `FontLoader` or `loadAppFonts` helper exists in `test/`, golden test output will differ from production rendering.

---

## goldenFileComparator Configuration

By default, Flutter's golden comparator compares on the CI host platform. For cross-platform CI stability, projects should configure a custom comparator:

```dart
// test/flutter_test_config.dart
import 'package:flutter_test/flutter_test.dart';

Future<void> testExecutable(FutureOr<void> Function() testMain) async {
  goldenFileComparator = LocalFileComparator(
    Uri.file('test/goldens/'),
  );
  await testMain();
}
```

**Audit signal:** If `goldenFileComparator` is never set and `matchesGoldenFile` calls exist, golden baselines will be generated relative to the test file path — this causes divergence when file structure changes.

---

## AnimationController in Golden Tests

Widgets with `AnimationController` require special handling:

```dart
// VIOLATION — late AnimationController with no vsync override
class FadeCard extends StatefulWidget { ... }
class _FadeCardState extends State<FadeCard> with SingleTickerProviderStateMixin {
  late final _controller = AnimationController(
    vsync: this, duration: const Duration(milliseconds: 300),
  );
  @override
  void initState() {
    super.initState();
    _controller.forward(); // starts animation in initState
  }
}
```

In a `testWidgets`, `pumpAndSettle()` will wait for `_controller` to complete. If the animation is bound to a real ticker (not a test ticker), it may hang indefinitely in golden tests.

**Correct approach in test:**

```dart
await tester.pumpWidget(MaterialApp(home: FadeCard()));
await tester.pumpAndSettle(); // waits for animation to complete
await expectLater(find.byType(FadeCard), matchesGoldenFile('fade_card.png'));
```

**Audit signal:** Widgets with `AnimationController` started in `initState` without an injectable vsync abstraction or explicit `ticker → completes` contract are MEDIUM golden test risk.

---

## Platform Divergence Detection Patterns

The following bash patterns detect platform branches inside presentational widgets:

```bash
# Check for platform-conditional rendering inside widgets
grep -rn "Platform\.isIOS\|Platform\.isAndroid\|defaultTargetPlatform" \
  lib/ --include="*.dart" | grep -v test | grep -v "theme\|ThemeData"
```

```bash
# Check for kIsWeb used in layout decisions
grep -rn "kIsWeb" lib/**/widgets/ lib/**/screens/ --include="*.dart" | grep -v test
```

**Severity calibration:**

| Branch location | Severity | Rationale |
|---|---|---|
| Inside a `ThemeData` or platform-specific `CupertinoWidget` wrapper | LOW | Expected platform adaptation |
| Inside a layout widget changing padding or size | MEDIUM | Golden baseline diverges on different CI platforms |
| Inside a color or text style decision | MEDIUM | Visual diff between macOS and Linux CI |
| Inside a content/copy decision | HIGH | Semantic output differs by platform |

---

## Golden Test Readiness Checklist

An auditor should assess:

- [ ] No `DateTime.now()` or `Random()` rendered directly in widget `build()` methods
- [ ] No `Platform.is*` or `defaultTargetPlatform` checks altering layout or colors in presentational widgets
- [ ] Custom fonts are loaded in a test helper (`FontLoader` or `loadFonts`)
- [ ] `goldenFileComparator` is configured in `flutter_test_config.dart` or equivalent
- [ ] `Image.network` calls in presentational widgets are replaceable with asset-based or mock images in tests
- [ ] Animated widgets settle within `pumpAndSettle` without infinite loops
- [ ] Existing `matchesGoldenFile` count in `test/` > 0 for a project claiming golden coverage

---

## Severity Table — Golden Test Blockers

| Finding | Severity | Rationale |
|---|---|---|
| `DateTime.now()` rendered inline in widget | HIGH | Golden fails every run by definition |
| `Random()` output used in visual rendering | HIGH | Non-deterministic pixel output |
| `Platform.isIOS` branch changes layout | MEDIUM | CI platform divergence |
| Custom fonts used but no `loadFonts` helper | MEDIUM | Text renders in fallback font on CI |
| `AnimationController` not settled before capture | MEDIUM | Captures in-flight frame |
| `Image.network` with no mock in golden test | MEDIUM | Missing image area breaks baseline |
| 0 `matchesGoldenFile` calls in project with 20+ screens | MEDIUM | Golden coverage entirely absent |
