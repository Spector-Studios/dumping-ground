Nice — here’s a complete **Kotlin DSL** `build.gradle.kts` for `:rust-android-build` that:

* keeps everything as a *pure native AAR* (no Java/Kotlin, no manifest in source),
* auto-detects ABIs (multi-consumer union),
* **builds each ABI in parallel** by creating independent Exec tasks per ABI,
* copies each ABI’s `.so` into `build/jniLibs/<abi>` (AGP reads that as jniLibs),
* wires everything into AGP so `preBuild` depends on the copy tasks.

> **Note about parallelism:** Gradle will only run independent tasks in parallel if your Gradle configuration allows it. Run with `./gradlew ... --parallel` or set `org.gradle.parallel=true` in `gradle.properties` and give Gradle several workers (`org.gradle.workers.max`). If you want a more robust worker-based implementation (using the Worker API), I can switch to that — it gives better isolation and works even without `--parallel`.

---

## `rust-android-build/build.gradle.kts` (parallel per-ABI tasks)

```kotlin
plugins {
    id("com.android.library")
}

android {
    // Required syntactically; unused for a pure native AAR
    namespace = "com.example.rustplaceholder"
    compileSdk = 34

    defaultConfig {
        minSdk = 21
    }

    // Use build/jniLibs for native output
    sourceSets["main"].jniLibs.srcDir(layout.buildDirectory.dir("jniLibs"))

    // No Java/Kotlin sources
    sourceSets["main"].java.srcDirs(emptyList<String>())

    // No manifest in sources (AGP will auto-generate minimal manifest)
    sourceSets["main"].manifest.srcFile(file("nonexistent.xml"))

    buildFeatures {
        buildConfig = false
        aidl = false
        renderScript = false
        resValues = false
        shaders = false
    }
}

fun abiToTargetTriple(abi: String) = when (abi) {
    "arm64-v8a" -> "aarch64-linux-android"
    "armeabi-v7a" -> "armv7-linux-androideabi"
    "x86_64" -> "x86_64-linux-android"
    else -> error("Unknown ABI: $abi")
}

fun sanitizeTaskNamePart(abi: String) =
    abi.replace(Regex("[^A-Za-z0-9]"), "_") // e.g. arm64-v8a -> arm64_v8a

/**
 * Detect ABIs to build:
 * 1. Collect ABI filters from all Android app modules that depend on this module (union)
 * 2. Else use this module's ndk.abiFilters
 * 3. Else fallback to debug/release defaults
 */
fun detectAbis(): List<String> {
    val appAbiFilters = project.rootProject.subprojects
        .filter { it.plugins.hasPlugin("com.android.application") }
        .filter { appProject ->
            // only apps that depend on this module
            appProject.configurations.any { cfg ->
                cfg.dependencies.any { dep ->
                    dep is ProjectDependency && dep.dependencyProject == project
                }
            }
        }
        .mapNotNull { appProject ->
            appProject.extensions.findByType(
                com.android.build.gradle.AppExtension::class.java
            )?.defaultConfig?.ndk?.abiFilters?.takeIf { it.isNotEmpty() }
        }
        .flatten()
        .toSet()

    if (appAbiFilters.isNotEmpty()) {
        println("Detected ABIs from consuming apps -> $appAbiFilters")
        return appAbiFilters.toList()
    }

    val localFilters = android.defaultConfig.ndk.abiFilters
    if (localFilters.isNotEmpty()) {
        println("Detected ABIs from rust-android-build module -> $localFilters")
        return localFilters.toList()
    }

    val taskNames = gradle.startParameter.taskNames.joinToString(" ").lowercase()
    return when {
        taskNames.contains("release") -> listOf("arm64-v8a", "armeabi-v7a", "x86_64")
        taskNames.contains("debug") -> listOf("arm64-v8a")
        else -> listOf("arm64-v8a", "armeabi-v7a", "x86_64")
    }
}

// === Create per-ABI build tasks (Exec) and per-ABI copy tasks (Copy) ===
val abisToBuild = detectAbis()

// Keep a list of all build tasks and copy tasks so we can dependOn them as a group
val buildTasks = mutableListOf<TaskProvider<out Task>>()
val copyTasks = mutableListOf<TaskProvider<out Task>>()

abisToBuild.forEach { abi ->
    val part = sanitizeTaskNamePart(abi) // e.g. arm64_v8a
    val targetTriple = abiToTargetTriple(abi)

    // Exec task that builds Rust for this ABI
    val buildTaskName = "buildRustFor_${part}"
    val buildTask = tasks.register(buildTaskName, Exec::class) {
        group = "rust"
        description = "Build Rust (cargo ndk) for ABI $abi"
        // Adjust workingDir to the location of Cargo.toml; here we assume project root has rust dir
        workingDir = rootDir.resolve("rust-android-build")
        // Using cargo ndk; adjust args if you use a different workflow/script
        commandLine = listOf(
            "cargo", "ndk",
            "--target", targetTriple,
            "build", "--release"
        )
        // Fail the build on non-zero exit
        isIgnoreExitValue = false
    }
    buildTasks += buildTask

    // Copy task to put the produced .so into build/jniLibs/<abi>
    val copyTaskName = "copyRustLibsFor_${part}"
    val copyTask = tasks.register(copyTaskName, Copy::class) {
        group = "rust"
        description = "Copy Rust .so for ABI $abi into build/jniLibs/$abi"
        dependsOn(buildTask) // ensure build finished for this ABI
        val srcSo = rootDir.resolve("rust-android-build/target/$targetTriple/release/libyourlib.so")
        // If you produce different filename(s), adapt the from() accordingly
        from(srcSo) {
            // fail fast if missing? Copy will be empty if missing; you might want to check existence
        }
        into(layout.buildDirectory.dir("jniLibs/$abi"))
    }
    copyTasks += copyTask
}

// Aggregate tasks that depend on all per-ABI tasks
val buildRustAll = tasks.register("buildRust") {
    group = "rust"
    description = "Build Rust for all detected ABIs: $abisToBuild"
    dependsOn(buildTasks)
}

val copyRustLibsAll = tasks.register("copyRustLibs") {
    group = "rust"
    description = "Copy Rust .so files for all detected ABIs into build/jniLibs/"
    dependsOn(copyTasks)
}

// Make AGP packaging wait for the jni libs to be present
tasks.named("preBuild") {
    dependsOn(copyRustLibsAll)
}
```

---

### Quick checklist & tips

* Replace `"libyourlib.so"` with your actual library file name(s) if different.
* Ensure Cargo & `cargo-ndk` are installed and on PATH for the environment running Gradle.
* To get parallel execution:

  * Run e.g. `./gradlew :rust-android-build:assembleRelease --parallel --max-workers=4` **or**
  * Add `org.gradle.parallel=true` and `org.gradle.workers.max=<n>` to `gradle.properties`.
* If you want the faster Worker API-based approach (more robust and better isolation), I can convert the `Exec` tasks to use `WorkerExecutor` (and handle stdout/stderr capture / reproducibility).

Want me to:

* switch to the Worker API implementation now, or
* adjust the script to build multiple library filenames (if your crate produces multiple `.so`s)?
* 
