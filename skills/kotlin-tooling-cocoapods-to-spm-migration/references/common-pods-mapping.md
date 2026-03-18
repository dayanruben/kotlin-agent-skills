# Common Pods to SwiftPM Mapping

Reference for migrating common CocoaPods dependencies to Swift Package Manager.

## Firebase Suite

All Firebase products come from:

`https://github.com/firebase/firebase-ios-sdk.git`

Key rules:

- Do not mix Firebase across CocoaPods and SPM.
- Product names usually match CocoaPods pod names.
- Some Objective-C Clang modules use different names and require
  `importedModules`.
- Firebase often needs `discoverModulesImplicitly = false` because transitive
  C++ modules fail cinterop generation.

### Firebase quick reference

| CocoaPods pod | SPM product | `importedModules` note |
| --- | --- | --- |
| `FirebaseAnalytics` | `FirebaseAnalytics` | `FirebaseAnalytics`, `FirebaseCore` |
| `FirebaseAuth` | `FirebaseAuth` | `FirebaseAuth`, `FirebaseCore` |
| `FirebaseCrashlytics` | `FirebaseCrashlytics` | `FirebaseCrashlytics` |
| `FirebaseDatabase` | `FirebaseDatabase` | Use `FirebaseDatabaseInternal` |
| `FirebaseFirestore` | `FirebaseFirestore` | Use `FirebaseFirestoreInternal` |
| `FirebaseMessaging` | `FirebaseMessaging` | `FirebaseMessaging` |
| `FirebaseRemoteConfig` | `FirebaseRemoteConfig` | Use `FirebaseRemoteConfigInternal` |
| `FirebaseStorage` | `FirebaseStorage` | `FirebaseStorage` |
| `FirebaseInAppMessaging` | `FirebaseInAppMessaging-Beta` | Use `FirebaseInAppMessagingInternal` |
| `FirebaseAppDistribution` | `FirebaseAppDistribution-Beta` | Use `FirebaseAppDistribution` |
| `FirebaseABTesting` | no product | Module-only dependency |
| `FirebaseAILogic` | `FirebaseAI` | Swift-only product |
| `FirebaseFunctions` | `FirebaseFunctions` | Swift-only product for Kotlin projects |

### Combined Firebase example

```kotlin
swiftPMDependencies {
    discoverModulesImplicitly = false

    swiftPackage(
        url = url("https://github.com/firebase/firebase-ios-sdk.git"),
        version = from("12.6.0"),
        products = listOf(
            product("FirebaseAnalytics"),
            product("FirebaseAuth"),
            product("FirebaseFirestore"),
            product("FirebaseCrashlytics"),
            product("FirebaseMessaging"),
            product("FirebaseFunctions"),
        ),
        importedModules = listOf(
            "FirebaseAnalytics",
            "FirebaseAuth",
            "FirebaseCore",
            "FirebaseCrashlytics",
            "FirebaseFirestoreInternal",
            "FirebaseMessaging",
        ),
    )
}
```

### Crashlytics dSYM upload script

If the app uses Crashlytics, update the build phase to the SPM checkout path:

```bash
"${BUILD_DIR%/Build/*}/SourcePackages/checkouts/firebase-ios-sdk/Crashlytics/run"
```

## Google Maps

Repository:

`https://github.com/googlemaps/ios-maps-sdk.git`

Rules:

- iOS-only package
- Prefer `exact(...)`
- Requires a platform constraint

```kotlin
swiftPackage(
    url = url("https://github.com/googlemaps/ios-maps-sdk.git"),
    version = exact("10.6.0"),
    products = listOf(
        product("GoogleMaps", platforms = setOf(iOS()))
    ),
)
```

## Google Sign-In

Repository:

`https://github.com/google/GoogleSignIn-iOS.git`

```kotlin
swiftPackage(
    url = url("https://github.com/google/GoogleSignIn-iOS.git"),
    version = from("8.0.0"),
    products = listOf(product("GoogleSignIn")),
)
```

## Simple one-to-one example

```kotlin
swiftPackage(
    url = url("https://github.com/lukaskubanek/LoremIpsum.git"),
    version = from("2.0.1"),
    products = listOf(product("LoremIpsum")),
)
```

## KMP wrapper libraries with bundled klibs

Some KMP libraries wrap Apple SDKs and ship pre-built cinterop klibs using the
`cocoapods.*` namespace. After migration, those imports must remain unchanged.

### KMPNotifier

- Maven artifact: `io.github.mirzemehdi:kmpnotifier`
- Bundled namespace: `cocoapods.FirebaseMessaging`
- Consequence: keep imports such as
  `import cocoapods.FirebaseMessaging.FIRMessaging`

Example:

```kotlin
import cocoapods.FirebaseMessaging.FIRMessaging
import swiftPMImport.com.example.app.GIDSignIn
```

### `dev.gitlive:firebase-*`

These libraries typically use high-level Kotlin APIs rather than direct
`cocoapods.*` imports, but their klibs still carry CocoaPods-era linker
metadata.

Common fixes:

1. Set `isStatic = true`.
2. Add `linkerOpts("-F", ...)` or Xcode `FRAMEWORK_SEARCH_PATHS`.
3. Re-run `integrateLinkagePackage` after adjusting products.

## Researching unmapped pods

If a pod is not listed here:

1. Check whether the upstream repository has a `Package.swift`.
2. Inspect the CocoaPods spec to find the source repository.
3. Search Swift Package Index.
4. If needed, inspect Clang module names or use build errors to discover the
   generated classes.
