# Publish Workflow

A single GitHub Actions workflow that publishes both `flowise-embed` and `flowise-embed-react` to npm with matching version numbers.

## Flow

```
workflow_dispatch (bump, tag, optional: custom_version / recovery_version)
         |
         v
  dry-run job (no approval needed):
    validate inputs → build both packages → npm publish --dry-run → show package contents + version in summary
         |
         v
  reviewer approves (npm-publish environment gate)
         |
         v
  publish job:
    flowise-embed: set version → install → build → npm publish --tag <tag> → create version bump PR
         |
         v
    poll npm registry until new version is available
         |
         v
    flowise-embed-react: update dep + version → yarn upgrade → install → verify lock file → build → npm publish --tag <tag> → restore dep tag to latest → create version bump PR
```

## Usage

### Publishing a new version

1. Go to **Actions** > **"Publish flowise-embed + flowise-embed-react"** > **Run workflow**
2. Set `bump`:
   - `patch` — 3.1.3 → 3.1.4
   - `minor` — 3.1.3 → 3.2.0
   - `major` — 3.1.3 → 4.0.0
   - `prerelease` — 3.1.3 → 3.1.4-dev.0
   - `custom` — set any exact version in the `custom_version` field
3. Set `tag`:
   - `latest` (default) — all `npm install flowise-embed` users get this version
   - `dev` — only `npm install flowise-embed@dev` users get this version
4. Leave `recovery_version` empty
5. **Review the dry-run job summary** — check the resolved version, package contents, and dry-run publish output
6. **Approve the publish job** via the `npm-publish` environment gate
7. Both packages publish to npm and version bump PRs are created in both repos

Or via CLI:

```shell
gh workflow run publish.yml -f bump=patch -f tag=latest
gh workflow run publish.yml -f bump=prerelease -f tag=dev
gh workflow run publish.yml -f bump=custom -f custom_version=3.2.0 -f tag=latest
```

### Recovery

If `flowise-embed` published successfully but `flowise-embed-react` failed:

1. Go to **Actions** > **"Publish flowise-embed + flowise-embed-react"** > **Run workflow**
2. Leave `bump` at default (it won't be used)
3. Set `recovery_version` to the version already published, e.g. `3.1.4`
4. Review dry-run and approve the publish job
5. All `flowise-embed` steps are skipped — only `flowise-embed-react` is built, published, and gets a version bump PR

### Prerelease versions

Use `bump: prerelease` with `tag: dev` to publish dev versions:

| Current version | After prerelease bump |
| --------------- | --------------------- |
| `3.1.3`         | `3.1.4-dev.0`         |
| `3.1.4-dev.0`   | `3.1.4-dev.1`         |
| `3.1.4-dev.1`   | `3.1.4-dev.2`         |

Promote to stable with `bump: patch` (or `minor`/`major`) and `tag: latest`.

### Verifying a publish

After the publish job completes, confirm both packages are on npm with matching versions:

```shell
# Check specific version exists
npm view flowise-embed@<version> version
npm view flowise-embed-react@<version> version

# Check dist-tag points to the right version
npm view flowise-embed@latest version
npm view flowise-embed-react@dev version

# Check the version bump PRs were created
gh pr list --search "chore: bump flowise-embed"
gh pr list --repo FlowiseAI/FlowiseEmbedReact --search "chore: bump flowise-embed-react"
```

### Important: merge PRs before next publish

The workflow reads the current version from `package.json` on `main`. Since version bumps are delivered via PR (not direct push), you **must merge the version bump PRs before running the workflow again** — otherwise it will resolve to the same version and fail because it already exists on npm.

## Setup

### Secrets

Add to **FlowiseChatEmbed** repo settings (repository-level, not environment-level):

| Secret       | Description                                                                                                                                         |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NPM_TOKEN`  | npm **Automation** token with publish access to both `flowise-embed` and `flowise-embed-react`. Use the "Automation" type so it bypasses 2FA in CI. |
| `PAT_GITHUB` | GitHub Personal Access Token with write access to both `FlowiseChatEmbed` and `FlowiseEmbedReact` repos.                                            |

**Important:** Secrets must be **repository-level** (not environment-level) so the dry-run job can access them.

#### PAT options

- **Classic PAT:** `repo` scope
- **Fine-grained PAT (recommended):** Scope to the FlowiseAI org, select both repos, grant `Contents: Read and write` and `Pull requests: Read and write`

## How it works

### Two-job structure

Every publish goes through two jobs:

1. **dry-run** — builds both packages, runs `npm pack --dry-run` and `npm publish --dry-run`, displays the resolved version and package contents in the job summary. No approval needed — runs immediately.
2. **publish** — gated behind the `npm-publish` environment. Only runs after a reviewer approves. Publishes to npm and creates version bump PRs.

### Version resolution

`npx semver` computes the new version from the current `package.json` version and the bump type. For `custom`, the input is validated as valid semver. For `prerelease`, versions use the `dev` pre-id (e.g. `3.1.4-dev.0`).

### Dist-tags

Both packages are published with the same dist-tag. Use `latest` for stable releases and `dev` for pre-releases. The workflow warns (but doesn't block) if a prerelease version is published with `latest` or a stable version with `dev`.

### Dependency update in FlowiseEmbedReact

The workflow pins `devDependencies.flowise-embed` to the exact new version using `npm pkg set`, then runs `yarn upgrade flowise-embed@<version>` to explicitly force yarn to resolve and lock that version — this is more reliable than relying on `yarn install` alone to detect the `package.json` change. A verification step then greps `yarn.lock` to confirm the correct version was resolved, failing the build if not.

After publishing, the workflow restores the dependency specifier back to `"latest"` before creating the version bump PR. This means the PR commits `package.json` with `"flowise-embed": "latest"` and `yarn.lock` with the resolved version pinned.

### npm registry propagation

After publishing `flowise-embed`, there's a short delay before the version is available on the registry. The workflow polls `npm view` every 10 seconds for up to 2 minutes before proceeding to the `flowise-embed-react` steps.

### Version bump PRs

After publishing, the workflow creates version bump PRs in both repos instead of pushing directly to main:

| Repo              | PR branch                                  | Files changed               |
| ----------------- | ------------------------------------------ | --------------------------- |
| FlowiseChatEmbed  | `chore/bump-flowise-embed-<version>`       | `package.json`              |
| FlowiseEmbedReact | `chore/bump-flowise-embed-react-<version>` | `package.json`, `yarn.lock` |

### Husky suppression

`HUSKY=0` is set during `yarn install` to prevent the `prepare` script (`husky install`) from failing in CI.

### What if PR creation fails?

If npm publish succeeds but PR creation fails (e.g. permissions issue, branch already exists), the packages are already on npm — no rollback needed. You can manually create the version bump PR or push the version change directly. The PR step failing does not affect the published packages.
