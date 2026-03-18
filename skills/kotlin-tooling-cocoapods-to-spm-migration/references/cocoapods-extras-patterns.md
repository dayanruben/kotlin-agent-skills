# CocoaPods Extras Patterns

Patterns to look for in `build.gradle.kts` and related project files beyond the
normal `cocoapods {}` block.

These are usually CocoaPods-era workarounds and cleanup candidates.

## Detection Patterns

- Custom tasks wired to `podInstall`, `podSetup`, or generated podspec tasks.
- Code that patches `Pods.xcodeproj/project.pbxproj`.
- `cocoapods.summary`, `cocoapods.homepage`, `cocoapods.version`, or
  `cocoapods.name`.
- `cocoapods.podfile = project.file(...)`.
- `cocoapods.extraSpecAttributes`.
- `pod("...", extraOpts = ...)` or `pod("...", moduleName = ...)`.
- `noPodspec()`.
- Any build logic that references `Pods/`, `.xcworkspace`, or `.podspec` files.
- Compiler flags or linker settings added only for CocoaPods interoperability.

## Safe to remove in Phase 6

- Podspec metadata fields such as `summary`, `homepage`, `version`, `name`.
- Explicit `podfile` references.
- `extraSpecAttributes`.
- `noPodspec()`.
- Tasks that only exist to patch or clean up CocoaPods outputs.
- Any logic that edits `Pods.xcodeproj`.
- References to `Pods/`, `.xcworkspace`, or generated podspecs that are no
  longer part of the build.

Example:

```kotlin
tasks.register("fixXcodeProject") {
    doLast {
        val podsProject = project.file("../iosApp/Pods/Pods.xcodeproj/project.pbxproj")
        // CocoaPods-only patching
    }
}
tasks.named("podInstall") { finalizedBy("fixXcodeProject") }
```

The full block above should be removed after the project stops using CocoaPods.

## Requires analysis before removal

- `extraOpts` on `pod(...)`, which may indicate missing compiler flags or other
  integration details that need a SwiftPM equivalent.
- `moduleName` overrides, which often signal that the Clang module name differs
  from the pod or product name.
- Custom `defFile` configurations for pod headers.
- CocoaPods-specific linker flags that may still be required with SPM.
- Build logic that references CocoaPods outputs but may encode business-critical
  behavior rather than simple cleanup.

When unsure, explain what the code does, identify the likely SwiftPM analogue,
and consult the user before deleting it.
