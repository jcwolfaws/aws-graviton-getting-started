# Graviton Agent Skills

Portable Agent Skills that help developers migrate codebases to AWS Graviton (ARM64) processors.

## What are Agent Skills?

Agent Skills are an open standard for packaging expert instructions in a format AI coding assistants can follow. Each skill is a folder containing instructions, references, and scripts that teach an agent how to perform a specific task — in this case, migrating applications to Graviton.

For the full specification, see [agentskills.io](https://agentskills.io).

## Platform Compatibility

Each skill folder contains two entry point files to support the broadest range of AI tools:

| File | Platforms | Format |
|------|-----------|--------|
| `SKILL.md` | Claude Code, Cursor, Codex, Windsurf, Gemini CLI, GitHub Copilot, and [20+ others](https://agentskills.io) | [Agent Skills spec](https://agentskills.io/specification) |
| `POWER.md` | Kiro | Kiro Powers format (`displayName`, `keywords`, `author` frontmatter) |

Both files contain the same instructions — only the frontmatter differs. Your platform will pick up the file it recognizes and ignore the other.

## Available Skills

| Skill | Language | Status | Description |
|-------|----------|--------|-------------|
| [java-x86-to-graviton](languages/java-x86-to-graviton/) | Java | Stable | Full x86-to-Graviton migration: dependency audit, native library validation, JVM optimization, ARM64 build validation |

## How to Install

### Kiro

Paste the GitHub folder URL into Kiro's power installer:

```
https://github.com/jcwolfaws/aws-graviton-getting-started/tree/main/tools/skills/languages/java-x86-to-graviton
```

### Claude Code, Cursor, Codex, and other Agent Skills platforms

Copy the skill folder into your project:

```bash
cp -r tools/skills/languages/java-x86-to-graviton/ /path/to/your/project/.skills/java-x86-to-graviton/
```

Your AI coding assistant will detect the `SKILL.md` and follow the instructions when triggered.

### Direct reference

You can also point your AI assistant to the skill without installing:

> Analyze this Java project for Graviton migration using the skill at tools/skills/languages/java-x86-to-graviton/SKILL.md

## Skill Folder Structure

Each skill folder contains:

```
<skill-name>/
├── SKILL.md                    # Entry point (Agent Skills format)
├── POWER.md                    # Entry point (Kiro format)
├── summaries.md                # File index for agent discovery
├── phases/                     # Detailed phase instructions
├── document_references/        # Scope guardrails and output standards
└── README.md                   # Human-readable overview
```

## Creating New Skills

See the [skill template](_templates/SKILL.template.md) for the boilerplate structure. When creating a new skill:

1. Create a folder under `languages/` named to match the skill's `name` field
2. Create both `SKILL.md` (Agent Skills frontmatter) and `POWER.md` (Kiro frontmatter) with the same body content
3. Add a `summaries.md` listing all files in the skill for agent discovery
4. Add a `README.md` with a human-readable overview
5. Keep the main entry point under 500 lines — split detailed instructions into `phases/`, `references/`, or `scripts/` subdirectories

Fill in language-specific details based on the existing [Graviton documentation](../../README.md) for each language.
