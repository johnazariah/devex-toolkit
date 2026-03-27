# Skill: Repo Onboard

## Purpose

Bootstrap a new repository with production-grade developer experience infrastructure. Discovers the project's language, stack, and conventions, then generates both invariant (same every time) and tailored (project-specific) files.

## Configuration

| Setting | Description |
|---------|-------------|
| Toolkit Root | Location of devex-toolkit (this repo) |
| Target Repo | The repository being onboarded |
| Templates Dir | `skills/repo-onboard/templates/` |
| Standards Dir | `skills/repo-onboard/standards/` |

## Invariant Files (copied verbatim, minor variable substitution)

These files are the same for every project:

| File | Source Template | Description |
|------|---------------|-------------|
| `.githooks/pre-commit` | `templates/githooks/pre-commit` | Secrets detection, CRLF fix, build check, testing register guard |
| `.githooks/pre-push` | `templates/githooks/pre-push` | Build + test gate before push |
| `.github/ISSUE_TEMPLATE/bug.md` | `templates/github/ISSUE_TEMPLATE/bug.md` | Bug report template |
| `.github/ISSUE_TEMPLATE/feature.md` | `templates/github/ISSUE_TEMPLATE/feature.md` | Feature request template |
| `.github/ISSUE_TEMPLATE/task.md` | `templates/github/ISSUE_TEMPLATE/task.md` | Task template |
| `.github/PULL_REQUEST_TEMPLATE.md` | `templates/github/PULL_REQUEST_TEMPLATE.md` | PR template with AI attribution |
| `.github/prompts/commit.prompt.md` | `templates/github/prompts/commit.prompt.md` | Conventional commit workflow |
| `.github/prompts/pick-next-work.prompt.md` | `templates/github/prompts/pick-next-work.prompt.md` | Work prioritisation |
| `.github/prompts/pr-prep.prompt.md` | `templates/github/prompts/pr-prep.prompt.md` | PR creation workflow |
| `.github/prompts/release.prompt.md` | `templates/github/prompts/release.prompt.md` | Release ceremony |
| `.github/prompts/debug-test-failure.prompt.md` | `templates/github/prompts/debug-test-failure.prompt.md` | Test failure investigation |
| `.project/testing-register.md` | `templates/project/testing-register.md` | Test catalog (empty scaffold) |
| `.project/STATUS.md` | Generated | Project status dashboard |
| `.project/learnings.md` | `templates/project/learnings.md` | Institutional memory — gotchas |
| `.project/future-enhancement-ideas.md` | `templates/project/future-enhancement-ideas.md` | Parking lot for ideas |
| `CHANGELOG.md` | `templates/root/CHANGELOG.md` | Keep a Changelog format |
| `LICENSE` | Generated | MIT license with owner name and year |

## Tailored Files (generated from discovery)

These files are customised per project:

| File | What's Customised |
|------|-------------------|
| `.github/copilot-instructions.md` | Architecture, tech stack, conventions, key files, build commands |
| `.github/workflows/ci.yml` | Build/test commands, toolchain setup, OS matrix |
| `.editorconfig` | Language-specific indent rules, encoding |
| `.gitattributes` | Language-specific extensions, binary declarations |
| `.gitignore` | Language/framework-specific ignores |
| `.vscode/settings.json` | Formatters, linters, tab sizes per language |
| `README.md` | Badges, description, architecture, commands, status |

---

## Flow

### Phase 1: Discovery

Ask the user or infer from existing files. Collect ALL of these before generating anything:

**1.1 — Project Identity**
- Project name
- One-line description
- GitHub owner/org (for badges, repo URL)
- License preference (default: MIT)
- Author name (for LICENSE)

**1.2 — Language & Stack**
- Primary language(s) and version (e.g. ".NET 10 / F#", "Python 3.12", "TypeScript 5")
- For .NET projects, **default to F# for core logic, C# for UI** (Avalonia/MAUI XAML bindings).
  Exception: if the project uses **Orleans**, default to C# throughout (Orleans serialization/code generation is C#-first and F# interop is painful).
- Build tool / commands (`dotnet build`, `npm run build`, `make`, etc.)
- Test tool / commands (`dotnet test`, `pytest`, `npm test`, etc.)
- Formatter (Fantomas, ruff, prettier, dotnet format, etc.)
- Linter (if separate from formatter)
- Package manager (NuGet, pip/uv, npm/pnpm, etc.)

**1.3 — Project Structure**
- Source directory layout (e.g. `src/`, `lib/`, flat)
- Test directory layout (e.g. `tests/`, `test/`, alongside source)
- Documentation location (e.g. `docs/`, wiki, none yet)
- Config file format (YAML, JSON, TOML, XML)

