# Phase 1: Static Compatibility Analysis

Analyze the project without making changes. All findings are documented in `graviton-validation/` files.

## 1.1 Project Structure Analysis

> **Output: `graviton-validation/01-project-assessment.md`** and **`graviton-validation/raw/dependency-tree-full.txt`**

### Determine Deployment Type

- Dockerfile or container config present: **Containerized**
- systemd/init scripts or direct JAR/WAR: **Host-based**
- Some applications support both

### Detect Multi-Module Structure

- Maven: parent POM with `<modules>` section
- Gradle: `settings.gradle` / `settings.gradle.kts` with `include` directives
- If multi-module: enumerate all submodules, analyze each independently
- Native libraries may reside in child modules; do NOT limit to root build file

### Generate Dependency Tree

```bash
# Maven
mvn dependency:tree -DoutputType=text > graviton-validation/raw/dependency-tree-full.txt

# Gradle
./gradlew dependencies --configuration runtimeClasspath > graviton-validation/raw/dependency-tree-full.txt
```

### Categorize Components by Risk

- **CRITICAL**: Native code (JNI/JNA), .so files, architecture-specific optimizations
- **HIGH**: Dependencies with known x86-only versions, crypto libraries
- **MEDIUM**: Build configs, deployment scripts, architecture detection code
- **LOW**: Pure Java code without architecture dependencies

## 1.2 Native Library Validation (.so File Analysis)

> **Output: `graviton-validation/02-native-library-report.md`**

Native libraries incompatible with ARM64 exist in two forms: **statically bundled** inside JARs and **dynamically extracted at runtime**. Both must be validated.

### 1.2.1 Statically Bundled .so Scanning

Scan all JARs for native libraries, including nested JARs in fat/uber JARs:

```bash
tmpdir=$(mktemp -d)
trap "rm -rf $tmpdir" EXIT

# Standard JARs
for jar in $(find . -name "*.jar" -not -path "*/build/*" -not -path "*/target/*"); do
  echo "=== Scanning: $jar ==="
  unzip -l "$jar" 2>/dev/null | grep "\.so$"
done

# Spring Boot fat JARs (nested JARs in BOOT-INF/lib/)
for jar in $(find . -name "*.jar" -not -path "*/build/*" -not -path "*/target/*"); do
  nested=$(unzip -l "$jar" 2>/dev/null | grep "BOOT-INF/lib/.*\.jar$" | awk '{print $NF}')
  if [ -n "$nested" ]; then
    echo "=== Fat JAR detected: $jar ==="
    unzip -q "$jar" -d "$tmpdir/fatjar" 2>/dev/null
    for nested_jar in $(find "$tmpdir/fatjar/BOOT-INF/lib" -name "*.jar" 2>/dev/null); do
      so_files=$(unzip -l "$nested_jar" 2>/dev/null | grep "\.so$")
      if [ -n "$so_files" ]; then
        echo "--- Nested JAR: $(basename $nested_jar) ---"
        echo "$so_files"
      fi
    done
    rm -rf "$tmpdir/fatjar"
  fi
done

# Validate architecture of discovered .so files
for jar in $(find . -name "*.jar" -not -path "*/build/*" -not -path "*/target/*"); do
  unzip -q "$jar" -d "$tmpdir/extract" 2>/dev/null
  find "$tmpdir/extract" -name "*.so" -exec file {} \;
  rm -rf "$tmpdir/extract"
done
```

**Fat/Uber JAR types:** Spring Boot fat JARs have native libs in `BOOT-INF/lib/`. Maven Shade and Gradle Shadow flatten into single JARs (standard scan covers them). WAR files: check `WEB-INF/lib/`.

### 1.2.2 Runtime-Extracted Native Library Detection

Some libraries extract native code at runtime rather than bundling .so files.

```bash
# Source code patterns
grep -rn "Native.load\|Native.loadLibrary\|System.loadLibrary\|System.load(" \
  --include="*.java" src/ || true

grep -rn "jnr\.\|LibraryLoader" --include="*.java" src/ || true

# Build file dependencies known to extract native code
grep -n "netty-transport-native\|netty-tcnative\|io.grpc.*netty\|conscrypt\|jnr-ffi\|jnr-posix\|leveldbjni\|rocksdbjni\|sqlite-jdbc\|lz4-java\|zstd-jni\|snappy-java" \
  pom.xml build.gradle build.gradle.kts 2>/dev/null || true
```

