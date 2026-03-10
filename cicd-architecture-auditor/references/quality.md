# Quality Gate Configuration Reference

This reference provides SonarQube wiring, lcov configuration, and Dart analyzer placement patterns used in Step 5 of the audit. Use it when the user's coverage reports are incorrect, quality gate configuration is misconfigured, or analyzer output is not reaching Sonar.

---

## SonarQube Job Requirements

SonarQube analysis has specific requirements that differ from all other jobs in the pipeline. These requirements must be applied to the Sonar job — and **only** to the Sonar job.

| Requirement | YAML Setting | Reason |
|---|---|---|
| Full git history | `fetch-depth: 0` on `actions/checkout` | SCM blame, new code period detection, PR decoration |
| Java runtime | `actions/setup-java` | `sonar-scanner` CLI requires JRE 11+ |
| Merged `lcov.info` | Downloaded from all test shard artifacts | Sonar must receive complete coverage data |
| `SONAR_TOKEN` | Step-level `env:` | Scoped to scanner step only |

Misplace any of these in a test or build job and you either waste time (JRE in a test job adds 30–90 sec) or produce incorrect data (partial coverage from one shard).

---

## Sonar Job YAML Template

```yaml
jobs:
  sonar:
    runs-on: ubuntu-latest
    needs: [test]              # Must wait for all test jobs
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0       # Full history — Sonar job only

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          cache: true

      - uses: actions/cache@v4
        with:
          path: ~/.pub-cache
          key: pub-${{ runner.os }}-${{ hashFiles('**/pubspec.lock') }}
          restore-keys: pub-${{ runner.os }}-

      - run: flutter pub get

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - uses: actions/download-artifact@v4
        with:
          pattern: coverage-shard-*
          path: coverage/shards
          merge-multiple: false

      - name: Install lcov and merge shards
        run: |
          sudo apt-get install -y lcov
          lcov $(find coverage/shards -name '*.info' | sort | sed 's/^/-a /') \
            -o coverage/merged.info
          lcov --remove coverage/merged.info \
            '**/*.g.dart' \
            '**/*.freezed.dart' \
            '**/*.mocks.dart' \
            -o coverage/merged.info

      - name: Run Dart analyzer (non-blocking)
        run: flutter analyze || true
        # || true is intentional — analyzer output feeds Sonar; Sonar decides pass/fail

      - name: Run SonarQube scanner
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        run: sonar-scanner
```

---

## sonar-project.properties Reference

Place this file at the repository root. Adjust paths for monorepo layouts.

```properties
# Required identification
sonar.projectKey=com.example.myapp
sonar.projectName=My Flutter App
sonar.projectVersion=1.0

# Source and test paths
sonar.sources=lib
sonar.tests=test
sonar.sourceEncoding=UTF-8

# Coverage report path (merged from all shards)
sonar.dart.coverage.reportPaths=coverage/merged.info

# Exclude generated code from analysis
sonar.exclusions=**/*.g.dart,**/*.freezed.dart,**/*.mocks.dart,**/generated/**
sonar.coverage.exclusions=**/*.g.dart,**/*.freezed.dart,**/*.mocks.dart

# For monorepo with packages/
# sonar.sources=lib,packages/package_a/lib,packages/package_b/lib
# sonar.tests=test,packages/package_a/test,packages/package_b/test
```

---

## Coverage Path Correction for Monorepo

When packages live under `packages/`, the relative paths in `lcov.info` may not match what Sonar expects. Add path normalization after the merge:

```bash
# Normalize paths so Sonar can map coverage back to source files
sed -i 's|SF:packages/package_a/|SF:|g' coverage/merged.info
```

Or configure `sonar.dart.coverage.reportPaths` to point to individual package lcov files:

```properties
sonar.dart.coverage.reportPaths=coverage/merged.info,packages/package_a/coverage/lcov.info
```

---

## Dart Analyzer Placement Decision Table

| Scenario | Placement | Reason |
|---|---|---|
| Standalone PR quality check | Separate parallel job | Does not block tests; runs concurrently |
| Input to SonarQube issue detection | Inside Sonar job with `\|\| true` | Sonar reads the output; Sonar decides pass/fail |
| Blocking gate before merge | Separate `analyze` job, `needs:` from merge job | Explicit quality gate; fails pipeline if violations found |
| Part of monorepo per-package check | Matrix job over packages | Each package analyzed independently |

Never run `flutter analyze` in a test job. It adds 30–90 sec to the test job critical path for work that can run in parallel.

---

## Quality Gate Misconfiguration Signals

| Observation | Root Cause | Fix |
|---|---|---|
| Sonar shows 15–25% coverage | Only one shard's `lcov.info` uploaded | Add `actions/upload-artifact` to every shard, merge before Sonar |
| Sonar shows 0% coverage | `lcov.info` path incorrect in `sonar-project.properties` | Confirm `sonar.dart.coverage.reportPaths` matches actual output path |
| `lcov: command not found` in merge step | lcov not installed before use | Add `sudo apt-get install -y lcov` before merge step |
| `sonar-scanner: not found` | sonar-scanner CLI not in PATH | Install via `apt-get` or use `SonarSource/sonarcloud-github-action` |
| `NO_CHANGES_SINCE` error in Sonar | `fetch-depth` not 0 on Sonar job | Add `fetch-depth: 0` to checkout in the Sonar job |
| Generated file hits inflate coverage | `*.g.dart` not excluded | Add `lcov --remove` for generated patterns before Sonar step |
