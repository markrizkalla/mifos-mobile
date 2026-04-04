# KMP Codebase Architecture & Build Speed Audit Report

This report provides a comprehensive audit of the provided Kotlin Multiplatform (KMP) codebase with a strict focus on speeding up the build time and identifying critical build configuration errors.

## 1. Issue [1]: Missing KSP Optimization Flags

* **Severity:** High
* **Location:** `gradle.properties` (Missing) & `build-logic/convention/src/main/kotlin/KMPKoinConventionPlugin.kt`
* **Description:** The project uses KSP extensively (via Koin annotations `koin.ksp.compiler`). By default, KSP might not use its fastest incremental processing features or KSP2. Enabling KSP incremental processing and KSP2 (which runs directly on K2 compiler frontend) dramatically speeds up symbol processing tasks. Currently, `KMPKoinConventionPlugin.kt` applies KSP but does not configure these optimization flags in the root project.
* **Code Fix:** Add KSP optimization flags to `gradle.properties`.

```properties
<<<<<<< SEARCH
# Kotlin code style for this project: "official" or "obsolete":
kotlin.code.style=official
=======
# Kotlin code style for this project: "official" or "obsolete":
kotlin.code.style=official

# KSP Optimizations
ksp.useKSP2=true
ksp.incremental=true
ksp.incremental.intermodule=true
>>>>>>> REPLACE
```

## 2. Issue [2]: Excessive Targets in Multiplatform Setup

* **Severity:** High
* **Location:** `build-logic/convention/src/main/kotlin/org/mifos/mobile/KotlinMultiplatform.kt`
* **Description:** The base KMP configuration adds targets for `desktop`, `androidTarget`, `iosSimulatorArm64`, `iosX64`, `iosArm64`, `js`, and `wasmJs`. Whenever a build or sync happens without specific target constraints, the Kotlin compiler may attempt to configure and sometimes compile tasks for all these platforms. If a developer is only working on Android or Desktop, configuring Wasm, JS, and iOS targets is a waste of time and memory.
* **Code Fix:** Implement a "target tiering" system or use local properties to disable unneeded targets during local development.

*(Note: The provided fix adds a way to conditionally disable targets via project properties, which developers can set in `~/.gradle/gradle.properties` or command line, e.g., `-PskipIos=true`).*

```kotlin
<<<<<<< SEARCH
        jvm("desktop")
        androidTarget()
        iosSimulatorArm64()
        iosX64()
        iosArm64()
        js(IR) {
            this.nodejs()
            binaries.executable()
        }
        wasmJs {
            browser()
            nodejs()
        }
=======
        val skipDesktop = project.hasProperty("skipDesktop")
        val skipIos = project.hasProperty("skipIos")
        val skipWeb = project.hasProperty("skipWeb")

        if (!skipDesktop) {
            jvm("desktop")
        }
        androidTarget()
        if (!skipIos) {
            iosSimulatorArm64()
            iosX64()
            iosArm64()
        }
        if (!skipWeb) {
            js(IR) {
                this.nodejs()
                binaries.executable()
            }
            wasmJs {
                browser()
                nodejs()
            }
        }
>>>>>>> REPLACE
```

## 3. Issue [3]: Static Analysis Plugins (Detekt/Spotless) Blocking Configuration/Build

* **Severity:** High
* **Location:** `build-logic/convention/src/main/kotlin/CMPFeatureConventionPlugin.kt` and `build-logic/convention/src/main/kotlin/KMPLibraryConventionPlugin.kt`
* **Description:** Currently, `mifos.detekt.plugin` and `mifos.spotless.plugin` are applied unconditionally to every feature and library module. These plugins can add significant overhead during the configuration phase and the execution phase (if they run on every build). It is highly recommended to only run static analysis locally when requested (e.g., via a git pre-commit hook or explicit task) or strictly on CI, rather than burdening the daily inner-loop development cycle.
* **Code Fix:** Apply these analysis plugins conditionally based on a property, so developers can opt-in when needed, or they can be enabled strictly on CI.

```kotlin
<<<<<<< SEARCH
            pluginManager.apply {
                apply("org.convention.kmp.library")
                apply("org.convention.kmp.koin")
                apply("org.jetbrains.kotlin.plugin.compose")
                apply("org.jetbrains.compose")
                apply("mifos.detekt.plugin")
                apply("mifos.spotless.plugin")
            }
=======
            pluginManager.apply {
                apply("org.convention.kmp.library")
                apply("org.convention.kmp.koin")
                apply("org.jetbrains.kotlin.plugin.compose")
                apply("org.jetbrains.compose")

                // Only apply heavy static analysis if explicitly requested or on CI
                if (project.hasProperty("enableStaticAnalysis") || System.getenv("CI") == "true") {
                    apply("mifos.detekt.plugin")
                    apply("mifos.spotless.plugin")
                }
            }
>>>>>>> REPLACE
```
*(Apply similar logic to `KMPLibraryConventionPlugin.kt`)*

## 4. Issue [4]: Over-fetching in Feature Convention Plugin

* **Severity:** Medium
* **Location:** `build-logic/convention/src/main/kotlin/CMPFeatureConventionPlugin.kt`
* **Description:** `CMPFeatureConventionPlugin` forces *every* feature module to depend on `core:ui`, `core-base:ui`, `core:designsystem`, `core:data`, `core-base:designsystem`, and multiple `jb.*` compose libraries. This creates a massive, tightly coupled dependency graph. If `core:data` changes, *every single feature module* must be recompiled, ruining incremental compilation.
* **Code Fix:** Remove ubiquitous dependencies from the convention plugin. Feature modules should explicitly declare only the dependencies they actually use in their own `build.gradle.kts`. At a minimum, remove data layer dependencies from the UI convention plugin.

```kotlin
<<<<<<< SEARCH
            dependencies {
                add("commonMainImplementation", project(":core:ui"))
                add("commonMainImplementation", project(":core-base:ui"))
                add("commonMainImplementation", project(":core:designsystem"))
//                add("commonMainImplementation", project(":core:testing"))
                add("commonMainImplementation", project(":core:data"))
                add("commonMainImplementation", project(":core-base:designsystem"))
//                add("commonMainImplementation", project(":core:analytics"))
=======
            dependencies {
                // Modules should depend on what they specifically need in their build.gradle.kts.
                // However, as a base UI convention, we provide the core design system and UI elements.
                add("commonMainImplementation", project(":core:ui"))
                add("commonMainImplementation", project(":core-base:ui"))
                add("commonMainImplementation", project(":core:designsystem"))
                add("commonMainImplementation", project(":core-base:designsystem"))
                // Removed :core:data to prevent invalidating all feature modules when data layer changes
>>>>>>> REPLACE
```

## 5. Issue [5]: Missing Gradle Daemon & Memory Tuning

* **Severity:** Medium
* **Location:** `gradle.properties`
* **Description:** The JVM arguments (`org.gradle.jvmargs=-Xmx6g...`) are reasonable for a large project, but we can enable additional optimizations like `org.gradle.daemon=true` (which is default, but good to make explicit) and caching configurations. Furthermore, `org.gradle.caching=true` is enabled, but we should ensure Kotlin build reports and incremental compilation are strictly enforced.
* **Code Fix:** Add Kotlin specific daemon and compilation flags.

```properties
<<<<<<< SEARCH
# Enable caching between builds.
org.gradle.caching=true
=======
# Enable caching between builds.
org.gradle.caching=true

# Kotlin Build Performance
kotlin.incremental.useClasspathSnapshot=true
kotlin.daemon.jvmargs=-Xmx3g -XX:+UseParallelGC
>>>>>>> REPLACE
```
