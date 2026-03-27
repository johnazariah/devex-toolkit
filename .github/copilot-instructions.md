# DevEx Toolkit — Copilot Instructions

## Working Style

**Default: discuss, don't code.** This is a template/standards repo — changes affect all future projects.

## What This Repo Is

A reusable developer experience toolkit that bootstraps new repos with production-grade infrastructure. Contains:

- **Agent** (`agents/repo-onboarder.agent.md`): AI agent that onboards new repos interactively
- **Skill** (`skills/repo-onboard/SKILL.md`): Full onboarding flow with discovery, generation, and review gates
- **Templates** (`skills/repo-onboard/templates/`): Ready-to-copy files (hooks, issue templates, prompts, etc.)
- **Standards** (`skills/repo-onboard/standards/`): Rules for generating tailored files (code quality, testing, docs, git workflow)

## Structure

```
agents/
└── repo-onboarder.agent.md

skills/repo-onboard/
├── SKILL.md
├── standards/
│   ├── git-workflow.md
│   ├── testing.md
│   ├── documentation.md
│   └── code-quality.md
└── templates/
    ├── githooks/          # pre-commit, pre-push
    ├── github/            # Issue templates, PR template, prompts, CI
    ├── project/           # Testing register, learnings, future ideas
    └── root/              # CHANGELOG

docs/
└── workspace-integration.md
```

## How Changes Work

1. Templates in `templates/` are **invariant** — same for every project, minor variable substitution
2. Standards in `standards/` are **rules** — the onboarding agent reads them when generating tailored files
3. To change what gets generated: edit the standard. To change what gets copied: edit the template.

## Contributing

When improving the toolkit:
- Prove the pattern in a real project first
- Generalise before committing here
- Note which project inspired the change in the commit message
