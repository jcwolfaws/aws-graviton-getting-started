# Skill File Index

* "SKILL.md": "Main entry point. Contains scope guardrails, entry/exit criteria, transformation workflow overview with phase routing, test failure handling, and documentation output mapping. Read this first."

* "phases/phase1-static-analysis.md": "Detailed steps for Phase 1: project structure analysis, native library validation (bundled and runtime-extracted .so files), tiered FAIL/WARN/PASS policy, dependency ARM64 compatibility analysis including transitive dependencies, architecture-specific code detection, and Java version check."

* "phases/phase2-resolution.md": "Detailed steps for Phase 2: native library resolution (cross-compilation or fallback), dependency updates for MUST UPGRADE items only (Maven dependencyManagement and Gradle resolutionStrategy patterns for transitive deps), architecture detection code updates, Dockerfile platform annotations, and version-gated Graviton JVM flag configuration."

* "phases/phase3-validation.md": "Detailed steps for Phase 3: Java runtime alignment for annotation processor compatibility (session-scoped only), build validation strategy with INFRA/ARM64/PRE-EXISTING failure classification, container and host-based validation, functional testing, startup checks, and summary generation."

* "document_references/agent-scope-boundaries.md": "CRITICAL guardrails. Decision tree for every dependency analysis, IN SCOPE vs OUT OF SCOPE definitions, common pure Java libraries (always compatible), red flags for scope creep, and case studies (JUnit wrong vs Protoc correct). MUST be read before starting."

* "document_references/documentation-standards.md": "Defines the mandatory graviton-validation/ output folder structure, canonical filenames (00-summary.md through 06-build-test-results.md plus raw/), required sections for each file, and mapping from transformation steps to output files. Read before Phase 1."
