# Phase 3: ARM64 Validation & Testing

Build and test on ARM64 architecture. Supported platforms: Linux, macOS, WSL.

## Container Runtime Detection

Before skipping build validation, check for local container runtimes. Apple Silicon Macs run ARM64 containers natively via Docker or Finch, making full ARM64 validation possible without a remote Graviton instance.

```bash
# Detect available container runtimes
CONTAINER_CMD=""
if command -v finch &>/dev/null; then
  CONTAINER_CMD="finch"
elif command -v docker &>/dev/null; then
  CONTAINER_CMD="docker"
elif command -v nerdctl &>/dev/null; then
  CONTAINER_CMD="nerdctl"
elif command -v podman &>/dev/null; then
  CONTAINER_CMD="podman"
fi

if [ -n "$CONTAINER_CMD" ]; then
  echo "Container runtime found: $CONTAINER_CMD"
else
  echo "No container runtime found"
fi

# Check host architecture
ARCH=$(uname -m)
echo "Host architecture: $ARCH"
```

**Decision logic:**
- **ARM64 host (aarch64/arm64) + container runtime:** Build and validate in containers using `$CONTAINER_CMD`. This is the ideal path.
- **ARM64 host (aarch64/arm64) + no container runtime:** Build and validate directly on host.
- **x86 host + container runtime with ARM64 support:** Use `--platform linux/arm64` to build and validate. Docker Desktop and Finch on Apple Silicon support this natively. On x86 Linux, `docker buildx` with QEMU emulation may work but is slower.
- **x86 host + no container runtime:** Static analysis only. Document that ARM64 build validation requires an ARM64 environment and recommend the user run validation on a Graviton instance or ARM64 Mac.

**Do NOT skip build validation if a container runtime is available.** Even on macOS, if Docker or Finch is installed, attempt the ARM64 container build. Only recommend external validation as a last resort when no container runtime is found.

Throughout Phase 3, replace `docker` with `$CONTAINER_CMD` in all commands (e.g., `$CONTAINER_CMD build`, `$CONTAINER_CMD run`).

## 3.0 Build Environment Preparation

> **Output: `graviton-validation/06-build-test-results.md`** (Build Environment sections)

### Java Runtime Alignment

Build-time tools (annotation processors, compiler plugins) may not support the latest Java compiler versions. Runtime compatibility != compile-time tooling compatibility.

**Common annotation processor sensitivities:**
- **Lombok:** < 1.18.36 fails with Java 25
- **MapStruct:** < 1.5.0 may fail with Java 17+
- **Dagger:** < 2.40 may fail with Java 16+

**If build fails with annotation processor errors:**
1. Identify processor from `<annotationProcessorPaths>` or `annotationProcessor` in build files
2. Check compatibility with current Java runtime
3. Apply session-scoped Java alignment below
4. Document the processor and version requiring alignment

### Detect Project Target Version

```bash
# Maven
PROJECT_TARGET=$(grep -oP '(?<=<release>)[0-9]+' pom.xml 2>/dev/null || \
                 grep -oP '(?<=<target>)[0-9]+' pom.xml 2>/dev/null || \
                 grep -oP '(?<=<java.version>)[0-9]+' pom.xml 2>/dev/null)

# Gradle (Groovy)
if [ -z "$PROJECT_TARGET" ]; then
  PROJECT_TARGET=$(grep -oP 'sourceCompatibility\s*[=:]\s*['\''"]?\K[0-9]+' build.gradle 2>/dev/null || \
                   grep -oP 'JavaVersion\.VERSION_\K[0-9]+' build.gradle 2>/dev/null || \
                   grep -oP 'jvmToolchain\s*\(\s*\K[0-9]+' build.gradle 2>/dev/null)
fi

# Gradle (Kotlin DSL)
if [ -z "$PROJECT_TARGET" ]; then
  PROJECT_TARGET=$(grep -oP 'jvmToolchain\(\K[0-9]+' build.gradle.kts 2>/dev/null || \
                   grep -oP 'languageVersion\.set\(JavaLanguageVersion\.of\(\K[0-9]+' build.gradle.kts 2>/dev/null)
fi

# gradle.properties fallback
if [ -z "$PROJECT_TARGET" ]; then
  PROJECT_TARGET=$(grep -oP 'javaVersion\s*=\s*\K[0-9]+' gradle.properties 2>/dev/null)
fi

echo "Project targets Java: $PROJECT_TARGET"
```

### Session-Scoped Java Switching

If runtime significantly exceeds target (e.g., Java 25 with Java 17 target), use subshells:

