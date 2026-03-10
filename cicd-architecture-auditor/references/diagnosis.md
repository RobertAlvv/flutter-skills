# Pipeline Diagnosis Reference

This reference provides the anti-pattern catalog, severity classification, and wall-clock time estimation baselines used in Steps 1–3 of the audit. Use it when the user shares a workflow YAML or describes pipeline slowness or intermittent failures.

---

## Anti-Pattern Catalog

### Topology Anti-Patterns

| Anti-Pattern | Symptom | Wall-Clock Cost | Severity |
|---|---|---|---|
| Single monolithic job | All steps from checkout to Sonar in one job | Sum of all steps with no parallelism | HIGH |
| Test and build in same job | Build waits for all tests to complete | Android build adds 3–6 min to test critical path | HIGH |
| Linear `needs:` chain | `A → B → C` instead of `[A, B] → C` | Each chain link adds full job duration to critical path | MEDIUM |
| No `concurrency:` group | Stale PR workflows run to completion | Wastes runner minutes on superseded pushes | LOW |

### Caching Anti-Patterns

| Anti-Pattern | Symptom | Wall-Clock Cost | Severity |
|---|---|---|---|
| No pub cache (`~/.pub-cache`) | `flutter pub get` downloads from pub.dev every run | +1–3 min per job | HIGH |
| `flutter clean` before tests | Destroys `.dart_tool` that pub get just populated | +1–3 min of re-download on next step | HIGH |
| Static cache key | Key never invalidates — stale packages or always misses | Either stale behavior or no cache benefit | MEDIUM |
| No Gradle cache | Android build re-downloads all Gradle dependencies | +2–4 min per Android build job | MEDIUM |
| No CocoaPods cache | `pod install` runs from scratch on every macOS runner | +3–6 min per iOS build job | MEDIUM |

### Toolchain Anti-Patterns

| Anti-Pattern | Symptom | Wall-Clock Cost | Severity |
|---|---|---|---|
| Dual `build_runner` invocation | Both `dart run build_runner` and `flutter packages pub run build_runner` | 4–10 min wasted (full second code-gen run) | HIGH |
| Java setup in test job | `actions/setup-java` before `flutter test` | +30–90 sec before tests start | MEDIUM |
| `flutter pub get` called twice | Same job re-fetches pub packages | +30–90 sec | MEDIUM |
| `flutter analyze` in test job | Analyze blocks tests from starting | +30–90 sec (could run in parallel) | MEDIUM |

---

## Wall-Clock Time Estimation Baselines

Use these baselines to estimate current vs. optimized pipeline duration when the user cannot provide actual run times.

### Setup Phase (per job)

| Operation | ubuntu-latest | macos-latest |
|---|---|---|
| `actions/checkout@v4` | 10–20 sec | 15–30 sec |
| `subosito/flutter-action` (cold) | 2–4 min | 3–5 min |
| `subosito/flutter-action` (warm cache) | 15–30 sec | 20–40 sec |
| `flutter pub get` (cold) | 1–3 min | 1–3 min |
| `flutter pub get` (warm cache) | 10–20 sec | 10–20 sec |
| `dart run build_runner` | 2–8 min | 2–8 min |
| `actions/setup-java` | 30–60 sec | 45–90 sec |

### Test Phase

| Test Volume | Single Runner | 2 Shards | 4 Shards | 8 Shards |
|---|---|---|---|---|
| 1,000 tests | 2–4 min | 1–2 min | 1 min | 1 min |
| 5,000 tests | 8–15 min | 4–8 min | 2–4 min | 1–2 min |
| 10,000 tests | 15–25 min | 8–13 min | 4–7 min | 2–4 min |
| 20,000 tests | 30–50 min | 15–25 min | 8–13 min | 4–7 min |

Shard count recommendation: aim for 3–5 minutes per shard. Diminishing returns above 8 shards due to setup overhead per runner.

### Build Phase

| Build Target | Runner | Cold | With Cache |
|---|---|---|---|
| `flutter build apk --release` | ubuntu-latest | 4–7 min | 2–4 min |
| `flutter build appbundle --release` | ubuntu-latest | 4–7 min | 2–4 min |
| `flutter build ipa --release` | macos-latest | 8–15 min | 5–10 min |
| `flutter build web --release` | ubuntu-latest | 2–4 min | 1–2 min |

### Quality Gate Phase

| Operation | Duration |
|---|---|
| `sonar-scanner` | 2–5 min |
| `lcov` merge (4 shards) | 15–30 sec |
| `dart analyze` / `flutter analyze` | 30–90 sec |
| Coverage artifact download (4 shards) | 15–30 sec |

---

## Critical Path Analysis Method

The critical path is the longest sequence of dependent steps in the job dependency graph. Optimizing a step that is NOT on the critical path has zero wall-clock impact.

**Steps:**
1. Draw the job dependency graph from `needs:` declarations
2. For each job, sum the step durations (use baselines above if actual data unavailable)
3. Walk the longest path from start to the final job
4. Report current critical path duration and which jobs are on it
5. Show critical path duration after applying all recommendations

**Example:**

```
Current (sequential):
  ci: [setup(3m) + build_runner(5m) + tests(20m) + build(6m) + sonar(3m)] = 37 min

Optimized (parallel):
  test-app: [setup(1m) + tests_sharded(5m)] = parallel critical path 6 min
  build:    [setup(1m) + build(4m)]        = parallel
  sonar:    needs [test, build] + [download(0.5m) + merge(0.3m) + scan(3m)] = 6 + 3.8 = ~10 min

  Wall-clock = ~10 min (73% reduction)
```
