# Design Token Usage Reference

## Token System Options in Flutter

Flutter projects commonly implement design tokens in one of three ways. All three are valid; the problem arises when they are mixed without a documented strategy.

| Approach | Mechanism | When appropriate |
|---|---|---|
| **Flutter-native** | `ThemeData`, `ColorScheme`, `TextTheme`, `MaterialStateProperty` | Projects that stay close to Material Design and benefit from system-level theming (dark mode, dynamic color) |
| **Custom token classes** | `AppColors`, `AppSpacing`, `AppTextStyles`, `AppRadius` | Projects with a custom design language that diverges from Material; multi-platform (Flutter Web + Mobile) |
| **Hybrid** | `ColorScheme` for semantic colors + `AppSpacing` for spacing + custom `AppTextStyles` | Projects migrating from one approach to the other; must be intentional and documented |

The following counts as a violation only when tokens from **two uncoordinated strategies** are mixed:

```dart
// VIOLATION — two uncoordinated color sources in the same atom
class AppCard extends StatelessWidget {
  @override
  Widget build(BuildContext context) => Container(
    color: AppColors.surface,           // custom token
    child: Text(
      label,
      style: TextStyle(color: Theme.of(context).colorScheme.primary), // Flutter-native
    ),
  );
}
// Mixed without a documented bridge — changes to AppColors don't propagate to colorScheme
```

---

## Token Classes — Canonical Structure

```dart
// lib/ui/tokens/app_colors.dart
import 'package:flutter/material.dart';

abstract final class AppColors {
  // Brand
  static const primary = Color(0xFF1A73E8);
  static const primaryVariant = Color(0xFF1557B0);
  static const secondary = Color(0xFF34A853);

  // Semantic
  static const error = Color(0xFFD93025);
  static const warning = Color(0xFFF9AB00);
  static const success = Color(0xFF1E8E3E);

  // Surface
  static const background = Color(0xFFF8F9FA);
  static const surface = Color(0xFFFFFFFF);
  static const onSurface = Color(0xFF202124);

  // Text
  static const textPrimary = Color(0xFF202124);
  static const textSecondary = Color(0xFF5F6368);
  static const textDisabled = Color(0xFFBDC1C6);
}
```

```dart
// lib/ui/tokens/app_spacing.dart
abstract final class AppSpacing {
  static const double xs = 4.0;
  static const double sm = 8.0;
  static const double md = 16.0;
  static const double lg = 24.0;
  static const double xl = 32.0;
  static const double xxl = 48.0;
}
```

```dart
// lib/ui/tokens/app_text_styles.dart
import 'package:flutter/material.dart';

abstract final class AppTextStyles {
  static const displayLarge = TextStyle(fontSize: 57, fontWeight: FontWeight.w400, letterSpacing: -0.25);
  static const headlineMedium = TextStyle(fontSize: 28, fontWeight: FontWeight.w400);
  static const titleLarge = TextStyle(fontSize: 22, fontWeight: FontWeight.w400);
  static const labelLarge = TextStyle(fontSize: 14, fontWeight: FontWeight.w500, letterSpacing: 0.1);
  static const bodyMedium = TextStyle(fontSize: 14, fontWeight: FontWeight.w400, letterSpacing: 0.25);
  static const bodySmall = TextStyle(fontSize: 12, fontWeight: FontWeight.w400, letterSpacing: 0.4);
}
```

---

## Raw Value Detection Patterns

### Color violations

| Pattern | Severity | Fix |
|---|---|---|
| `Color(0xFF...)` in an atom or molecule | HIGH (> 5 occurrences) / MEDIUM | Replace with `AppColors.*` or `Theme.of(context).colorScheme.*` |
| `Colors.red`, `Colors.blue` (named Flutter colors) | MEDIUM | These are not design tokens; use semantic color from token class |
| `Colors.transparent` | LOW | Acceptable — no meaningful semantic token equivalent |
| `Colors.white`, `Colors.black` | LOW | Acceptable when design system has no explicit override |

### Spacing violations

| Pattern | Severity | Fix |
|---|---|---|
| `EdgeInsets.all(17)` — non-round number | MEDIUM | Non-scale value; replace with nearest spacing token |
| `EdgeInsets.all(8)` — round but hardcoded | LOW | Preferably replace with `AppSpacing.sm`; not critical |
| `SizedBox(height: 13)` | MEDIUM | Non-scale value |
| `SizedBox(height: 16)` | LOW | Matches common spacing scale; LOW because it's likely intentional |

Round numbers that match a spacing scale (4, 8, 12, 16, 24, 32, 48) are LOW severity. Odd numbers (13, 17, 7, 22) are MEDIUM — they indicate a developer measured per-pixel rather than using the scale.

### Typography violations

| Pattern | Severity | Fix |
|---|---|---|
| `TextStyle(fontSize: 15, fontWeight: FontWeight.w600)` inside an atom | MEDIUM | Replace with `AppTextStyles.labelLarge` |
| `TextStyle(fontSize: 14)` in an atom | LOW | Replace with `AppTextStyles.bodyMedium` |
| `DefaultTextStyle.merge(style: TextStyle(...))` in templates | MEDIUM | Use token-based style; merge is acceptable mechanism |

### Radius violations

| Pattern | Severity | Fix |
|---|---|---|
| `BorderRadius.circular(8)` — round, matches scale | LOW | Acceptable; preferably `AppRadius.sm` |
| `BorderRadius.circular(13)` — non-standard | MEDIUM | Odd radius; replace with scale token |

---

## ThemeData Integration Pattern

When using Flutter-native tokens, the `ThemeData` is the token source and atoms consume it via `Theme.of(context)`:

```dart
// lib/app/app_theme.dart
ThemeData buildAppTheme() => ThemeData(
  colorScheme: const ColorScheme.light(
    primary: Color(0xFF1A73E8),
    secondary: Color(0xFF34A853),
    error: Color(0xFFD93025),
    surface: Color(0xFFFFFFFF),
  ),
  textTheme: const TextTheme(
    labelLarge: TextStyle(fontSize: 14, fontWeight: FontWeight.w500),
    bodyMedium: TextStyle(fontSize: 14),
  ),
  elevatedButtonTheme: ElevatedButtonThemeData(
    style: ElevatedButton.styleFrom(
      minimumSize: const Size.fromHeight(48),
      shape: const StadiumBorder(),
    ),
  ),
);

// lib/ui/atoms/app_button.dart — consuming native tokens
@override
Widget build(BuildContext context) {
  final cs = Theme.of(context).colorScheme;
  return ElevatedButton(
    style: ElevatedButton.styleFrom(backgroundColor: cs.primary),
    ...
  );
}
```

---

## Token Generation Tooling

Large enterprise projects generate token files from Figma or design tool exports. Common tools:

| Tool | Output | Verdict |
|---|---|---|
| Style Dictionary | `app_colors.dart`, `app_spacing.dart` | Best practice for teams with design-engineering sync |
| Figma Tokens Plugin + `build_runner` | Generated Dart constants | Acceptable; reduces manual sync errors |
| Manual maintenance | Handwritten `AppColors` class | Acceptable for small teams; drifts from design over time |

Flag as LOW technical debt when token files are manually maintained but no generation tooling exists — this is a future risk, not an immediate violation.