```bash
# macOS
(
  export JAVA_HOME=$(/usr/libexec/java_home -v 21)
  java -version
  ${BUILD_CMD} clean install
)

# Linux
JAVA_21_HOME=$(find /usr/lib/jvm /usr/java -maxdepth 1 -type d \( -name "*java-21*" -o -name "*jdk-21*" -o -name "*corretto-21*" \) 2>/dev/null | head -1)
(
  export JAVA_HOME="$JAVA_21_HOME"
  export PATH="$JAVA_HOME/bin:$PATH"
  java -version
  ${BUILD_CMD} clean install
)

# SDKMAN
(
  source "$HOME/.sdkman/bin/sdkman-init.sh"
  sdk use java 21.0.9-amzn
  ${BUILD_CMD} clean install
)

# Single-command scope (any Unix)
JAVA_HOME=/path/to/java-21 ${BUILD_CMD} clean install
```

**ALLOWED:** `export JAVA_HOME` in subshells `()`, single-command env, `bash -c`, SDKMAN `sdk use`.

**FORBIDDEN:** Writing to `~/.zshrc`/`~/.bash_profile`/`~/.bashrc`, `update-alternatives --set`, `sdk default`, any persistent changes.

**Version preference:** Java 21 > 17 > 11. After transformation, user's `java -version` must match pre-transformation.

**If no compatible version found:** Document requirement, provide install commands, do NOT install automatically:
```bash
# Amazon Corretto
# macOS: brew install --cask corretto@21
# Amazon Linux/RHEL: sudo yum install java-21-amazon-corretto-devel
# Ubuntu/Debian: see apt.corretto.aws setup
```

## 3.1 ARM64 Build Validation

> **Output: `graviton-validation/06-build-test-results.md`** (Build Attempts, Test Failure Classification)

### Build Strategy

1. **First attempt** - full build with tests:
   - Gradle: `./gradlew clean build`
   - Maven: `mvn clean install`

2. **If tests fail**, classify root cause:
   - `INFRA` - Missing DB, Docker, env vars (non-blocking)
   - `ARM64` - Architecture failure (blocking)
   - `PRE-EXISTING` - Existed before migration (non-blocking)

3. **INFRA or PRE-EXISTING failures:** Document, then build without tests:
   - Gradle: `./gradlew clean build -x test`
   - Maven: `mvn clean install -DskipTests`
   - This becomes the **final build**

4. **ARM64 failures:** Do NOT skip tests. Build fails, requires resolution.

The final build command determines the build score.

### Container Validation (if containerized)

**Docker ENTRYPOINT handling:** Override entrypoint for validation commands.
- Wrong: `$CONTAINER_CMD run app:arm64 java -version` (appends to entrypoint)
- Correct: `$CONTAINER_CMD run --entrypoint java app:arm64 -version`

```bash
# Build
$CONTAINER_CMD build --platform linux/arm64 -t app:arm64 .

# Validate architecture
$CONTAINER_CMD run --rm --platform linux/arm64 --entrypoint java app:arm64 -version
$CONTAINER_CMD run --rm --platform linux/arm64 --entrypoint java app:arm64 -XshowSettings:properties -version 2>&1 | grep os.arch
# Must show: os.arch = aarch64
```

Verify output shows the SAME JDK distribution and version as the original application.

### Host-Based Validation

```bash
java -version  # Verify same distribution and version
java -XshowSettings:properties -version | grep os.arch  # Must show aarch64

./gradlew build  # or mvn clean package
```

## 3.2 Functional Testing on ARM64

> **Output: update `graviton-validation/06-build-test-results.md`**

**Containerized:**
```bash
$CONTAINER_CMD run --rm --platform linux/arm64 --entrypoint ./gradlew app:arm64 test
# or
$CONTAINER_CMD run --rm --platform linux/arm64 --entrypoint mvn app:arm64 test
```

**Host-based:**
```bash
./gradlew test  # or mvn test
```

Classify all failures (INFRA/ARM64/PRE-EXISTING). Test architecture-specific functionality: native library loading, crypto operations, file I/O, multi-threading.

**Final build determination:**
- All tests pass: test build is final build
- INFRA/PRE-EXISTING failures: `./gradlew clean build -x test` or `mvn clean install -DskipTests`
- ARM64 failures: failing build is final build (do not skip tests)

## 3.3 Startup Validation

> **Output: update `graviton-validation/06-build-test-results.md`** (Startup Validation)

Verify:
1. Application starts without errors
2. JVM flags accepted without errors
3. No immediate runtime crashes

Recommend to user for independent testing: performance benchmarking, load testing, resource utilization measurement.

## Write Summary

> **Output: `graviton-validation/00-summary.md`**

After all phases complete, write the summary using the template from [../document_references/documentation-standards.md](../document_references/documentation-standards.md). This file consolidates exit criteria status and references (not duplicates) detail in files 01-06.
