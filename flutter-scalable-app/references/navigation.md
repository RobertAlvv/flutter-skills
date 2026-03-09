# Navigation — auto_route 10.x

## Setup

```yaml
dependencies:
  auto_route: ^10.2.2
dev_dependencies:
  auto_route_generator: ^10.2.2
  build_runner: ^2.4.14
```

---

## AppRouter — Full Configuration

```dart
// core/router/app_router.dart
import 'package:auto_route/auto_route.dart';
import 'package:injectable/injectable.dart';
part 'app_router.gr.dart';

@lazySingleton            // register in get_it via injectable
@AutoRouterConfig(replaceInRouteName: 'Page,Route')
class AppRouter extends RootStackRouter {
  @override
  List<AutoRoute> get routes => [
    // Auth flow
    AutoRoute(page: SplashRoute.page, initial: true),
    AutoRoute(page: LoginRoute.page),
    AutoRoute(page: RegisterRoute.page),

    // Main shell with bottom nav
    AutoRoute(
      page: ShellRoute.page,
      children: [
        AutoRoute(page: HomeRoute.page),
        AutoRoute(page: SearchRoute.page),
        AutoRoute(page: ProfileRoute.page),
      ],
    ),

    // Feature routes
    AutoRoute(page: ProductDetailRoute.page),
    AutoRoute(page: CheckoutRoute.page),
  ];

  // Global route guard — all routes pass through here
  @override
  List<AutoRouteGuard> get guards => [getIt<AuthGuard>()];
}
```

---

## Registering a Page

```dart
// features/product/presentation/screen/product_detail_screen.dart
@RoutePage()
class ProductDetailScreen extends StatelessWidget {
  // Path params are auto-detected by auto_route from constructor
  final String productId;

  const ProductDetailPage({
    super.key,
    @PathParam('id') required this.productId,
  });

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => getIt<ProductDetailBloc>()
        ..add(ProductDetailStarted(productId: productId)),
      child: const _ProductDetailView(),
    );
  }
}
```

---

## Navigation Methods

```dart
// Push a route
context.router.push(ProductDetailRoute(productId: '123'));

// Replace current route
context.router.replace(const HomeRoute());

// Replace entire stack (e.g., after login)
context.router.replaceAll([const ShellRoute()]);

// Pop current route
context.router.maybePop();

// Pop with result
context.router.maybePop<bool>(true);

// Navigate to tab index (AutoTabsRouter)
context.tabsRouter.setActiveIndex(2);

// Navigate without context (via get_it)
getIt<AppRouter>().push(const ProfileRoute());
```

**Rule:** Always pass only IDs, never full objects:
```dart
// ✅ Correct
context.router.push(ProductDetailRoute(productId: product.id));

// ❌ Incorrect — object is not serializable, breaks deep links
context.router.push(ProductDetailRoute(product: product));
```

---

## Route Guards

```dart
// core/router/guards/auth_guard.dart
@injectable
class AuthGuard extends AutoRouteGuard {
  final AuthRepository _auth;
  AuthGuard(this._auth);

  @override
  void onNavigation(NavigationResolver resolver, StackRouter router) {
    final isAuthenticated = _auth.currentUser != null;
    if (isAuthenticated) {
      resolver.next(true);
    } else {
      // Redirect to login, then continue original navigation
      router.push(LoginRoute(onLoginCallback: resolver.next));
    }
  }
}
```

Apply guards per route (not globally):
```dart
AutoRoute(
  page: ProfileRoute.page,
  guards: [AuthGuard()],  // injectable instance via get_it
),
```

---

## Tab Navigation with AutoTabsScaffold

```dart
@RoutePage()
class ShellPage extends StatelessWidget {
  const ShellPage({super.key});

  @override
  Widget build(BuildContext context) {
    return AutoTabsScaffold(
      routes: const [
        HomeRoute(),
        SearchRoute(),
        ProfileRoute(),
      ],
      bottomNavigationBuilder: (_, tabsRouter) {
        return BottomNavigationBar(
          currentIndex: tabsRouter.activeIndex,
          onTap: tabsRouter.setActiveIndex,
          items: const [
            BottomNavigationBarItem(
              icon: Icon(Icons.home_outlined),
              activeIcon: Icon(Icons.home),
              label: 'Home',
            ),
            BottomNavigationBarItem(
              icon: Icon(Icons.search_outlined),
              activeIcon: Icon(Icons.search),
              label: 'Search',
            ),
            BottomNavigationBarItem(
              icon: Icon(Icons.person_outline),
              activeIcon: Icon(Icons.person),
              label: 'Profile',
            ),
          ],
        );
      },
    );
  }
}
```

---

## Deep Links — Android & iOS Setup

### Android (`android/app/src/main/AndroidManifest.xml`)
```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="https" android:host="myapp.com" />
</intent-filter>
```

### iOS (`ios/Runner/Info.plist`)
```xml
<key>FlutterDeepLinkingEnabled</key>
<true/>
```

### Router config with deep link transformer
```dart
routerConfig: getIt<AppRouter>().config(
  deepLinkBuilder: (deepLink) {
    // Custom deep link handling if needed
    return deepLink;
  },
),
```

---

## Nested Navigation — Feature Sub-Stacks

```dart
// When a feature has its own navigation stack
@RoutePage()
class OnboardingPage extends StatelessWidget {
  const OnboardingPage({super.key});

  @override
  Widget build(BuildContext context) {
    // AutoRouter renders the current child route
    return const AutoRouter();
  }
}

// Router config
AutoRoute(
  page: OnboardingRoute.page,
  children: [
    AutoRoute(page: OnboardingStep1Route.page, initial: true),
    AutoRoute(page: OnboardingStep2Route.page),
    AutoRoute(page: OnboardingStep3Route.page),
  ],
),
```

---

## Custom Page Transitions

```dart
@RoutePage()
class ModalPage extends StatelessWidget {
  const ModalPage({super.key});

  @override
  Widget build(BuildContext context) => const _ModalView();
}

// In AppRouter routes:
CustomRoute(
  page: ModalRoute.page,
  transitionsBuilder: TransitionsBuilders.slideBottom,
  durationInMilliseconds: 300,
),
```

---

## Route-Aware Widget Lifecycle

```dart
@RoutePage()
class VideoPage extends StatefulWidget implements AutoRouteAware {
  const VideoPage({super.key});

  @override
  State<VideoPage> createState() => _VideoPageState();
}

class _VideoPageState extends State<VideoPage> with AutoRouteAware {
  @override
  void didInitTabRoute(TabPageRoute? previousRoute) {
    // Tab became active — start video
  }

  @override
  void didChangeTabRoute(TabPageRoute previousRoute) {
    // Tab switched away — pause video
  }
}
```