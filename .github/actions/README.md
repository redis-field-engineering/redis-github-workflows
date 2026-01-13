# Composite Actions for Gradle Releases

This directory contains modular, reusable composite actions for building and releasing Gradle projects.

## Available Actions

**Core Actions:**
- 🔧 **setup-gradle** - Setup Java and Gradle with caching
- 🏷️ **create-release-tag** - Create release tag using Axion
- 🏗️ **gradle-build** - Build and publish with Gradle

**Release & Deploy:**
- 🚀 **jreleaser** - GitHub release, Maven Central deploy, Slack notifications

**Documentation & Distribution:**
- 📝 **update-antora-version** - Update docs/antora.yml version
- 📋 **clone-to-dist-repo** - Clone release to -dist repository

---

### 🔧 setup-gradle
Setup Java and Gradle with caching.

**Inputs:**
- `java-version` (optional, default: `17`) - Java version to use
- `java-distribution` (optional, default: `temurin`) - Java distribution

**Example:**
```yaml
- uses: redis-field-engineering/redis-github-workflows/.github/actions/setup-gradle@main
  with:
    java-version: '17'
```

---

### 🏷️ create-release-tag
Create a release tag using Axion Release Plugin.

**Inputs:**
- `version` (optional) - Specific version to release
- `version-increment` (optional, default: `patch`) - Version increment type (major, minor, patch)
- `git-access-token` (required) - GitHub token for pushing tags

**Outputs:**
- `version` - The version that was released

**Example:**
```yaml
- uses: redis-field-engineering/redis-github-workflows/.github/actions/create-release-tag@main
  id: release
  with:
    version-increment: 'minor'
    git-access-token: ${{ secrets.GITHUB_TOKEN }}
```

---

### 🏗️ gradle-build
Build and optionally publish with Gradle.

**Inputs:**
- `tasks` (optional, default: `build`) - Gradle tasks to run
- `gpg-secret-key` (optional) - GPG secret key for signing
- `gpg-public-key` (optional) - GPG public key
- `gpg-passphrase` (optional) - GPG passphrase
- `sonatype-username` (optional) - Sonatype username
- `sonatype-password` (optional) - Sonatype password

**Example:**
```yaml
- uses: redis-field-engineering/redis-github-workflows/.github/actions/gradle-build@main
  with:
    tasks: 'build publish'
    gpg-secret-key: ${{ secrets.GPG_SECRET_KEY }}
    gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
    sonatype-username: ${{ secrets.SONATYPE_USERNAME }}
    sonatype-password: ${{ secrets.SONATYPE_PASSWORD }}
```

---

### 🚀 jreleaser
Run JReleaser to create GitHub release, deploy to Maven Central, and send notifications.

**Inputs:**
- `version` (required) - Project version
- `git-access-token` (required) - GitHub token for creating release
- `gpg-secret-key` (optional) - GPG secret key for signing
- `gpg-public-key` (optional) - GPG public key
- `gpg-passphrase` (optional) - GPG passphrase
- `sonatype-username` (optional) - Sonatype username
- `sonatype-password` (optional) - Sonatype password
- `slack-webhook` (optional) - Slack webhook URL
- `jreleaser-version` (optional, default: `latest`) - JReleaser version
- `arguments` (optional, default: `full-release`) - JReleaser arguments

**What it does:**
- Creates GitHub release with changelog
- Signs and uploads artifacts
- Deploys to Maven Central
- Sends Slack notifications
- All configured via `jreleaser.yml` in your project

**Example:**
```yaml
- uses: redis-field-engineering/redis-github-workflows/.github/actions/jreleaser@main
  with:
    version: ${{ steps.release.outputs.version }}
    git-access-token: ${{ secrets.GITHUB_TOKEN }}
    gpg-secret-key: ${{ secrets.GPG_SECRET_KEY }}
    gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
    sonatype-username: ${{ secrets.SONATYPE_USERNAME }}
    sonatype-password: ${{ secrets.SONATYPE_PASSWORD }}
    slack-webhook: ${{ secrets.SLACK_WEBHOOK }}
```

---

### 📝 update-antora-version
Update docs/antora.yml version to match release version.

**Inputs:**
- `version` (required) - Version to set in antora.yml
- `git-access-token` (required) - GitHub token for pushing changes

