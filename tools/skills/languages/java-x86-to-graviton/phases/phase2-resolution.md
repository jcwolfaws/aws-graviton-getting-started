# Phase 2: Compatibility Resolution

Apply fixes for ARM64-blocking issues identified in Phase 1. Update ONLY what is required for ARM64 compatibility.

## 2.1 Native Library Resolution

> **Output: update `graviton-validation/02-native-library-report.md`** (Resolution Details, Pure Java Fallbacks)

For each x86-only .so identified in Phase 1:

**If source code available:**
```bash
# Cross-compilation
CC=aarch64-linux-gnu-gcc make

# Or on ARM64 machine (distro defaults are correct for Graviton)
make

# Verify
file libname.so  # Should show "ARM aarch64"
```

**If source unavailable:** Prompt user for ARM64-compatible version or confirm pure Java fallback.

**Update native library loading logic** to handle ARM64:
```java
String arch = System.getProperty("os.arch");
String libName;

if ("aarch64".equals(arch)) {
    libName = "libchatbot-arm64.so";
} else if ("amd64".equals(arch) || "x86_64".equals(arch)) {
    libName = "libchatbot-amd64.so";
} else {
    throw new UnsupportedOperationException("Unsupported architecture: " + arch);
}

try {
    nativeLib = (NativeLib) Native.load(libName, NativeLib.class);
} catch (UnsatisfiedLinkError e) {
    logger.warn("Native library not available, using Java fallback", e);
}
```

## 2.2 Dependency Compatibility Updates

> **Output: update `graviton-validation/03-dependency-compatibility-report.md`**

Update ONLY dependencies flagged as MUST UPGRADE. Do NOT upgrade compatible dependencies.

**Direct dependencies (Maven):**
```xml
<dependency>
    <groupId>net.java.dev.jna</groupId>
    <artifactId>jna</artifactId>
    <version>5.14.0</version> <!-- Minimum 5.8.0 for ARM64 -->
</dependency>
```

**Transitive dependencies (Maven):**
```xml
<!-- Option A: dependencyManagement override -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.xerial.snappy</groupId>
            <artifactId>snappy-java</artifactId>
            <version>1.1.10.5</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Option B: Exclude and re-add -->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>${kafka.version}</version>
    <exclusions>
        <exclusion>
            <groupId>org.xerial.snappy</groupId>
            <artifactId>snappy-java</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.xerial.snappy</groupId>
    <artifactId>snappy-java</artifactId>
    <version>1.1.10.5</version>
</dependency>
```

**Transitive dependencies (Gradle):**
```groovy
configurations.all {
    resolutionStrategy {
        force 'org.xerial.snappy:snappy-java:1.1.10.5'
    }
}
```

## 2.3 Architecture Detection Code Updates

> **Output: update `graviton-validation/04-code-scan-findings.md`** (Changes Applied)

Add `aarch64` handling to all architecture detection:
```java
// Before
if (System.getProperty("os.arch").equals("amd64")) {
    // x86-specific code
}

// After
String arch = System.getProperty("os.arch");
if ("aarch64".equals(arch)) {
    // ARM64-specific code path
} else if ("amd64".equals(arch) || "x86_64".equals(arch)) {
    // x86-specific code
} else {
    // Generic fallback
}
```

## 2.4 Build Configuration Updates

**Maven ARM64 profile:**
```xml
<profiles>
    <profile>
        <id>arm64</id>
        <activation>
            <os><arch>aarch64</arch></os>
        </activation>
        <properties>
            <native.arch>aarch64</native.arch>
        </properties>
    </profile>
</profiles>
```

**Dockerfile updates (containerized deployments only):**

PRESERVE the current base image distribution and version. Do NOT change JDK distribution or version.

Single-stage:
```dockerfile
FROM --platform=$BUILDPLATFORM <current-base-image>:<current-version>
```

Multi-stage:
```dockerfile
# Builder: BUILDPLATFORM for native build speed
FROM --platform=$BUILDPLATFORM <current-builder-image>:<current-version> AS builder
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

# Runtime: TARGETPLATFORM for ARM64 execution
FROM --platform=$TARGETPLATFORM <current-runtime-image>:<current-version>
COPY --from=builder /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Key: Builder stage uses `$BUILDPLATFORM`, runtime stage uses `$TARGETPLATFORM`. If original has no `--platform` annotations, add them. Host-based deployments skip Docker steps.

## 2.5 Graviton-Specific JVM Recommendations

> **Output: `graviton-validation/05-jvm-configuration.md`**

Do NOT apply JVM flags automatically. Graviton runs well with default JVM settings. Document recommendations in the report for the team to evaluate during performance testing.

**JDK 21+:** Most Graviton optimizations are already enabled by default (AES-CTR intrinsics, improved tiered compilation). No flags to recommend unless performance testing shows a specific issue.

**JDK 11-17:** Document the following as recommendations to try if performance testing shows a regression:

| Flag | Workload type | Notes |
|------|---------------|-------|
| `-XX:+UnlockDiagnosticVMOptions -XX:+UseAESCTRIntrinsics` | Crypto-heavy (TLS termination, encryption) | Enabled by default in 21+ |
| `-XX:-TieredCompilation` | Long-running services (not latency-sensitive startup) | Trades startup time for steady-state throughput |
| `-XX:ReservedCodeCacheSize=64M` | Large apps with many classes | Only if code cache pressure observed |

The report should note the project's JDK version and which recommendations (if any) are relevant to its workload type.
