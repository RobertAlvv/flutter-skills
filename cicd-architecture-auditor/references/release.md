# Release Automation Reference

This reference provides Play Store, App Store Connect, GitHub Releases, and Fastlane patterns used in Step 8 of the audit. Use it when the user needs to configure or fix release automation.

---

## Release Trigger Patterns

Release jobs must never run on every push. Use one of these trigger strategies:

### Version tag trigger (recommended)

```yaml
on:
  push:
    tags:
      - 'v*.*.*'       # Matches v1.0.0, v2.3.1, etc.
```

In job conditions:

```yaml
jobs:
  deploy:
    if: startsWith(github.ref, 'refs/tags/v')
```

### Main branch trigger (for continuous delivery)

```yaml
on:
  push:
    branches:
      - main

jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
```

---

## Release Job Gate Pattern

Every release job must declare `needs:` covering all test and build jobs. This is the minimum required gate:

```yaml
jobs:
  test:         {}  # no needs
  build-android: {}  # no needs
  build-ios:    {}  # no needs

  deploy-android:
    needs: [test, build-android]    # Will not run if test or build fails
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-release-aab
          path: build/
```

---

## Play Store Upload via Fastlane Supply

### Job template

```yaml
jobs:
  deploy-android:
    runs-on: ubuntu-latest
    needs: [test, build-android]
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - uses: actions/download-artifact@v4
        with:
          name: app-release-aab
          path: build/app/outputs/bundle/release/

      - name: Upload to Play Store (internal track)
        env:
          SUPPLY_JSON_KEY_DATA: ${{ secrets.PLAY_STORE_JSON_KEY }}
        run: |
          bundle exec fastlane supply \
            --aab build/app/outputs/bundle/release/app-release.aab \
            --track internal \
            --release_status draft
```

### `android/Gemfile`

```ruby
source 'https://rubygems.org'

gem 'fastlane'
```

### `android/Fastfile`

```ruby
lane :deploy_internal do
  supply(
    aab: '../build/app/outputs/bundle/release/app-release.aab',
    track: 'internal',
    release_status: 'draft',
    json_key_data: ENV['SUPPLY_JSON_KEY_DATA']
  )
end
```

---

## App Store Connect Upload via Fastlane Pilot

### Job template

```yaml
jobs:
  deploy-ios:
    runs-on: macos-latest
    needs: [test, build-ios]
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
          working-directory: ios

      - uses: actions/download-artifact@v4
        with:
          name: app-release-ipa
          path: build/ios/ipa/

      - name: Upload to TestFlight
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.ASC_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY_CONTENT: ${{ secrets.ASC_PRIVATE_KEY }}
        run: |
          cd ios && bundle exec fastlane pilot upload \
            --ipa ../build/ios/ipa/Runner.ipa \
            --skip_waiting_for_build_processing true
```

### App Store Connect API Key setup

Generate a key at `App Store Connect → Users and Access → Keys`. Store three secrets:
- `ASC_KEY_ID` — the 10-character key identifier
- `ASC_ISSUER_ID` — the UUID from the issuer section
- `ASC_PRIVATE_KEY` — the full `.p8` file content (including header/footer lines)

---

## GitHub Releases

Create a GitHub Release that attaches all platform binaries after all platform deploys succeed:

```yaml
jobs:
  create-release:
    runs-on: ubuntu-latest
    needs: [deploy-android, deploy-ios]
    if: startsWith(github.ref, 'refs/tags/v')
    permissions:
      contents: write        # Required to create GitHub releases
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          pattern: app-release-*
          merge-multiple: true
          path: release-assets/

      - name: Create GitHub Release
        run: |
          gh release create "${{ github.ref_name }}" \
            release-assets/* \
            --title "Release ${{ github.ref_name }}" \
            --generate-notes \
            --draft
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Version Extraction from pubspec.yaml

Automated versioning ensures the version code embedded in the binary matches the tag:

```yaml
      - name: Extract version from pubspec.yaml
        id: version
        run: |
          VERSION=$(grep '^version:' pubspec.yaml | sed 's/version: //' | tr -d ' ')
          VERSION_NAME=$(echo $VERSION | cut -d'+' -f1)
          VERSION_CODE=$(echo $VERSION | cut -d'+' -f2)
          echo "name=$VERSION_NAME" >> $GITHUB_OUTPUT
          echo "code=$VERSION_CODE" >> $GITHUB_OUTPUT

      - name: Build with version
        run: |
          flutter build appbundle --release \
            --build-name=${{ steps.version.outputs.name }} \
            --build-number=${{ steps.version.outputs.code }}
```

---

## Release Anti-Patterns

| Anti-Pattern | Risk | Fix |
|---|---|---|
| Release job with no `needs:` | Deploys even if tests fail | Add `needs: [test, build-*]` |
| Release triggers on all branches | Feature branch pushes trigger store upload | Add `if: startsWith(github.ref, 'refs/tags/v')` |
| Release job rebuilds the binary | Different binary than what was tested | Download artifact from build job |
| `PLAY_STORE_JSON_KEY` as plain JSON in env | Credential exposed in workflow logs | Store full JSON as a secret, reference via `${{ secrets.NAME }}` |
| No `--release_status draft` on first upload | App goes live immediately | Use `draft` or `internal` track on initial integration |
