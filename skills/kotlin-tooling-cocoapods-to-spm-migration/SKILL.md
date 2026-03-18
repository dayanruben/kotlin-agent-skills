---
name: kotlin-tooling-cocoapods-to-spm-migration
description: >
  Migrate Kotlin Multiplatform (KMP) projects from CocoaPods integration to
  Swift Package Manager import. Covers swiftPMDependencies adoption,
  CocoaPods cleanup, Kotlin import rewrites, Xcode reconfiguration, and common
  interoperability issues such as bundled klibs and Firebase or Google SDK
  migration. Use when replacing kotlin("native.cocoapods"), migrating pod()
  declarations, or updating imports from cocoapods.* to swiftPMImport.*.
license: Apache-2.0
metadata:
  author: JetBrains
  version: "1.0.0"
---

# CocoaPods to SwiftPM Migration for KMP

Migrate Kotlin Multiplatform projects from `kotlin("native.cocoapods")` to
`swiftPMDependencies {}`.

## Requirements

- **Kotlin**: Version with Swift Import support (for example `2.4.0-Beta1` or later)
- **Xcode**: `16.4` or `26.0+`
- **iOS deployment target**: `16.0+` recommended

## Migration Overview

**Important:** Keep the `cocoapods {}` block and plugin active until Phase 6.
The migration adds `swiftPMDependencies {}` alongside the existing CocoaPods
setup first, reconfigures Xcode, and only then removes CocoaPods.

| Phase | Action |
| --- | --- |
| 1 | Analyze the existing CocoaPods configuration |
| 2 | Update Gradle configuration (repositories, Kotlin version) |
| 3 | Add `swiftPMDependencies {}` alongside `cocoapods {}` |
| 4 | Transform Kotlin imports |
| 5 | Reconfigure the iOS project and deintegrate CocoaPods |
| 6 | Remove CocoaPods plugin and Gradle configuration |
| 7 | Verify Gradle and Xcode builds |
| 8 | Write `MIGRATION_REPORT.md` |

## Phase 1: Pre-Migration Analysis

### 1.0 Verify the project builds

Before changing anything, identify the module that uses CocoaPods and confirm it
builds successfully.

1. Find `build.gradle.kts` files containing `cocoapods`.
2. Extract the KMP module name from the path.
3. Build only that module with `./gradlew :moduleName:build`.
4. If the targeted build fails, ask the user either:
   - to provide the correct build command, or
   - to confirm the module is in a working state and it is safe to proceed.

If the user confirms without a successful pre-migration build, record that
verification could not be completed and call it out again in Phase 7.

### 1.0a Confirm Kotlin version with Swift Import support

Ask the user whether the project already uses a Kotlin version with
`swiftPMDependencies` support.

- If **yes**, read the current Kotlin version and skip Phase 2.2.
- If **no**, ask for:
  - the target Kotlin version, and
  - whether that version requires a custom Maven repository.

Compare the current and target **major.minor** Kotlin versions. If the change is
large, warn that the version jump may introduce unrelated breaking changes and
recommend updating Kotlin first, verifying the build, and then re-running the
migration.

### 1.1 Check for deprecated CocoaPods workaround property

Search `gradle.properties` for:

```properties
kotlin.apple.deprecated.allowUsingEmbedAndSignWithCocoaPodsDependencies=true
```

Record whether it exists. After migration, this property is no longer needed and
must be removed in Phase 6.

### 1.2 Check for EmbedAndSign disablers

Search all `build.gradle.kts` files for logic that disables `EmbedAndSign`
tasks, such as `taskGraph.whenReady` filters or `tasks.matching` blocks.
These CocoaPods-era workarounds will break `integrateEmbedAndSign` and must be
removed before Phase 5 succeeds.

See [references/troubleshooting.md](references/troubleshooting.md).

### 1.3 Check for third-party KMP libraries with bundled cinterop klibs

Some KMP libraries ship pre-built cinterop klibs that use the
`cocoapods.*` namespace. Those imports must be preserved after migration,
because the bindings come from the library's bundled klib rather than actual
CocoaPods infrastructure.

Known example:

| Library | Maven artifact | Bundled namespace |
| --- | --- | --- |
| KMPNotifier | `io.github.mirzemehdi:kmpnotifier` | `cocoapods.FirebaseMessaging` |

To detect this:

1. Search Gradle dependencies for known KMP wrapper libraries.
2. Find all `import cocoapods.*` statements in Kotlin sources.
3. Cross-reference the imports against the bundled namespaces provided by those
   libraries.
4. Mark the imports that must stay unchanged in Phase 4.

If needed, inspect downloaded klibs with `klib dump-metadata-signatures`.

### Record these findings

1. CocoaPods configuration blocks.
2. Pod names, versions, and `linkOnly` flags.
3. `cocoapods.framework {}` settings such as `baseName`, `isStatic`, and
   deployment target.
4. All `cocoapods.*` imports and whether they should be migrated or preserved.
5. The iOS project directory containing `Podfile` and `.xcworkspace`.
6. Whether the project uses non-KMP CocoaPods in the iOS app.
7. Whether the Xcode project has commented-out or broken `embedAndSign` phases.
8. Whether Crashlytics dSYM upload scripts point at CocoaPods paths.
9. Any extra CocoaPods-specific build logic. See
   [references/cocoapods-extras-patterns.md](references/cocoapods-extras-patterns.md).

## Phase 2: Gradle Configuration

Do **not** upgrade Gradle, KSP, or unrelated dependencies during this migration.
Only make the changes needed for SwiftPM import support.

### 2.1 Add custom Maven repository when required

If the chosen Kotlin version needs a custom repository, add it to
`settings.gradle.kts` in both `pluginManagement.repositories` and
`dependencyResolutionManagement.repositories`.

### 2.2 Update Kotlin version when required

If the project is not already on a Kotlin version with Swift Import support,
update the Kotlin version in the version catalog or build scripts to the value
confirmed in Phase 1.0a.

### 2.3 Add buildscript constraint if `swiftPMDependencies` is unresolved

```kotlin
buildscript {
    dependencies.constraints {
        "classpath"("org.jetbrains.kotlin:kotlin-gradle-plugin:<kotlin-version>!!")
    }
}
```

## Phase 3: Add `swiftPMDependencies` While Keeping CocoaPods

Do **not** remove the `cocoapods {}` block or `kotlin("native.cocoapods")`
plugin yet.

### 3.1 Add `group`

Ensure the module sets a stable `group`, because the generated Kotlin import
namespace depends on it.

```kotlin
group = "org.example.myproject"
```

### 3.2 Add `swiftPMDependencies`

Add a `swiftPMDependencies {}` block that mirrors the existing pod usage.
Use [references/common-pods-mapping.md](references/common-pods-mapping.md) to
map each pod to:

- the Swift package repository URL,
- the SPM products to link, and
- the `importedModules` list when the Clang module name differs from the
  product name.

Key rules:

- `products` control linking.
- `importedModules` control cinterop generation when
  `discoverModulesImplicitly = false`.
- Do not split a shared library suite across CocoaPods and SPM. Migrate the
  full suite together, especially Firebase.

### 3.3 Move framework configuration out of `cocoapods`

If the project configures `framework {}` inside `cocoapods`, move that
configuration to `binaries.framework {}` on the iOS targets.

Prefer `isStatic = true`. It is strongly recommended in general and required for
some CocoaPods-era KMP wrapper libraries such as `dev.gitlive:firebase-*`.

### 3.4 Handle wrapper libraries such as `dev.gitlive:firebase-*`

If the project uses wrapper libraries with CocoaPods-era linker metadata:

1. Switch the framework to `isStatic = true`.
2. Add framework search paths or `linkerOpts` as needed.
3. Re-run `integrateLinkagePackage` after product changes.

See [references/common-pods-mapping.md](references/common-pods-mapping.md) and
[references/troubleshooting.md](references/troubleshooting.md).

### 3.5 Add language settings

```kotlin
sourceSets.configureEach {
    languageSettings {
        optIn("kotlinx.cinterop.ExperimentalForeignApi")
    }
}
```

For DSL details, see [references/dsl-reference.md](references/dsl-reference.md).

## Phase 4: Kotlin Source Updates

