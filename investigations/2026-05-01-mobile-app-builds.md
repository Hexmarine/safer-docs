# Mobile App Build Investigation

- Date: 2026-05-01
- Scope: How the SensorSyn mobile applications are built for Android and iOS
- Related systems: `code/smokealarm-android`, `code/smokealarm-ios`, Firebase/GCP, Google Play, Apple Developer/App Store Connect, Bitrise or other mobile CI/CD

## Objective

Understand the repository evidence for how the Android and iOS mobile apps are built, configured by environment, signed, and released. This was a local source-code investigation only; no cloud, store, or CI account access was used.

## Sources Used

- Repository files:
  - `code/smokealarm-android/README.MD`
  - `code/smokealarm-android/build.gradle`
  - `code/smokealarm-android/app/build.gradle`
  - `code/smokealarm-android/settings.gradle`
  - `code/smokealarm-android/gradle/wrapper/gradle-wrapper.properties`
  - `code/smokealarm-android/app/src/main/AndroidManifest.xml`
  - `code/smokealarm-android/.github/workflows/gitguardian.yaml`
  - `code/smokealarm-ios/README.md`
  - `code/smokealarm-ios/Sensor Alarm/Podfile`
  - `code/smokealarm-ios/Sensor Alarm/Sensor Alarm.xcodeproj/project.pbxproj`
  - `code/smokealarm-ios/Sensor Alarm/Sensor Alarm.xcodeproj/xcshareddata/xcschemes/Sensor Alarm Sandbox.xcscheme`
  - `code/smokealarm-ios/Sensor Alarm/Sensor Alarm/Application Service Layer/CIBaseURL.swift`
  - `code/smokealarm-ios/Sensor Alarm/Sensor Alarm/Application Service Layer/EndPoints.swift`
  - `code/smokealarm-ios/.github/workflows/gitguardian.yaml`
  - `docs/gaps-detailed.md`
- AWS commands or consoles: none
- External references: none

## Findings

The mobile apps are native apps, not wrappers around the Angular portals.

Android lives in `code/smokealarm-android`. It is a single-module Gradle project named `Sensor` with module `:app`. The app is Kotlin-based and the README describes a Clean Architecture / MVVM / Repository pattern using Android Jetpack, Retrofit/OkHttp/Gson, Firebase, AppAuth, reCAPTCHA, LaunchDarkly, Google Places, ZXing, Glide, and related Android libraries.

Android build tooling is Gradle wrapper 7.5, Android Gradle Plugin 7.4.2, Kotlin 1.7.20, compile SDK 34, target SDK 34, minimum SDK 23, and Java/Kotlin target 11. The normal local command is `./gradlew build` from `code/smokealarm-android`, or opening the project in Android Studio.

Android environment selection is done with Gradle product flavors under flavor dimension `Apis`: `dev`, `qa`, `sandbox`, and `live`. Each flavor sets a different application ID, API base URL, SSO callback URL, email signature URL, support URL, e-learning URL, and app version metadata through generated `BuildConfig` fields. Runtime API calls are built from `BuildConfig.BASE_URL` plus `/api/v1/`.

Android package IDs in the current Gradle file are:

| Flavor | Application ID | API base |
| --- | --- | --- |
| `dev` | `com.sensorglobal.hub.dev` | `https://api-development.sensorglobal.com` |
| `qa` | `com.sensorglobal.hub.qa` | `https://api-qa.sensorglobal.com` |
| `sandbox` | `com.sensorglobal.hub.sandbox` | `https://api-sandbox.sensorglobal.com` |
| `live` | `com.sensorglobal.hub` | `https://api.sensorglobal.com` |

Android has Firebase config files committed under each environment source set: `clientDev`, `dev`, `qa`, `sandbox`, and `live`. These files contain multiple registered Android package names, including both older `com.app.sensor*` IDs and current `com.sensorglobal.hub*` IDs. Firebase ownership and active app mappings still need confirmation from the GCP/Firebase project.

Android release signing is not defined in `app/build.gradle`. There is no committed Android keystore, `.jks`, or Gradle `signingConfigs` block in the checked files. That means signed Play Store builds likely depend on Android Studio local signing, CI secrets, Google Play App Signing, or an external keystore process that is not captured in this repo snapshot.

iOS lives in `code/smokealarm-ios`. It is a native Swift/Xcode project using CocoaPods. Local setup is documented as installing CocoaPods, running `pod install` inside `code/smokealarm-ios/Sensor Alarm/`, opening `Sensor Alarm.xcworkspace`, and building from Xcode. The workspace must be used rather than the bare `xcodeproj`.

iOS dependency management is via CocoaPods with minimum platform iOS 13.0 in the `Podfile`. The project build settings use iOS deployment target 15.0 for app targets, Swift 5.0, and manual code signing.

iOS has four native app targets:

| Target | Bundle ID | Version metadata in project |
| --- | --- | --- |
| `Sensor Alarm` | `com.sensorglobal.sensor` | marketing `1.0.22`, build `1.2` |
| `Sensor Alarm Sandbox` | `com.sensorglobal.sensor-sandbox` | marketing `1.0.0`, build `1.2` |
| `Sensor Alarm Dev` | `com.sensorglobal.sensor-development` | marketing `1.0.0`, build `5` |
| `Sensor Alarm QA` | `com.sensorglobal.sensor-qa` | marketing `1.0.0`, build `1.6` |

