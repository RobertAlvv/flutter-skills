# Platform Build Configuration Reference

This reference provides signing configuration templates, runner requirements, and artifact patterns used in Step 7 of the audit. Use it when the user needs to add or fix Android, iOS, or Web build steps.

---

## Runner Requirements

| Build Target | Required Runner | Notes |
|---|---|---|
| `flutter build apk` | `ubuntu-latest` | Android SDK available on ubuntu |
| `flutter build appbundle` | `ubuntu-latest` | Preferred over APK for Play Store |
| `flutter build ipa` | `macos-latest` | Xcode only available on macOS — always flag ubuntu as an error |
| `flutter build web` | `ubuntu-latest` | No platform toolchain needed |
| Fastlane (Android) | `ubuntu-latest` | Ruby + Bundler available on ubuntu |
| Fastlane (iOS) | `macos-latest` | Must match iOS build runner |

---

## Android Build Configuration

### Keystore signing flow

Android release builds require a keystore file and four secrets: the base64-encoded JKS, the store password, the key alias, and the key password. Store the JKS as a base64-encoded GitHub Secret:

```bash
# Generate base64 string to store as KEYSTORE_BASE64 secret
base64 -i android/app/release.keystore | pbcopy
```

### Android build job template

```yaml
jobs:
  build-android:
    runs-on: ubuntu-latest
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
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: android-${{ runner.os }}-${{ hashFiles('**/pubspec.lock', '**/build.gradle') }}
          restore-keys: android-${{ runner.os }}-
      - run: flutter pub get
      - run: dart run build_runner build --delete-conflicting-outputs

      - name: Decode keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/app/release.jks

      - name: Build release AAB
        env:
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          STORE_PASSWORD: ${{ secrets.STORE_PASSWORD }}
        run: |
          flutter build appbundle --release \
            --dart-define=ENV=production

      - uses: actions/upload-artifact@v4
        with:
          name: app-release-aab
          path: build/app/outputs/bundle/release/app-release.aab
          retention-days: 7
```

### `android/app/build.gradle` signing config

The YAML secrets must map to `signingConfigs` in `build.gradle`:

```groovy
android {
    signingConfigs {
        release {
            storeFile file("release.jks")
            storePassword System.getenv("STORE_PASSWORD")
            keyAlias System.getenv("KEY_ALIAS")
            keyPassword System.getenv("KEY_PASSWORD")
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

---

## iOS Build Configuration

### Fastlane Match certificate flow

Fastlane Match stores certificates in a private git repository. In CI, always use `--readonly` to prevent accidental certificate regeneration.

```yaml
jobs:
  build-ios:
    runs-on: macos-latest      # Required — Xcode only on macOS
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
            ios/Pods
          key: ios-${{ runner.os }}-${{ hashFiles('**/pubspec.lock', 'ios/Podfile.lock') }}
          restore-keys: ios-${{ runner.os }}-
      - run: flutter pub get

      - name: Set up Ruby and Bundler
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
          working-directory: ios

      - name: Install CocoaPods
        run: cd ios && pod install --repo-update

      - name: Setup SSH for Match
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.MATCH_SSH_KEY }}

      - name: Fetch certificates via Match (readonly)
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
        run: cd ios && bundle exec fastlane match appstore --readonly

      - name: Build IPA
        run: |
          flutter build ipa --release \
            --export-options-plist=ios/ExportOptions.plist

      - uses: actions/upload-artifact@v4
        with:
          name: app-release-ipa
          path: build/ios/ipa/*.ipa
          retention-days: 7
```

### `ios/ExportOptions.plist` for App Store distribution

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "...">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>uploadBitcode</key>
    <false/>
    <key>uploadSymbols</key>
    <true/>
    <key>compileBitcode</key>
    <false/>
</dict>
</plist>
```

---

## Web Build Configuration

Web builds require no signing. The main concern is `dart-define` for environment injection and artifact upload for deployment or PR preview.

```yaml
jobs:
  build-web:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
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
      - run: |
          flutter build web --release \
            --dart-define=ENV=production \
            --dart-define=API_URL=${{ secrets.API_URL }}
      - uses: actions/upload-artifact@v4
        with:
          name: web-build
          path: build/web/
          retention-days: 7
```

---

## Secrets Required Per Platform

| Secret Name | Platform | Description |
|---|---|---|
| `KEYSTORE_BASE64` | Android | Base64-encoded `.jks` keystore file |
| `KEY_ALIAS` | Android | Alias of the signing key in the keystore |
| `KEY_PASSWORD` | Android | Password for the signing key |
| `STORE_PASSWORD` | Android | Password for the keystore file |
| `MATCH_SSH_KEY` | iOS | SSH private key for Fastlane Match certificates repo |
| `MATCH_PASSWORD` | iOS | Encryption password for Match certificates |
| `API_URL` | Web | Optional — environment-specific API base URL |
