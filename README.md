# DevEx Toolkit

**Managed developer experience infrastructure. One agent to bootstrap, maintain, and upgrade any repo.**

## TL;DR

You clone a repo, open VS Code, type `@repo-onboarder setup` in Copilot chat, answer 2–3 questions, and get a fully configured project with language agents, git hooks, CI/CD, issue templates, idiom enforcement, and a release process. Everything stays up to date automatically.

---

## Getting Started

### Option A: New Project (recommended)

```bash
# 1. Create your project repo on GitHub and clone it
git clone https://github.com/you/my-project.git
cd my-project

# 2. Clone devex-toolkit alongside it
cd ..
git clone https://github.com/johnazariah/devex-toolkit.git

# 3. Open both in VS Code as a multi-root workspace
#    File → Add Folder to Workspace → select devex-toolkit
#    (or create a .code-workspace file)

# 4. In Copilot chat:
@repo-onboarder setup
```

The agent will:
1. Scan your project files and detect your language stack + topology
2. Ask you to confirm (e.g., *"Detected orleans-aspire-react. Correct?"*)
3. Ask for license type and GitHub owner
4. Show you every file it will create
5. Wait for your approval
6. Write everything, activate hooks, and verify the build

### Option B: Existing Project

Same steps, but `setup` is smart about existing files — it won't overwrite your README or clobber custom CI. It fills in what's missing.

### Option C: No Clone Needed

If you don't want to clone devex-toolkit locally, it still works. Your project's agent files have **GitHub fetch fallback** — they pull standards directly from this repo's `main` branch at runtime. You just won't get the multi-root workspace benefits (instant load, no network dependency).

---

## What You Get

After `@repo-onboarder setup`, your project has:

| Category | Files | Purpose |
|----------|-------|---------|
| **Language agents** | `.github/agents/{lang}-dev.agent.md` | Write, review, refactor code with idiom enforcement |
| **Git hooks** | `.githooks/pre-commit`, `pre-push` | Secrets detection, build gates, test register checks |
| **CI/CD** | `.github/workflows/ci.yml` | Build + test on push/PR, matrix OS support |
| **Issue templates** | `.github/ISSUE_TEMPLATE/` | Bug, feature, and task templates |
| **PR template** | `.github/PULL_REQUEST_TEMPLATE.md` | Checklist with AI attribution |
| **Prompt library** | `.github/prompts/` | Commit messages, PR prep, debug failures, releases, phase starts |
| **Copilot instructions** | `.github/copilot-instructions.md` | Always-on idiom enforcement + project context |
| **Project tracking** | `.project/` | Testing register, learnings, ideas, phases, status |
| **Changelog** | `CHANGELOG.md` | Conventional-commit powered changelog |
| **Config** | `.editorconfig`, `.gitattributes`, `.gitignore` | Consistent formatting across editors |

---

## Agent Verbs

Talk to the agents in Copilot chat:

| Command | What It Does |
|---------|-------------|
| `@repo-onboarder setup` | Full bootstrap — detect topology, generate all files |
| `@repo-onboarder add <language>` | Drop in a language agent (e.g., `add python`) |
| `@repo-onboarder add topology <name>` | Layer a topology (e.g., `add topology dotnet-lib` for NuGet publishing) |
| `@repo-onboarder upgrade` | Diff managed files against latest, apply on approval |
| `@repo-onboarder status` | Show which managed files are current, stale, or missing |
| `@repo-onboarder release` | Analyse commits since last tag, suggest semver bump, update changelog, tag + push |
| `@fsharp-dev` | Write/review/refactor F# code with 17 idiom rules |
| `@csharp-dev` | Write/review/refactor C# with 14 idiom rules + Orleans/Aspire/NuGet patterns |
| `@python-dev` | Write/review/refactor Python with 13 idiom rules + PyPI patterns |
| `@typescript-dev` | Write/review/refactor TypeScript with 12 idiom rules + React/Vite patterns |

---

## Topologies

A **topology** describes your project's shape. `setup` auto-detects it from your project files:

| Topology | What It Is | Languages | Detection Signal |
|----------|-----------|-----------|-----------------|
| `dotnet-app` | .NET application (console, service, desktop) | C# | `.csproj` or `.fsproj` |
| `dotnet-lib` | .NET library published to NuGet | C# or F# | `<IsPackable>true</IsPackable>` in csproj |
| `python-app` | Python application | Python | `.py` files or `pyproject.toml` without `[build-system]` |
| `python-lib` | Python library published to PyPI | Python | `pyproject.toml` with `[build-system]` |
| `orleans-aspire-react` | C#/Orleans/Aspire 13 + Vite/React frontend | C# + TypeScript | Orleans + Aspire NuGet refs + `package.json` |

**Topologies are additive.** Start as `dotnet-app`, later run `@repo-onboarder add topology dotnet-lib` to layer in NuGet publishing without losing what you have.

### What Topologies Add

