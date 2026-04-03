# Standard: Git Workflow

## CI Cost Control

**Self-hosted runners are mandatory for macOS and Windows CI.** GitHub-hosted macOS runners cost 10x Linux minutes; Windows costs 2x. Only use `ubuntu-latest` for Linux-only jobs.

**Artifact retention: 3 days.** Set `retention-days: 3` on all `actions/upload-artifact` steps. GitHub artifact storage is not for long-term retention.

```yaml
# Example: cross-platform CI matrix with self-hosted runners
# Runner labels from the shared BuildBeast pool
jobs:
  build:
    strategy:
      matrix:
        include:
          - os: ubuntu-latest                              # GitHub-hosted
          - os: [self-hosted, macOS, ARM64]                # self-hosted Mac
          - os: [self-hosted, Windows, X64, buildbeast]    # self-hosted Windows
    runs-on: ${{ matrix.os }}
```

## Branch Naming

**Default branch: `main`.** All new repos must use `main` as the default branch, not `master`. Configure globally with:

```bash
git config --global init.defaultBranch main
```

**Enforcement:** `@repo-onboarder setup` detects if the repo is on `master` and renames to `main` automatically. The pre-commit hook warns if it detects a `master` branch.

Feature branch format: `{type}/{issue-id}-{short-description}`

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

Examples:
- `feat/42-email-sync`
- `fix/17-crlf-handling`
- `docs/3-architecture-update`

## Conventional Commits

Subject: `<type>: <imperative description>`

| Type | When |
|------|------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `chore` | Build, CI, config, tooling |

### Rules
- Subject line: imperative mood, lowercase, no trailing period, max 72 chars
- Body: wrap at 72 chars, explain what and why (not how)
- One logical change per commit

### AI Attribution Trailers

When commits are AI-assisted, include ALL three trailers (all-or-none):

```
Co-authored-by: GitHub Copilot <noreply@github.com>
agent: github-copilot
model: <model-name>
```

## Pull Requests

- Title follows conventional commit format
- Description uses the PR template (What/Why + Changes table + AI tools used)
- All CI checks must pass before merge
- Squash merge by default; merge commit for large features with meaningful history

## Hooks

| Hook | Gate |
|------|------|
| pre-commit | Secrets scan, CRLF fix, build, testing register sync |
| pre-push | Build + full test suite |

Hooks are activated via `git config core.hooksPath .githooks`.

For .NET projects, auto-activate in `Directory.Build.targets`:
```xml
<Target Name="EnsureGitHooks" BeforeTargets="PrepareForBuild">
  <Exec Command="git config core.hooksPath .githooks" />
</Target>
```
