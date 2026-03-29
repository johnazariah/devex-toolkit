# Design: Managed Developer Experience (v2)

> **Status:** Draft
> **Origin:** Evolved from hermes onboarding — generalising the pattern.

## Problem

When someone clones a repo, the AI-assisted developer experience only works if they:
1. Know about devex-toolkit
2. Clone it alongside the project
3. Set up a multi-root workspace
4. Understand which agents exist and how to invoke them

That's four steps too many. The goal: **clone, open, work.**

## Vision

devex-toolkit is a managed content distribution system. One agent (`@repo-onboarder`) handles the full lifecycle: bootstrap, add capabilities, upgrade, and status check. Every managed file in a project traces back to a versioned source in devex-toolkit on GitHub.

## Design Principles

1. **Clone-and-go.** A fresh clone must provide a functional AI-assisted workflow with zero additional setup.
2. **Single source of truth.** Standards, agents, and templates live in devex-toolkit on GitHub. Projects carry thin copies with GitHub-fetch fallback — never full duplicates.
3. **Graceful degradation.** No internet? Inline self-check tables still work. No devex-toolkit? copilot-instructions still enforces key rules. No Copilot? Hooks and CI still run.
4. **Zero-friction.** The onboarder asks 2–3 questions and generates everything. No manual file juggling.
5. **Upgradable.** Managed content can be refreshed from GitHub without losing project-specific customisation.

## Architecture

### Content Types

| Type | Description | Upgrade Strategy |
|------|-------------|------------------|
| **invariant** | Identical across projects. Copied verbatim. | Full replace on upgrade |
| **template** | Variable substitution (`{{project_name}}`, `{{license}}`). No user edits expected. | Full replace on upgrade |
| **agent** | Language agent with inline fallback + project-specific context section. | Replace managed section; preserve project context |
| **generated** | Produced by rules engine from project discovery. Mix of managed + custom sections. | Replace managed sections only |
| **scaffolded** | Created once as a starting point. Fully user-owned after creation. | Never auto-upgraded. `status` shows version drift. |

### Managed Section Markers

Files that mix managed and custom content use markers:

```markdown
<!-- devex:begin type=idiom-standards version=1.2.0 -->
... managed content ...
<!-- devex:end -->
```

`upgrade` replaces content between markers. Everything outside markers is user-owned.

### Topologies

A **topology** describes a project's shape — its runtime(s), build pipeline, testing approach, and publishing lifecycle. Topologies are **additive**: a project starts with one and can layer more via `@repo-onboarder add topology <name>`.

Each topology selects:
- **Languages** → which agents get installed
- **Build/test commands** → wired into hooks + CI
- **Extra files** → topology-specific CI workflows, config, standards
- **Extra standards** → domain-specific idiom rules loaded by language agents

```yaml
topologies:
  dotnet-app:
    description: ".NET application (console, service, desktop)"
    languages: [csharp]           # or [fsharp, csharp]
    build: "dotnet build"
    test: "dotnet test"

  dotnet-lib:
    description: ".NET library published to NuGet"
    languages: [csharp]           # or [fsharp]
    build: "dotnet build"
    test: "dotnet test"
    publish: "dotnet pack → dotnet nuget push"
    extra_files:
      - dest: .github/workflows/publish-nuget.yml
        type: generated
      - dest: Directory.Build.props
        type: template
        managed_sections: [package-metadata]
    extra_standards: [nuget-library]

  python-app:
    description: "Python application"
    languages: [python]
    build: "python -m build"
    test: "pytest"

  python-lib:
    description: "Python library published to PyPI"
    languages: [python]
    build: "python -m build"
    test: "pytest"
    publish: "twine upload"
    extra_files:
      - dest: .github/workflows/publish-pypi.yml
        type: generated
      - dest: pyproject.toml
        type: scaffolded
    extra_standards: [pypi-library]

  orleans-aspire-react:
    description: "C#/Orleans/Aspire 13 backend + Vite/React frontend"
    languages: [csharp, typescript]
    build: "dotnet build && npm run build"
    test: "dotnet test && npm test"
    extra_files:
      - dest: .github/workflows/ci.yml
        type: generated             # dual-runtime CI
      - dest: Directory.Build.props
        type: template
    extra_standards: [orleans-grains, aspire-patterns, react-vite]
```

Topologies compose. A project with `[dotnet-lib, orleans-aspire-react]` gets NuGet publishing CI **and** the Orleans/Aspire/React agents + standards. The lockfile records active topologies:

```json
{
  "discovery": {
    "topologies": ["orleans-aspire-react"],
    "languages": ["csharp", "typescript"],
    ...
  }
}
```

**`setup` detects topology** by scanning project files:

| Signal | Inferred Topology |
|--------|-------------------|
| `.fsproj` or `.csproj` only | `dotnet-app` |
| `<IsPackable>true</IsPackable>` or `<PackageId>` in csproj | `dotnet-lib` |
| `pyproject.toml` with `[build-system]` | `python-lib` |
| `pyproject.toml` without `[build-system]` or just `.py` files | `python-app` |
| `Microsoft.Orleans.*` in csproj + `Aspire.Hosting` + `package.json` | `orleans-aspire-react` |

The agent confirms detection: *"Detected orleans-aspire-react (Orleans + Aspire + React). Correct?"*

### Release Process

All topologies get a **git-tag based release process** powered by conventional commits:

#### How It Works

1. **Analyse** — scan `git log` since the last semver tag
2. **Classify** — parse conventional commit prefixes:
   - `fix:` → **patch** bump
   - `feat:` → **minor** bump
   - `BREAKING CHANGE:` in footer or `!` suffix → **major** bump
   - `docs:`, `chore:`, `test:`, `refactor:` → no version bump (but included in changelog)
3. **Suggest** — propose the next version with rationale:
   ```
   Current: v1.3.2
   Since last tag: 4 feat, 2 fix, 1 BREAKING CHANGE
   Suggested: v2.0.0 (breaking change detected)
   
   Changelog preview:
   ## [2.0.0] - 2026-03-30
   ### ⚠ BREAKING CHANGES
   - Removed legacy grain state API (#45)
   ### Features
   - Add batch document processing (#42)
   - Orleans dashboard integration (#43)
   ...
   ```
4. **🛑 Gate** — wait for approval (user can override version)
5. **Execute**:
   - Update `CHANGELOG.md` (prepend new section)
   - Update version in project files (csproj `<Version>`, pyproject.toml `version`, package.json `version`)
   - Create git commit: `chore(release): vX.Y.Z`
   - Create annotated git tag: `vX.Y.Z`
   - Push commit + tag (triggers publish CI)

#### Topology-Specific Publish Triggers

| Topology | Tag triggers |
|----------|-------------|
| `dotnet-app` | CI builds release artifacts, creates GitHub Release |
| `dotnet-lib` | `publish-nuget.yml` — `dotnet pack` + `dotnet nuget push` to nuget.org |
| `python-lib` | `publish-pypi.yml` — `python -m build` + `twine upload` via trusted publisher |
| `orleans-aspire-react` | CI builds containers, creates GitHub Release with Docker tags |

#### Agent Integration

The release flow is exposed as:
- **`@repo-onboarder release`** — new verb, runs the analyse → suggest → execute flow
- **`.github/prompts/release.prompt.md`** — already exists as invariant template, updated to invoke the release skill

### File Manifest

The manifest (`manifest.yml`) describes every distributable file:

