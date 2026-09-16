# fidelta-v/ci-workflows

Reusable GitHub Actions workflows for the fidelta-v Vite + React + TypeScript
projects. One place to change PR review rules and CI checks; every consumer
repo picks up the update automatically.

## What's inside

| Workflow | Purpose |
|---|---|
| `.github/workflows/claude-review.yml` | Runs Claude Code as an automated PR reviewer against a strict fidelta-v rulebook (16 sections: file placement, naming, common components, Zod, RTK Query, Redux, section order, hardcoded values, TS hygiene, React hygiene, styling, i18n, a11y, file hygiene, correctness, security). |
| `.github/workflows/vite-ci.yml` | Runs `lint`, `format:check` (Prettier), and `build` (typecheck + Vite build) in parallel. Each job toggleable. |

## Setup checklist for a new consumer repo

1. Repo must expose these npm scripts: `lint`, `format:check`, `build`.
   (See "pharmacy-app baseline" section below for the standard config.)
2. Repo must have `CLAUDE_CODE_OAUTH_TOKEN` as an Actions secret
   (repo-level or, better, org-level). Generate with `claude setup-token`.
3. Add the two caller workflows below to `.github/workflows/`.
4. Open a PR to `main` — Claude will review, and CI will run lint / format / build.

## Caller: Claude PR Review

Copy this into `<consumer-repo>/.github/workflows/claude-review.yml`:

```yaml
name: Claude Review
on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

jobs:
  review:
    uses: fidelta-v/ci-workflows/.github/workflows/claude-review.yml@v1
    with:
      project_description: "my-repo-name (one-line context)"
      # model: claude-sonnet-5   # override to claude-haiku-4-5 or claude-opus-5 if needed
      # additional_rules: |      # optional repo-specific rules
      #   - This repo uses react-hook-form for forms; flag manual form state.
      #   - This repo uses tRPC on the API side; ignore section E (RTK Query).
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `project_description` | string | `a fidelta-v Vite + React + TypeScript project` | One-line context Claude uses in its prompt. |
| `model` | string | `claude-sonnet-5` | Claude model. Options: `claude-sonnet-5` (recommended), `claude-haiku-4-5` (cheapest), `claude-opus-5` (highest quality). |
| `additional_rules` | string | `""` | Multiline block of extra review rules appended to the base prompt. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | yes | Claude Code OAuth token from `claude setup-token` (Pro/Max) OR an Anthropic API key. |

## Caller: Vite CI (lint + format + build)

Copy this into `<consumer-repo>/.github/workflows/ci.yml`:

```yaml
name: CI
on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

jobs:
  ci:
    uses: fidelta-v/ci-workflows/.github/workflows/vite-ci.yml@v1
    # with:
    #   node_version: "20"
    #   run_lint: true
    #   run_format_check: true
    #   run_build: true
    #   package_manager: npm     # or pnpm / yarn
```

### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `node_version` | string | `"20"` | Node major version. |
| `run_lint` | boolean | `true` | Run `npm run lint`. |
| `run_format_check` | boolean | `true` | Run `npm run format:check`. Set false if the repo hasn't been Prettier-baselined yet. |
| `run_build` | boolean | `true` | Run `npm run build` (typecheck + build). |
| `package_manager` | string | `"npm"` | `npm`, `pnpm`, or `yarn`. |

## pharmacy-app baseline (standard scripts, deps, config)

Every consumer repo must have this in `package.json` for the CI to work:

```json
{
  "scripts": {
    "lint": "eslint .",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "build": "tsc -b && vite build"
  },
  "devDependencies": {
    "prettier": "^3.3.3",
    "eslint-config-prettier": "^9.1.0"
  }
}
```

Plus a `.prettierrc` and `.prettierignore` at the repo root — see
`examples/prettierrc.json` and `examples/prettierignore` in this repo.

## Versioning

Consumers pin to a tag. Bump the tag when the review rulebook changes:

```
git tag -a v1 -m "Initial reusable workflows"
git push origin v1
```

Consumers using `@v1` pick up patches to that tag automatically. For a
breaking change, cut `v2` and let consumers migrate on their own schedule.

## When to update this repo vs. a consumer repo

Update this repo when:
- The review rulebook should change for every fidelta-v Vite project.
- CI defaults should change (Node version, script names, etc.).
- A new rule applies broadly.

Update the consumer repo (via `additional_rules` in its caller) when:
- The rule only applies to that repo's stack (e.g. it uses react-hook-form
  instead of Zod-only forms).
- The rule refers to a name unique to that repo.

## Cost

Per-review cost with the default model (Sonnet 5):
- Small PR (<200 LOC): ~$0.04
- Medium PR (200-1000 LOC): ~$0.13
- Large PR (>1000 LOC): ~$0.30

With Claude Pro OAuth (`claude setup-token`), reviews consume the Pro
subscription's rolling 5-hour quota instead of billing per-token.
