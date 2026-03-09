# Stitch → Flutter Pipeline

Complete guide for translating Stitch (Firebase/Google) designs into production Flutter code via the MCP.

---

## What Stitch MCP Provides

- **Design tokens** — colors (hex), typography (font family, size, weight), spacing scales
- **Component tree** — layout hierarchy, nesting, component types
- **Screen descriptions** — what each screen shows, navigation triggers
- **Exported assets** — SVG icons, PNG images

Stitch MCP does **not** generate Flutter code. This pipeline maps its output to production-quality Dart.

---

## Phase 0: Project Foundation

Before touching any screen, ensure the project has:

```
✅ pubspec.yaml with all dependencies at latest versions
✅ main.dart bootstraps: dotenv → Firebase → configureDependencies() → runApp
✅ AppColors extracted from Stitch design tokens
✅ AppTypography extracted from Stitch design tokens
✅ AppTheme using Material 3 + AppColors
✅ AppRouter initialized as @lazySingleton
✅ injection.dart with @InjectableInit
✅ .env file with required variables (gitignored)
✅ build_runner runs cleanly with no errors
```

---

## Phase 1: Extract Design Tokens from Stitch

Via MCP, read the design tokens and map them to Dart constants.

### Colors
```dart
// core/theme/app_colors.dart
// Exact hex values from Stitch — never approximate
class AppColors {
  AppColors._();
  static const primary        = Color(0xFF5B4FFF); // Stitch: brand-primary
  static const primaryVariant = Color(0xFF3D35CC); // Stitch: brand-primary-dark
  static const secondary      = Color(0xFF03DAC6); // Stitch: brand-secondary
  static const surface        = Color(0xFFFFFFFF); // Stitch: surface
  static const background     = Color(0xFFF5F5F5); // Stitch: background
  static const error          = Color(0xFFB00020); // Stitch: semantic-error
  static const textPrimary    = Color(0xFF1A1A2E); // Stitch: text-primary
  static const textSecondary  = Color(0xFF6B7280); // Stitch: text-secondary
}
```

### Typography
```dart
// core/theme/app_typography.dart
// Font: as defined in Stitch (e.g., Inter, Roboto, DM Sans)
class AppTypography {
  AppTypography._();
  static const String _family = 'Inter'; // From Stitch font token

  static const displayLarge = TextStyle(
    fontFamily: _family, fontSize: 57, fontWeight: FontWeight.w400,
  );
  static const headlineLarge = TextStyle(
    fontFamily: _family, fontSize: 32, fontWeight: FontWeight.w700,
  );
  static const headlineMedium = TextStyle(
    fontFamily: _family, fontSize: 24, fontWeight: FontWeight.w600,
  );
  static const titleLarge = TextStyle(
    fontFamily: _family, fontSize: 22, fontWeight: FontWeight.w500,
  );
  static const bodyLarge = TextStyle(
    fontFamily: _family, fontSize: 16, fontWeight: FontWeight.w400,
  );
  static const bodyMedium = TextStyle(
    fontFamily: _family, fontSize: 14, fontWeight: FontWeight.w400,
  );
  static const labelLarge = TextStyle(
    fontFamily: _family, fontSize: 14, fontWeight: FontWeight.w500,
    letterSpacing: 0.1,
  );
  static const caption = TextStyle(
    fontFamily: _family, fontSize: 12, fontWeight: FontWeight.w400,
    color: AppColors.textSecondary,
  );
}
```

---

## Phase 2: Screen Inventory → Route Map

For every screen in Stitch, create one route entry. Build the complete inventory before coding any UI:

