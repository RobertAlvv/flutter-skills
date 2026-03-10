# GitHub Actions Patterns Reference

This reference provides caching patterns, matrix strategies, artifact handling, and concurrency configuration used across all steps of the audit. Use it when the user needs help with GitHub Actions-specific YAML mechanics.

---

## Caching Patterns

### Flutter SDK + pub packages (layered)

The two cache layers must be configured separately. `subosito/flutter-action cache: true` covers the SDK binary only. Pub packages require an explicit `actions/cache` step.

```yaml
# Layer 1 — Flutter SDK binary
- uses: subosito/flutter-action@v2
  with:
    flutter-version: '3.24.0'
    channel: stable
    cache: true

# Layer 2 — pub packages and .dart_tool
- uses: actions/cache@v4
  with:
    path: |
      ~/.pub-cache
      .dart_tool
    key: pub-${{ runner.os }}-${{ hashFiles('**/pubspec.lock') }}
    restore-keys: pub-${{ runner.os }}-
```

### Gradle cache (Android build jobs)

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: gradle-${{ runner.os }}-
```

### CocoaPods cache (iOS build jobs)

```yaml
- uses: actions/cache@v4
  with:
    path: ios/Pods
    key: pods-${{ runner.os }}-${{ hashFiles('ios/Podfile.lock') }}
    restore-keys: pods-${{ runner.os }}-
```

### Cache key design rules

| Rule | Correct pattern | Wrong pattern |
|---|---|---|
| Content-addressed | `hashFiles('**/pubspec.lock')` | `${{ github.sha }}` (too granular — no reuse) |
| Restore fallback | `restore-keys: pub-${{ runner.os }}-` | No restore-keys (cold start on key miss) |
| OS-aware | `${{ runner.os }}-` prefix | No prefix (Linux cache consumed on macOS job) |
| Scope-limited | Separate keys per build target | One shared key for all jobs (invalidation spreads) |

---

## Matrix Strategy Patterns

### Sharded test matrix

```yaml
strategy:
  fail-fast: false       # Required — prevents one shard failure from cancelling siblings
  matrix:
    shard: [0, 1, 2, 3]
steps:
  - run: flutter test --total-shards=4 --shard-index=${{ matrix.shard }}
```

### Multi-package matrix

```yaml
strategy:
  fail-fast: false
  matrix:
    package: [package_a, package_b, package_c]
steps:
  - run: |
      cd packages/${{ matrix.package }}
      flutter pub get
      flutter test
```

### Combined matrix (shards × platforms)

```yaml
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-latest, macos-latest]
    shard: [0, 1]
runs-on: ${{ matrix.os }}
steps:
  - run: flutter test --total-shards=2 --shard-index=${{ matrix.shard }}
```

### Dynamic matrix from file (advanced)

When the package list is large or changes frequently, generate the matrix from a script:

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      packages: ${{ steps.list.outputs.packages }}
    steps:
      - uses: actions/checkout@v4
      - id: list
        run: |
          PACKAGES=$(ls packages/ | jq -R -s -c 'split("\n")[:-1]')
          echo "packages=$PACKAGES" >> $GITHUB_OUTPUT

  test-packages:
    needs: [setup]
    strategy:
      fail-fast: false
      matrix:
        package: ${{ fromJson(needs.setup.outputs.packages) }}
```

---

## Artifact Patterns

### Upload per shard (short retention)

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: coverage-shard-${{ matrix.shard }}
    path: coverage/lcov.info
    retention-days: 1          # Only needed for merge job — minimize storage cost
```

### Download and merge multiple artifacts

```yaml
- uses: actions/download-artifact@v4
  with:
    pattern: coverage-shard-*   # Glob — downloads all matching
    path: coverage/shards
    merge-multiple: false        # Keep as separate subdirectories

# After download, files are at:
# coverage/shards/coverage-shard-0/lcov.info
# coverage/shards/coverage-shard-1/lcov.info
# etc.
```

### Upload build binary (longer retention)

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: app-release-aab
    path: build/app/outputs/bundle/release/app-release.aab
    retention-days: 7           # Available for release job within the same workflow run
```

---

## Concurrency Configuration

Cancel in-progress runs when a new push to the same branch arrives. This prevents stale PR workflows from consuming runner minutes.

```yaml
# At workflow level (cancels entire workflow)
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

For release workflows, never cancel in-progress deployments:

```yaml
# Separate group for deploy jobs — never cancel
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false
```

---

## Job Dependency (needs:) Patterns

### Fan-in (correct — maximizes parallelism)

```yaml
jobs:
  test-app:   {}   # parallel, no needs
  test-pkg:   {}   # parallel, no needs
  build:      {}   # parallel, no needs
  sonar:
    needs: [test-app, test-pkg, build]   # waits for all three
```

### Linear chain (anti-pattern — no parallelism)

```yaml
jobs:
  test:       {}
  build:
    needs: [test]       # Waits for test — but build is independent of test
  sonar:
    needs: [build]      # Critical path: test + build + sonar = sum of all
```

### Pass outputs between jobs

```yaml
jobs:
  version:
    outputs:
      tag: ${{ steps.get_version.outputs.tag }}
    steps:
      - id: get_version
        run: echo "tag=$(cat pubspec.yaml | grep '^version:' | cut -d' ' -f2)" >> $GITHUB_OUTPUT

  build:
    needs: [version]
    steps:
      - run: flutter build appbundle --build-name=${{ needs.version.outputs.tag }}
```

---

## Environment and Secret Scoping

Always scope secrets to the step that needs them, not the job:

```yaml
# Correct — step-level scope
steps:
  - name: Upload to Play Store
    env:
      SUPPLY_JSON_KEY_DATA: ${{ secrets.PLAY_STORE_JSON_KEY }}
    run: bundle exec fastlane supply

# Wrong — job-level scope (all steps inherit the secret)
jobs:
  deploy:
    env:
      PLAY_STORE_JSON_KEY: ${{ secrets.PLAY_STORE_JSON_KEY }}  # Exposed to every step
```
