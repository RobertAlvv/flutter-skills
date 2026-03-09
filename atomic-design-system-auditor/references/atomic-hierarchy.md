# Atomic Hierarchy Reference

## Widget Classification Criteria

Use these observable characteristics to classify a widget at the correct atomic level — regardless of folder name or class name.

### Atom

| Criterion | Must satisfy |
|---|---|
| Renders | A single UI primitive (text, icon, button, divider, badge, avatar, chip, switch, checkbox) |
| State | Stateless or minimal ephemeral state (focus, hover highlight) |
| Imports | Only other atoms, token classes, or Dart core packages |
| Flutter package imports | `widgets.dart`, `material.dart`, `cupertino.dart` — no feature packages |
| Domain imports | None — no model classes, no repository interfaces |
| Test requirement | Renderable with `WidgetTester` without any `BlocProvider` or `RepositoryProvider` wrapping |

### Molecule

| Criterion | Must satisfy |
|---|---|
| Renders | 2–5 atoms composed into a self-contained interaction pattern |
| State | May have local `FocusNode`, hover flag, or toggle — no async state |
| Imports | Atoms + token classes. May import `flutter_bloc` only for `BlocBuilder` on a single field, not full state |
| Domain imports | None — receives all data via typed constructor parameters |
| Feature coupling | Zero — the same molecule must be usable in any feature |

### Organism

| Criterion | Must satisfy |
|---|---|
| Renders | A complete, meaningful UI section (a form, a card, a header section, a list item with full content) |
| State | Receives all state data via constructor parameters or callbacks — does not call `BlocProvider.of<X>()` internally |
| Imports | Molecules + atoms. May import domain model types for parameter typing |
| Feature coupling | Typically feature-specific, but makes no navigation calls and issues no repository calls |
| Size | Up to ~150 lines `build()` method; decompose further if exceeded |

### Template

| Criterion | Must satisfy |
|---|---|
| Renders | Layout skeleton — defines scroll regions, app bar slot, body slot, FAB slot |
| Content | All content injected via `Widget` parameters (named slots) — no hardcoded content |
| State | Stateless — zero business logic, no `BlocBuilder`, no conditional content |
| Imports | Only other templates, atoms for structural helpers (dividers, safe area), and core Flutter |

### Page

| Criterion | Must satisfy |
|---|---|
| Renders | The route endpoint — corresponds to a `GoRouter` route or `Navigator.pushNamed` destination |
| Wiring | Contains `BlocProvider`, `RepositoryProvider`, `Provider`, or `ProviderScope` |
| Build size | Short (< 50 lines) — delegates all layout to templates and organisms |
| Navigation | Only layer allowed to call `context.go()`, `Navigator.push()`, or `context.read<X>().add(NavEvent())` |

---

## Import Direction Rules

```
atoms ──► molecules ──► organisms ──► templates ──► pages
  ▲              ▲              ▲              ▲
  └── token classes only
```

**Allowed imports at each level:**

| Level | May import |
|---|---|
| Atom | Other atoms, token classes (`AppColors`, `AppSpacing`, `AppTextStyles`), Dart core, Flutter material/widgets |
| Molecule | Atoms, token classes |
| Organism | Molecules, atoms, domain model types (for parameter typing only) |
| Template | Organisms, molecules, atoms, token classes |
| Page | Templates, organisms, molecules, atoms, BLoC/Cubit/Riverpod providers, repositories |

---

## Severity Table: Hierarchy Violations

