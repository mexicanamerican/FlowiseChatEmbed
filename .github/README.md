# Publish Workflow

A single GitHub Actions workflow that publishes both `flowise-embed` and `flowise-embed-react` to npm with matching version numbers.

## Flow

```
workflow_dispatch (bump_type: patch / minor / major / exact version)
         |
         v
  reviewer approves (environment gate)
         |
         v
  flowise-embed: bump version -> install -> build -> npm publish -> commit + tag + push
         |
         v
  poll npm registry until new version is available
         |
         v
  flowise-embed-react: update dep + version -> install (updates yarn.lock) -> build -> npm publish -> commit + tag + push
```

## Usage

### Publishing a new version

1. Go to **Actions** > **"Publish flowise-embed + flowise-embed-react"** > **Run workflow**
2. Set `bump_type`:
   - `patch` — 3.1.2 -> 3.1.3
   - `minor` — 3.1.2 -> 3.2.0
   - `major` — 3.1.2 -> 4.0.0
   - Or an exact version like `3.2.0`
3. Leave `recovery_version` empty
4. Approve the environment gate when prompted
5. Both packages publish to npm and both repos receive a version commit + git tag

### Recovery

If `flowise-embed` published successfully but `flowise-embed-react` failed:

1. Go to **Actions** > **"Publish flowise-embed + flowise-embed-react"** > **Run workflow**
2. Leave `bump_type` at default (it won't be used)
3. Set `recovery_version` to the version already published, e.g. `3.1.3`
4. Approve the environment gate
5. All `flowise-embed` steps are skipped — only `flowise-embed-react` is built and published

## Setup

### Secrets

Add to **FlowiseChatEmbed** repo settings:

| Secret       | Description                                                                                                                                         |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NPM_TOKEN`  | npm **Automation** token with publish access to both `flowise-embed` and `flowise-embed-react`. Use the "Automation" type so it bypasses 2FA in CI. |
| `PAT_GITHUB` | GitHub Personal Access Token with access to both `FlowiseChatEmbed` and `FlowiseEmbedReact` repos.                                                  |

#### PAT options

- **Classic PAT:** `repo` scope
- **Fine-grained PAT (recommended):** Scope to the FlowiseAI org, select both repos, grant `Contents: Read and write`

### Environment

Create a `production` environment in **FlowiseChatEmbed** repo:

This gates every workflow run behind a human approval step.

## How it works

### Version bump

`npm version <bump_type> --no-git-tag-version` updates `package.json` without creating npm's default `v3.1.3` tag. The workflow controls the exact tag format: `flowise-embed@3.1.3`.

### Dependency update in FlowiseEmbedReact

The workflow sets `devDependencies.flowise-embed` to the exact new version using `npm pkg set`. This changes the specifier (e.g. from `"latest"` to `"3.1.3"`), which forces yarn to re-resolve from the registry and update `yarn.lock`. Both `package.json` and `yarn.lock` are committed back to the repo.

### npm registry propagation

After publishing `flowise-embed`, there's a short delay before the version is available on the registry. The workflow polls `npm view` every 10 seconds for up to 2 minutes before proceeding to the `flowise-embed-react` steps.

### Husky suppression

`HUSKY=0` is set during `yarn install` for FlowiseChatEmbed to prevent the `prepare` script (`husky install`) from failing in CI where there's no git hook context.

## What gets committed

| Repo              | Files committed             | Tag format                  |
| ----------------- | --------------------------- | --------------------------- |
| FlowiseChatEmbed  | `package.json`              | `flowise-embed@3.1.3`       |
| FlowiseEmbedReact | `package.json`, `yarn.lock` | `flowise-embed-react@3.1.3` |