**Common runtime-extracting libraries:**
- **Netty** (`netty-transport-native-epoll`, `netty-tcnative`): Extracts .so based on `os.arch`
- **Conscrypt**: Native crypto libraries at runtime
- **RocksDB** (`rocksdbjni`): Platform-specific .so at first use
- **SQLite JDBC** (`sqlite-jdbc`): Bundles and extracts native SQLite
- **LZ4/Zstd/Snappy** (`lz4-java`, `zstd-jni`, `snappy-java`): Native compression accelerators
- **LevelDB** (`leveldbjni`): Native key-value store bindings

For each: check if current version includes ARM64 binaries. If missing, flag as MUST UPGRADE. If pure Java fallback exists (e.g., Netty NIO vs native epoll), document it.

### 1.2.3 Tiered Validation Policy

**FAIL immediately if:**
- .so shows x86-64 only AND no ARM64 version in JAR AND no source code available AND user cannot provide ARM64 version

**WARN but proceed if:**
- Multi-arch JAR with both x86 and ARM64, OR pure Java fallback exists, OR source available for recompilation

**PASS if:**
- .so shows "ARM aarch64" OR JAR contains .so in both `/linux/amd64/` and `/linux/aarch64/`

For x86-only .so files: check for source in repo, document recompilation needs, or prompt user. Validate with:
```bash
file libname.so  # Must show "ARM aarch64"
```

## 1.3 Dependency ARM64 Compatibility Analysis

> **Output: `graviton-validation/03-dependency-compatibility-report.md`** and **`graviton-validation/raw/dependency-tree-native.txt`**

**IMPORTANT:** ARM64-incompatible native code can be introduced through transitive dependencies. A pure Java direct dependency may pull in a transitive with native code. Analyze the full tree.

### Generate Filtered Tree

```bash
# Maven
mvn dependency:tree -Dincludes=net.java.dev.jna,io.netty,org.xerial.snappy,org.lz4,com.github.luben,org.rocksdb,org.xerial,org.conscrypt,com.google.protobuf \
  > graviton-validation/raw/dependency-tree-native.txt

# Gradle
./gradlew dependencies --configuration runtimeClasspath | grep -E "jna|netty-transport-native|snappy|lz4|zstd|rocksdb|sqlite|conscrypt|protobuf|jnr|leveldbjni" \
  > graviton-validation/raw/dependency-tree-native.txt
```

### Classify Each Dependency

**MUST UPGRADE (Blocking):** Current version lacks ARM64 binaries, contains x86-only native code, or has known critical ARM64 bugs.

**RECOMMENDED UPGRADE (Non-blocking):** ARM64 works but has known performance issues or bug fixes in newer version.

**COMPATIBLE (No action):** Full ARM64 support, no known issues, or pure Java with no architecture dependencies.

For transitive dependencies: identify which direct dependency pulls it in. Resolution may require updating the parent.

### Document Findings

```
Dependency: net.java.dev.jna:jna
Current Version: 5.6.0
Status: MUST UPGRADE
Reason: Version 5.6.0 lacks ARM64 native libraries
Minimum ARM64 Version: 5.8.0
Recommended Version: 5.14.0

Dependency: org.xerial.snappy:snappy-java (transitive via org.apache.kafka:kafka-clients)
Current Version: 1.1.7.3
Status: MUST UPGRADE
Reason: Lacks linux-aarch64 native binary
Minimum ARM64 Version: 1.1.8.1
Resolution: dependencyManagement override or exclusion+re-add
```

## 1.4 Architecture-Specific Code Detection

> **Output: `graviton-validation/04-code-scan-findings.md`**

Scan for:
```java
System.getProperty("os.arch")
System.getProperty("os.name")
Native.load()
System.loadLibrary()
```

Flag code that checks for "amd64" or "x86_64" without "aarch64" handling, loads native libraries without architecture-aware paths, or contains x86-specific assumptions.

## 1.5 Java Version Compatibility Check

> **Output: `graviton-validation/01-project-assessment.md`** (Java Environment section)

1. Document current Java version AND JDK distribution (e.g., "OpenJDK 21", "Corretto 17")
2. JDK 8: Supported with limitations. JDK 11+: Fully supported.
3. Flag known Graviton compatibility issues
4. **DO NOT change** the JDK distribution or Java version
5. All major distributions support ARM64 for Java 8+
