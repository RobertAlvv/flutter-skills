# Architecture Reference — Ideal Flutter Project Structure

Use this file to calibrate audit findings. Compare the audited project against these patterns to determine severity of architectural violations.

---

## Core Dependency Rule

Dependencies only point **inward**. This rule must never be violated:

```
presentation  →  domain  ←  data
                   ↑
                  core (available to all layers)
```

| Layer | May import | Must NOT import |
|---|---|---|
| `presentation` | `domain`, `core` | `data` directly |
| `domain` | `core/error`, `core/utils` | `presentation`, `data`, Flutter SDK, Firebase |
| `data` | `domain`, `core` | `presentation` |
| `core` | external packages | any `feature` |

---

## Ideal Project Tree

```
lib/
├── app/                        # App entry point, MaterialApp.router
├── core/                       # Shared infrastructure — no feature logic
│   ├── di/                     # GetIt + Injectable wiring
│   ├── router/                 # Centralized route config + guards
│   ├── network/                # Dio client + interceptors
│   ├── error/                  # Failure hierarchy (Equatable)
│   ├── theme/                  # AppTheme, AppColors, AppTypography
│   └── utils/                  # NoParams, validators, extensions
└── features/
    └── [feature_name]/
        ├── data/
        │   ├── datasource/         # API / local DB access — works with Model
        │   ├── model/              # Extends entity, adds toJson/fromJson
        │   └── repository_impl/    # Implements domain interface, maps exceptions → Failure
        ├── domain/
        │   ├── entity/             # Pure Dart + Equatable — zero external deps
        │   ├── repository/         # abstract interface — no implementation
        │   └── use_case/           # One file per use case, @injectable
        └── presentation/
            ├── screen/             # @RoutePage() + BlocProvider — no UI logic
            ├── bloc/               # event + state + bloc (@injectable, @freezed)
            ├── cubit/              # (optional) for simple linear state
            ├── view/               # Scaffold + BlocBuilder (optional split)
            └── widgets/            # Reusable UI components scoped to feature
```

---

## Architecture Patterns — Ranked by Scalability

| Pattern | Appropriate for | Scalability |
|---|---|---|
| Clean Architecture (feature-first) | Teams of 3+, growing product | ★★★★★ |
| Feature-driven (without full Clean) | Teams of 2–3, stable scope | ★★★★☆ |
| Layer-first (presentation/data/domain at root) | Single developer, small app | ★★★☆☆ |
| MVC / MVVM without layers | Prototypes | ★★☆☆☆ |
| Unstructured (flat screens/ widgets/) | Throwaway code only | ★☆☆☆☆ |

---

## Violation Severity Guide

### HIGH — fundamental separation broken

- `Widget` or `BuildContext` imported inside `domain/` or `data/` files
- Direct `http.get()` or `dio.post()` calls inside widget `build()` or `initState()`
- Repository implementations placed inside `presentation/`
- Feature importing directly from another feature (`features/a/` → `features/b/`)
- No `domain/repository/` interfaces — Bloc calls datasource directly

### MEDIUM — maintainability reduced

- Use cases skipped — Bloc calls repository directly without use case
- Domain entities contain `toJson`/`fromJson` (serialization belongs in `model/`)
- State management mixed across features (some BLoC, some `setState`)
- No `core/error/` — exceptions handled ad-hoc per feature with raw try/catch

### LOW — code quality and consistency

- Generic catch-all folders at feature level (`utils/`, `helpers/`, `common/`)
- Missing `core/router/` — routing defined inline in widget trees
- Inconsistent naming conventions across similar files
- `data/` and `presentation/` layers present but `domain/` layer absent

---

## Key Files to Inspect During Audit

| File / Pattern | What to look for |
|---|---|
| `pubspec.yaml` | State management, DI, routing packages declared |
| `lib/core/di/injection.dart` | Presence of DI wiring (`GetIt`, `Injectable`) |
| `lib/core/router/app_router.dart` | Centralized, named routing |
| `lib/core/error/failures.dart` | Typed `Failure` hierarchy vs raw exceptions |
| `lib/features/*/domain/repository/*.dart` | Abstract interfaces (correct) vs missing (violation) |
| `lib/features/*/data/datasource/*.dart` | Isolation of network calls |
| `lib/features/*/presentation/bloc/*.dart` | State management placement and coupling |
| Any file with `import 'package:http` or `import 'package:dio'` | Verify which layer it lives in |
| Any file with `import 'package:flutter/material.dart'` in `domain/` or `data/` | HIGH violation |

---

## Naming Conventions — Expected Signals

Consistent naming is a proxy for architectural discipline. Check for:

| Type | Expected pattern | Example |
|---|---|---|
| Entity | `*Entity` | `ProductEntity` |
| Model (DTO) | `*Model` | `ProductModel` |
| Repository interface | `*Repository` | `ProductRepository` |
| Repository impl | `*RepositoryImpl` | `ProductRepositoryImpl` |
| Datasource interface | `*Datasource` | `ProductDatasource` |
| Datasource impl | `*DatasourceImpl` | `ProductDatasourceImpl` |
| Use case | `Get*UseCase`, `Create*UseCase` | `GetProductsUseCase` |
| Bloc | `*Bloc`, `*Event`, `*State` | `ProductBloc` |
| Cubit | `*Cubit`, `*State` | `CheckoutStepCubit` |
| Screen | `*Screen` | `ProductScreen` |

Presence of these patterns = architecture is intentional.  
Absence or inconsistency = architecture is improvised.
