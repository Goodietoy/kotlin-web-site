[//]: # (title: Gradle)

In order to build a Kotlin project with Gradle, you should [apply the Kotlin Gradle plugin to your project](#plugin-and-versions)
and [configure the dependencies](#configuring-dependencies).

## Plugin and versions

Apply the Kotlin Gradle plugin by using the [Gradle plugins DSL](https://docs.gradle.org/current/userguide/plugins.html#sec:plugins_block).

The Kotlin Gradle plugin and the `kotlin-multiplatform` plugin %kotlinVersion% require Gradle %minGradleVersion% or later.

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
plugins {
  kotlin("<...>") version "%kotlinVersion%"
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
plugins {
  id 'org.jetbrains.kotlin.<...>' version '%kotlinVersion%'
}
```

</tab>
</tabs>

The placeholder `<...>` should be replaced with the name of one of the plugins that will be discussed in subsequent sections.

## Targeting multiple platforms

Projects targeting [multiple platforms](mpp-supported-platforms.md), called [multiplatform projects](mpp-get-started.md),
require the `kotlin-multiplatform` plugin. [Learn more about the plugin](mpp-discover-project.md#multiplatform-plugin).

>The `kotlin-multiplatform` plugin works with Gradle %minGradleVersion% or later.
>
{type="note"}

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
plugins {
  kotlin("multiplatform") version "%kotlinVersion%"
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
plugins {
  id 'org.jetbrains.kotlin.multiplatform' version '%kotlinVersion%'
}
```

</tab>
</tabs>

## Targeting the JVM

To target the JVM, apply the Kotlin JVM plugin.

<tabs group="build-script">
    <tab title="Kotlin" group-key="kotlin">

```kotlin
plugins {
    kotlin("jvm") version "%kotlinVersion%"
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
plugins {
    id "org.jetbrains.kotlin.jvm" version "%kotlinVersion%"
}
```

</tab>
</tabs>

The `version` should be literal in this block, and it cannot be applied from another build script.

Alternatively, you can use the older `apply plugin` approach:

```groovy
apply plugin: 'kotlin'
```

Applying Kotlin plugins with `apply` in the Kotlin Gradle DSL is not recommended – [see why](#using-the-gradle-kotlin-dsl).

### Kotlin and Java sources

Kotlin sources and Java sources can be stored in the same folder, or they can be placed in different folders. The default convention is to use different folders:

```groovy
project
    - src
        - main (root)
            - kotlin
            - java
```

The corresponding `sourceSets` property should be updated if you are not using the default convention:

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
sourceSets.main {
    java.srcDirs("src/main/myJava", "src/main/myKotlin")
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
sourceSets {
    main.kotlin.srcDirs += 'src/main/myKotlin'
    main.java.srcDirs += 'src/main/myJava'
}
```

</tab>
</tabs>

### Check for JVM target compatibility of related compile tasks

In the build module, you may have related compile tasks, for example:
* `compileKotlin` and `compileJava`
* `compileTestKotlin` and `compileTestJava`

> `main` and `test` source set compile tasks are not related.
>
{type="note"}

For such related tasks, the Kotlin Gradle plugin checks for JVM target compatibility. Different values of `jvmTarget` in the `kotlin` extension
and [`targetCompatibility`](https://docs.gradle.org/current/userguide/java_plugin.html#sec:java-extension)
in the `java` extension cause incompatibility. For example:
the `compileKotlin` task has `jvmTarget=1.8`, and
the `compileJava` task has (or [inherits](https://docs.gradle.org/current/userguide/java_plugin.html#sec:java-extension)) `targetCompatibility=15`.

Control the behavior of this check by setting the `kotlin.jvm.target.validation.mode` property in the `build.gradle` file equal to:
* `warning` – the default value; the Kotlin Gradle plugin will print a warning message.
* `error` – the plugin will fail the build.
* `ignore` – the plugin will skip the check and won't produce any messages.

### Set custom JDK home

By default, Kotlin compile tasks use the current Gradle JDK. 
If you need to change the JDK by some reason, you can set the JDK home in the following ways:
* For Gradle 6.7 and later – with [Java toolchains](#gradle-java-toolchains-support) or the [Task DSL](#setting-jdk-version-with-the-task-dsl) to set a local JDK.
* For earlier Gradle versions without Java toolchains (up to 6.6) – with the [`UsesKotlinJavaToolchain` interface and the Task DSL](#setting-jdk-version-with-the-task-dsl).

> The `jdkHome` compiler option is deprecated since Kotlin 1.5.30.
>
{type="warning"}

When you use a custom JDK, note that [kapt task workers](kapt.md#running-kapt-tasks-in-parallel)
use [process isolation mode](https://docs.gradle.org/current/userguide/worker_api.html#changing_the_isolation_mode) only,
and ignore the `kapt.workers.isolation` property.

### Gradle Java toolchains support

Gradle 6.7 introduced [Java toolchains support](https://docs.gradle.org/current/userguide/toolchains.html).
Using this feature, you can:
* Use a JDK and a JRE that are different from the Gradle ones to run compilations, tests, and executables.
* Compile and test code with a not-yet-released language version.

With toolchains support, Gradle can autodetect local JDKs and install missing JDKs that Gradle requires for the build.
Now Gradle itself can run on any JDK and still reuse the [remote build cache feature](#gradle-build-cache-support)
for tasks that depend on a major JDK version.

The Kotlin Gradle plugin supports Java toolchains for Kotlin/JVM compilation tasks. JS and Native tasks don't use toolchains.
The Kotlin compiler always uses the JDK the Gradle daemon is running on.
A Java toolchain:
* Sets the [`jdkHome` option](#attributes-specific-to-jvm) available for JVM targets.
* Sets the [`kotlinOptions.jvmTarget`](#attributes-specific-to-jvm) to the toolchain's JDK version
  if the user doesn't set the `jvmTarget` option explicitly.
  If the user doesn't configure the toolchain, the `jvmTarget` field will use the default value.
  Learn more about [JVM target compatibility](#check-for-jvm-target-compatibility-of-related-compile-tasks).
* Affects which JDK [`kapt` workers](kapt.md#running-kapt-tasks-in-parallel) are running on.

Use the following code to set a toolchain. Replace the placeholder `<MAJOR_JDK_VERSION>` with the JDK version you would like to use:

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
kotlin {
    jvmToolchain {
        (this as JavaToolchainSpec).languageVersion.set(JavaLanguageVersion.of(<MAJOR_JDK_VERSION>)) // "8" 
    }
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
kotlin {
    jvmToolchain {
        languageVersion.set(JavaLanguageVersion.of(<MAJOR_JDK_VERSION>)) // "8"
    }
}
```

</tab>
</tabs>

Note that setting a toolchain via the `kotlin` extension will update the toolchain for Java compile tasks as well.

> To understand which toolchain Gradle uses, run your Gradle build with the [log level `--info`](https://docs.gradle.org/current/userguide/logging.html#sec:choosing_a_log_level)
> and find a string in the output starting with `[KOTLIN] Kotlin compilation 'jdkHome' argument:`.
> The part after the colon will be the JDK version from the toolchain.
>
{type="note"}

To set any JDK (even local) for the specific task, use the Task DSL.

### Setting JDK version with the Task DSL

If you use the a Gradle version earlier than 6.7, there is no [Java toolchains support](#gradle-java-toolchains-support). 
You can use the Task DSL that allows setting any JDK version for any task implementing the `UsesKotlinJavaToolchain` interface.
At the moment, these tasks are `KotlinCompile` and `KaptTask`.
If you want Gradle to search for the major JDK version, replace the `<MAJOR_JDK_VERSION>` placeholder in your build script:

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
val service = project.extensions.getByType<JavaToolchainService>()
val customLauncher = service.launcherFor {
    it.languageVersion.set(JavaLanguageVersion.of(<MAJOR_JDK_VERSION>)) // "8"
}
project.tasks.withType<UsesKotlinJavaToolchain>().configureEach {
    kotlinJavaToolchain.toolchain.use(customLauncher)
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
JavaToolchainService service = project.getExtensions().getByType(JavaToolchainService.class)
Provider<JavaLauncher> customLauncher = service.launcherFor {
    it.languageVersion.set(JavaLanguageVersion.of(<MAJOR_JDK_VERSION>)) // "8"
}
tasks.withType(UsesKotlinJavaToolchain::class).configureEach { task ->
    task.kotlinJavaToolchain.toolchain.use(customLauncher)
}
```

</tab>
</tabs>

Or you can specify the path to your local JDK and replace the placeholder `<LOCAL_JDK_VERSION>` with this JDK version:

```kotlin
tasks.withType<UsesKotlinJavaToolchain>().configureEach {
    kotlinJavaToolchain.jdk.use(
        "/path/to/local/jdk", // Put a path to your JDK
        JavaVersion.<LOCAL_JDK_VERSION> // For example, JavaVersion.17
    )
}
```

## Targeting JavaScript

When targeting only JavaScript, use the `kotlin-js` plugin. [Learn more](js-project-setup.md)

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
plugins {
    kotlin("js") version "%kotlinVersion%"
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
plugins {
    id 'org.jetbrains.kotlin.js' version '%kotlinVersion%'
}
```

</tab>
</tabs>

### Kotlin and Java sources for JavaScript

This plugin only works for Kotlin files, so it is recommended that you keep Kotlin and Java files separate (if the
project contains Java files). If you don't store them separately, specify the source folder in the `sourceSets` block:

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
kotlin {
    sourceSets["main"].apply {    
        kotlin.srcDir("src/main/myKotlin") 
    }
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
kotlin {
    sourceSets {
        main.kotlin.srcDirs += 'src/main/myKotlin'
    }
}
```

</tab>
</tabs>

## Targeting Android

It's recommended to use Android Studio for creating Android applications. [Learn how to use Android Gradle plugin](https://developer.android.com/studio/releases/gradle-plugin).

## Configuring dependencies

To add a dependency on a library, set the dependency of the required [type](#dependency-types) (for example, `implementation`) in the
`dependencies` block of the source sets DSL.

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
kotlin {
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("com.example:my-library:1.0")
            }
        }
    }
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
kotlin {
    sourceSets {
        commonMain {
            dependencies {
                implementation 'com.example:my-library:1.0'
            }
        }
    }
}
```

</tab>
</tabs>

Alternatively, you can [set dependencies at the top level](#set-dependencies-at-the-top-level).

### Dependency types

Choose the dependency type based on your requirements.

<table>
    <tr>
        <th>Type</th>
        <th>Description</th>
        <th>When to use</th>
    </tr>
    <tr>
        <td><code>api</code></td>
        <td>Used both during compilation and at runtime and is exported to library consumers.</td>
        <td>If any type from a dependency is used in the public API of the current module, use an <code>api</code> dependency.
        </td>
    </tr>
    <tr>
        <td><code>implementation</code></td>
        <td>Used during compilation and at runtime for the current module, but is not exposed for compilation of other modules
            depending on the one with the `implementation` dependency.</td>
        <td>
            <p>Use for dependencies needed for the internal logic of a module.</p>
            <p>If a module is an endpoint application which is not published, use <code>implementation</code> dependencies instead
                of <code>api</code> dependencies.</p>
        </td>
    </tr>
    <tr>
        <td><code>compileOnly</code></td>
        <td>Used for compilation of the current module and is not available at runtime nor during compilation of other modules.</td>
        <td>Use for APIs which have a third-party implementation available at runtime.</td>
    </tr>
    <tr>
        <td><code>runtimeOnly</code></td>
        <td>Available at runtime but is not visible during compilation of any module.</td>
        <td></td>
    </tr>
</table>

### Dependency on the standard library

A dependency on the standard library (`stdlib`) is added automatically to each source set. The version
of the standard library used is the same as the version of the Kotlin Gradle plugin.

For platform-specific source sets, the corresponding platform-specific variant of the library is used, while a common standard
library is added to the rest. The Kotlin Gradle plugin will select the appropriate JVM standard library depending on
the `kotlinOptions.jvmTarget` [compiler option](#compiler-options) of your Gradle build script.

If you declare a standard library dependency explicitly (for example, if you need a different version), the Kotlin Gradle
plugin won't override it or add a second standard library.

If you do not need a standard library at all, you can add the opt-out option to the `gradle.properties`:

```kotlin
kotlin.stdlib.default.dependency=false
```

### Set dependencies on test libraries

The [`kotlin.test`](https://kotlinlang.org/api/latest/kotlin.test/) API is available for testing Kotlin projects on
all supported platforms.
Add the dependency `kotlin-test` to the `commonTest` source set, and the Gradle plugin will infer the corresponding
test dependencies for each test source set:
* `kotlin-test-common` and `kotlin-test-annotations-common` for common source sets
* `kotlin-test-junit` for JVM source sets
* `kotlin-test-js` for Kotlin/JS source sets

Kotlin/Native targets do not require additional test dependencies, and the `kotlin.test` API implementations are built-in.

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
kotlin {
    sourceSets {
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test")) // This brings all the platform dependencies automatically
            }
        }
    }
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
kotlin {
    sourceSets {
        commonTest {
            dependencies {
                implementation kotlin("test") // This brings all the platform dependencies automatically
            }
        }
    }
}
```

</tab>
</tabs>

> You can use shorthand for a dependency on a Kotlin module, for example, kotlin("test") for "org.jetbrains.kotlin:kotlin-test".
>
{type="note"}

You can use the `kotlin-test` dependency in any shared or platform-specific source set as well.

For Kotlin/JVM, Gradle uses JUnit 4 by default. Therefore, the `kotlin("test")` dependency resolves to the variant for
JUnit 4, namely `kotlin-test-junit`.

You can choose JUnit 5 or TestNG by calling
[`useJUnitPlatform()`]( https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/testing/Test.html#useJUnitPlatform)
or [`useTestNG()`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/testing/Test.html#useTestNG) in the
test task of your build script.
The following example is for an MPP project:

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
kotlin {
    jvm {
        testRuns["test"].executionTask.configure {
            useJUnitPlatform()
        }
    }
    sourceSets {
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
            }
        }
    }
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
kotlin {
    jvm {
        testRuns["test"].executionTask.configure {
            useJUnitPlatform()
        }
    }
    sourceSets {
        commonTest {
            dependencies {
                implementation kotlin("test")
            }
        }
    }
}
```

</tab>
</tabs>

The following example is for a JVM project:

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
dependencies {
    testImplementation(kotlin("test"))
}

tasks {
    test {
        useTestNG()
    }
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
dependencies {
    testImplementation 'org.jetbrains.kotlin:kotlin-test'
}

test {
    useTestNG()
}
```

</tab>
</tabs>

[Learn how to test code using JUnit on the JVM](jvm-test-using-junit.md).

If you need to use a different JVM test framework, disable automatic testing framework selection by
adding the line `kotlin.test.infer.jvm.variant=false` to the project's `gradle.properties` file.
After doing this, add the framework as a Gradle dependency.

If you had used a variant of `kotlin("test")` in your build script explicitly and project build stopped working with
a compatibility conflict,
see [this issue in the Compatibility Guide](compatibility-guide-15.md#do-not-mix-several-jvm-variants-of-kotlin-test-in-a-single-project).

### Set a dependency on a kotlinx library

If you use a kotlinx library and need a platform-specific dependency, you can use platform-specific variants
of libraries with suffixes such as `-jvm` or `-js`, for example, `kotlinx-coroutines-core-jvm`. You can also use the library's
base artifact name instead – `kotlinx-coroutines-core`.

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
kotlin {
    sourceSets {
        val jvmMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core-jvm:%coroutinesVersion%")
            }
        }
    }
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
kotlin {
    sourceSets {
        jvmMain {
            dependencies {
                implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core-jvm:%coroutinesVersion%'
            }
        }
    }
}
```

</tab>
</tabs>

If you use a multiplatform library and need to depend on the shared code, set the dependency only once, in the shared
source set. Use the library's base artifact name, such as `kotlinx-coroutines-core` or `ktor-client-core`.

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
kotlin {
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:%coroutinesVersion%")
            }
        }
    }
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
kotlin {
    sourceSets {
        commonMain {
            dependencies {
                implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:%coroutinesVersion%'
            }
        }
    }
}
```

</tab>
</tabs>

### Set dependencies at the top level

Alternatively, you can specify the dependencies at the top level, using the following pattern for the configuration names:
`<sourceSetName><DependencyType>`. This can be helpful for some Gradle built-in dependencies, like `gradleApi()`, `localGroovy()`,
or `gradleTestKit()`, which are not available in the source sets' dependency DSL.

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
dependencies {
    "commonMainImplementation"("com.example:my-library:1.0")
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
dependencies {
    commonMainImplementation 'com.example:my-library:1.0'
}
```

</tab>
</tabs>

## Annotation processing

Kotlin supports annotation processing via the Kotlin annotation processing tool [`kapt`](kapt.md).

## Incremental compilation

The Kotlin Gradle plugin supports incremental compilation. Incremental compilation tracks changes to source files between
builds so only files affected by these changes are compiled.

Incremental compilation is supported for Kotlin/JVM and Kotlin/JS projects and is enabled by default.

There are several ways to switch off incremental compilation:

* `kotlin.incremental=false` for Kotlin/JVM.
* `kotlin.incremental.js=false` for Kotlin/JS projects.
* Use `-Pkotlin.incremental=false` or `-Pkotlin.incremental.js=false` as a command line parameter.

  The parameter should be added to each subsequent build, and any build with incremental
  compilation disabled invalidates incremental caches.

The first build is never incremental.

## Gradle build cache support

The Kotlin plugin uses the [Gradle build cache](https://docs.gradle.org/current/userguide/build_cache.html), which stores
the build outputs for reuse in future builds.

To disable caching for all Kotlin tasks, set the system property flag `kotlin.caching.enabled` to `false`
(run the build with the argument `-Dkotlin.caching.enabled=false`).

If you use [kapt](kapt.md), note that kapt annotation processing tasks are not cached by default. However, you can
[enable caching for them manually](kapt.md#gradle-build-cache-support).

## Gradle configuration cache support

> The configuration cache is available in Gradle 6.5 and later as an experimental feature.
> You can check the [Gradle releases page](https://gradle.org/releases/) to see whether it has been promoted to stable.
>
{type="note"}

The Kotlin plugin uses the [Gradle configuration cache](https://docs.gradle.org/current/userguide/configuration_cache.html),
which speeds up the build process by reusing the results of the configuration phase.

See the [Gradle documentation](https://docs.gradle.org/current/userguide/configuration_cache.html#config_cache:usage)
to learn how to enable the configuration cache. After you enable this feature, the Kotlin Gradle plugin will automatically
start using it.

## Compiler options

Use the `kotlinOptions` property of a Kotlin compilation task to specify additional compilation options.

When targeting the JVM, the tasks are called `compileKotlin` for production code and `compileTestKotlin`
for test code. The tasks for custom source sets are named according to their `compile<Name>Kotlin` patterns.

The names of the tasks in Android Projects contain [build variant](https://developer.android.com/studio/build/build-variants.html) names and follow the `compile<BuildVariant>Kotlin` pattern, for example, `compileDebugKotlin` or `compileReleaseUnitTestKotlin`.

When targeting JavaScript, the tasks are called `compileKotlinJs` for production code and `compileTestKotlinJs` for test code, and `compile<Name>KotlinJs` for custom source sets.

To configure a single task, use its name. Examples:

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile
// ...

val compileKotlin: KotlinCompile by tasks

compileKotlin.kotlinOptions.suppressWarnings = true
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
compileKotlin {
    kotlinOptions.suppressWarnings = true
}

//or

compileKotlin {
    kotlinOptions {
        suppressWarnings = true
    }
}
```

</tab>
</tabs>

Note that with the Gradle Kotlin DSL, you should get the task from the project's `tasks` first.

Use the `Kotlin2JsCompile` and `KotlinCompileCommon` types for JS and common targets, respectively.

It is also possible to configure all of the Kotlin compilation tasks in the project:

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile>().configureEach {
    kotlinOptions { /*...*/ }
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
tasks.withType(org.jetbrains.kotlin.gradle.tasks.KotlinCompile).configureEach {
    kotlinOptions { /*...*/ }
}
```
</tab>
</tabs>

Here is a complete list of options for Gradle tasks:

### Attributes common to JVM, JS, and JS DCE

| Name | Description | Possible values |Default value |
|------|-------------|-----------------|--------------|
| `allWarningsAsErrors` | Report an error if there are any warnings |  | false |
| `suppressWarnings` | Don't generate warnings |  | false |
| `verbose` | Enable verbose logging output. Works only when the [Gradle debug log level enabled](https://docs.gradle.org/current/userguide/logging.html) |  | false |
| `freeCompilerArgs` | A list of additional compiler arguments |  | [] |

### Attributes common to JVM and JS

| Name | Description | Possible values |Default value |
|------|-------------|-----------------|--------------|
| `apiVersion` | Restrict the use of declarations to those from the specified version of bundled libraries | "1.3" (DEPRECATED), "1.4" (DEPRECATED), "1.5", "1.6", "1.7" (EXPERIMENTAL) |  |
| `languageVersion` | Provide source compatibility with the specified version of Kotlin | "1.4" (DEPRECATED), "1.5", "1.6", "1.7" (EXPERIMENTAL) |  |

### Attributes specific to JVM

| Name | Description | Possible values |Default value |
|------|-------------|-----------------|--------------|
| `javaParameters` | Generate metadata for Java 1.8 reflection on method parameters |  | false |
| `jdkHome` | Include a custom JDK from the specified location into the classpath instead of the default JAVA_HOME. Direct setting is deprecated sinсe 1.5.30, use [other ways to set this option](#set-custom-jdk-home).  |  |  |
| `jvmTarget` | Target version of the generated JVM bytecode | "1.6" (DEPRECATED), "1.8", "9", "10", "11", "12", "13", "14", "15", "16", "17" | "%defaultJvmTargetVersion%" |
| `noJdk` | Don't automatically include the Java runtime into the classpath |  | false |
| `useOldBackend` | Use the [old JVM backend](whatsnew15.md#stable-jvm-ir-backend) |  | false |

### Attributes specific to JS

| Name | Description | Possible values |Default value |
|------|-------------|-----------------|--------------|
| `friendModulesDisabled` | Disable internal declaration export |  | false |
| `main` | Define whether the `main` function should be called upon execution | "call", "noCall" | "call" |
| `metaInfo` | Generate .meta.js and .kjsm files with metadata. Use to create a library |  | true |
| `moduleKind` | The kind of JS module generated by the compiler | "umd", "commonjs", "amd", "plain"  | "umd" |
| `noStdlib` | Don't automatically include the default Kotlin/JS stdlib in compilation dependencies |  | true |
| `outputFile` | Destination *.js file for the compilation result |  | "\<buildDir>/js/packages/\<project.name>/kotlin/\<project.name>.js" |
| `sourceMap` | Generate source map |  | true |
| `sourceMapEmbedSources` | Embed source files into the source map | "never", "always", "inlining" |  |
| `sourceMapPrefix` | Add the specified prefix to paths in the source map |  |  |
| `target` | Generate JS files for specific ECMA version | "v5" | "v5" |
| `typedArrays` | Translate primitive arrays to JS typed arrays |  | true |

## Generating documentation

To generate documentation for Kotlin projects, use [Dokka](https://github.com/Kotlin/dokka);
please refer to the [Dokka README](https://github.com/Kotlin/dokka/blob/master/README.md#using-the-gradle-plugin)
for configuration instructions. Dokka supports mixed-language projects and can generate output in multiple
formats, including standard JavaDoc.

## OSGi

For OSGi support see the [Kotlin OSGi page](kotlin-osgi.md).

## Using the Gradle Kotlin DSL

When using [Gradle Kotlin DSL](https://github.com/gradle/kotlin-dsl), apply Kotlin plugins using the `plugins { ... }` block. 
If you apply them with `apply { plugin(...) }` instead, you may encounter unresolved references to the extensions generated 
by Gradle Kotlin DSL. To resolve that, you can comment out the erroneous usages, run the Gradle task `kotlinDslAccessorsSnapshot`, 
then uncomment the usages back and rerun the build or reimport the project into the IDE.

## Kotlin daemon and using it with Gradle

The Kotlin daemon:
* Runs along with the Gradle daemon to compile the project.
* Runs separately when you compile the project with an IntelliJ IDEA built-in build system.

The Kotlin daemon starts at the Gradle [execution stage](https://docs.gradle.org/current/userguide/build_lifecycle.html#sec:build_phases)
when one of Kotlin compile tasks starts compiling the sources.
The Kotlin daemon stops along with the Gradle daemon or after two idle hours with no Kotlin compilation.

The Kotlin daemon uses the same JDK that the Gradle daemon does.

### Setting Kotlin daemon's JVM arguments

Each of the options in the following list overrides the ones that came before it:
* If nothing is specified, the Kotlin daemon inherits arguments from the Gradle daemon.
  For example, in the `gradle.properties` file:

 ```properties
  org.gradle.jvmargs=-Xmx1500m -Xms=500m
  ```

* If the Gradle daemon's JVM arguments have the `kotlin.daemon.jvm.options` system property – use it in the `gradle.properties` file:

 ```properties
  org.gradle.jvmargs=-Dkotlin.daemon.jvm.options=-Xmx1500m -Xms=500m
  ```

* You can add the`kotlin.daemon.jvmargs` property in the `gradle.properties` file:

 ```properties
  kotlin.daemon.jvmargs=-Xmx1500m -Xms=500m
  ```

* You can specify arguments in the `kotlin` extension:

  <tabs group="build-script">
  <tab title="Kotlin" group-key="kotlin">
  
  ```kotlin
  kotlin {
      kotlinDaemonJvmArgs = listOf("-Xmx486m", "-Xms256m", "-XX:+UseParallelGC")
  }
  ```

  </tab>
  <tab title="Groovy" group-key="groovy">

  ```groovy
  kotlin {
      kotlinDaemonJvmArgs = ["-Xmx486m", "-Xms256m", "-XX:+UseParallelGC"]
  }
  ```
  
  </tab>
  </tabs>

* You can specify arguments for a specific task:

  <tabs group="build-script">
  <tab title="Kotlin" group-key="kotlin">
  
  ```kotlin
  tasks.withType<CompileUsingKotlinDaemon>().configureEach {
      kotlinDaemonJvmArguments.set(listOf("-Xmx486m", "-Xms256m", "-XX:+UseParallelGC"))
      }
  ```
  
  </tab>
  <tab title="Groovy" group-key="groovy">

  ```groovy
  tasks.withType(CompileUsingKotlinDaemon::class).configureEach { task ->
      task.kotlinDaemonJvmArguments.set(["-Xmx1g", "-Xms512m"])
      }
  ```
  
  </tab>
  </tabs>

  > In this case a new Kotlin daemon instance can start on task execution. Learn more about [Kotlin daemon's behavior with JVM arguments](#kotlin-daemon-s-behavior-with-jvm-arguments).
  >
  {type="note"}

#### Kotlin daemon's behavior with JVM arguments

When configuring the Kotlin daemon's JVM arguments, note that:

* It is expected to have multiple instances of the Kotlin daemon running at the same time when different subprojects or tasks have different sets of JVM arguments.
* A new Kotlin daemon instance starts only when Gradle runs a related compilation task and existing Kotlin daemons do not have the same set of JVM arguments.
  Imagine that your project has a lot of subprojects. Most of them require some heap memory for a Kotlin daemon, but one module requires a lot (though it is rarely compiled).
  In this case, you should provide a different set of JVM arguments for such a module, so a Kotlin daemon with a larger heap size would start only for developers who touch this specific module.
  > If you are already running a Kotlin daemon that has enough heap size to handle the compilation request,
  > even if other requested JVM arguments are different, this daemon will be reused instead of starting a new one.
  >
  {type="note"}
* If the `Xmx` is not specified, the Kotlin daemon will inherit it from the Gradle daemon.