- **`dotnet-lib`**: NuGet publish CI workflow, `Directory.Build.props` with package metadata, MinVer for git-tag versioning, NuGet library standards
- **`python-lib`**: PyPI publish CI workflow, `pyproject.toml` scaffold, hatch-vcs for git-tag versioning, PyPI library standards
- **`orleans-aspire-react`**: Dual-runtime CI, Orleans grain patterns (modern `IPersistentState<T>`, `[GenerateSerializer]`, `[Alias]`), Aspire 13 patterns (NuGet-only, service discovery, OpenTelemetry), React+Vite patterns (feature folders, TanStack Query)

---

## Release Process

All topologies get a **git-tag based release process**:

```
@repo-onboarder release
```

The agent:
1. Scans `git log` since the last semver tag
2. Classifies conventional commits (`feat:` → minor, `fix:` → patch, `BREAKING CHANGE:` → major)
3. Suggests the next version with a changelog preview
4. **Waits for your approval** (you can override the version)
5. Updates `CHANGELOG.md`, version in project files, creates a git tag, pushes

Pushing the tag triggers topology-specific CI:
- **`dotnet-lib`** → `dotnet pack` + push to nuget.org
- **`python-lib`** → `python -m build` + upload to PyPI via trusted publishers
- **`orleans-aspire-react`** → container builds + GitHub Release

---

## How Standards Stay Current

Your project's agent files are **thin wrappers** with a three-tier loading strategy:

```
1. Workspace peer    → reads from devex-toolkit if it's in your VS Code workspace (instant, no network)
2. GitHub fetch      → pulls from raw.githubusercontent.com/johnazariah/devex-toolkit/main/... (always latest)
3. Inline fallback   → self-check table baked into the agent file (works offline, no dependencies)
```

Standards improve in devex-toolkit → every project sees the improvement next time an agent runs. No sync step, no version mismatch.

---

## Content Types

| Type | Description | On Upgrade |
|------|-------------|------------|
| **invariant** | Same for every project (hooks, templates, prompts) | Full replace |
| **template** | Variable substitution (LICENSE, .editorconfig) | Full replace |
| **agent** | Language agent with inline fallback + project context | Replace managed section only |
| **generated** | Rules engine output (copilot-instructions, CI, README) | Replace managed sections only |
| **scaffolded** | Starting point (testing register, learnings) | Never auto-upgraded |

---

## Repository Structure

```
manifest.yml                              # Content distribution manifest

.github/agents/
├── repo-onboarder.agent.md               # Bootstrap/upgrade/release agent
├── fsharp-dev.agent.md                   # F# write/review/refactor
├── csharp-dev.agent.md                   # C# write/review/refactor
├── python-dev.agent.md                   # Python write/review/refactor
└── typescript-dev.agent.md               # TypeScript write/review/refactor

skills/
├── repo-onboard/                         # Onboarding flow + templates + standards
│   ├── SKILL.md                          # Full onboarding skill with 🛑 gates
│   ├── standards/                        # Cross-cutting rules (quality, docs, git, testing)
│   └── templates/                        # Invariant files (hooks, issue templates, prompts)
├── csharp-dev/
│   └── standards/
│       ├── idiomatic-csharp.md           # 14-rule C# idiom standard
│       ├── orleans-patterns.md           # Orleans 8+ grain design
│       ├── aspire-patterns.md            # Aspire 13 (no workloads)
│       └── nuget-library.md              # NuGet packaging + publishing
├── fsharp-dev/
│   └── standards/idiomatic-fsharp.md     # 17-rule F# idiom standard
├── python-dev/
│   └── standards/
│       ├── idiomatic-python.md           # 13-rule Python idiom standard
│       └── pypi-library.md              # PyPI packaging + publishing
└── typescript-dev/
    └── standards/
        ├── idiomatic-typescript.md       # 12-rule TypeScript idiom standard
        └── react-vite-patterns.md        # React + Vite patterns

docs/
├── design-v2-managed-devex.md            # v2 architecture (manifest, lockfile, verbs, topologies)
└── workspace-integration.md              # Multi-root workspace setup
```

---

## FAQ

**Q: Do I need devex-toolkit cloned locally?**
No. Agent files in your project have GitHub fetch fallback. Cloning locally gives you faster loading and offline support.

**Q: What if I'm offline?**
Agent files carry inline self-check tables — they work without network access. You just won't get the full detailed standards until you're back online.

**Q: Can I customise the generated files?**
Yes. Files use managed section markers (`<!-- devex:begin -->` ... `<!-- devex:end -->`). Everything outside markers is yours. `upgrade` only touches content between markers.

**Q: What if I don't use conventional commits?**
The release process still works — it falls back to manual version input. But conventional commits give you automatic changelog generation and version suggestions.

**Q: Can I use this for a monorepo?**
Not yet designed for it. Each repo gets one topology (or an additive stack). Monorepo support is a future consideration.

**Q: How do I add my own standards?**
Add them to your project's `.github/copilot-instructions.md` outside the managed sections. They'll survive upgrades.

---

## Origin

Synthesized from patterns across production repos (Aura, Pelican, Emic, Encoding, Hermes), battle-tested on .NET/F#, C#, Python, and TypeScript projects.
