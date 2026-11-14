# Actions

This repository contains a collection of GitHub Actions that can be used in your workflows.

# Usage
To use an action in your workflow, you must include it in your workflow file.

## Examples
### Changelog (deprecated)
> Deprecated: use `release.yml` (see below).
```yaml
# Path: .github/workflows/changelog.yml
name: Changelog
on:
  workflow_dispatch:
    inputs:
      next_version:
        description: 'Next version tag to use instead of UNRELEASED'
        type: string
        required: false

      config_dir:
        description: 'git-chglog configuration directory'
        type: string
        default: '.chglog'
        required: false

      filename:
        description: 'Filename to write the changelog to'
        type: string
        default: 'CHANGELOG.md'
        required: false

      path:
        description: 'Optional path to follow for directory'
        default: ''
        type: string
        required: false

      tag_query:
        description: 'Optional tag query to use for changelog generation'
        default: ''
        type: string
        required: false

      commit_message:
        type: string
        description: "Commit message"
        required: false

permissions:
    contents: write

jobs:
    changelog:
        uses: maicol07/actions/.github/workflows/changelog.yml@v1
        with:
            next_version: ${{ github.event.inputs.next_version }}
            config_dir: ${{ github.event.inputs.config_dir }}
            filename: ${{ github.event.inputs.filename }}
            path: ${{ github.event.inputs.path }}
            tag_query: ${{ github.event.inputs.tag_query }}
            commit_message: ${{ github.event.inputs.commit_message }}
```

### Release & Publish (deprecated)
> Deprecated: use `release.yml` + `npm-publish.yml` (see below).

> Note: You will need a `NPM_TOKEN` secret in your repository to publish the package. You can create one from the [NPM website](https://www.npmjs.com/settings/your-username/tokens). You can also use the [NPM CLI](https://docs.npmjs.com/creating-and-viewing-authentication-tokens) to create one.
```yaml
# Path: .github/workflows/release.yml
name: Release & Publish
on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write

jobs:
  release_publish:
    uses: maicol07/actions/.github/workflows/release_publish_npm.yml@v1
    with:
      release_channel: ${{ github.event.inputs.release_channel }}
    secrets: inherit # Or pass just what secret you want. This is required to pass the NPM_TOKEN needed to publish the package
```

### CI pipeline (security, commit lint, PHP, release PR, Docker)
GitHub port of the Trust-IT `websites/utils/ci-templates` flow, one reusable workflow per job group:

| Workflow | Jobs | Inputs |
|---|---|---|
| `security.yml` | `secret_detection` (Betterleaks), `lockfile_scan` + `config_scan` (Trivy) | - |
| `commit-lint.yml` | `commit_lint`: Conventional Commits, gitmoji after the colon | - |
| `php.yml` | PHP + cached `composer install` + a command | `php_version`, `extensions`, `run` |
| `js.yml` | Bun (or Node.js + pnpm/npm without a `bun.lock`) + install + a command | `run`, `node_version` |
| `gradle.yml` | JDK + Gradle (cached) + a command | `java_version`, `run` |
| `release.yml` | Release PR on `release/next` (git-cliff), GitHub Release once merged | `changelog`, `tag_pattern`, `next_version`, `bump`; secret `token`; output `tag` |
| `docker.yml` | Build & push of the `prod` target, Trivy image scan (report only) | `registry`, `image`, `tag`, `target`, `build_args`; secrets `registry_username`, `registry_password` |
| `npm-publish.yml` | `bun publish` of a release tag | `tag`, `dist_tag`; secret `npm_token` |
| `portainer.yml` | New tag in a Portainer Swarm stack file, redeploy, smoke test | `stack_id`, `endpoint_id`, `image`, `tag`, `url`; secrets `portainer_url`, `portainer_api_key` |

The `setup-js` composite action (`maicol07/actions/setup-js@v1`) is the JS setup of `js.yml` and `npm-publish.yml`.
Reference everything at `@v1` (a moving major tag): Dependabot's `github-actions` ecosystem then proposes `v2`.

To override a job, replace its `uses:` with your own steps under the same job name, or leave it out.
`release.yml` uses the project's `cliff.toml`, or this repo's one. A `fork-point` tag limits the changelog
and the commit lint to the commits after it (e.g. a fork of an upstream project).
Tags and PRs created with `GITHUB_TOKEN` trigger no workflow: chain the image build on the `tag` output.

```yaml
# Path: .github/workflows/ci.yml
on:
  push:
    branches: [main]
  pull_request:

jobs:
  security:
    uses: maicol07/actions/.github/workflows/security.yml@v1
  commit_lint:
    uses: maicol07/actions/.github/workflows/commit-lint.yml@v1
  tests:
    uses: maicol07/actions/.github/workflows/php.yml@v1
    with:
      php_version: '8.5'
      run: vendor/bin/pest --ci
  release:
    needs: [security, tests]
    if: github.event_name == 'push'
    uses: maicol07/actions/.github/workflows/release.yml@v1
    permissions:
      contents: write
      pull-requests: write
  docker:
    needs: release
    if: needs.release.outputs.tag
    uses: maicol07/actions/.github/workflows/docker.yml@v1
    permissions:
      packages: write
    with:
      registry: ghcr.io
      image: owner/app
      tag: ${{ needs.release.outputs.tag }}
    secrets:
      registry_username: ${{ github.actor }}
      registry_password: ${{ secrets.GITHUB_TOKEN }}
```

### Dependabot
Dependabot has no shared config: each repo needs its own `.github/dependabot.yml`, e.g.:
```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    groups:
      github-actions:
        patterns: ['*']
  # + composer, bun, gradle, docker, docker-compose... as the repo needs
```
