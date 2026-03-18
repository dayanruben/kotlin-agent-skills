# Troubleshooting Guide

Common issues when migrating from CocoaPods to SwiftPM import in Kotlin
Multiplatform projects.

## Gradle Issues

### `swiftPMDependencies` not found

Symptom:

`Unresolved reference: swiftPMDependencies`

Fix:

1. Verify the Kotlin version supports Swift Import.
2. Add the required Maven repository if the chosen Kotlin build needs one.
3. Add a strict Kotlin Gradle plugin constraint if needed:

```kotlin
buildscript {
    dependencies.constraints {
        "classpath"("org.jetbrains.kotlin:kotlin-gradle-plugin:<kotlin-version>!!")
    }
}
```

### Import not found after migration

Check the generated namespace:

```text
swiftPMImport.<group>.<module>.<ClassName>
```

Remember to replace `-` with `.` in both the group and module names.

If the class still does not exist, confirm whether the import should actually
remain `cocoapods.*` because it comes from a bundled third-party klib.

## Linker and Packaging Issues

### Undefined symbols or framework not found

Typical causes:

- `integrateLinkagePackage` was not run.
- The package was added but not linked in Xcode.
- The framework should be static but still uses `isStatic = false`.
- Wrapper libraries such as `dev.gitlive:firebase-*` require framework search
  paths or `linkerOpts("-F", ...)`.

### `No such module` in Xcode

Try:

1. Clean the Xcode build folder.
2. Re-run `integrateLinkagePackage`.
3. Restart Xcode.
4. Re-open the correct project file:
   - `.xcodeproj` after full CocoaPods removal
   - `.xcworkspace` if non-KMP CocoaPods remain

## Build Phase Issues

### User Script Sandboxing

Symptom:

`checkSandboxAndWriteProtection` fails during Xcode builds.

Fix:

1. Set `ENABLE_USER_SCRIPT_SANDBOXING = NO` in the Xcode project.
2. Restart the Gradle daemon with `./gradlew --stop`.

### `integrateEmbedAndSign` does nothing

Cause:

The project disables `EmbedAndSign` tasks.

Example of problematic code:

```kotlin
project.gradle.taskGraph.whenReady {
    allTasks
        .filter { it::class.simpleName?.contains("EmbedAndSign") == true }
        .forEach { it.enabled = false }
}
```

Remove the disabler and run the integration again.

### `embedAndSignAppleFrameworkForXcode` remains commented out

If the build phase exists but the Gradle invocation is commented out, uncomment
it manually in `project.pbxproj`.

## Bundled Klib Problems

### `cocoapods.*` class missing after converting to `swiftPMImport.*`

Cause:

A third-party KMP dependency already provides the binding through a bundled
klib, and `swiftPMDependencies` intentionally skipped generating a duplicate.

Fix:

Revert the affected import back to `cocoapods.*`.

Example:

```kotlin
import cocoapods.FirebaseMessaging.FIRMessaging
```

To verify this, inspect the dependency klib with
`klib dump-metadata-signatures`.

## Firebase-Specific Issues

### Cinterop fails on Firebase transitive dependencies

Cause:

Implicit module discovery tries to process transitive C++ modules such as gRPC,
abseil, or leveldb.

Fix:

Set `discoverModulesImplicitly = false` and declare only the required Firebase
modules in `importedModules`.

### Firebase classes not found

Several Firebase products need internal Clang module names:

| Product | Correct `importedModules` entry |
| --- | --- |
| `FirebaseDatabase` | `FirebaseDatabaseInternal` |
| `FirebaseFirestore` | `FirebaseFirestoreInternal` |
| `FirebaseRemoteConfig` | `FirebaseRemoteConfigInternal` |
| `FirebaseInAppMessaging-Beta` | `FirebaseInAppMessagingInternal` |

### Crashlytics upload script still points at `${PODS_ROOT}`

Update the script to:

```bash
"${BUILD_DIR%/Build/*}/SourcePackages/checkouts/firebase-ios-sdk/Crashlytics/run"
```

### `dyld` crash after partial Firebase migration

If some Firebase libraries still come from CocoaPods and others come from SPM,
the app may build and then crash at runtime.

Fix:

Migrate the entire Firebase suite together, including Swift-only products that
share the same repository.

## Manual Integration Fallback

If automatic discovery of the integration command fails:

1. Find the directory containing `Podfile`.
2. Find the app `.xcodeproj` in that directory.
3. Find the KMP module that now contains `swiftPMDependencies`.
4. Run:

```bash
XCODEPROJ_PATH="/abs/path/to/App.xcodeproj" \
GRADLE_PROJECT_PATH=":shared" \
./gradlew :shared:integrateEmbedAndSign :shared:integrateLinkagePackage
```

## Last Resort

Do not immediately revert the migration. First:

1. Read the full error log.
2. Re-check each migration phase.
3. Confirm the correct Xcode project type is being opened.
4. Confirm `cocoapods.*` imports were only replaced where appropriate.
5. Present the remaining options to the user if the issue is still ambiguous.