```dart
@AutoRouterConfig(replaceInRouteName: 'Page,Route')
class AppRouter extends RootStackRouter {
  @override
  List<AutoRoute> get routes => [
    // Auth flow (unauthenticated)
    AutoRoute(page: SplashRoute.page, initial: true),
    AutoRoute(page: LoginRoute.page),
    AutoRoute(page: RegisterRoute.page),
    AutoRoute(page: ForgotPasswordRoute.page),

    // Main shell (authenticated)
    AutoRoute(
      page: ShellRoute.page,
      guards: [AuthGuard()],
      children: [
        AutoRoute(page: HomeRoute.page),
        AutoRoute(page: ExploreRoute.page),
        AutoRoute(page: CartRoute.page),
        AutoRoute(page: ProfileRoute.page),
      ],
    ),

    // Feature routes
    AutoRoute(page: ProductDetailRoute.page),
    AutoRoute(page: CheckoutRoute.page),
    AutoRoute(page: OrderConfirmationRoute.page),
    // ... one entry per Stitch screen
  ];
}
```

---

## Phase 3: Analyze Each Screen — Decision Matrix

Before writing any code for a screen, answer:

| Question | If Yes → | If No → |
|---|---|---|
| Does it show data from backend? | UseCase + Repository + BLoC | Static/hardcoded widget |
| Can user perform actions? | BLoC Events for each action | Read-only BlocBuilder |
| Does it navigate to other screens? | AutoRoute push/replace | — |
| Does it have a form? | TextEditingController + validation | — |
| Does it show a list? | ListView.builder + BlocBuilder | Column |
| Does it have real-time data? | Stream + Watch UseCase + BLoC stream sub | Future once |
| Is it behind auth? | AuthGuard on route | Open route |

---

## Phase 4: Stitch Layout → Flutter Widget Mapping

| Stitch Element | Flutter Widget | Notes |
|---|---|---|
| Frame / Container | `Container` or `SizedBox` | |
| Frame with padding | `Padding` | |
| Frame with scroll (vertical) | `SingleChildScrollView` or `ListView` | Use `ListView.builder` for dynamic |
| Frame with scroll (horizontal) | `SingleChildScrollView(scrollDirection: Axis.horizontal)` | |
| Auto-layout row | `Row` | Set `mainAxisAlignment`, `crossAxisAlignment` |
| Auto-layout column | `Column` | Same |
| Auto-layout with gap | `Column/Row` + `SizedBox(height/width: gap)` | Or `gap` in Flutter 3.27+ |
| Wrap / flow layout | `Wrap` | |
| Stack of layers | `Stack` + `Positioned` | |
| Grid | `GridView.builder` | |
| Card with shadow | `Card` or `Container` + `BoxDecoration(boxShadow: ...)` | |
| Circular image | `ClipOval` wrapping `CachedNetworkImage` | |
| Rounded image | `ClipRRect` with `borderRadius` | |
| Bottom sheet | `showModalBottomSheet` | |
| Drawer | `Drawer` in `Scaffold` | |
| Tab bar | `AutoTabsScaffold` | |
| Bottom nav bar | `AutoTabsScaffold.bottomNavigationBuilder` | |
| Divider line | `Divider` or `SizedBox(height: 1)` + color | |
| Badge / chip | `Chip` or custom `Container` | |
| Avatar | `CircleAvatar` | |
| Icon button | `IconButton` | |
| Primary CTA | `FilledButton` (Material 3) | |
| Secondary CTA | `OutlinedButton` | |
| Text button | `TextButton` | |
| Text input | `TextField` with `InputDecoration` | |
| Dropdown | `DropdownButtonFormField` | |
| Toggle | `Switch` | |
| Checkbox | `Checkbox` | |
| Radio | `Radio<T>` | |
| Slider | `Slider` | |
| Progress indicator | `LinearProgressIndicator` or `CircularProgressIndicator` | |
| Shimmer/skeleton | `shimmer` package | |
| App bar | `AppBar` in `Scaffold` | |
| Floating action button | `FloatingActionButton` in `Scaffold` | |
| Snackbar | `ScaffoldMessenger.of(context).showSnackBar` | |
| Dialog | `showDialog` + `AlertDialog` | |

---

## Phase 5: Translating a Stitch Card Component