### Import namespace formula

```
swiftPMImport.<group>.<module>.<ClassName>
```

Where:

- `group` comes from `build.gradle.kts`, with `-` converted to `.`
- `module` comes from the Gradle module name, with `-` converted to `.`
- `ClassName` is the Objective-C class name

### Example

```kotlin
// BEFORE
import cocoapods.FirebaseAnalytics.FIRAnalytics

// AFTER
import swiftPMImport.org.example.myproject.shared.FIRAnalytics
```

### Preserve bundled klib imports

Do **not** replace `cocoapods.*` imports that come from bundled third-party
klibs identified in Phase 1.3. Those imports should stay unchanged.

### Bulk replacement approach

Use a scoped regex replacement across Kotlin source files, excluding preserved
imports:

```text
Find:    cocoapods\.\w+\.
Replace: swiftPMImport.<group>.<module>.
```

Then manually restore any imports that should remain `cocoapods.*`.

## Phase 5: iOS Project Reconfiguration

### 5.1 Run the integration tasks

Build the iOS workspace to capture the Gradle integration command. Then run the
project-specific `integrateEmbedAndSign` and `integrateLinkagePackage` tasks.

After running them:

1. Verify `embedAndSignAppleFrameworkForXcode` is active in the Xcode project.
2. Disable `ENABLE_USER_SCRIPT_SANDBOXING`.
3. Restart the Gradle daemon with `./gradlew --stop`.

If `integrateEmbedAndSign` is skipped or the build phase remains commented out,
check again for EmbedAndSign disablers and see
[references/troubleshooting.md](references/troubleshooting.md).

### 5.2 Update Crashlytics scripts when needed

If the iOS app uses Crashlytics, update dSYM upload scripts from CocoaPods paths
to the SPM checkout path.

### 5.3 Deintegrate CocoaPods

Choose one of these paths:

- **Option A:** Full deintegration when CocoaPods was only used for the KMP
  framework.
- **Option B:** Partial removal when non-KMP CocoaPods remain in the iOS app.

Do not guess. Base the cleanup path on the findings from Phase 1.

### 5.4 Manual fallback

If automatic integration fails, use the manual steps from
[references/troubleshooting.md](references/troubleshooting.md).

## Phase 6: Remove CocoaPods from Gradle

Only after Xcode has been reconfigured successfully:

1. Remove `kotlin("native.cocoapods")` from `plugins`.
2. Delete the `cocoapods {}` block.
3. Remove deprecated properties from `gradle.properties`.
4. Remove obsolete CocoaPods-specific tasks and build logic.

For cleanup patterns, see
[references/cocoapods-extras-patterns.md](references/cocoapods-extras-patterns.md).

## Phase 7: Verification

Run the migrated module build:

```bash
./gradlew :moduleName:build
./gradlew :moduleName:linkDebugFrameworkIosSimulatorArm64
```

Then build the iOS app with `xcodebuild`, using:

- `.xcodeproj` if all CocoaPods were removed, or
- `.xcworkspace` if non-KMP CocoaPods remain.

If the pre-migration build was not verified, explicitly warn that current build
failures may include pre-existing issues.

Do **not** revert the migration as a first response. Read the failure logs,
re-check the migration phases, and use the troubleshooting guide.

## Phase 8: Migration Report

Write `MIGRATION_REPORT.md` in the project root using
[references/migration-report-template.md](references/migration-report-template.md).

The report must cover:

1. Pre-migration state and project-specific anomalies.
2. Exact changes made in each migration phase.
3. Every import transformation, including preserved `cocoapods.*` imports.
4. Errors encountered, root causes, and fixes.
5. Non-trivial decisions such as `isStatic`, preserved imports, and framework
   search paths.
6. A complete file list of created, modified, and deleted files.

## Additional Resources

- [references/dsl-reference.md](references/dsl-reference.md)
- [references/common-pods-mapping.md](references/common-pods-mapping.md)
- [references/cocoapods-extras-patterns.md](references/cocoapods-extras-patterns.md)
- [references/troubleshooting.md](references/troubleshooting.md)
- [references/migration-report-template.md](references/migration-report-template.md)
