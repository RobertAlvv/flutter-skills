# flutter-skills

AI skills focused on Flutter development — prompts, workflows, and tools designed to help AI agents build, audit, refactor, and maintain Flutter applications efficiently.

## Table of Contents

- [Overview](#overview)
- [Available Skills](#available-skills)
- [Quick Reference](#quick-reference)
- [Usage Examples](#usage-examples)
- [Repository Structure](#repository-structure)
- [Resources](#resources)

---

## Overview

This repository contains two categories of AI-powered Flutter skills:

**Builder skills** — guide an AI agent to implement production-grade Flutter code from scratch, following clean architecture, advanced state management, and industry best practices.

**Auditor skills** — guide an AI agent to perform structured, evidence-based analysis of an existing Flutter codebase, producing a prioritized report with findings and recommendations. Each auditor skill has a focused scope and does not overlap with the others.

---

## Available Skills

### 🏗️ Flutter Scalable App

A senior-level implementation skill for building enterprise-grade Flutter applications.

**Use for:** building new screens or features, setting up architecture from scratch, implementing state management with BLoC/Cubit, configuring Firebase, integrating Stitch/Figma designs into Flutter code.

**References:** [architecture](flutter-scalable-app/references/architecture.md) · [bloc-patterns](flutter-scalable-app/references/bloc-patterns.md) · [dependency-injection](flutter-scalable-app/references/dependency-injection.md) · [firebase](flutter-scalable-app/references/firebase.md) · [navigation](flutter-scalable-app/references/navigation.md) · [performance](flutter-scalable-app/references/performance.md) · [stitch-to-flutter](flutter-scalable-app/references/stitch-to-flutter.md)

---

### 🔍 Architecture Auditor

Audits the overall architectural quality of a Flutter project.

**Use for:** evaluating layer separation, feature modularization, dependency boundaries, state management strategy, and long-term scalability. Produces an Architecture Score (1–10) with a maturity level and prioritized recommendations.

**Does not cover:** runtime performance, widget rebuild cost, state management internals, design system compliance, testability.

**References:** [architecture](architecture-auditor/references/architecture.md) · [bloc-patterns](architecture-auditor/references/bloc-patterns.md) · [dependency-injection](architecture-auditor/references/dependency-injection.md) · [navigation](architecture-auditor/references/navigation.md)

---

### ⚡ Widget Performance Analyzer

Audits widget rebuild behavior, render tree efficiency, layout costs, and scroll performance.

**Use for:** detecting unnecessary rebuilds, expensive `build()` methods, inefficient list implementations, `saveLayer` misuse, InheritedWidget propagation issues, Sliver performance problems, and GC allocation pressure.

**Does not cover:** isolate usage, async efficiency, architecture quality, design system compliance.

**References:** [rebuild-patterns](widget-performance-analyzer/references/rebuild-patterns.md) · [savelayer-and-compositing](widget-performance-analyzer/references/savelayer-and-compositing.md) · [scroll-performance](widget-performance-analyzer/references/scroll-performance.md) · [layout-cost](widget-performance-analyzer/references/layout-cost.md)

---

### 🗂️ State Management Auditor

Audits state management architecture — how state is created, scoped, mutated, and consumed.

**Use for:** detecting rebuild inefficiencies, inconsistent pattern usage across features, state scoping and lifecycle issues, BLoC/Cubit/Riverpod architectural coupling, unification strategy when multiple solutions are mixed.

**Does not cover:** widget rendering performance, architecture layer separation, DI quality, testability.

**References:** [state-patterns](state-management-auditor/references/state-patterns.md) · [scoping-and-lifecycle](state-management-auditor/references/scoping-and-lifecycle.md) · [rebuild-efficiency](state-management-auditor/references/rebuild-efficiency.md) · [isolation-and-coupling](state-management-auditor/references/isolation-and-coupling.md)

---

### 🚀 Runtime Performance Auditor

Audits Dart runtime performance — isolate usage, async efficiency, memoization, caching, and event loop health.

**Use for:** detecting main-thread blocking work, sequential async calls that should be parallel, missing memoization in expensive computations, cache absence for repeated I/O, `scheduleMicrotask` misuse, and `Timer.periodic` event loop saturation.

**Does not cover:** widget rebuild costs, frame budget, architecture quality, state management.

**References:** [isolate-patterns](runtime-performance-auditor/references/isolate-patterns.md) · [async-efficiency](runtime-performance-auditor/references/async-efficiency.md) · [memoization-and-caching](runtime-performance-auditor/references/memoization-and-caching.md) · [event-loop-health](runtime-performance-auditor/references/event-loop-health.md)

---

### 🎨 Atomic Design System Auditor

Audits whether the Flutter UI follows Atomic Design principles and maintains a healthy design system.

**Use for:** detecting missing atomic hierarchy (atoms/molecules/organisms/templates/pages), import direction violations, duplicated UI primitives, raw hardcoded values instead of design tokens, oversized organisms, and design system scalability gaps.

**Does not cover:** runtime performance, architecture quality, state management, testability.

**References:** [atomic-hierarchy](atomic-design-system-auditor/references/atomic-hierarchy.md) · [token-usage](atomic-design-system-auditor/references/token-usage.md) · [component-composition](atomic-design-system-auditor/references/component-composition.md) · [duplication-detection](atomic-design-system-auditor/references/duplication-detection.md)

---

### 🧪 Testability Architecture Auditor

Audits whether the architecture enables scalable, reliable testing across all test types.

**Use for:** detecting architectural barriers to unit tests, bloc tests, widget tests, golden tests, and integration tests — DI coupling, hidden dependencies, business logic in widgets, `BuildContext` in Blocs, non-deterministic widget rendering, missing repository abstractions.

**Does not cover:** runtime performance, widget rebuild cost, design system quality, architecture layer quality (those belong to other auditors).

**References:** [unit-test-architecture](testability-architecture-auditor/references/unit-test-architecture.md) · [bloc-test-patterns](testability-architecture-auditor/references/bloc-test-patterns.md) · [widget-test-patterns](testability-architecture-auditor/references/widget-test-patterns.md) · [golden-test-prerequisites](testability-architecture-auditor/references/golden-test-prerequisites.md)

---

## Quick Reference

### Skill Selection Guide

| Question | Skill to use |
|---|---|
| "Build a screen / feature / integration" | `flutter-scalable-app` |
| "Is our architecture scalable?" | `architecture-auditor` |
| "Why is the UI janky / rebuilding too much?" | `widget-performance-analyzer` |
| "Is our BLoC/Riverpod usage correct?" | `state-management-auditor` |
| "Why is our app slow on heavy data operations?" | `runtime-performance-auditor` |
| "Is our design system consistent?" | `atomic-design-system-auditor` |
| "Can we write unit / widget / golden tests?" | `testability-architecture-auditor` |

### Auditor Scope Matrix

| Concern | Architecture | Widget Perf | State Mgmt | Runtime Perf | Atomic Design | Testability |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Layer separation | ✅ | | | | | |
| Modularization | ✅ | | | | | |
| Widget rebuild cost | | ✅ | | | | |
| Scroll / layout performance | | ✅ | | | | |
| BLoC/Cubit/Riverpod patterns | | | ✅ | | | |
| State rebuild propagation | | | ✅ | | | |
| Isolate / async efficiency | | | | ✅ | | |
| Caching / memoization | | | | ✅ | | |
| Atomic hierarchy | | | | | ✅ | |
| Design token adoption | | | | | ✅ | |
| DI / constructor injection | | | | | | ✅ |
| Bloc test readiness | | | | | | ✅ |
| Widget / golden test readiness | | | | | | ✅ |

---

## Usage Examples

**Build a new feature:**
> "Build a product listing screen using clean architecture with BLoC state management and Firebase Firestore. Include error handling and pagination."
> → use `flutter-scalable-app`

**Audit architecture:**
> "Review the architecture of this Flutter app. Detect layer violations, inconsistent state management, and scalability risks."
> → use `architecture-auditor`

**Diagnose UI jank:**
> "The product list screen drops frames during scroll. Analyze widget rebuild patterns and layout costs."
> → use `widget-performance-analyzer`

**Audit state management:**
> "We use BLoC in some features and Riverpod in others. Audit the state management architecture and recommend a unification strategy."
> → use `state-management-auditor`

**Diagnose slow data operations:**
> "Our app freezes when processing large datasets. Audit isolate usage and async efficiency."
> → use `runtime-performance-auditor`

**Audit design system:**
> "We have duplicate button and text components across features. Audit our atomic design compliance and token usage."
> → use `atomic-design-system-auditor`

**Audit testability:**
> "We can't write unit tests for our Blocs and widget tests always need real repositories. Audit what's blocking us."
> → use `testability-architecture-auditor`

---

## Repository Structure

```
flutter/
├── README.md
├── flutter-scalable-app/
│   ├── SKILL.md
│   └── references/
│       ├── architecture.md
│       ├── bloc-patterns.md
│       ├── dependency-injection.md
│       ├── firebase.md
│       ├── navigation.md
│       ├── performance.md
│       └── stitch-to-flutter.md
├── architecture-auditor/
│   ├── SKILL.md
│   └── references/
│       ├── architecture.md
│       ├── bloc-patterns.md
│       ├── dependency-injection.md
│       └── navigation.md
├── widget-performance-analyzer/
│   ├── SKILL.md
│   └── references/
│       ├── rebuild-patterns.md
│       ├── savelayer-and-compositing.md
│       ├── scroll-performance.md
│       └── layout-cost.md
├── state-management-auditor/
│   ├── SKILL.md
│   └── references/
│       ├── state-patterns.md
│       ├── scoping-and-lifecycle.md
│       ├── rebuild-efficiency.md
│       └── isolation-and-coupling.md
├── runtime-performance-auditor/
│   ├── SKILL.md
│   └── references/
│       ├── isolate-patterns.md
│       ├── async-efficiency.md
│       ├── memoization-and-caching.md
│       └── event-loop-health.md
├── atomic-design-system-auditor/
│   ├── SKILL.md
│   └── references/
│       ├── atomic-hierarchy.md
│       ├── token-usage.md
│       ├── component-composition.md
│       └── duplication-detection.md
└── testability-architecture-auditor/
    ├── SKILL.md
    └── references/
        ├── unit-test-architecture.md
        ├── bloc-test-patterns.md
        ├── widget-test-patterns.md
        └── golden-test-prerequisites.md
```

---

## Resources

- [Flutter Documentation](https://flutter.dev/docs)
- [BLoC Library](https://bloclibrary.dev/)
- [Riverpod](https://riverpod.dev/)
- [GetIt](https://pub.dev/packages/get_it) / [Injectable](https://pub.dev/packages/injectable)
- [auto_route](https://pub.dev/packages/auto_route)
- [Freezed](https://pub.dev/packages/freezed)
- [Firebase Flutter](https://firebase.flutter.dev/)
- [Flutter Performance](https://flutter.dev/perf)
- [Flutter Test](https://docs.flutter.dev/testing)