```yaml
version: "1.0.0"
github: johnazariah/devex-toolkit

files:
  # --- Invariant files (same for every project) ---
  - src: templates/githooks/pre-commit
    dest: .githooks/pre-commit
    type: invariant
    vars: [build_command]

  - src: templates/githooks/pre-push
    dest: .githooks/pre-push
    type: invariant
    vars: [build_command, test_command]

  - src: templates/github/ISSUE_TEMPLATE/bug.md
    dest: .github/ISSUE_TEMPLATE/bug.md
    type: invariant

  - src: templates/github/ISSUE_TEMPLATE/feature.md
    dest: .github/ISSUE_TEMPLATE/feature.md
    type: invariant

  - src: templates/github/ISSUE_TEMPLATE/task.md
    dest: .github/ISSUE_TEMPLATE/task.md
    type: invariant

  - src: templates/github/PULL_REQUEST_TEMPLATE.md
    dest: .github/PULL_REQUEST_TEMPLATE.md
    type: invariant

  - src: templates/github/prompts/commit.prompt.md
    dest: .github/prompts/commit.prompt.md
    type: invariant

  - src: templates/github/prompts/pick-next-work.prompt.md
    dest: .github/prompts/pick-next-work.prompt.md
    type: invariant

  - src: templates/github/prompts/pr-prep.prompt.md
    dest: .github/prompts/pr-prep.prompt.md
    type: invariant

  - src: templates/github/prompts/release.prompt.md
    dest: .github/prompts/release.prompt.md
    type: invariant

  - src: templates/github/prompts/debug-test-failure.prompt.md
    dest: .github/prompts/debug-test-failure.prompt.md
    type: invariant

  - src: templates/root/CHANGELOG.md
    dest: CHANGELOG.md
    type: invariant

  # --- Scaffolded files (user-owned after creation) ---
  - src: templates/project/testing-register.md
    dest: .project/testing-register.md
    type: scaffolded

  - src: templates/project/learnings.md
    dest: .project/learnings.md
    type: scaffolded

  - src: templates/project/future-enhancement-ideas.md
    dest: .project/future-enhancement-ideas.md
    type: scaffolded

  # --- Language agents (managed + project context) ---
  - src: .github/agents/fsharp-dev.agent.md
    dest: .github/agents/fsharp-dev.agent.md
    type: agent
    language: fsharp
    requires: [dotnet]

  - src: .github/agents/csharp-dev.agent.md
    dest: .github/agents/csharp-dev.agent.md
    type: agent
    language: csharp
    requires: [dotnet]

  - src: .github/agents/python-dev.agent.md
    dest: .github/agents/python-dev.agent.md
    type: agent
    language: python
    requires: [python]

  - src: .github/agents/typescript-dev.agent.md
    dest: .github/agents/typescript-dev.agent.md
    type: agent
    language: typescript
    requires: [node]

  # --- Generated files (rules engine + discovery) ---
  - dest: .github/copilot-instructions.md
    type: generated
    managed_sections: [idiom-standards]

  - dest: .github/workflows/ci.yml
    type: generated

  - dest: .editorconfig
    type: template

  - dest: .gitattributes
    type: template

  - dest: .gitignore
    type: template

  - dest: .vscode/settings.json
    type: generated

  - dest: README.md
    type: generated
    managed_sections: [badges, tech-stack]

  - dest: LICENSE
    type: template
    vars: [author_name, year, license_type]

  - dest: "{{project_name}}.code-workspace"
    type: template
```

### Devex Lockfile

Each managed project gets a `.devex-lock.json` tracking what was installed:

```json
{
  "devex_version": "1.0.0",
  "installed": "2026-03-30T10:00:00Z",
  "files": {
    ".githooks/pre-commit": {
      "type": "invariant",
      "sha256": "abc123...",
      "source_version": "1.0.0"
    },
    ".github/agents/fsharp-dev.agent.md": {
      "type": "agent",
      "sha256": "def456...",
      "source_version": "1.0.0"
    }
  },
  "discovery": {
    "project_name": "hermes",
    "languages": ["fsharp", "csharp"],
    "license": "MIT",
    "build_command": "dotnet build",
    "test_command": "dotnet test"
  }
}
```

This enables:
- `status`: compare local SHA against latest manifest
- `upgrade`: know which files are managed vs user-modified
- `setup` on re-run: skip questions already answered

## Agent Verbs

### `@repo-onboarder setup`

Full bootstrap. The zero-friction flow:

1. **Detect** — scan for existing files (`.fsproj`, `package.json`, `pyproject.toml`, `Cargo.toml`, Orleans/Aspire references)
2. **Infer topology** — match detected signals to known topologies (see Topologies table above)
3. **Ask** — 2–3 questions max:
   - "Detected orleans-aspire-react topology. Correct?" (or "What project shape?")
   - "License? (MIT / Apache-2.0 / proprietary)"
   - "GitHub owner? (for badges and CI)" — infer from git remote if possible
4. **Present plan** — show every file that will be created, grouped by type, with topology-specific extras highlighted
5. **🛑 Gate** — wait for approval
6. **Write** — create all files, activate hooks, write lockfile (with topology + languages)
7. **Verify** — run build command, report result

### `@repo-onboarder add <language>`

Drop a language agent into an existing project:

1. Fetch agent file from GitHub (or workspace peer)
2. Append project-specific context section (infer from copilot-instructions)
3. Update copilot-instructions idiom summary (managed section)
4. Update lockfile

### `@repo-onboarder add topology <name>`

Layer a topology onto an existing project:

