# Duplication Detection Reference

## Why Duplication Happens in Flutter Projects

Without design system governance, each feature team creates widgets they need independently. The same atom is reimplemented because:
1. No shared component directory is known or enforced
2. The existing atom doesn't quite fit (wrong size, wrong color) — instead of parameterizing it, a new one is created
3. Copy-paste from another feature's widget with a new name
4. No design system PR review checklist

The result: a codebase with 5 button widgets, 4 text field variants, 3 avatar implementations — all slightly different, all unmaintainable.

---

## Duplication Severity Table

| Duplication level | Severity | Consequence |
|---|---|---|
| 2 implementations of the same atom | LOW | Visual inconsistency is possible but contained |
| 3–4 implementations | MEDIUM | Multiple teams maintaining separate bugs; impossible to apply a design change centrally |
| 5+ implementations | HIGH | The design system is functionally absent for this component type |

---

## Button Duplication Patterns

The most commonly duplicated atom in Flutter projects.

### Detection

```bash
grep -rn "class.*Button.*extends StatelessWidget\|class.*Button.*extends StatefulWidget" lib/ --include="*.dart" | grep -v "\.g\.dart\|test"
grep -rn "ElevatedButton\|TextButton\|OutlinedButton\|FilledButton\|GestureDetector.*onTap\|InkWell" lib/ --include="*.dart" | grep -v "\.g\.dart\|test" | grep -i "class\|widget" | head -30
```

### Signal Patterns

| Pattern | Verdict |
|---|---|
| `LoginButton`, `RegisterButton`, `SubmitButton` — all extend `StatelessWidget` and wrap `ElevatedButton` | Clear duplication; feature prefix is the only difference |
| `PrimaryButton` vs `FilledButton` — same visual, different name | Rename to canonical atom |
| `IconButton` with custom styling alongside `AppIconButton` | Check if Flutter's `IconButton` is being wrapped with the same style each time |
| `LoadingButton`, `ConfirmButton`, `DeleteButton` — different states but same base structure | Should be one `AppButton` with `variant` and `isLoading` parameters |

### Canonical Parameterized Atom

```dart
enum AppButtonVariant { primary, secondary, destructive, ghost }
enum AppButtonSize { sm, md, lg }

class AppButton extends StatelessWidget {
  final String label;
  final VoidCallback? onPressed;
  final AppButtonVariant variant;
  final AppButtonSize size;
  final bool isLoading;
  final Widget? leading; // optional icon

  const AppButton({
    super.key,
    required this.label,
    this.onPressed,
    this.variant = AppButtonVariant.primary,
    this.size = AppButtonSize.md,
    this.isLoading = false,
    this.leading,
  });
  ...
}
```

A single `AppButton` with these parameters replaces: `LoginButton`, `RegisterButton`, `CheckoutButton`, `DeleteButton`, `LoadingButton`, `SecondaryButton`, `GhostButton`.

---

## TextField / Input Duplication Patterns

### Detection

```bash
grep -rn "class.*TextField\|class.*Input\|class.*Field\b" lib/ --include="*.dart" | grep "extends StatelessWidget\|extends StatefulWidget" | grep -v "\.g\.dart\|test"
grep -rn "TextFormField\|TextField\b" lib/ --include="*.dart" | grep -v "\.g\.dart\|test\|class\|_text_field\|_input" | head -20
```

### Signal Patterns

| Pattern | Verdict |
|---|---|
| Raw `TextFormField(decoration: InputDecoration(...))` repeated 8+ times across feature files | No shared atom — HIGH |
| `EmailTextField`, `PasswordTextField`, `SearchTextField` — same border/style, different config | 3 implementations of one atom; should be 1 `AppTextField` with type enum |
| `decoration: InputDecoration(border: OutlineInputBorder(borderSide: BorderSide(color: Color(0xFF...))))` repeated | Duplicated styling without tokens |

### Canonical Parameterized Atom

