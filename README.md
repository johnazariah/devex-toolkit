# DevEx Toolkit

**Reusable developer experience infrastructure for new repositories.**

An opinionated set of templates, standards, and an AI onboarding agent that bootstraps any new repo with production-grade devex:

- Testing register with pre-commit enforcement
- Git hooks (secrets, build gate, documentation sync)
- CI/CD pipeline (GitHub Actions)
- Issue & PR templates with AI attribution
- Copilot instructions + prompt library
- Institutional memory (learnings, future ideas)
- README with badges, architecture, status
- Changelog, license, .editorconfig, .gitattributes

## Usage

### Option A: Run the onboarding agent in VS Code

1. Clone this repo alongside your project
2. In VS Code with Copilot, open your target repo
3. Ask: `@repo-onboarder onboard this repo`
4. The agent discovers your project context and generates everything

### Option B: Copy templates manually

Browse `skills/repo-onboard/templates/` and copy what you need.

## What's Inside

```
agents/
└── repo-onboarder.agent.md         # AI agent — thin router to the skill

skills/repo-onboard/
├── SKILL.md                         # Full onboarding flow with 🛑 gates
├── standards/                       # Rules for customization
│   ├── git-workflow.md              # Branch naming, commits, PRs, AI attribution
│   ├── testing.md                   # Register format, coverage, test naming
│   ├── documentation.md             # README structure, changelog, copilot-instructions
│   └── code-quality.md              # EditorConfig, line endings, formatting
└── templates/                       # Ready-to-copy files
    ├── githooks/                    # pre-commit, pre-push
    ├── github/                      # Issue templates, PR template, prompts, CI
    ├── project/                     # STATUS.md, testing register, learnings, ideas
    ├── vscode/                      # VS Code settings
    └── root/                        # .editorconfig, .gitattributes, CHANGELOG, LICENSE
```

## Origin

Synthesized from patterns across 5 production repos (Aura, Pelican, Emic, Encoding, Hypervelocity Engineering Toolkit), battle-tested on .NET/F#, Python, and mixed-language projects.
