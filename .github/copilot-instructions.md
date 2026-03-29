# DevEx Toolkit — Copilot Instructions

## Working Style

**Default: discuss, don't code.** This is a template/standards repo — changes affect all future projects.

## What This Repo Is

A managed developer experience toolkit. One agent (`@repo-onboarder`) bootstraps, maintains, and upgrades repos with production-grade infrastructure. All distributable content is described in `manifest.yml`.

### Core Components

- **Manifest** (`manifest.yml`): Every distributable file, its type, and destination
- **Agents** (`.github/agents/`): Language-specific dev agents + the repo onboarder
- **Skills** (`skills/`): Onboarding flow and language-specific write/review/refactor workflows
- **Standards** (`skills/*/standards/`): Idiom rules for F#, C#, TypeScript, Python, and cross-cutting quality
- **Templates** (`skills/repo-onboard/templates/`): Invariant files (hooks, issue templates, prompts)
- **Design** (`docs/`): Architecture docs including the v2 managed devex design

## Structure

```
manifest.yml                          # Content distribution manifest

.github/agents/
├── repo-onboarder.agent.md           # Bootstrap/upgrade agent
├── fsharp-dev.agent.md               # F# write/review/refactor
├── csharp-dev.agent.md               # C# write/review/refactor
├── python-dev.agent.md               # Python write/review/refactor
└── typescript-dev.agent.md           # TypeScript write/review/refactor

skills/
├── repo-onboard/
│   ├── SKILL.md                      # Full onboarding flow with 🛑 gates
│   ├── standards/                    # Cross-cutting rules
│   │   ├── code-quality.md           # EditorConfig, Tagless-Final, formatting
│   │   ├── documentation.md          # README, changelog, copilot-instructions
│   │   ├── git-workflow.md           # Branches, commits, PRs, AI attribution
│   │   └── testing.md               # Register, coverage, naming
│   └── templates/                    # Invariant files
│       ├── githooks/                 # pre-commit, pre-push
│       ├── github/                   # Issue templates, PR template, prompts
│       ├── project/                  # Testing register, learnings, ideas
│       └── root/                     # CHANGELOG
├── fsharp-dev/
│   ├── SKILL.md                      # F# write/review/refactor flow
│   └── standards/idiomatic-fsharp.md # 17-rule F# idiom standard
├── csharp-dev/
│   ├── SKILL.md                      # C# write/review/refactor flow
│   └── standards/
│       ├── idiomatic-csharp.md       # 14-rule C# idiom standard
│       ├── orleans-patterns.md       # Orleans 8+ grain design (topology: orleans-aspire-react)
│       ├── aspire-patterns.md        # Aspire 13 no-workload patterns (topology: orleans-aspire-react)
│       └── nuget-library.md          # NuGet packaging + publishing (topology: dotnet-lib)
├── python-dev/
│   ├── SKILL.md                      # Python write/review/refactor flow
│   └── standards/
│       ├── idiomatic-python.md       # 13-rule Python idiom standard
│       └── pypi-library.md           # PyPI packaging + publishing (topology: python-lib)
└── typescript-dev/
    ├── SKILL.md                      # TypeScript write/review/refactor flow
    └── standards/
        ├── idiomatic-typescript.md   # 12-rule TypeScript idiom standard
        └── react-vite-patterns.md    # React + Vite patterns (topology: orleans-aspire-react)

docs/
├── workspace-integration.md
└── design-v2-managed-devex.md        # v2 architecture — manifest, lockfile, verbs
```

## Content Types

| Type | Description | Upgrade Strategy |
|------|-------------|------------------|
| **invariant** | Same for every project. Copied verbatim. | Full replace |
| **template** | Variable substitution. No user edits expected. | Full replace |
| **agent** | Language agent with inline fallback + project context. | Replace managed section; preserve project context |
| **generated** | Rules engine output. Mix of managed + custom. | Replace managed sections only |
| **scaffolded** | Starting point. Fully user-owned after creation. | Never auto-upgraded |

## How Changes Work

1. To change what gets **copied**: edit the template in `templates/`
2. To change what gets **generated**: edit the standard in `standards/`
3. To add a **new distributable file**: add entry to `manifest.yml` + create source file
4. To improve a **language agent**: edit in `.github/agents/` — projects pull latest via GitHub fetch

## Contributing

When improving the toolkit:
- Prove the pattern in a real project first
- Generalise before committing here
- Note which project inspired the change in the commit message
- Update `manifest.yml` if adding/removing distributable files
