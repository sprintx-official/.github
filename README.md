# SprintX Organization Shared Workflows

This repository contains shared GitHub Actions workflows for all repositories in the `sprintx-official` organization.

## Available Workflows

### AI PR Review

Automated code review powered by Gemini AI. Reviews PRs against your project's coding conventions.

#### Usage

Add this to `.github/workflows/pr-review.yml` in any repo:

```yaml
name: AI PR Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  review:
    uses: sprintx-official/.github/.github/workflows/pr-review.yml@main
    secrets: inherit
```

#### Custom Configuration

```yaml
jobs:
  review:
    uses: sprintx-official/.github/.github/workflows/pr-review.yml@main
    with:
      conventions_path: 'docs/CONVENTIONS.md'  # Custom conventions file path
      project_path: 'docs/PROJECT.md'          # Custom project file path
      model: 'gemini-1.5-flash'                # Faster model (default: gemini-1.5-pro)
    secrets: inherit
```

#### What It Reviews

- **Convention Compliance**: File/component naming, imports, code style
- **Code Quality**: Clean code, error handling, DRY, typing
- **Security**: Hardcoded secrets, input validation, XSS/injection
- **Performance**: Re-renders, optimizations, bundle size
- **Best Practices**: Framework patterns, hooks, accessibility

## Setup (Organization Admin)

1. Add `GEMINI_API_KEY` as an organization secret:
   - Go to: Organization Settings > Secrets and variables > Actions
   - Create new organization secret
   - Grant access to all (or selected) repositories

2. Ensure this repository is **public** (required for reusable workflows)