Stitch describes: *"Card — 16px padding, 12px radius, 2dp shadow. Left: 80x80 circular image. Right column: title (bodyLarge bold) + subtitle (caption). Trailing: chevron icon."*

```dart
class ProductCard extends StatelessWidget {
  final ProductEntity product;
  final VoidCallback onTap;

  const ProductCard({
    super.key,
    required this.product,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
      ),
      elevation: 2,
      child: InkWell(
        onTap: onTap,
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Row(
            children: [
              // Circular image — 80x80
              ClipOval(
                child: CachedNetworkImage(
                  imageUrl: product.imageUrl,
                  width: 80,
                  height: 80,
                  fit: BoxFit.cover,
                  memCacheWidth: 160, // 2x for retina
                  placeholder: (_, __) => Container(
                    width: 80,
                    height: 80,
                    color: AppColors.background,
                  ),
                  errorWidget: (_, __, ___) => const Icon(
                    Icons.image_not_supported_outlined,
                    color: AppColors.textSecondary,
                  ),
                ),
              ),
              const SizedBox(width: 16),
              // Title + subtitle
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      product.name,
                      style: AppTypography.bodyLarge.copyWith(
                        fontWeight: FontWeight.w700,
                      ),
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                    ),
                    const SizedBox(height: 4),
                    Text(
                      product.category,
                      style: AppTypography.caption,
                      maxLines: 1,
                      overflow: TextOverflow.ellipsis,
                    ),
                  ],
                ),
              ),
              const SizedBox(width: 8),
              const Icon(
                Icons.chevron_right,
                color: AppColors.textSecondary,
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## Phase 6: Screen Checklist (Per Screen)

Before marking any screen as complete:

```
□ @RoutePage() decorator present
□ Route registered in AppRouter + build_runner run
□ BlocProvider wraps the page (not a global provider)
□ Freezed events defined with @freezed
□ Freezed states: initial / loading / loaded / error all handled in UI
□ BLoC registered with @injectable in get_it
□ All images via CachedNetworkImage with memCacheWidth set
□ All lists via ListView.builder (never Column + .map for dynamic data)
□ const used on all static widgets
□ Overflow handled on all Text widgets (maxLines + overflow)
□ Navigation uses context.router, never Navigator.of(context)
□ Only IDs passed through routes (no full objects)
□ All colors from AppColors, all text styles from AppTypography
□ Error state shows actionable message (retry button)
□ Empty state shows appropriate illustration/message
```

---

## Common Stitch → Flutter Translation Errors

### 1. Hardcoded colors
```dart
// ❌ Breaks theming, impossible to maintain
color: Color(0xFF5B4FFF),
// ✅ Single source of truth
color: AppColors.primary,
```

### 2. Text overflow in dynamic content
```dart
// ❌ Crashes with long names
Text(user.displayName),
// ✅ Handles overflow gracefully
Text(
  user.displayName,
  maxLines: 1,
  overflow: TextOverflow.ellipsis,
),
```

### 3. Using Image.network for remote images
```dart
// ❌ No cache, no placeholder, full-resolution decode
Image.network(url, width: 80, height: 80),
// ✅ Cached, sized, with placeholder
CachedNetworkImage(imageUrl: url, width: 80, height: 80, memCacheWidth: 160),
```

### 4. Building large widget trees inside build()
```dart
// ❌ Entire product list rebuilt on every state change
Widget build(BuildContext context) {
  return Column(
    children: products.map((p) => _buildProductCard(p)).toList(),
  );
}
// ✅ Extract to const StatelessWidget
Widget build(BuildContext context) {
  return Column(
    children: products.map((p) => ProductCard(product: p)).toList(),
  );
}
```

### 5. setState mixed with BLoC
```dart
// ❌ Creates parallel state sources
setState(() => _isLoading = true);
// ✅ All state through BLoC
context.read<ProductBloc>().add(const ProductRefreshed());
```