1. Validate compatibility (e.g., can't add `python-lib` to a project with no Python)
2. Fetch topology-specific files (CI workflows, config templates, standards)
3. Install extra language agents if needed (e.g., `orleans-aspire-react` adds TypeScript agent)
4. Update lockfile with new topology in `discovery.topologies` array
5. Show what was added and any manual steps needed

### `@repo-onboarder upgrade`

Refresh managed content from latest devex-toolkit:

1. Read lockfile → know what's managed and at what version
2. Fetch latest manifest from GitHub
3. For each managed file:
   - **invariant/template**: compare SHA. If different, show diff, apply on approval.
   - **agent**: replace managed portion, preserve project context section.
   - **generated with managed sections**: replace between markers only.
   - **scaffolded**: show drift warning only. Never auto-update.
4. Update lockfile with new SHAs and version.

### `@repo-onboarder release`

Git-tag based release (see Release Process section above):

1. Parse conventional commits since last semver tag
2. Suggest version bump (patch/minor/major) with rationale
3. Show changelog preview
4. **🛑 Gate** — wait for approval (user can override version)
5. Update CHANGELOG.md, version in project files
6. Create release commit + annotated tag
7. Push (triggers topology-specific publish CI)

### `@repo-onboarder status`

Read-only health check:

```
devex-toolkit v1.0.0 → v1.2.0 available

Managed files:
  ✅ .githooks/pre-commit          (current)
  ⚠️  .github/prompts/commit.prompt.md  (v1.0.0 → v1.2.0 available)
  ✅ .github/agents/fsharp-dev.agent.md (current)
  📌 .project/learnings.md         (scaffolded, user-owned)
  ❌ .github/agents/python-dev.agent.md (not installed)

Run '@repo-onboarder upgrade' to update stale files.
```

## Versioning

devex-toolkit uses **semver tags** on GitHub:
- **Patch** (1.0.x): typo fixes, minor template tweaks
- **Minor** (1.x.0): new files, new rules, new agents
- **Major** (x.0.0): breaking changes to manifest schema or agent interface

The lockfile records which version each file came from. `upgrade` shows the version jump and changelog.

## Migration Path

### Phase 0 — What exists today (done)
- Agents with three-tier cascade (workspace → GitHub → inline)
- SKILL.md with full onboarding flow
- Templates directory

### Phase 1 — Manifest + lockfile
- Create `manifest.yml` ✅
- Add topologies to manifest schema
- Add lockfile generation to the existing SKILL.md flow
- Add managed section markers to copilot-instructions template
- Tag v1.0.0

### Phase 2 — Verb surface
- Rewrite SKILL.md to support setup/add/upgrade/status/release modes
- `setup` = current onboarding flow + topology detection + lockfile
- `add` = language agent + `add topology` for layering
- `upgrade` = diff + apply flow
- `status` = lockfile comparison
- `release` = conventional commit analysis + tag + publish

### Phase 3 — Topologies + Publishing
- Orleans/Aspire/React topology standards and CI templates
- NuGet publish workflow (`publish-nuget.yml`)
- PyPI publish workflow (`publish-pypi.yml`)
- Release process skill (conventional commits → semver → changelog → tag)
- Domain-specific standards: orleans-grains, aspire-patterns, react-vite, nuget-library, pypi-library

### Phase 4 — Polish
- Changelog generation for upgrades ("here's what changed since your version")
- `upgrade --dry-run` for preview
- Batch upgrade with single approval gate
- Integration with `start-phase.prompt.md` to suggest missing agents

### Phase 5 — GitHub Template Repo
- Create `johnazariah/project-template` as a GitHub template repository
- Pre-baked with all skeleton files, thin agent stubs (all 4 languages), managed section markers, and a lockfile
- Template is a **release artifact** of devex-toolkit — regenerated on each semver tag
- `@repo-onboarder setup` detects template-derived projects (via lockfile) and runs a lighter customization flow:
  1. "Project name?" → renames workspace file, updates README/LICENSE/copilot-instructions
  2. "Which languages?" → removes unused agent files
  3. "GitHub owner?" → updates badges and CI
- Template repo README says: "Generated from devex-toolkit vX.Y.Z. Run `@repo-onboarder upgrade` to update."

## Open Questions

1. **Should the lockfile be committed?** Yes — it's part of the project's devex contract. Other contributors can see what's managed and at what version. `.devex-lock.json` in the repo root.

2. **What about private repos?** GitHub raw URLs require auth for private repos. The agent would need a GitHub token. For now, assume devex-toolkit is public.

3. **Should agents in devex-toolkit carry project-context templates?** E.g., a `<!-- PROJECT CONTEXT: Replace this section with your project-specific details -->` comment block that the `add` verb fills in from copilot-instructions. Yes — this is the cleanest way to separate managed from custom.

4. **Should we support multiple devex-toolkit sources?** E.g., a company fork with proprietary standards. Future consideration — v2. For now, hardcode `johnazariah/devex-toolkit`.

5. **~~Duplicated agents in devex-toolkit (`agents/` and `.github/agents/`).~~** Resolved — consolidated to `.github/agents/` only. Old `agents/` directory removed.
