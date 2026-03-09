# flutter-skills

AI skills focused on Flutter development — prompts, workflows, and tools designed to help AI agents build, audit, refactor, and maintain Flutter applications efficiently.

## Table of Contents

- [Overview](#overview)
- [Available Skills](#available-skills)
- [Quick Reference](#quick-reference)
- [Usage Examples](#usage-examples)
- [Repository Structure](#repository-structure)
- [Resources](#resources)

## Overview

This repository contains a collection of AI-powered Flutter development skills. Each skill is a comprehensive guide that combines architectural patterns, best practices, code examples, and configuration recommendations to help AI agents (and developers) build production-grade Flutter applications.

The **`flutter-scalable-app`** skill is a complete senior-level engineer's guide to building scalable Flutter applications using clean architecture, advanced state management, and industry best practices.

## Available Skills

### 🎯 Flutter Scalable App

A comprehensive skill for building enterprise-grade Flutter applications with production-ready architecture.

**Use this skill for:**
- Building new Flutter projects from scratch
- Implementing features using clean architecture (domain/data/presentation layers)
- Setting up complex state management with BLoC and Cubit
- Configuring Firebase authentication and real-time databases
- Optimizing app performance and user experience
- Integrating UI designs (Stitch/Figma) into Flutter code
- Setting up dependency injection with get_it + injectable

**Includes:**
- Complete architectural patterns and folder structure
- State management patterns (BLoC vs Cubit decision matrix)
- Dependency injection setup and configuration
- Firebase integration (Auth, Firestore CRUD, Storage, Cloud Functions)
- Navigation with auto_route (routing, guards, deep linking, tab navigation)
- Performance optimization techniques and DevTools profiling
- Translation guide: from Stitch designs to Flutter implementation

---

## Quick Reference

### Reference Guides

| Reference | Topic | Best For |
|-----------|-------|----------|
| [Architecture](skills/flutter-scalable-app/references/architecture.md) | Project structure, layers, folder organization | Understanding the clean architecture setup |
| [BLoC Patterns](skills/flutter-scalable-app/references/bloc-patterns.md) | State management, BLoC vs Cubit, events, states | Implementing state management |
| [Dependency Injection](skills/flutter-scalable-app/references/dependency-injection.md) | get_it, @injectable, modules, scopes | Setting up DI in your project |
| [Firebase Integration](skills/flutter-scalable-app/references/firebase.md) | Auth, Firestore, Storage, Cloud Functions | Implementing backend services |
| [Navigation](skills/flutter-scalable-app/references/navigation.md) | auto_route setup, routing guards, deep linking | Configuring app navigation |
| [Performance Optimization](skills/flutter-scalable-app/references/performance.md) | const widgets, BlocSelector, ListViews, profiling | Optimizing app performance |
| [Stitch to Flutter](skills/flutter-scalable-app/references/stitch-to-flutter.md) | Design tokens, routes, layouts, widgets | Translating designs into code |

### Key Versions Specified

- **Flutter:** 3.27+ with Dart 3.6+
- **State Management:** `flutter_bloc: ^9.1.0`, `cubit: ^9.1.0`
- **Navigation:** `auto_route: ^10.2.2`
- **DI:** `get_it: ^8.0.3`, `injectable: ^2.5.0`
- **Code Generation:** `freezed: ^2.5.8`, `build_runner: ^2.4.0`
- **Backend:** `firebase_core: ^3.13.0`, `cloud_firestore: ^5.4.0`

---

## Usage Examples

### Example 1: Build a Flutter Screen in Clean Architecture
**Prompt:** "Build a user profile screen using clean architecture with BLoC state management. Include Firebase Firestore data loading and error handling."

**What the skill provides:**
- Domain layer: Use cases and repository abstractions
- Data layer: Firebase implementation and models
- Presentation layer: BLoC event/state definitions and UI
- Dependency injection setup

### Example 2: Set Up Firebase Authentication
**Prompt:** "Set up Firebase authentication with sign-up, login, and logout using get_it for dependency injection. Include error handling and user session management."

**What the skill provides:**
- Firebase configuration steps
- Authentication repository pattern
- BLoC for auth state management
- get_it module registration
- Session persistence strategies

### Example 3: Optimize a List View
**Prompt:** "Optimize a large list of products. The list shows images, titles, and prices. Currently experiencing frame drops during scroll."

**What the skill provides:**
- `ListView.builder` implementation (lazy loading)
- `const` widget optimization
- BlocSelector to rebuild only affected items
- Image caching strategies
- DevTools profiling guidance

### Example 4: Translate a Stitch Design to Flutter
**Prompt:** "Convert this Stitch design into Flutter. Create the navigation structure, extract design tokens, and build reusable components."

**What the skill provides:**
- Design token extraction process
- Route and navigation structure setup
- Custom widget creation from design components
- Responsive layout patterns
- Theme integration

---

## Technical Requirements

### Minimum Environment
- **Flutter:** 3.27 or higher
- **Dart:** 3.6 or higher
- **macOS/Linux/Windows** with Flutter SDK installed
- **IDE:** VS Code, Android Studio, or IntelliJ

### Required Packages (pubspec.yaml)

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_bloc: ^9.1.0
  auto_route: ^10.2.2
  get_it: ^8.0.3
  freezed_annotation: ^2.4.0
  firebase_core: ^3.13.0
  cloud_firestore: ^5.4.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner: ^2.4.0
  freezed: ^2.5.8
  injectable_generator: ^2.5.0
  auto_route_generator: ^10.2.0
```

### Knowledge Prerequisites
- Basic Flutter and Dart understanding
- Familiarity with state management concepts
- Experience with async/await patterns
- Understanding of Object-Oriented Programming

---

## Repository Structure

```
flutter-skills/
├── README.md                           # This file
├── skills/                             # Collection of AI skills
│   └── flutter-scalable-app/
│       ├── SKILL.md                    # Skill definition and metadata
│       └── references/                 # Comprehensive reference guides
│           ├── architecture.md         # Clean architecture patterns
│           ├── bloc-patterns.md        # State management guide
│           ├── dependency-injection.md # DI setup with get_it
│           ├── firebase.md             # Firebase integration
│           ├── navigation.md           # auto_route routing
│           ├── performance.md          # Optimization techniques
│           └── stitch-to-flutter.md    # Design-to-code workflow
└── .github/
    └── workflows/
        └── validate-skills.yml         # CI/CD validation pipeline
```

### What Each Directory Contains

- **`skills/`** — AI skill definitions with complete documentation
- **`references/`** — Deep-dive guides on specific topics with code examples
- **`.github/workflows/`** — CI/CD pipelines for validating skill completeness

---

## Resources

### Official Documentation
- [Flutter Documentation](https://flutter.dev/docs)
- [BLoC Library Docs](https://bloclibrary.dev/)
- [GetIt Package](https://pub.dev/packages/get_it)
- [auto_route Package](https://pub.dev/packages/auto_route)
- [Firebase Flutter Docs](https://firebase.flutter.dev/)
- [Freezed Code Generation](https://pub.dev/packages/freezed)

### Learning Resources
- [Flutter Clean Architecture](https://resocoder.com/clean-architecture-tdd)
- [BLoC Pattern Explanation](https://www.youtube.com/playlist?list=PLAXnLdK94zTxZ7tMwkWWGV6iRFxdQGiqG)
- [Firebase Best Practices](https://firebase.google.com/docs/firestore/best-practices)
- [Flutter Performance Tips](https://flutter.dev/perf)

---

## Contributing

When adding new skills or references:
1. Follow the existing structure and naming conventions
2. Include comprehensive code examples (copy-paste ready)
3. Document anti-patterns with ❌ vs ✅ comparisons
4. Add a checklist for implementation validation
5. Update this README with links to new resources

---

## License

These AI skills are provided as-is for development and learning purposes. Always refer to official Flutter and package documentation for the latest best practices.
