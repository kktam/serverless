---
name: project-analysis
description: Systematic deep-dive analysis of a large monorepo — parallel sub-agent exploration, structured multi-document output organized by architectural concern
source: auto-skill
extracted_at: '2026-07-03T02:15:09.780Z'
---

# Project Analysis: Systematic Deep-Dive Workflow

A repeatable approach for analyzing a large unfamiliar monorepo and producing structured documentation.

## When to Use

- The user says "analyze this project" or asks for a comprehensive understanding of a codebase.
- You need to produce reference documentation for a project you'll work on repeatedly.
- The project has multiple packages/modules with non-obvious relationships.

## Workflow

### Phase 1: Surface Survey

1. **Read top-level configuration** — `package.json`, `eslint.config.js`, `prettier.config.js`, `tsconfig.json`, `.editorconfig`, `pyproject.toml`, etc. These reveal the tech stack, package manager, linting conventions, and project name/version.

2. **Read all package manifests** — Glob for `packages/*/package.json` (or equivalent `*/setup.py`, `*.csproj`). Note each package's name, version, entry point, scripts, and inter-package dependencies (`"*"` or `"workspace:*"` patterns reveal local workspace links).

3. **Read root documentation** — `README.md`, `CONTRIBUTING.md`, `TESTING.md`, `RELEASE_PROCESS.md`, `VERSIONING.md`, `SECURITY.md`, `AGENTS.md`. These explain the project's purpose, development workflow, release process, and conventions.

### Phase 2: Parallel Package Exploration (Sub-Agents)

1. **Identify all packages** from the workspace manifests in Phase 1.

2. **Launch one sub-agent per major package** using the Explore sub-agent type with `thoroughness: "very thorough"`. Each agent should:
   - Read the package's `package.json` (or equivalent manifest).
   - Read the main entry point (`src/index.js`, `lib/main.dart`, etc.).
   - List the full directory tree of `src/` and `lib/`.
   - Read key orchestration/dispatcher files.
   - Read representative files to understand patterns (testing, configuration, error handling).
   - Look at test directory structure and testing conventions.

3. **For cross-cutting concerns** (e.g., CI/CD, release scripts, documentation), launch a sub-agent covering those.

### Phase 3: Synthesis into Structured Documents

Organize findings into a set of Markdown documents in a `./doc_analysis/` directory (or similar). Structure by architectural concern, not by package name:

| Document | Focus |
|---|---|
| **`01-Overview.md`** | Project identity, monorepo structure diagram, tech stack table, package dependency graph, overall architecture patterns |
| **`02-<Core-Orchestrator>.md`** | The entry point / dispatcher / router — how commands flow from user input through to execution |
| **`03-<Major-Subsystem-1>.md`** | Deep dive into the most architecturally complex subsystem (e.g., variable resolution, plugin engine, state machine) |
| **`04-<Major-Subsystem-2>.md`** | Second major subsystem (e.g., traditional framework, plugin lifecycle, deployment pipeline) |
| **`05-<Major-Subsystem-3>.md`** | Third major subsystem (e.g., container engine, deployment strategies) |
| **`06-<Cross-Cutting>.md`** | Remaining pieces: MCP servers, binary installers, CI/CD workflows, testing strategy, shared utilities |

For each document:
- Use **tables** for comparisons (package versions, environments, command listings).
- Use **Mermaid** `flowchart TD` for execution flows (but keep them as text — the user renders them).
- Use **code blocks** for key APIs, constructor signatures, or configuration shapes.
- Use **numbered lists** for sequential pipelines (e.g., deploy steps, release stages).
- Use **bold section headers** to delineate subsystems.

### Phase 4: Save a Reference Memory

Save a reference memory pointing to the `./doc_analysis/` directory so future sessions know the documentation exists:

```markdown
---
name: Project documentation analysis
description: Comprehensive analysis documents saved in ./doc_analysis/
type: reference
---

Overview of what was produced: [one-line per document with focus]
```

## Tips

- **Read the full config schema files** — they reveal the full surface area of what the framework accepts.
- **Look for `static` detection methods** on classes — these are how the system discovers which runner/plugin/provider to use (e.g., `static configFileNames()`, `static shouldRun()`).
- **Identify the "brain" classes** — classes that exceed ~800 lines are likely central orchestrators (e.g., `manager.js` at 1357 lines, `provider.js` at 4232 lines). Prioritize reading these.
- **Trace the credential chain** — how does auth flow from user input → token/key → API call? This is critical in any cloud tool.
- **Note the testing strategy layers** — unit → resolver/golden → integration → e2e — this reveals what parts of the system are trusted vs. risk-prone.
