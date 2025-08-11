Alright — here’s the **final, fully-synced version** of your `rust-android-build/build.gradle.kts`
where the ABI list is pulled directly from your **app module’s `ndk.abiFilters`** if it’s set, so you never duplicate ABI settings between modules.

---

## **`rust-android-build/build.gradle.kts`**

```kotlin
plugins {
    id("com.android.library")
}

android {
    namespace = "com.example.rustandroidbuild" // change to your namespace
    compileSdk = 34

    defaultConfig {
        minSdk = 21
    }

    // Store JNI libs in buildDir instead of src/
    sourceSets["main"].jniLibs.srcDir(layout.buildDirectory.dir("jniLibs"))
    sourceSets["main"].java.srcDirs("")
}

fun abiToTargetTriple(abi: String) = when (abi) {
    "arm64-v8a" -> "aarch64-linux-android"
    "armeabi-v7a" -> "armv7-linux-androideabi"
    "x86_64" -> "x86_64-linux-android"
    else -> error("Unknown ABI: $abi")
}

/**
 * Detect which ABIs to build:
 *  1. If `app` module sets ndk.abiFilters → use that
 *  2. Else if this module sets ndk.abiFilters → use that
 *  3. Else auto-detect based on build type:
 *      - Release builds → all ABIs
 *      - Debug builds → only arm64-v8a for speed
 */
fun detectAbis(): List<String> {
    // Try reading from :app module if present
    val appAbiFilters: Set<String>? = project.rootProject
        .findProject(":app")
        ?.extensions
        ?.findByType(com.android.build.gradle.AppExtension::class.java)
        ?.defaultConfig
        ?.ndk
        ?.abiFilters
        ?.takeIf { it.isNotEmpty() }

    if (appAbiFilters != null) {
        println("Detected ABIs from :app module → $appAbiFilters")
        return appAbiFilters.toList()
    }

    // Fallback: this module's own ABI filters
    val localFilters = android.defaultConfig.ndk.abiFilters
    if (localFilters.isNotEmpty()) {
        println("Detected ABIs from rust-android-build module → $localFilters")
        return localFilters.toList()
    }

    // Auto-detect based on task name
    val taskNames = gradle.startParameter.taskNames.joinToString(" ").lowercase()
    return when {
        taskNames.contains("release") -> listOf("arm64-v8a", "armeabi-v7a", "x86_64")
        taskNames.contains("debug") -> listOf("arm64-v8a")
        else -> listOf("arm64-v8a", "armeabi-v7a", "x86_64")
    }
}

val buildRust by tasks.registering {
    val abis = detectAbis()
    doLast {
        println("Building Rust for ABIs: $abis")
        abis.forEach { abi ->
            val targetTriple = abiToTargetTriple(abi)
            exec {
                workingDir = rootDir.resolve("rust-android-build") // Path to Cargo.toml
                commandLine(
                    "cargo", "ndk",
                    "--target", targetTriple,
                    "build", "--release"
                )
            }
        }
    }
}

val copyRustLibs by tasks.registering(Copy::class) {
    dependsOn(buildRust)
    val abis = detectAbis()
    abis.forEach { abi ->
        val targetTriple = abiToTargetTriple(abi)
        from(rootDir.resolve("rust-android-build/target/$targetTriple/release/libyourlib.so"))
        into(layout.buildDirectory.dir("jniLibs/$abi"))
    }
}

tasks.named("preBuild") {
    dependsOn(copyRustLibs)
}
```

---

### **How it works now**

1. Checks **`app/build.gradle.kts`** for:

   ```kotlin
   android {
       defaultConfig {
           ndk {
               abiFilters += listOf("arm64-v8a", "x86_64")
           }
       }
   }
   ```

   If found, uses that ABI list automatically.

2. If `:app` has no ABI filters, it checks this module’s own config.

3. If neither has ABI filters:

   * **Debug builds** → only `arm64-v8a` (fast).
   * **Release builds** → all ABIs.

---

### **Why this is nice**

* You only set `abiFilters` **once** — in `app`.
* Rust module always matches your final APK’s ABI list.
* Fast debug builds, full release builds.
* No duplication or manual editing when changing ABIs.

---

I can also make this **pick up ABI filters dynamically from any consuming module**, not just `:app`, in case you use this Rust library in multiple Android apps inside the same project.

Do you want me to make that multi-consumer ABI detection?
