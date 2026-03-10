# Test Sharding and Coverage Reference

This reference provides calibration tables, YAML templates, and lcov merge patterns used in Steps 4 and 5 of the audit. Use it when the user's tests are slow, sharding needs to be configured, or coverage data is incomplete or missing from Sonar.

---

## Native Sharding Prerequisites

Flutter's native `--total-shards` / `--shard-index` flags are available in Flutter ≥ 3.7. Always confirm the Flutter version before recommending sharding.

```bash
# Confirm Flutter version supports sharding
grep -n "flutter-version:" .github/workflows/ci.yml
# Must be 3.7.0 or later
```

If the version is below 3.7, recommend upgrading rather than using third-party sharding packages.

---

## Shard Count Calibration Table

Target 3–5 minutes of test execution per shard. Setup overhead per runner (checkout, flutter-action, pub get) adds ~1–2 minutes per shard regardless of shard count — diminishing returns above 8 shards.

| Test Volume | Recommended Shards | Target Duration per Shard |
|---|---|---|
| < 1,000 tests | 1 (no sharding needed) | — |
| 1,000–3,000 tests | 2 | 2–4 min |
| 3,000–8,000 tests | 4 | 2–4 min |
| 8,000–15,000 tests | 6 | 3–5 min |
| 15,000–25,000 tests | 8 | 3–5 min |
| > 25,000 tests | 10–12 | 3–5 min |

---

## Sharding YAML Templates

### Basic 4-shard template

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false          # REQUIRED — prevents one failure from cancelling siblings
      matrix:
        shard: [0, 1, 2, 3]
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          channel: stable
          cache: true
      - uses: actions/cache@v4
        with:
          path: |
            ~/.pub-cache
            .dart_tool
          key: pub-${{ runner.os }}-${{ hashFiles('**/pubspec.lock') }}
          restore-keys: pub-${{ runner.os }}-
      - run: flutter pub get
      - run: dart run build_runner build --delete-conflicting-outputs
      - run: |
          flutter test \
            --total-shards=4 \
            --shard-index=${{ matrix.shard }} \
            --coverage \
            --reporter compact
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-shard-${{ matrix.shard }}
          path: coverage/lcov.info
          retention-days: 1      # Short retention — only needed for merge job
```

### Dynamic shard count (parametrized)

When the team wants to adjust shard count without editing the matrix manually:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [0, 1, 2, 3, 4, 5]    # 6 shards for mid-size suite
    steps:
      - run: |
          flutter test \
            --total-shards=6 \
            --shard-index=${{ matrix.shard }} \
            --coverage \
            --reporter compact
```

Important: the number in `--total-shards` must always equal the number of values in the `matrix.shard` list.

---

## Coverage Merge Patterns

### Standard lcov merge (4 shards → Sonar)

```yaml
jobs:
  sonar:
    runs-on: ubuntu-latest
    needs: [test]
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0              # Required for SCM blame — Sonar job only

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

      - uses: actions/download-artifact@v4
        with:
          pattern: coverage-shard-*
          path: coverage/shards
          merge-multiple: false       # Keep as separate directories

      - name: Install lcov and merge coverage
        run: |
          sudo apt-get install -y lcov
          # Build -a flags dynamically from all downloaded shard files
          lcov $(find coverage/shards -name '*.info' | sort | sed 's/^/-a /') \
            -o coverage/merged.info
          # Remove generated files from coverage
          lcov --remove coverage/merged.info \
            '**/*.g.dart' \
            '**/*.freezed.dart' \
            '**/*.mocks.dart' \
            -o coverage/merged.info

      - name: Run SonarQube scanner
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        run: sonar-scanner
```

### Coverage for monorepo (packages + root app)

```yaml
      - uses: actions/download-artifact@v4
        with:
          pattern: coverage-*
          path: coverage/all
          merge-multiple: false

      - name: Merge all coverage (app + packages)
        run: |
          sudo apt-get install -y lcov
          # Find all lcov.info files across app and package artifacts
          FILES=$(find coverage/all -name '*.info' | sort | sed 's/^/-a /')
          lcov $FILES -o coverage/merged.info
          lcov --remove coverage/merged.info \
            '**/*.g.dart' '**/*.freezed.dart' '**/*.mocks.dart' \
            -o coverage/merged.info
```

---

## Coverage Configuration for sonar-project.properties

```properties
sonar.projectKey=my-flutter-app
sonar.sources=lib
sonar.tests=test
sonar.dart.coverage.reportPaths=coverage/merged.info

# Exclude generated files from analysis
sonar.exclusions=**/*.g.dart,**/*.freezed.dart,**/*.mocks.dart

# For monorepo: include package sources
# sonar.sources=lib,packages/package_a/lib,packages/package_b/lib
# sonar.dart.coverage.reportPaths=coverage/merged.info
```

---

## Common lcov Failures and Fixes

| Error | Cause | Fix |
|---|---|---|
| `lcov: command not found` | lcov not installed before the merge step | Add `sudo apt-get install -y lcov` before any `lcov` command |
| `lcov: ERROR: no valid records found` | Empty or missing lcov.info from a shard | Confirm `--coverage` flag is present on `flutter test` and artifact upload path is correct |
| `lcov: WARNING: negative hit count` | Overlapping coverage records across shards | Use `--ignore-errors inconsistent` flag on the merge command |
| Coverage report shows ~25% on Sonar | Only one shard's lcov.info being reported | Confirm artifact upload from all shards and merge before Sonar |
| Generated files inflating coverage % | `*.g.dart` files included in report | Add `lcov --remove` step to strip generated files |
