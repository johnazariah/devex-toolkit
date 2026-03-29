# Workspace Integration

## How to use devex-toolkit with your projects

The toolkit is designed to live alongside your project repos in a multi-root VS Code workspace. This way:

1. The onboarding agent is always available via `@repo-onboarder`
2. Improvements to the toolkit are immediately available across all projects
3. Field improvements from any project can be pushed back to the toolkit

### Setup

#### Option A: Multi-root workspace (recommended)

Create a `.code-workspace` file:

```json
{
  "folders": [
    { "path": "." },
    { "path": "../devex-toolkit", "name": "devex-toolkit" }
  ],
  "settings": {}
}
```

Or add via VS Code: File → Add Folder to Workspace → select `devex-toolkit`.

#### Option B: Git submodule

```bash
git submodule add https://github.com/johnazariah/devex-toolkit.git .devex-toolkit
```

Agents in `.devex-toolkit/.github/agents/` need to be symlinked to `.github/agents/`:
```bash
ln -s ../../.devex-toolkit/.github/agents/repo-onboarder.agent.md .github/agents/repo-onboarder.agent.md
```

#### Option C: Clone alongside

```bash
cd ~/work
git clone https://github.com/johnazariah/devex-toolkit.git
git clone https://github.com/johnazariah/my-project.git
```

Open both in a multi-root workspace.

### Onboarding a new repo

1. Open the target repo in VS Code (with devex-toolkit in the workspace)
2. Chat: `@repo-onboarder onboard this repo`
3. Answer the discovery questions
4. Review the proposed files at each 🛑 gate
5. Approve — files are written, hooks activated, initial commit created

### Improving the toolkit

When you discover a better pattern in a real project:

1. Make the improvement in your project first — prove it works
2. Generalise it into the toolkit: update the template and/or standard
3. Commit to devex-toolkit with a note about which project inspired the change
4. Other projects pick up the improvement next time they sync

### Keeping templates in sync

The toolkit is a **generator, not a runtime dependency**. Once files are generated in your project, they're independent — you can customise them freely. The toolkit doesn't auto-update existing repos.

To pull in toolkit improvements to an existing project:
1. Re-run the onboarding agent (it will ask before overwriting existing files)
2. Or manually diff the toolkit template against your project's version
