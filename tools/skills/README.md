# Graviton Agent Skills

Portable Agent Skills that help developers migrate codebases to AWS Graviton (ARM64) processors.

## What are Agent Skills?

Agent Skills are an open standard for packaging expert instructions in a format AI coding assistants can follow. Each skill is a `SKILL.md` file (with optional supporting documents) that teaches an agent how to perform a specific task — in this case, migrating applications to Graviton.

Skills work on 20+ platforms: Claude Code, Kiro, Cursor, Codex, Windsurf, Gemini CLI, GitHub Copilot, and more.

For the full specification, see [agentskills.io](https://agentskills.io).

## Available Skills

| Skill | Language | Status | Description |
|-------|----------|--------|-------------|
| [java-x86-to-graviton](languages/java-x86-to-graviton/) | Java | Stable | Full x86-to-Graviton migration: dependency audit, native library validation, JVM optimization, ARM64 build validation |

## How to Use

### Import into your project

Copy the skill folder into your project repository:

```bash
# Copy the Java Graviton migration skill
cp -r tools/skills/languages/java-x86-to-graviton/ /path/to/your/project/.skills/java-x86-to-graviton/
```

Your AI coding assistant will detect the `SKILL.md` and follow the instructions when triggered.

### Direct reference

Point your AI assistant to the skill directly:

> Analyze this Java project for Graviton migration using the skill at tools/skills/languages/java-x86-to-graviton/SKILL.md

## Creating New Skills

See the [skill template](_templates/SKILL.template.md) for the boilerplate structure. Fill in language-specific details based on the existing [Graviton documentation](../../README.md) for each language.
