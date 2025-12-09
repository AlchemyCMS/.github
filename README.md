# Reusable Release Workflows for AlchemyCMS Gems

This repository contains reusable GitHub Actions workflows for automating gem releases across all AlchemyCMS repositories.

## Overview

The release process is fully automated with three workflows:

1. **prepare-release.yml** - Creates a release PR with version bump and changelog
2. **release.yml** - Publishes the gem to RubyGems and creates a GitHub release
3. **post-release.yml** - Bumps to next development version after release

## Adding Workflows to Your Gem

Create three workflow files in your gem's `.github/workflows/` directory:

### 1. `.github/workflows/prepare-release.yml`
```yaml
name: Prepare Release

on:
  workflow_dispatch:
    inputs:
      bump:
        description: 'Version bump type. Choose "release" for finalizing a pre-release (8.0.0.dev → 8.0.0), or patch/minor/major to simply bump version.'
        required: true
        type: choice
        default: 'patch'
        options:
          - release
          - patch
          - minor
          - major

jobs:
  prepare:
    uses: AlchemyCMS/.github/workflows/prepare-release.yml@main
    with:
      version_file_path: lib/alchemy/YOUR_GEM/version.rb  # UPDATE THIS
      bump: ${{ inputs.bump }}
    secrets:
      app_id: ${{ vars.ALCHEMY_BOT_APP_ID }}
      app_private_key: ${{ secrets.ALCHEMY_BOT_APP_PRIVATE_KEY }}
```

### 2. `.github/workflows/release.yml`
```yaml
name: Publish Release

on:
  workflow_dispatch:
  pull_request:
    types: [closed]
    branches:
      - main

jobs:
  publish:
    if: github.event_name == 'workflow_dispatch' || (github.event.pull_request.merged == true && startsWith(github.event.pull_request.head.ref, 'release/v'))
    uses: AlchemyCMS/.github/workflows/release.yml@main
    with:
      version_file_path: lib/alchemy/YOUR_GEM/version.rb  # UPDATE THIS
    secrets:
      app_id: ${{ vars.ALCHEMY_BOT_APP_ID }}
      app_private_key: ${{ secrets.ALCHEMY_BOT_APP_PRIVATE_KEY }}
```

### 3. `.github/workflows/post-release.yml`
```yaml
name: Post Release

on:
  workflow_run:
    workflows: ["Publish Release"]
    types:
      - completed

jobs:
  bump-dev-version:
    if: github.event.workflow_run.conclusion == 'success'
    uses: AlchemyCMS/.github/workflows/post-release.yml@main
    with:
      version_file_path: lib/alchemy/YOUR_GEM/version.rb  # UPDATE THIS
    secrets:
      app_id: ${{ vars.ALCHEMY_BOT_APP_ID }}
      app_private_key: ${{ secrets.ALCHEMY_BOT_APP_PRIVATE_KEY }}
```

### Configuration

**Update `version_file_path` in all three workflows** to match your gem's version file location. For example:
- `lib/alchemy/solidus/version.rb`
- `lib/alchemy/devise/version.rb`
- `lib/alchemy_cms/version.rb`

The workflows will automatically use the organization's `ALCHEMY_BOT_APP_ID` variable and `ALCHEMY_BOT_APP_PRIVATE_KEY` secret.

## How to Release

### Step 1: Prepare Release

1. Go to **Actions** tab in your gem repository
2. Select **Prepare Release** workflow
3. Click **Run workflow**
4. Choose the version bump type:
   - **release** - Finalize a pre-release (e.g., `8.0.0.dev` → `8.0.0`)
   - **patch** - Increment patch version (e.g., `1.2.3` → `1.2.4`)
   - **minor** - Increment minor version (e.g., `1.2.3` → `1.3.0`)
   - **major** - Increment major version (e.g., `1.2.3` → `2.0.0`)

This creates a PR with:
- Updated version file
- Generated changelog from GitHub release notes
- Release branch `release/vX.Y.Z`

### Step 2: Review and Merge

Review the release PR and merge it to `main`. This automatically triggers:
1. **Publish Release** - Publishes gem to RubyGems and creates GitHub release
2. **Post Release** - Bumps version to next dev version (e.g., `1.2.0` → `1.3.0.dev`)

## Requirements

Your gem repository must have:

1. **Version file** with `VERSION = "x.y.z"` constant (e.g., `lib/alchemy/solidus/version.rb`)
2. **CHANGELOG.md** file in the repository root
3. **Trusted publishing** configured on RubyGems for the gem
4. **Main branch** named `main`

These are standard across all AlchemyCMS gems.

## What Each Workflow Does

**prepare-release.yml**
- Calculates next version based on bump type and current version
- Strips pre-release suffixes (.dev, .alpha, etc.) when using "release" bump
- Creates release branch `release/vX.Y.Z`
- Generates changelog from GitHub release notes API
- Updates version file and prepends to CHANGELOG.md
- Creates PR to main branch

**release.yml**
- Triggers when a `release/v*` PR is merged to main
- Sets up Ruby and installs dependencies
- Publishes gem to RubyGems using trusted publishing
- Creates GitHub release with auto-generated notes

**post-release.yml**
- Triggers after successful release
- Bumps to next minor dev version (e.g., `1.2.0` → `1.3.0.dev`)
- Commits version change directly to main branch

## Advanced: Version Pinning

The examples above use `@main` to always use the latest workflow version. You can also pin to specific versions:

```yaml
# Pin to a tag (recommended for stability)
uses: AlchemyCMS/.github/workflows/release.yml@v1

# Pin to a specific commit (maximum control)
uses: AlchemyCMS/.github/workflows/release.yml@abc123
```

Using `@main` ensures you automatically get updates and fixes, which is recommended for most AlchemyCMS gems.
