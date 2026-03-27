# Standard: Documentation

## Copilot Instructions (`.github/copilot-instructions.md`)

Every repo must have a copilot-instructions file. Structure:

```markdown
# {Project} — Copilot Instructions

## Working Style
"Default: discuss, don't code. Propose changes and wait for approval."

## Project Overview
Table: Concept | Description (key domain concepts)

## Architecture
ASCII diagram or brief description of components

## Technology Stack
Table: Component | Choice

## Solution Structure
Directory tree of source layout

## Development Commands
Code block with build, test, run, publish commands

## Code Conventions
Language-specific rules (naming, patterns, idioms)

## Testing Conventions
Framework, naming, categories, register

## Key Files
Table: File | Purpose

## Tips for AI Agents
Numbered list of things agents should know/do
```

## README Structure

```markdown
# {Project Name}

Badges: CI | License | Language/Version

One-liner description.

## What It Does
Bullet list of capabilities.

## Architecture
Diagram or brief description.

## Technology
Table: Component | Choice

## Development
Build/test/run commands.

## Project Status
Phase/milestone table with status.

## Documentation
Links to design docs, specs, etc.

## License
```

## Changelog

Follow [Keep a Changelog](https://keepachangelog.com/):
- `## [Unreleased]` section at top
- Sections: Added, Changed, Fixed, Removed
- Each entry is one line, links to commit or issue

## Institutional Memory

### `.project/learnings.md`
Categorised gotchas. Each entry:
- **Learning**: What was discovered
- **Pattern**: The practice that follows
- **Rationale**: Why it matters
- **Source**: Where/when discovered

### `.project/future-enhancement-ideas.md`
Out-of-scope ideas with unique IDs, context, priority, and date added.

## Status Tracking

`.project/STATUS.md` contains:
- Current phase / milestone
- Health indicator (Green/Yellow/Red)
- Quick stats (tests, coverage, LOC)
- Phase/milestone table with status
- Key decisions summary