**1.4 — Architecture**
- Architecture pattern: **Tagless-Final** (default). Abstract capabilities as records/interfaces of functions, parameterized over effect type. Concrete implementations wired at composition root. See `standards/code-quality.md` for examples.
- Confirm with user or note if they prefer a different pattern.

**1.6 — CI/CD**
- CI platform (GitHub Actions assumed; confirm)
- OS targets (Linux only, macOS + Windows, etc.)
- Deploy targets (if any — NuGet, PyPI, npm, Docker, etc.)

**1.7 — AI & Tools**
- Ollama / local AI usage (for copilot-instructions context)
- MCP servers (for VS Code MCP config)
- DevContainer needed? (default: no)

### 🛑 STOP — Discovery Review

Present a summary table of all discovered settings. Ask:
> "Here's what I've discovered about your project. Anything to correct or add before I generate files?"

Wait for confirmation before proceeding.

---

### Phase 2: Generate Invariant Files

Copy the following templates verbatim from `templates/` to the target repo:

1. `.githooks/pre-commit` — substitute build command
2. `.githooks/pre-push` — substitute build + test commands
3. `.github/ISSUE_TEMPLATE/bug.md`
4. `.github/ISSUE_TEMPLATE/feature.md`
5. `.github/ISSUE_TEMPLATE/task.md`
6. `.github/PULL_REQUEST_TEMPLATE.md`
7. `.github/prompts/commit.prompt.md`
8. `.github/prompts/pick-next-work.prompt.md`
9. `.github/prompts/pr-prep.prompt.md`
10. `.github/prompts/release.prompt.md`
11. `.github/prompts/debug-test-failure.prompt.md`
12. `.project/testing-register.md`
13. `.project/learnings.md`
14. `.project/future-enhancement-ideas.md`
15. `CHANGELOG.md`
16. `LICENSE` — substitute author name and year

### Phase 3: Generate Tailored Files

Using the discovery data, generate:

**3.1 — `.github/copilot-instructions.md`**
Follow the structure in `standards/documentation.md`:
- Working style ("discuss, don't code" default)
- Project overview table
- Architecture diagram (if known)
- Technology stack table
- Solution structure
- Development commands
- Code conventions (from language standards)
- Testing conventions
- Key files table
- Tips for AI agents

**3.2 — `.github/workflows/ci.yml`**
- Trigger: push/PR to main/master
- OS matrix from discovery
- Steps: checkout → setup toolchain → restore → build → test
- Use pinned action versions

**3.3 — `.editorconfig`**
Apply rules from `standards/code-quality.md` with language-specific indent sizes.

**3.4 — `.gitattributes`**
LF enforcement for all text files. Language-specific extensions. Binary declarations.

**3.5 — `.gitignore`**
Language-specific patterns. IDE patterns. OS patterns. Private project directories.

**3.6 — `.vscode/settings.json`**
- Copilot commit message instructions
- Format-on-save
- LF line endings
- Per-language tab sizes
- Formatter/linter integration

**3.7 — `README.md`**
Follow the structure in `standards/documentation.md`:
- CI badge, License badge, language/version badge
- One-liner description
- "What it Does" section
- Architecture section
- Technology table
- Development commands
- Project status (Not Started for all phases, or inferred)
- Documentation links
- License

**3.8 — `.project/STATUS.md`**
Project name, phase, health, quick stats, phase table (if phases exist).

### 🛑 STOP — Generation Review

Present a file list of everything that will be created:
```
Files to create (N total):
  .githooks/pre-commit          [invariant]
  .githooks/pre-push            [invariant]
  .github/copilot-instructions.md [tailored]
  .github/workflows/ci.yml      [tailored]
  ...
```

Ask:
> "Ready to write these N files to your repo? I won't overwrite any existing files without asking."

Wait for confirmation.

---

### Phase 4: Write Files

1. Check for existing files — if any target path already exists, ask before overwriting.
2. Write all approved files.
3. Report what was created.

---

### Phase 5: Git Setup

1. Activate hooks: `git config core.hooksPath .githooks`
2. Initial commit (if repo is new): `git add -A && git commit -m "chore: bootstrap devex infrastructure"`
3. Offer to push: "Push to origin?"

### 🛑 STOP — Final Review

Show summary:
> "Onboarding complete. Created N files. Hooks active. Run `dotnet build` / `npm test` / etc. to verify."

---

## Error Handling

- If a file already exists, ask before overwriting
- If a build command is unknown, ask the user
- If the repo has no code yet, skip architecture and structure sections in copilot-instructions (mark as TODO)
- If git is not initialised, run `git init` first