iOS environment selection appears split across target settings, launch-scheme environment variables, and code fallbacks. `CIBaseURL.swift` contains build-time placeholder strings such as `$(ENV_BASE_URL_KEY)` and `$(ENV_TYPE)`. `EndPoints.swift` reads runtime process environment variables first, falling back to those placeholder values. The committed shared Xcode scheme is only for `Sensor Alarm Sandbox` and includes sandbox-specific environment variables. The README also mentions changing `baseUrlSetting.plist`, but current code evidence points more strongly at environment variables and build placeholders.

iOS has Firebase plist files committed for production, dev, QA, and sandbox. `AppDelegate.swift` configures Firebase and Firebase Messaging at app startup.

iOS signing material is present in the repository snapshot under `code/smokealarm-ios/Certificate`, including provisioning profiles and `.p12` certificate files. The iOS README also includes certificate password notes. This is a significant custody and rotation concern and should be treated as sensitive even if the certificates are expired or no longer active.

The mobile CI/CD evidence in this repo is incomplete. Both mobile repos have GitHub Actions workflows, but they run GitGuardian secret scanning only. I found no repo-local mobile build workflow, `fastlane` folder, Bitrise config file, or App Store / Play Store deployment automation. Existing gap docs already list Bitrise account ownership, mobile signing-key custody, Apple Developer account, Google Play account, and Firebase/GCP ownership as open items.

## Build Commands From Source Evidence

Android local build:

```bash
cd code/smokealarm-android
./gradlew build
```

Common Android flavor builds likely follow standard Gradle variant names:

```bash
cd code/smokealarm-android
./gradlew assembleDevDebug
./gradlew assembleQaDebug
./gradlew assembleSandboxDebug
./gradlew assembleLiveRelease
```

These commands were inferred from the Gradle flavor/buildType configuration and were not executed during this investigation.

iOS local build:

```bash
cd "code/smokealarm-ios/Sensor Alarm"
pod install
open "Sensor Alarm.xcworkspace"
```

For command-line iOS builds, the likely shape is:

```bash
cd "code/smokealarm-ios/Sensor Alarm"
xcodebuild -workspace "Sensor Alarm.xcworkspace" -scheme "Sensor Alarm Sandbox" -configuration Release archive
```

Only the sandbox scheme is shared in the repo. Other schemes may exist locally in developer Xcode state or in CI, but they are not visible in this checkout.

## Risks Or Constraints

- iOS signing assets and certificate password notes are present in the repo snapshot. Confirm whether these credentials are still valid, then rotate/revoke as appropriate.
- Android release signing custody is unknown from source. The upload key, Play App Signing status, and backup/rotation process need confirmation from Google Play and CI/account owners.
- Firebase/GCP ownership is unresolved. Push notifications and Crashlytics depend on this ownership.
- The iOS README and code disagree on environment switching. The README points to `baseUrlSetting.plist`; code and the shared sandbox scheme point to environment variables/placeholders.
- Android `AndroidManifest.xml` contains a Google Maps API key literal and enables `usesCleartextTraffic=true`. Both need review against current platform security expectations.
- Android debug builds add full HTTP body logging; this is acceptable for local debug only, but release behavior and production observability should be confirmed.
- GitHub Actions currently provide secret scanning only, not reproducible mobile builds.
- The repo-local source does not prove which CI system currently creates production `.apk`/`.aab` or `.ipa` artifacts.

## Cost Optimization Opportunities

None identified. This investigation is mostly about build custody, release reproducibility, signing governance, and mobile platform ownership.

## Recommended Next Steps

1. Confirm current release authority for each platform:
   - Apple Developer team owner and App Store Connect admin
   - Google Play Console owner and whether Play App Signing is enabled
   - Firebase/GCP project owner for push notifications and Crashlytics
   - Bitrise or other mobile CI owner, if still used
2. Export or document the current production release process:
   - who can produce signed Android and iOS builds
   - where signing secrets live
   - how version numbers are incremented
   - how production submissions are approved
3. Validate whether the committed iOS certificate/provisioning files are active. If active, rotate and remove from source history where practical.
4. Decide whether to standardize CI through Bitrise, GitHub Actions, or another runner. The repo should have at least one reproducible build path per platform.
5. Align iOS environment switching docs with actual code behavior.
6. Confirm whether Android release output should be APK or Android App Bundle (`.aab`) for current Play Store publishing.

## Open Questions

- Which system actually produces the production Android and iOS release artifacts today?
- Where is the Android upload key or keystore stored?
- Are the committed iOS `.p12` and provisioning profile files still valid?
- Are App Store Connect, Google Play Console, Firebase/GCP, and Bitrise controlled by the current SensorSyn operating entity?
- Does Bitrise still exist for these apps, or is it only a historical/accounting gap?
- Are there private Xcode schemes or CI-only environment files missing from the checkout?
- Are the older `com.app.sensor*` Firebase package registrations still needed?
