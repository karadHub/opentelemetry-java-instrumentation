# Build System Documentation

This document explains the custom build configurations and workarounds in the OpenTelemetry Java Instrumentation project.

## Overview

The project uses Gradle 8.10.1 with Kotlin DSL for build scripts. It supports building with Java 21 while maintaining Java 8 bytecode compatibility for runtime.

## Gradle Properties

### Performance Settings

```properties
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.priority=low
```

**Rationale:** Enables parallel builds and caching for faster build times. Priority is set to "low" to avoid impacting system performance.

### JVM Memory

```properties
org.gradle.jvmargs=-XX:MaxMetaspaceSize=512m
```

**Rationale:** Gradle default of 256m causes build failures. The metaspace size was increased to 512m to handle the large number of modules.

### Dependency Resolution Timeouts

```properties
systemProp.org.gradle.internal.http.connectionTimeout=120000
systemProp.org.gradle.internal.http.socketTimeout=120000
systemProp.org.gradle.internal.repository.max.retries=10
systemProp.org.gradle.internal.repository.initial.backoff=500
```

**Rationale:** Workaround for Maven Central flakiness, particularly for `maven-metadata.xml` files. These settings increase reliability in CI environments.

### Kotlin Incremental Compilation

```properties
kotlin.incremental=false
```

