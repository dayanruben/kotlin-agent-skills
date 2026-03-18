# SwiftPM Import DSL Reference

Complete reference for the `swiftPMDependencies {}` DSL in Kotlin Multiplatform.

## Basic Structure

```kotlin
kotlin {
    iosArm64()
    iosSimulatorArm64()

    swiftPMDependencies {
        iosDeploymentVersion.set("16.0")
        macosDeploymentVersion.set("13.0")
        tvosDeploymentVersion.set("16.0")
        watchosDeploymentVersion.set("9.0")

        xcodeProjectPathForKmpIJPlugin.set(
            layout.projectDirectory.file("../iosApp/iosApp.xcodeproj")
        )

        discoverModulesImplicitly = true

        swiftPackage(...)
        localSwiftPackage(...)
    }
}
```

## Package Declarations

### Remote package by Git URL

```kotlin
swiftPackage(
    url = url("https://github.com/owner/repo.git"),
    version = from("1.0.0"),
    products = listOf(product("ProductName")),
)
```

### Remote package by registry id

```kotlin
swiftPackage(
    repository = id("scope.package-name"),
    version = from("1.0.0"),
    products = listOf(product("ProductName")),
)
```

### Local package

```kotlin
localSwiftPackage(
    directory = layout.projectDirectory.dir("../LocalPackage"),
    products = listOf("LocalPackage"),
)
```

## Version selectors

| Function | Meaning | Typical use |
| --- | --- | --- |
| `from("1.0.0")` | Minimum compatible version | Most packages |
| `exact("1.0.0")` | Exact version | Strict packages such as GoogleMaps |
| `branch("main")` | Track a branch | Development only |
| `revision("abc123")` | Pin a commit | Reproducible debugging |

## Products

### Single product

```kotlin
products = listOf(product("FirebaseAnalytics"))
```

### Multiple products from the same package

```kotlin
products = listOf(
    product("FirebaseAnalytics"),
    product("FirebaseAuth"),
    product("FirebaseFirestore"),
)
```

### Platform-constrained product

```kotlin
products = listOf(
    product("GoogleMaps", platforms = setOf(iOS()))
)
```

Available platform helpers:

- `iOS()`
- `macOS()`
- `tvOS()`
- `watchOS()`

## Module discovery

### Automatic discovery

`discoverModulesImplicitly = true` is the default. Kotlin will try to generate
cinterop bindings for every accessible Clang module.

### Explicit `importedModules`

Use `importedModules` when:

- the Clang module name differs from the product name,
- a package exposes multiple Objective-C modules, or
- you disable implicit discovery because transitive C or C++ modules fail
  cinterop generation.

```kotlin
swiftPMDependencies {
    discoverModulesImplicitly = false

    swiftPackage(
        url = url("https://github.com/firebase/firebase-ios-sdk.git"),
        version = from("12.6.0"),
        products = listOf(
            product("FirebaseAnalytics"),
            product("FirebaseFirestore"),
        ),
        importedModules = listOf(
            "FirebaseAnalytics",
            "FirebaseCore",
            "FirebaseFirestoreInternal",
        ),
    )
}
```

When `discoverModulesImplicitly = true`, `importedModules` is ignored.

## Deployment versions

```kotlin
swiftPMDependencies {
    iosDeploymentVersion.set("16.0")
    macosDeploymentVersion.set("13.0")
    tvosDeploymentVersion.set("16.0")
    watchosDeploymentVersion.set("9.0")
}
```

## IDE integration

If the project uses the KMP IntelliJ plugin to build the Apple app, point
`xcodeProjectPathForKmpIJPlugin` at the `.xcodeproj` that contains the
`embedAndSignAppleFrameworkForXcode` integration:

```kotlin
swiftPMDependencies {
    xcodeProjectPathForKmpIJPlugin.set(
        layout.projectDirectory.file("../iosApp/iosApp.xcodeproj")
    )
}
```

## Complete Example

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
}

group = "org.example.myproject"

kotlin {
    iosArm64()
    iosSimulatorArm64()

    swiftPMDependencies {
        iosDeploymentVersion.set("16.0")
        discoverModulesImplicitly = false

        xcodeProjectPathForKmpIJPlugin.set(
            layout.projectDirectory.file("../iosApp/iosApp.xcodeproj")
        )

        swiftPackage(
            url = url("https://github.com/firebase/firebase-ios-sdk.git"),
            version = from("12.6.0"),
            products = listOf(
                product("FirebaseAnalytics"),
                product("FirebaseAuth"),
                product("FirebaseFirestore"),
            ),
            importedModules = listOf(
                "FirebaseAnalytics",
                "FirebaseCore",
                "FirebaseAuth",
                "FirebaseFirestoreInternal",
            ),
        )

        swiftPackage(
            url = url("https://github.com/googlemaps/ios-maps-sdk.git"),
            version = exact("10.6.0"),
            products = listOf(product("GoogleMaps", platforms = setOf(iOS()))),
        )
    }

    listOf(iosArm64(), iosSimulatorArm64()).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "Shared"
            isStatic = true
        }
    }

    sourceSets.configureEach {
        languageSettings {
            optIn("kotlinx.cinterop.ExperimentalForeignApi")
        }
    }
}
```

## Notes on transitive dependencies

SwiftPM resolves transitive dependencies automatically. You can still declare
them explicitly if you need to pin versions or model package resolution more
deterministically.
