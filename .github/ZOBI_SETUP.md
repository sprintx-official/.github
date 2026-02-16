# ZOBI PR Review — Setup & Architecture

## What is ZOBI?
AI-powered PR review bot using Gemini API. Lives in `sprintx-official/.github` as a **reusable workflow** that all org repos call.

---

## Key Files

| File | Purpose |
|------|---------|
| `.github/workflows/pr-review.yml` | The shared reusable workflow (single source of truth) |
| `.github/ZOBI_PROMPT_TEMPLATE.md` | Reference copy of the prompt. **Actual prompt is stored as org secret `ZOBI_REVIEW_PROMPT`** |

---

## How Repos Use It

Each repo needs ONE file: `.github/workflows/zobi-review.yml`

```yaml
name: ZOBI PR Review

on:
  pull_request:
    types: [opened, synchronize]
  issue_comment:
    types: [created]

jobs:
  # Normal PR trigger
  review-on-pr:
    if: github.event_name == 'pull_request'
    uses: sprintx-official/.github/.github/workflows/pr-review.yml@main
    secrets: inherit

  # @zobi comment trigger — must fetch PR data first
  get-pr-data:
    if: >
      github.event_name == 'issue_comment' &&
      github.event.issue.pull_request &&
      contains(github.event.comment.body, '@zobi')
    runs-on: ubuntu-latest
    outputs:
      pr_number: ${{ steps.pr.outputs.pr_number }}
      pr_head_ref: ${{ steps.pr.outputs.head_ref }}
      pr_base_ref: ${{ steps.pr.outputs.base_ref }}
      pr_author: ${{ steps.pr.outputs.author }}
      pr_title: ${{ steps.pr.outputs.title }}
      pr_body: ${{ steps.pr.outputs.body }}
      comment_body: ${{ steps.pr.outputs.comment_body }}
    steps:
      - name: Get PR details
        id: pr
        uses: actions/github-script@v7
        with:
          script: |
            const { data: pr } = await github.rest.pulls.get({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number
            });
            core.setOutput('pr_number', pr.number);
            core.setOutput('head_ref', pr.head.ref);
            core.setOutput('base_ref', pr.base.ref);
            core.setOutput('author', pr.user.login);
            core.setOutput('title', pr.title);
            core.setOutput('body', pr.body || '');
            const comment = context.payload.comment.body;
            const instruction = comment.replace(/@zobi\s*/i, '').trim();
            core.setOutput('comment_body', instruction);

  review-on-comment:
    needs: get-pr-data
    uses: sprintx-official/.github/.github/workflows/pr-review.yml@main
    with:
      pr_number: ${{ fromJSON(needs.get-pr-data.outputs.pr_number) }}
      pr_head_ref: ${{ needs.get-pr-data.outputs.pr_head_ref }}
      pr_base_ref: ${{ needs.get-pr-data.outputs.pr_base_ref }}
      pr_author: ${{ needs.get-pr-data.outputs.pr_author }}
      pr_title: ${{ needs.get-pr-data.outputs.pr_title }}
      pr_body: ${{ needs.get-pr-data.outputs.pr_body }}
      reviewer_instruction: ${{ needs.get-pr-data.outputs.comment_body }}
    secrets: inherit
```

---

## @zobi Usage (in PR comments)

| Comment | Result |
|---------|--------|
| `@zobi` | Full standard review |
| `@zobi check if TypeScript types are used properly` | Review focused on types |
| `@zobi look for security issues` | Review focused on security |
| Any comment without `@zobi` | Nothing triggered |

---

## Required Org Secrets

| Secret | Required | Purpose |
|--------|----------|---------|
| `GEMINI_API_KEY` | Yes | Gemini API access |
| `ZOBI_REVIEW_PROMPT` | Yes | Full prompt template (copy from `ZOBI_PROMPT_TEMPLATE.md`) |
| `DEV_PROFILES_PAT` | Optional | Write access to `sprintx-official/dev-profiles` repo |

---

## Known Fixes Applied

### 1. `.cjs` extension fix
`review_script.js` renamed to `review_script.cjs` to avoid ES module conflicts when the target repo has `"type": "module"` in `package.json`. Affected repo: `website-v4`.

### 2. `issue_comment` context fix
When triggered via comment, `github.event.pull_request`, `github.head_ref`, `github.base_ref` are ALL empty. Fixed by:
- Caller workflow fetches PR data via GitHub API in `get-pr-data` job
- Passes as inputs: `pr_number`, `pr_head_ref`, `pr_base_ref`, `pr_author`, `pr_title`, `pr_body`
- Shared workflow uses `inputs.pr_head_ref || github.head_ref` pattern as fallback

### 3. `reviewer_instruction` support
Comment text after `@zobi` is extracted and passed as `reviewer_instruction` input → injected into prompt as `{{reviewerInstruction}}` → Gemini treats it as primary focus.

---

## Updating the Prompt

If you edit `ZOBI_PROMPT_TEMPLATE.md`, you must also update the `ZOBI_REVIEW_PROMPT` org secret:
1. Go to `https://github.com/organizations/sprintx-official/settings/secrets/actions`
2. Edit `ZOBI_REVIEW_PROMPT`
3. Paste the full content of `ZOBI_PROMPT_TEMPLATE.md`