**Rationale:** Workaround for [KT-34862](https://youtrack.jetbrains.com/issue/KT-34862). Incremental compilation is disabled to avoid build failures.

## Build Plugins

### GraalVM Native Image Plugin

The GraalVM plugin is applied with `apply false` at the root level:

```kotlin
id("org.graalvm.buildtools.native") apply false
```

**Rationale:** Workaround for [Gradle issue #17559](https://github.com/gradle/gradle/issues/17559). The plugin cannot be in `pluginManagement` due to metadata service initialization issues.

### Dependency Management

Dependencies are centralized in `dependencyManagement/build.gradle.kts` using BOMs (Bill of Materials):

- Jackson BOM
- Guava BOM
- OpenTelemetry BOMs (SDK and SDK-alpha)
- JUnit BOM
- Testcontainers BOM
- Spock BOM

**Rationale:** Reduces version conflicts and makes dependency management more predictable. This is especially important for IntelliJ IDEA indexing performance.

### Netty Version Alignment

Custom component metadata rules force Netty 4.0.x to 4.0.56.Final and 4.1.x to 4.1.113.Final:

```kotlin
abstract class NettyAlignmentRule : ComponentMetadataRule {
  override fun execute(ctx: ComponentMetadataContext) {
    with(ctx.details) {
      if (id.group == "io.netty" && id.name != "netty") {
        if (id.version.startsWith("4.1.")) {
          belongsTo("io.netty:netty-bom:4.1.113.Final", false)
        } else if (id.version.startsWith("4.0.")) {
          belongsTo("io.netty:netty-bom:4.0.56.Final", false)
        }
      }
    }
  }
}
```

**Rationale:** Netty 4.0 and 4.1 have compatibility issues. This ensures consistent versions within each major branch.

## Java Version Management

### Toolchain Configuration

The project uses Gradle toolchains to manage Java versions:

- **Default Build Version:** Java 21
- **Minimum Supported Runtime:** Java 8
- **Test Matrix:** Java 8, 11, 17, 21, 22

```kotlin
java {
  toolchain {
    languageVersion.set(
      otelJava.minJavaVersionSupported.map {
        val defaultJavaVersion = otelJava.maxJavaVersionSupported.getOrElse(DEFAULT_JAVA_VERSION).majorVersion.toInt()
        JavaLanguageVersion.of(Math.max(it.majorVersion.toInt(), defaultJavaVersion))
      }
    )
  }
}
```

### Bytecode Compatibility

Most modules compile to Java 8 bytecode using the `--release` flag:

```kotlin
tasks.withType<JavaCompile>().configureEach {
  with(options) {
    release.set(otelJava.minJavaVersionSupported.map { it.majorVersion.toInt() })
  }
}
```

**Note:** Some instrumentation modules may require higher Java versions if the instrumented library requires it.

## Gradle Build Cache

### Local Cache

Enabled by default via `org.gradle.caching=true`. Located in `~/.gradle/caches/`.

### Remote Cache (Gradle Enterprise)

The project uses Gradle Enterprise for build scans and remote build cache:

- **Production:** https://ge.opentelemetry.io (requires authentication)
- **Fallback:** https://scans.gradle.com (for unauthenticated CI builds)

Configuration in `settings.gradle.kts`:

```kotlin
val gradleEnterpriseServer = "https://ge.opentelemetry.io"
val isCI = System.getenv("CI") != null
val geAccessKey = System.getenv("GRADLE_ENTERPRISE_ACCESS_KEY") ?: ""
val useScansGradleCom = isCI && geAccessKey.isEmpty()
```

## Common Build Tasks

### Building the Agent

```bash
./gradlew assemble
```

Output: `javaagent/build/libs/opentelemetry-javaagent-<version>.jar`

### Running Tests

```bash
./gradlew test
```

### Running Tests with Latest Dependencies

```bash
./gradlew test -PtestLatestDeps=true
```

### Skipping Tests

```bash
./gradlew build -PskipTests=true
```

### Code Coverage

```bash
./gradlew test jacocoTestReport
```

Output: `build/reports/jacoco/`

### Code Formatting

```bash
./gradlew spotlessApply
```

### Publishing Locally

```bash
./gradlew publishToMavenLocal
```

## Troubleshooting

### Build Fails with "Could not determine the dependencies"

**Symptom:** Error related to GraalVM metadata service.

**Solution:** This is the known issue #17559. The workaround is already in place. Try cleaning the build:

```bash
./gradlew clean build
```

### IntelliJ Takes Forever to Index

**Symptom:** IntelliJ spends excessive time indexing dependencies.

**Solution:** This is due to dependency version explosion. The project already forces specific versions. If issues persist:

1. Invalidate caches: File → Invalidate Caches → Invalidate and Restart
2. Increase IntelliJ memory: Help → Edit Custom VM Options
3. Run `./gradlew intellijDeps` to see dependency tree

### Maven Central Connection Timeouts

**Symptom:** Build fails with connection timeouts to Maven Central.

**Solution:** The workaround is already configured in `gradle.properties`. If issues persist:

1. Check network connectivity
2. Try again (retries are automatic)
3. Use a VPN if Maven Central is blocked

### Kotlin Compilation Errors

**Symptom:** Errors related to incremental compilation.

**Solution:** Incremental compilation is already disabled. If issues persist:

```bash
./gradlew clean build --no-build-cache
```

## Performance Tips

### Use Gradle Daemon

The daemon is enabled by default. Verify with:

```bash
./gradlew --status
```

### Increase Daemon Memory

Edit `~/.gradle/gradle.properties`:

```properties
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=1g
```

### Use Parallel Execution

Already enabled via `org.gradle.parallel=true`.

### Remote JAR Version Numbers (Local Development)

Add to `~/.gradle/gradle.properties`:

```properties
removeJarVersionNumbers=true
```

This keeps the artifact name stable across versions: `opentelemetry-javaagent.jar` instead of `opentelemetry-javaagent-2.8.0-SNAPSHOT.jar`.

## Known Issues and Workarounds

| Issue | Workaround | Tracking |
|-------|------------|----------|
| Gradle #17559 (GraalVM plugin) | `apply false` at root | `build.gradle.kts` comment |
| KT-34862 (Kotlin incremental) | Disabled | `gradle.properties` |
| Maven Central flakiness | Increased timeouts/retries | `gradle.properties` |
| Netty version conflicts | Component metadata rules | `otel.java-conventions.gradle.kts` |

## References

- [Gradle Documentation](https://docs.gradle.org/current/userguide/userguide.html)
- [Gradle Build Cache](https://docs.gradle.org/current/userguide/build_cache.html)
- [Java Toolchains](https://docs.gradle.org/current/userguide/toolchains.html)
- [Gradle Enterprise](https://gradle.com/enterprise/)

---

**Last Updated:** October 8, 2025  
**Gradle Version:** 8.10.1  
**Kotlin Version:** 1.9.24
