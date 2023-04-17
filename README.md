# Actions

This repository contains a collection of GitHub Actions that can be used in your workflows.

# Usage
To use an action in your workflow, you must include it in your workflow file.

## Examples
### Changelog
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
        uses: maicol07/actions/.github/workflows/changelog.yml@main
        with:
            next_version: ${{ github.event.inputs.next_version }}
            config_dir: ${{ github.event.inputs.config_dir }}
            filename: ${{ github.event.inputs.filename }}
            path: ${{ github.event.inputs.path }}
            tag_query: ${{ github.event.inputs.tag_query }}
            commit_message: ${{ github.event.inputs.commit_message }}
```

### Release & Publish
> Note: You will need a `NPM_TOKEN` secret in your repository to publish the package. You can create one from the [NPM website](https://www.npmjs.com/settings/your-username/tokens). You can also use the [NPM CLI](https://docs.npmjs.com/creating-and-viewing-authentication-tokens) to create one.
```yaml
# Path: .github/workflows/release.yml
name: Release & Publish
on:
  workflow_dispatch:
    inputs:
      next_version:
        description: '[CHANGELOG] Next version tag to use instead of UNRELEASED'
        type: string
        required: false

      config_dir:
        description: '[CHANGELOG] git-chglog configuration directory'
        type: string
        default: '.chglog'
        required: false

      filename:
        description: '[CHANGELOG] Filename to write the changelog to'
        type: string
        required: false
        default: CHANGELOG.md

      path:
        description: '[CHANGELOG] Optional path to follow for directory'
        default: ''
        type: string
        required: false

      tag_query:
        description: '[CHANGELOG] Optional tag query to use for changelog generation'
        default: ''
        type: string
        required: false

      commit_message:
        type: string
        description: "[CHANGELOG] Commit message"
        required: false
      
      version_type:
        type: choice
        description: "Version bump type"
        required: false
        options:
          - patch
          - minor
          - major
          - rc
      release_channel:
        type: choice
        description: "Release channel"
        required: false
        options:
          - latest
          - next
          - beta
          - alpha

permissions:
  contents: write

jobs:
  changelog:
    uses: maicol07/actions/.github/workflows/changelog.yml@main
    with:
      next_version: ${{ github.event.inputs.next_version }}
      config_dir: ${{ github.event.inputs.config_dir }}
      filename: ${{ github.event.inputs.filename }}
      path: ${{ github.event.inputs.path }}
      tag_query: ${{ github.event.inputs.tag_query }}
      commit_message: ${{ github.event.inputs.commit_message }}
  release_publish:
    needs: changelog
    uses: maicol07/actions/.github/workflows/release_publish.yml@main
    with:
      version_type: ${{ github.event.inputs.version_type }}
      release_channel: ${{ github.event.inputs.release_channel }}
  secrets: inherit # Or pass just what secret you want. This is required to pass the NPM_TOKEN needed to publish the package
```