```dart
enum AppTextFieldType { text, email, password, search, multiline }

class AppTextField extends StatefulWidget {
  final String? label;
  final String? hint;
  final String? errorText;
  final AppTextFieldType type;
  final ValueChanged<String>? onChanged;
  final TextEditingController? controller;
  final bool enabled;

  const AppTextField({
    super.key,
    this.label,
    this.hint,
    this.errorText,
    this.type = AppTextFieldType.text,
    this.onChanged,
    this.controller,
    this.enabled = true,
  });
  ...
}
```

---

## Avatar / Profile Image Duplication

### Detection

```bash
grep -rn "class.*Avatar\|class.*ProfileImage\|class.*UserImage" lib/ --include="*.dart" | grep "extends StatelessWidget" | grep -v "\.g\.dart\|test"
grep -rn "CircleAvatar\|ClipOval" lib/ --include="*.dart" | grep -v "\.g\.dart\|test" | head -20
```

### Canonical Atom

```dart
class AppAvatar extends StatelessWidget {
  final String? imageUrl;
  final String? initials;
  final double size;

  const AppAvatar({super.key, this.imageUrl, this.initials, this.size = 40});
  ...
}
```

Replaces: `UserAvatar`, `ProfileAvatar`, `CommentAvatar`, `ParticipantIcon`.

---

## Text / Typography Wrapper Duplication

### Detection

```bash
grep -rn "class.*Text.*extends StatelessWidget\|class.*Label.*extends StatelessWidget\|class.*Heading.*extends StatelessWidget\|class.*Caption.*extends StatelessWidget" lib/ --include="*.dart" | grep -v "\.g\.dart\|test"
```

### Signal Patterns

| Pattern | Verdict |
|---|---|
| `BodyText`, `CaptionText`, `HeadlineText`, `SubtitleText` — all wrapping `Text` with a hardcoded `TextStyle` | 4 implementations of what should be 1 `AppText` with variant enum |
| `Text(value, style: TextStyle(fontSize: 14, color: Color(0xFF...)))` repeated 20+ times | No shared text atom at all — HIGH |

### Canonical Atom

```dart
enum AppTextVariant { displayLarge, headlineMedium, titleLarge, bodyMedium, bodySmall, labelLarge, caption }

class AppText extends StatelessWidget {
  final String text;
  final AppTextVariant variant;
  final Color? colorOverride;
  final int? maxLines;
  final TextOverflow? overflow;

  const AppText(this.text, {super.key, this.variant = AppTextVariant.bodyMedium, this.colorOverride, this.maxLines, this.overflow});
  ...
}
```

---

## Feature-Prefix Naming as a Duplication Signal

A feature-prefix in an atom-level widget name almost always indicates the widget was created for one feature and never shared:

| Name pattern | Signal |
|---|---|
| `CheckoutButton`, `CartButton`, `PaymentButton` | 3 buttons in feature folders — potential duplication |
| `AuthTextField`, `ProfileTextField`, `SearchTextField` | 3 inputs — likely same base style |
| `OrderStatusBadge`, `ShippingBadge`, `InventoryBadge` | 3 badge/chip atoms — candidate for `AppBadge(variant:)` |

Cross-check: open each class. If the visual structure (padding, font, color, shape) is the same or differs only in hardcoded magic numbers — it is duplication. If the structure is genuinely different (one is a chip, one is a `Container` badge, one is an icon) — they are different components.

---

## Extraction Checklist

Use this when recommending consolidation of duplicated atoms:

- [ ] Name the canonical atom (`AppButton`, `AppTextField`, `AppText`, `AppAvatar`)
- [ ] Define the variant/type enum covering all existing use cases
- [ ] Define all constructor parameters replacing hardcoded values
- [ ] Confirm the atom uses design tokens only (no raw values)
- [ ] Confirm zero imports from feature layer
- [ ] Confirm const constructor is possible
- [ ] Identify all call sites to update (output file list in the report)