**Example:**
```yaml
- uses: redis-field-engineering/redis-github-workflows/.github/actions/update-antora-version@main
  with:
    version: ${{ steps.release.outputs.version }}
    git-access-token: ${{ secrets.GITHUB_TOKEN }}
```

---

### 📋 clone-to-dist-repo
Clone GitHub release to a -dist repository.

**Inputs:**
- `git-access-token` (required) - GitHub token with access to both repos
- `source-repo` (required) - Source repository (e.g., owner/repo)
- `dest-repo` (required) - Destination repository (e.g., owner/repo-dist)

**Example:**
```yaml
- uses: redis-field-engineering/redis-github-workflows/.github/actions/clone-to-dist-repo@main
  with:
    git-access-token: ${{ secrets.GIT_ACCESS_TOKEN }}
    source-repo: ${{ github.repository }}
    dest-repo: ${{ github.repository }}-dist
```

---



## Complete Example Workflow

Here's a complete example of using these actions together:

```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Release version (optional)'
        required: false
      version-increment:
        description: 'Version increment type'
        default: 'incrementPatch'
        type: choice
        options:
          - incrementPatch
          - incrementMinor
          - incrementMajor

permissions:
  contents: write
  packages: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GIT_ACCESS_TOKEN }}

      - name: Setup Gradle
        uses: redis-field-engineering/redis-github-workflows/.github/actions/setup-gradle@main
        with:
          java-version: '17'

      - name: Create Release Tag
        id: release
        uses: redis-field-engineering/redis-github-workflows/.github/actions/create-release-tag@main
        with:
          version: ${{ inputs.version }}
          version-increment: ${{ inputs.version-increment }}
          git-access-token: ${{ secrets.GIT_ACCESS_TOKEN }}

      - name: Build
        uses: redis-field-engineering/redis-github-workflows/.github/actions/gradle-build@main
        with:
          tasks: 'build'

      - name: Release with JReleaser
        uses: redis-field-engineering/redis-github-workflows/.github/actions/jreleaser@main
        with:
          version: ${{ steps.release.outputs.version }}
          git-access-token: ${{ secrets.GIT_ACCESS_TOKEN }}
          gpg-secret-key: ${{ secrets.GPG_SECRET_KEY }}
          gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
          sonatype-username: ${{ secrets.SONATYPE_USERNAME }}
          sonatype-password: ${{ secrets.SONATYPE_PASSWORD }}
          slack-webhook: ${{ secrets.SLACK_WEBHOOK }}

      - name: Clone to Dist Repo
        uses: redis-field-engineering/redis-github-workflows/.github/actions/clone-to-dist-repo@main
        with:
          git-access-token: ${{ secrets.GIT_ACCESS_TOKEN }}
          source-repo: ${{ github.repository }}
          dest-repo: ${{ github.repository }}-dist

      - name: Update Antora Version
        uses: redis-field-engineering/redis-github-workflows/.github/actions/update-antora-version@main
        with:
          version: ${{ steps.release.outputs.version }}
          git-access-token: ${{ secrets.GIT_ACCESS_TOKEN }}
```

## Benefits of Modular Actions

✅ **Flexibility** - Use only the actions you need  
✅ **Composability** - Mix and match in any order  
✅ **Reusability** - Use in different workflows (not just releases)  
✅ **Visibility** - See exactly what each step does  
✅ **Customization** - Add your own steps between actions  
✅ **Testing** - Test individual actions in isolation  

## Migration from Reusable Workflow

If you're currently using the `gradle-release-axion.yml` reusable workflow, you can migrate to individual actions for more control.

**Before (Reusable Workflow):**
```yaml
jobs:
  release:
    uses: redis-field-engineering/redis-github-workflows/.github/workflows/gradle-release-axion.yml@main
    with:
      java-version: '17'
      gradle-build-tasks: 'build publish'
    secrets:
      git-access-token: ${{ secrets.GIT_ACCESS_TOKEN }}
```

**After (Individual Actions):**
```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: redis-field-engineering/redis-github-workflows/.github/actions/setup-gradle@main
      - uses: redis-field-engineering/redis-github-workflows/.github/actions/create-release-tag@main
      - uses: redis-field-engineering/redis-github-workflows/.github/actions/gradle-build@main
      # ... more actions
```

The reusable workflow remains available for projects that prefer the all-in-one approach.
