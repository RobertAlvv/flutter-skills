# Component Composition Reference

## Page Composition Rules

A Page is the only atomic level that may:
1. Instantiate `BlocProvider`, `RepositoryProvider`, `Provider`, or `ProviderScope`
2. Call navigation APIs (`context.go()`, `Navigator.push()`)
3. Listen to state for side effects via `BlocListener`

Everything else is delegated downward.

### Target Page Structure

```dart
// lib/features/auth/pages/login_page.dart
class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => LoginBloc(
        authRepository: context.read<AuthRepository>(),
      ),
      child: BlocListener<LoginBloc, LoginState>(
        listener: (context, state) {
          if (state is LoginSuccess) context.go('/home');
          if (state is LoginFailure) ScaffoldMessenger.of(context)
              .showSnackBar(SnackBar(content: Text(state.message)));
        },
        child: const _LoginView(), // delegates layout to template + organism
      ),
    );
  }
}

class _LoginView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocBuilder<LoginBloc, LoginState>(
      builder: (context, state) => AuthFlowTemplate( // template handles scaffold
        body: LoginFormOrganism(              // organism handles form structure
          isLoading: state is LoginLoading,
          emailError: state is LoginValidationError ? state.emailError : null,
          onEmailChanged: (v) => context.read<LoginBloc>().add(EmailChanged(v)),
          onPasswordChanged: (v) => context.read<LoginBloc>().add(PasswordChanged(v)),
          onSubmit: () => context.read<LoginBloc>().add(LoginSubmitted()),
        ),
      ),
    );
  }
}
```

### Page Size Threshold

| Page `build()` method length | Verdict |
|---|---|
| < 50 lines | Ideal — mostly wiring and provider setup |
| 50–100 lines | Acceptable — may have multiple `BlocListener` chains |
| 100–150 lines | MEDIUM — some inline layout is leaking into the page |
| > 150 lines | HIGH — page is acting as organism; extract layout to organisms/templates |

---

## Template Slot Pattern

Templates define layout structure through named `Widget` parameters. They have zero knowledge of what content fills the slots.

```dart
// lib/ui/templates/feed_page_template.dart
class FeedPageTemplate extends StatelessWidget {
  final Widget header;
  final Widget body;
  final Widget? floatingAction;
  final List<Widget> persistentFooterButtons;

  const FeedPageTemplate({
    super.key,
    required this.header,
    required this.body,
    this.floatingAction,
    this.persistentFooterButtons = const [],
  });

  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: PreferredSize(
      preferredSize: const Size.fromHeight(56),
      child: header,
    ),
    body: SafeArea(child: body),
    floatingActionButton: floatingAction,
    persistentFooterButtons: persistentFooterButtons,
  );
}
```

**Template violations to flag:**

```dart
// VIOLATION — template contains conditional content based on user type
class DashboardTemplate extends StatelessWidget {
  final bool isAdmin; // template should not know about roles
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(children: [
        if (isAdmin) AdminPanelOrganism(), // feature-aware — MEDIUM violation
        MainContentSlot(),
      ]),
    );
  }
}

// CORRECT — template accepts slots; caller decides what to inject
class DashboardTemplate extends StatelessWidget {
  final Widget? topBanner; // nullable slot — caller passes AdminPanel or null
  final Widget body;
  ...
}
```

---

## Organism Decomposition Thresholds

| Organism `build()` lines | Action |
|---|---|
| < 100 lines | Acceptable |
| 100–150 lines | Review for decomposition opportunity |
| 150–200 lines | MEDIUM — likely contains 2+ distinct sections; extract molecules |
| > 200 lines | MEDIUM–HIGH — confirm whether a distinct section can become its own organism or molecule |
| > 400 lines | HIGH — multiple responsibilities confirmed; mandatory decomposition |

### Decomposition Decision Rule

Split an organism when any two of these are true:
1. The organism has two distinct visual sections (e.g., a header area and an action area)
2. Either section appears in at least one other screen in the app
3. The organism `build()` exceeds 150 lines

### Decomposition Example

```dart
// BEFORE — UserProfileOrganism doing too much
class UserProfileOrganism extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(children: [
      // section 1: user identity (appears also in chat, comments)
      Row(children: [CircleAvatar(...), Column(children: [Text(user.name), Text(user.handle)])]),
      // section 2: stats (unique to profile)
      Row(children: [StatCounter('Posts', user.postCount), StatCounter('Followers', user.followerCount)]),
      // section 3: action buttons (appears also in user search results)
      Row(children: [FollowButton(isFollowing: user.isFollowing, onTap: onFollowTap), MessageButton(onTap: onMessageTap)]),
    ]);
  }
}

// AFTER — decomposed into 3 molecules
class UserProfileOrganism extends StatelessWidget {
  @override
  Widget build(BuildContext context) => Column(children: [
    UserIdentityMolecule(name: user.name, handle: user.handle, avatarUrl: user.avatarUrl),
    UserStatsMolecule(postCount: user.postCount, followerCount: user.followerCount),
    UserActionsMolecule(isFollowing: user.isFollowing, onFollow: onFollowTap, onMessage: onMessageTap),
  ]);
}
```

---

## BlocProvider Placement Rules

| Location | Verdict | Rationale |
|---|---|---|
| Inside `Page.build()` | Correct | Page is the wiring layer |
| Inside a named `_View` private widget within a page file | Correct | Organizational refinement; still within the page's scope |
| Inside an organism | HIGH violation | Organism must be usable in tests without DI setup |
| Inside a molecule | HIGH violation | Molecules must be feature-agnostic |
| Inside a template | HIGH violation | Templates must be pure layout |
| Inside `main.dart` / `App` for global concerns | Correct | `AuthBloc`, `ThemeBloc` — legitimately app-scoped |

---

## Navigation in Atomic Design

Navigation belongs exclusively in **pages** and **app-level routing**. It must never appear inside templates, organisms, or molecules.

```dart
// VIOLATION — organism handles navigation directly
class ProductCardOrganism extends StatelessWidget {
  @override
  Widget build(BuildContext context) => GestureDetector(
    onTap: () => Navigator.push(       // navigation from organism — MEDIUM violation
      context,
      MaterialPageRoute(builder: (_) => ProductDetailPage(id: product.id)),
    ),
    child: ...,
  );
}

// CORRECT — organism exposes callback; page decides what navigation to trigger
class ProductCardOrganism extends StatelessWidget {
  final VoidCallback onTap; // caller controls navigation

  @override
  Widget build(BuildContext context) => GestureDetector(
    onTap: onTap,
    child: ...,
  );
}

// In the page:
ProductCardOrganism(
  onTap: () => context.go('/products/${product.id}'),
)
```