| Violation | Severity | Rationale |
|---|---|---|
| Atom imports a molecule or organism | HIGH | Atom can no longer be reused without the organism's dependencies |
| Atom imports a page-level widget | HIGH | Circular dependency; atom now depends on the entire app context |
| Molecule imports an organism | HIGH | Molecule cannot be shared without pulling in feature-specific organism |
| Organism calls `BlocProvider.of<X>(context)` directly | HIGH | Organism is now coupled to the DI graph; cannot be used in isolation or tests |
| Organism imports a repository or service | HIGH | Business logic has leaked into the UI layer |
| Template contains conditional state-driven rendering | MEDIUM | Template becomes feature-aware; cannot be reused across features with different state shapes |
| Organism imports another organism from a different feature | MEDIUM | Feature-to-feature coupling via UI layer |
| Page `build()` > 150 lines with inline UI primitives | MEDIUM | Page is acting as organism; layout not reusable |
| No template layer — pages implement scaffold directly | MEDIUM | Layout cannot be reused across screens with same structure |
| Molecule has local async state | LOW | May signal it should be elevated to an organism |

---

## Flutter-Specific Examples

### Correct Atom

```dart
// lib/ui/atoms/app_button.dart
import 'package:flutter/material.dart';
import '../tokens/app_colors.dart';
import '../tokens/app_spacing.dart';
import '../tokens/app_text_styles.dart';

enum AppButtonVariant { primary, secondary, destructive }

class AppButton extends StatelessWidget {
  final String label;
  final VoidCallback? onPressed;
  final AppButtonVariant variant;

  const AppButton({
    super.key,
    required this.label,
    this.onPressed,
    this.variant = AppButtonVariant.primary,
  });

  @override
  Widget build(BuildContext context) => ElevatedButton(
    onPressed: onPressed,
    style: _styleFor(variant),
    child: Text(label, style: AppTextStyles.labelLarge),
  );

  ButtonStyle _styleFor(AppButtonVariant v) => switch (v) {
    AppButtonVariant.primary => ElevatedButton.styleFrom(
        backgroundColor: AppColors.primary,
        padding: EdgeInsets.symmetric(horizontal: AppSpacing.md, vertical: AppSpacing.sm),
      ),
    AppButtonVariant.secondary => ElevatedButton.styleFrom(
        backgroundColor: AppColors.surface,
        foregroundColor: AppColors.primary,
      ),
    AppButtonVariant.destructive => ElevatedButton.styleFrom(
        backgroundColor: AppColors.error,
      ),
  };
}
```

### Correct Molecule

```dart
// lib/ui/molecules/labeled_text_field.dart
import 'package:flutter/material.dart';
import '../atoms/app_text.dart';
import '../atoms/app_text_field.dart';
import '../tokens/app_spacing.dart';

class LabeledTextField extends StatelessWidget {
  final String label;
  final String? hint;
  final ValueChanged<String>? onChanged;
  final TextEditingController? controller;

  const LabeledTextField({super.key, required this.label, this.hint, this.onChanged, this.controller});

  @override
  Widget build(BuildContext context) => Column(
    crossAxisAlignment: CrossAxisAlignment.start,
    children: [
      AppText(label, variant: AppTextVariant.label),
      SizedBox(height: AppSpacing.xs),
      AppTextField(hint: hint, onChanged: onChanged, controller: controller),
    ],
  );
}
```

### Correct Organism — receives data, does not pull from BLoC

```dart
// lib/features/auth/organisms/login_form_organism.dart
import 'package:flutter/material.dart';
import '../../../ui/molecules/labeled_text_field.dart';
import '../../../ui/atoms/app_button.dart';

class LoginFormOrganism extends StatelessWidget {
  final String? emailError;
  final String? passwordError;
  final bool isLoading;
  final ValueChanged<String> onEmailChanged;
  final ValueChanged<String> onPasswordChanged;
  final VoidCallback onSubmit;

  const LoginFormOrganism({/* ... */});

  @override
  Widget build(BuildContext context) => Column(
    children: [
      LabeledTextField(label: 'Email', hint: 'you@example.com', onChanged: onEmailChanged),
      LabeledTextField(label: 'Password', onChanged: onPasswordChanged),
      AppButton(label: isLoading ? 'Logging in…' : 'Log in', onPressed: isLoading ? null : onSubmit),
    ],
  );
}
```
