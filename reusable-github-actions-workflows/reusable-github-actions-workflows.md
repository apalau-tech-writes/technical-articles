# How to Create Reusable CI/CD Pipeline Templates with GitHub Actions

---

<!--
## COLLABORATION INSTRUCTIONS

This file is actively being drafted. You can edit it directly in VS Code and use the following markers to request changes. When done, tell the AI assistant "process my edits" and it will action every marker and save the updated file.

MARKERS:
  <<REMOVE>>                         — delete this block
  <<REWRITE: your instructions>>     — rewrite with specific guidance
  <<SHORTEN>>                        — make this more concise
  <<EXPAND: what to add>>            — add more detail on something
  <<NOTE: your comment>>             — a note for consideration, no direct action required

EXAMPLE:
  <<REWRITE: make the tone less formal and cut the last two sentences>>

-->

---

## Introduction

You've seen it happen. A new microservice gets created, someone copies the CI/CD pipeline YAML from another repo, tweaks a few names, and pushes it. Six months later there are fifteen repos with fifteen slightly-different versions of the same pipeline — each one a fork of the original that has since drifted in its own direction. One repo got the updated Node version. Another got the fixed caching logic. A third has a security scanning step nobody else knows about. When a compliance requirement forces a change across all of them, someone has to open fifteen pull requests and hope nothing gets missed.

This is the copy-paste problem in CI/CD, and it's more common than most teams want to admit. The DRY principle — Don't Repeat Yourself — is well understood at the code level, but it often gets ignored the moment engineers open a `.github/workflows/` directory.

GitHub Actions reusable workflows are the solution. They let you define a pipeline once in a central repository and call it from any other repository in your organization. The called workflow runs exactly as written — with its own jobs, runners, and secrets — and the calling workflow simply passes in the inputs it needs. When you need to update that pipeline, you update it in one place. Every repo that references it picks up the change on its next run.

This guide walks you through building a real-world shared workflows setup from scratch. You'll create a dedicated repository to host your reusable workflows, write a parameterized workflow using the `workflow_call` trigger, call it from a separate application repository, and version it properly using git tags. Along the way, you'll also see where reusable workflows fit versus GitHub's other reuse mechanism — composite actions — so you can reach for the right tool in the right situation.

By the end, you'll have a pattern you can apply immediately to stop the copy-paste cycle and start treating your CI/CD pipelines as the shared infrastructure they actually are.

**Prerequisites:** Familiarity with GitHub Actions basics — you should be comfortable reading and writing workflow YAML, understand what jobs and steps are, and have a GitHub account. No prior experience with reusable workflows is required.

---

## Section 1: Reusable Workflows vs. Composite Actions

Before diving into building your first reusable workflow, it's worth understanding that GitHub Actions gives you two different mechanisms for sharing pipeline logic: **reusable workflows** and **composite actions**. They solve related but distinct problems, and picking the wrong one creates friction down the road.

### Reusable Workflows

A reusable workflow is a complete workflow file — with `on`, `jobs`, runners, and steps — that another workflow can invoke using the `workflow_call` trigger. When it runs, it executes as its own independent workflow: it defines its own runners, manages its own environment, and appears in the Actions UI with its own job entries and full step-by-step logs.

Think of a reusable workflow as a **pipeline template**. It's the right tool when you want to standardize an entire CI/CD process — a build-test-deploy sequence, a security scanning pipeline, a release workflow — and call that same pipeline from multiple repositories.

<!-- SCREENSHOT PLACEHOLDER 1
     What to capture: GitHub Actions UI showing a completed workflow run that called a reusable workflow.
     The key thing to show: the workflow run page where both the caller job AND the called workflow's jobs
     appear as distinct, expandable job entries with their own step-level logs.
     Suggested caption: "Figure 1: A reusable workflow run in the GitHub Actions UI — each job and step
     in the called workflow appears as a first-class entry with its own logs." -->

### Composite Actions

A composite action bundles multiple steps into a single reusable action, which then runs as one step inside a caller workflow's job. Unlike a reusable workflow, a composite action doesn't define its own runner — it runs on whatever machine the caller's job is already running on. In the Actions UI, the entire composite action appears as a single collapsed step in the calling job's log; you don't see the individual steps inside it unless you expand it.

Think of a composite action as a **shared task template**. It's the right tool when you want to package a set of steps that should always run together as a logical unit — setting up a custom environment, running a standard linting pass, publishing an artifact — and plug that unit into different jobs.

<!-- SCREENSHOT PLACEHOLDER 2
     What to capture: GitHub Actions UI showing a job that uses a composite action.
     The key thing to show: the job's step list where the composite action appears as a single step
     (collapsed), with the internal steps only visible when expanded.
     Suggested caption: "Figure 2: A composite action in the GitHub Actions UI — the entire action
     appears as a single step in the calling job, with internal steps nested inside." -->

### Key Differences at a Glance

| | Reusable Workflow | Composite Action |
|---|---|---|
| **Unit of reuse** | Full workflow (multiple jobs) | Steps within a job |
| **Runner control** | Defines its own runners | Inherits the caller's runner |
| **Secrets access** | Via `secrets` context | Must be passed as explicit inputs |
| **Input types** | Typed (`string`, `boolean`, `number`) | Always strings |
| **Logging** | Full job and step logs | Single collapsed step |
| **Parallelism** | Can run multiple jobs in parallel | Runs sequentially within a job |
| **Nesting** | Up to 4 levels deep | Can be used inside reusable workflows |

### How to Choose

Use a **reusable workflow** when:
- You want to standardize an entire pipeline across multiple repositories
- Your reusable logic needs to run on a specific type of runner (for example, a Windows runner for a .NET build, or a larger runner for a resource-heavy test suite)
- You need parallel jobs within the shared logic
- You want full job and step visibility in the Actions UI

Use a **composite action** when:
- You want to share a group of steps that will run inside an existing job
- The steps must run on the same runner as the calling job
- You're packaging setup logic (installing tools, configuring environments) that different workflows need as a step, not as a separate job

In practice, these two aren't mutually exclusive. A reusable workflow can call composite actions inside its own steps — and this is actually a common pattern. Your organization might have a composite action that handles environment setup, which gets called from inside a reusable workflow that handles the full build-test-deploy cycle.

For the rest of this guide, we'll focus on reusable workflows, since they're the right tool for the "shared pipeline template" problem we're solving.

---

## Section 2: Setting Up a Shared Workflows Repository

The first step is creating a dedicated repository to host your reusable workflows. This is the single source of truth that every other repository in your organization will reference — so it's worth setting it up properly from the start.

### Creating the Repository

Create a new repository in your GitHub organization. The name should be self-explanatory — something any engineer new to the org can discover and understand without having to ask around. A vague or overly generic name creates confusion about ownership and purpose, which matters when this repository becomes load-bearing infrastructure for every team's pipelines.

For this guide, we'll use `shared-workflows`.

### Choosing Visibility

How you set the repository's visibility determines which repositories can call your reusable workflows:

**Public** — Any repository, inside or outside your organization, can reference your workflows. This is the simplest option if your organization's code is already public, or if you intentionally want to share your workflows with the broader community.

**Internal** (GitHub Enterprise/organization accounts) — The repository is visible to all members of your organization but not to the public. Any repository within the organization can call its workflows without additional configuration. This is the recommended option for most teams on GitHub Enterprise — you get broad internal access without exposing anything externally.

**Private** — Only accessible to repositories you explicitly allow. This gives you the tightest control, but it requires an extra configuration step: you must go to the shared workflows repository's **Settings → Actions → General → Access** and set it to allow access from repositories in your organization.

<!-- SCREENSHOT PLACEHOLDER 3
     What to capture: GitHub repository Settings page, navigated to Actions > General, showing the
     "Access" section with the option selected to allow access from repositories in the organization.
     Suggested caption: "Figure 3: Enabling cross-repository access for a private shared workflows
     repository — Settings → Actions → General → Access." -->

> **A note on private shared repos:** If you enable access from your organization, be aware that contributors in caller repositories can view the workflow run logs, which may include output from your shared workflow's steps. If your shared workflows emit sensitive values (they shouldn't, but it happens), this is worth keeping in mind.

For most teams, **internal** (if available on your plan) or **public within a private org** is the practical choice. Don't let visibility configuration become a blocker — you can change it later.

### Repository Structure

Reusable workflows must live in the `.github/workflows/` directory of the repository — the same directory used for regular workflows. There's no special subdirectory or naming convention required by GitHub, but a clear file naming scheme saves everyone from digging through YAML to understand what a workflow does.

A clean starting structure looks like this:

```
shared-workflows/
├── .github/
│   └── workflows/
│       ├── build-node.yml         # Reusable build workflow for Node.js services
│       ├── build-docker.yml       # Reusable Docker build and push workflow
│       ├── deploy-kubernetes.yml  # Reusable Kubernetes deployment workflow
│       └── security-scan.yml     # Reusable security scanning workflow
├── CODEOWNERS
└── README.md
```

### Protecting the Repository

Because every team's pipelines depend on this repository, treat it like infrastructure. A few practices that pay off quickly:

**Set up a `CODEOWNERS` file.** Require review from your platform or DevOps team for any change to the `.github/workflows/` directory. A mistake here can break CI across every repo that calls the affected workflow.

```
# .github/CODEOWNERS
.github/workflows/   @your-org/platform-team
```

**Enable branch protection on `main`.** Require pull request reviews and passing status checks before anything merges. This is the same discipline you'd apply to application code, and it matters just as much here.

**Use a clear PR process.** Because changes to a reusable workflow take effect immediately for any caller pinned to a branch, it helps to document in your README how changes are tested and what the expected rollout process is. Even a short paragraph sets expectations across teams.

With the repository created and configured, you're ready to write your first reusable workflow.

---

## Section 3: Creating Your First Reusable Workflow

With the `shared-workflows` repository ready, it's time to write an actual reusable workflow. In this section we'll build a parameterized Node.js build-and-test workflow that any application repository can call. It covers the three pillars of the `workflow_call` interface — inputs, secrets, and outputs — so you'll have a pattern you can adapt to any stack.

### The `workflow_call` Trigger

What makes a workflow reusable is simple: instead of (or in addition to) `push`, `pull_request`, or `schedule`, it listens on `on: workflow_call`. That's the entry point that other workflows use to invoke it.

```yaml
on:
  workflow_call:
```

A workflow with only this trigger can't run on its own — it has no other event to wake it up. In practice, it's common to also add `workflow_dispatch` during development so you can trigger the workflow manually to test it without needing a caller:

```yaml
on:
  workflow_call:
  workflow_dispatch:  # useful for testing in isolation
```

> **Note:** Remove `workflow_dispatch` before promoting to production if you don't want the workflow to be triggerable outside of a caller.

### Defining Inputs

Inputs let the caller customize the workflow's behavior at runtime — think of them as function parameters. You define them under `on.workflow_call.inputs`, and each one requires a `type`. The supported types for `workflow_call` are `string`, `boolean`, and `number`.

```yaml
on:
  workflow_call:
    inputs:
      node-version:
        description: "Node.js version to use for the build"
        type: string
        required: false
        default: "20"
      run-lint:
        description: "Whether to run the linter as part of the build"
        type: boolean
        required: false
        default: true
      upload-artifact:
        description: "Whether to upload the build artifact"
        type: boolean
        required: false
        default: false
```

Inside the workflow's jobs and steps, you reference inputs through the `inputs` context:

```yaml
steps:
  - uses: actions/setup-node@v4
    with:
      node-version: ${{ inputs.node-version }}
```

A few things worth knowing up front:

- **Always provide defaults for optional inputs.** Callers will omit optional inputs regularly. A missing default for a `string` input evaluates to an empty string, a missing `boolean` defaults to `false`, and a missing `number` defaults to `0`. Be explicit so the behavior is obvious.
- **Don't overload inputs.** A reusable workflow with fifteen inputs is harder to call correctly than two workflows with eight inputs each. If your workflow is growing a long input list, it's usually a sign it's trying to do too much.

### Defining Secrets

Secrets follow the same pattern as inputs but live under `on.workflow_call.secrets`. Inside the workflow, you access them via the `secrets` context.

```yaml
on:
  workflow_call:
    secrets:
      npm-token:
        description: "NPM registry token for installing private packages"
        required: false
```

Referencing a secret in a step:

```yaml
steps:
  - name: Install dependencies
    run: npm ci
    env:
      NPM_TOKEN: ${{ secrets.npm-token }}
```

If you'd rather pass all of the caller's secrets through at once instead of mapping each one explicitly, GitHub also supports a `secrets: inherit` shorthand — see the [official documentation](https://docs.github.com/en/actions/sharing-automations/reusing-workflows#passing-secrets-to-called-workflows) for details.

### Defining Outputs

Outputs let the called workflow surface values back to the caller — for example, a build artifact name, a version string, or a test result summary. This is where the plumbing gets a little more involved.

GitHub Actions outputs flow in one direction: **step → job → workflow**. You can't reference a step output directly at the workflow level. You first promote the step output to a job output, then map that job output to the workflow's `on.workflow_call.outputs`.

```yaml
on:
  workflow_call:
    outputs:
      artifact-name:
        description: "Name of the uploaded build artifact"
        value: ${{ jobs.build.outputs.artifact-name }}
```

Then in the job:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.set-artifact-name.outputs.artifact-name }}
    steps:
      - id: set-artifact-name
        run: echo "artifact-name=my-app-${{ github.sha }}" >> $GITHUB_OUTPUT
```

> **Watch out:** If you skip the job-level outputs mapping and try to reference a step output directly in `on.workflow_call.outputs`, GitHub won't throw an error — it will silently pass an empty string to the caller. This is one of the more frustrating silent failures in reusable workflows.

### A Complete Example

Putting it all together, here's a complete `build-node.yml` to save in `shared-workflows/.github/workflows/`:

```yaml
name: Build and Test (Node.js)

on:
  workflow_call:
    inputs:
      node-version:
        description: "Node.js version to use"
        type: string
        required: false
        default: "20"
      run-lint:
        description: "Whether to run the linter"
        type: boolean
        required: false
        default: true
      upload-artifact:
        description: "Whether to upload the build artifact"
        type: boolean
        required: false
        default: false
    secrets:
      npm-token:
        description: "NPM registry token for private packages"
        required: false
    outputs:
      artifact-name:
        description: "Name of the uploaded build artifact"
        value: ${{ jobs.build.outputs.artifact-name }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.set-artifact-name.outputs.artifact-name }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: "npm"

      - name: Configure NPM token
        if: secrets.npm-token != ''
        run: echo "//registry.npmjs.org/:_authToken=${{ secrets.npm-token }}" > ~/.npmrc

      - name: Install dependencies
        run: npm ci

      - name: Lint
        if: ${{ inputs.run-lint }}
        run: npm run lint

      - name: Build
        run: npm run build

      - name: Test
        run: npm test

      - name: Set artifact name
        id: set-artifact-name
        run: echo "artifact-name=${{ github.event.repository.name }}-${{ github.sha }}" >> $GITHUB_OUTPUT

      - name: Upload artifact
        if: ${{ inputs.upload-artifact }}
        uses: actions/upload-artifact@v4
        with:
          name: ${{ steps.set-artifact-name.outputs.artifact-name }}
          path: dist/
```

This workflow is ready to be called from any repository that has a Node.js project with `lint`, `build`, and `test` scripts in `package.json`. In Section 4, we'll write the caller side.

---

## Section 4: Calling the Reusable Workflow from Another Repository

With `build-node.yml` in place in `shared-workflows`, any repository in your organization can now call it. Here's how the caller side works.

### The `uses` Keyword

In a caller workflow, you invoke a reusable workflow at the **job level** using `uses` — not at the step level, which is where you reference actions. The full syntax is:

```
uses: {organization}/{repo}/.github/workflows/{filename}@{ref}
```

Where `{ref}` is a branch name, a git tag, or a full commit SHA. A minimal caller looks like this:

```yaml
jobs:
  build:
    uses: your-org/shared-workflows/.github/workflows/build-node.yml@main
```

### Passing Inputs and Secrets

Inputs are passed via `with`, exactly like you'd pass inputs to an action. Secrets are passed via `secrets`, with each one mapped by name:

```yaml
jobs:
  build:
    uses: your-org/shared-workflows/.github/workflows/build-node.yml@main
    with:
      node-version: "18"
      run-lint: true
      upload-artifact: true
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

The names in `with` and `secrets` must match the input and secret names defined in the reusable workflow's `on.workflow_call` block exactly.

### Consuming Outputs

If the reusable workflow defines outputs, a downstream job in the caller can access them via the `needs` context. The calling job must be listed in the downstream job's `needs` array first:

```yaml
jobs:
  build:
    uses: your-org/shared-workflows/.github/workflows/build-node.yml@main
    with:
      node-version: "20"
      upload-artifact: true
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
```

### Pinning to a Tag or SHA

In the example above, the caller references `@main`. That works, but it means any change merged to `main` in `shared-workflows` immediately affects every caller pinned to that branch — with no review gate on the consuming side. For production pipelines, it's better to pin to a tag or SHA.

Pin to a tag:
```yaml
uses: your-org/shared-workflows/.github/workflows/build-node.yml@v1.2.0
```

Pin to a commit SHA:
```yaml
uses: your-org/shared-workflows/.github/workflows/build-node.yml@a3f8d21c
```

Pinning to a SHA is the most stable option — a tag can be moved, but a SHA is immutable. We'll cover how to manage tags and releases for your shared workflows in Section 5.

### A Complete Caller Workflow

Here's a full example of what a caller workflow looks like in an application repository, saved as `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  build-and-test:
    uses: your-org/shared-workflows/.github/workflows/build-node.yml@v1.2.0
    with:
      node-version: "20"
      run-lint: true
      upload-artifact: ${{ github.ref == 'refs/heads/main' }}
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}

  deploy:
    needs: build-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: echo "Deploying artifact ${{ needs.build-and-test.outputs.artifact-name }}"
```

A couple of things worth noting in this example. The `upload-artifact` input uses an expression to conditionally upload only on pushes to `main` — this is a practical pattern for avoiding unnecessary artifact storage on PR builds. And the `deploy` job gates on both `needs: build-and-test` and `if: github.ref == 'refs/heads/main'`, so it only runs when the build succeeded and we're on the main branch.

---

## Section 5: Versioning Your Shared Workflows

How callers pin to your shared workflows determines how much control you have over rollout and how much risk callers take on when you make changes. Getting this right is what separates a shared workflows setup that teams trust from one they quietly work around.

### The Three Pinning Options

You already saw these in Section 4 — here's the full tradeoff picture:

**Branch (`@main`)** — Always resolves to the latest commit on that branch. Changes you merge to `main` take effect immediately for every caller pinned there. Useful during early development when you're iterating quickly across the shared repo and the callers at the same time. Not appropriate for production pipelines.

**Tag (`@v1.2.0`)** — Resolves to a specific tagged commit. Callers don't pick up changes until they explicitly update their `uses` line to a newer tag. This gives teams control over when they adopt changes and makes it straightforward to roll back by pointing to an older tag. The tradeoff: tags are mutable by default — someone with push access can move a tag to a different commit.

**Full commit SHA (`@a3f8d21c...`)** — The most stable option. A SHA is immutable — it points to exactly one commit, forever, and no one can change that. This is the right choice when security or reproducibility requirements are strict. GitHub's own security hardening documentation recommends SHA pinning for this reason. The downside is that SHAs aren't human-readable, and updating them requires looking up the new commit hash rather than just bumping a version number.

> **2025 update:** GitHub now supports immutable releases — once marked immutable, a release's assets and git tag cannot be changed or deleted. If your organization uses immutable releases, tags become as stable as SHAs for the workflows they point to. Check your organization's GitHub plan for availability.

For most teams, **semantic version tags** are the right default: they're readable, give callers an explicit upgrade step, and are easy to reason about in PR reviews.

### Creating and Pushing Tags

When you're ready to cut a release from your `shared-workflows` repo:

```bash
# Create an annotated tag
git tag -a v1.0.0 -m "Initial stable release"

# Push the tag to GitHub
git push origin v1.0.0
```

Then create a GitHub Release from that tag — this is optional but recommended. A release gives you a place to write changelog notes, which your callers will appreciate when deciding whether to upgrade.

### A Practical Versioning Strategy

The pattern GitHub itself uses for its own Actions is worth adopting:

- **Full semver tags** (`v1.0.0`, `v1.1.0`, `v2.0.0`) for every release — these are the immutable reference points.
- **Floating major version tags** (`v1`, `v2`) that always point to the latest stable release within that major version.

This lets callers choose their own risk tolerance. A team that wants stability pins to `v1.0.0` and never gets surprised. A team that's comfortable with non-breaking changes within a major version pins to `v1` and gets improvements automatically.

For your `ci.yml` callers, a reasonable policy is:
- Internal dev/staging workflows: pin to `v1` (float within major)
- Production workflows: pin to the full `v1.x.x` tag, and update deliberately via PR

---

## Section 6: Common Gotchas and Best Practices

### Common Gotchas

Reusable workflows have a few sharp edges that tend to bite teams when they're first setting things up. These are the ones worth knowing before you hit them in production.

**Environment variables don't propagate from caller to called.** If you set variables in the `env` context at the workflow level in your caller, they are not available in the called workflow. This catches people off guard because it's the opposite of what you'd expect coming from shell scripting or most CI systems. The correct pattern is to pass values explicitly as typed inputs via `with` — not as env vars.

**`GITHUB_TOKEN` permissions can only be downgraded, never upgraded.** If the caller workflow grants `GITHUB_TOKEN` a limited permission set, the called workflow cannot exceed those permissions — even if its own permission block asks for more. Plan your permission model at the caller level and make sure it covers everything the reusable workflow needs.

**Secrets don't pass transitively through nested workflows.** In a chain of A → B → C, workflow C only receives the secrets that B explicitly passes to it. If A passes a secret to B but B doesn't forward it to C, C won't have it — and there's no error, it just won't be there. When you nest reusable workflows, audit the full secret chain.

**A job with `uses` cannot also have `steps`.** A job is either a caller of a reusable workflow or a job with its own steps — never both. This is a hard constraint, not a style guideline. If you need to run additional steps after a reusable workflow completes, put them in a separate downstream job using `needs`.

**Outputs silently fail if you skip the job-level mapping.** Step outputs must be promoted to job outputs before they can be exposed as workflow outputs. Referencing a step output directly in `on.workflow_call.outputs` won't error — it will just surface as an empty string to the caller. If your outputs are coming back blank, this is the first thing to check.

**The `uses` value must be a literal string.** You can't use an expression or a variable to dynamically construct the path to a reusable workflow. `uses: ${{ env.WORKFLOW_PATH }}` is not valid. This is intentional — GitHub enforces literal values to make it possible to audit and restrict which workflows can be called.

**Nesting limits.** You can chain reusable workflows up to four levels deep, and a single caller workflow can reference a maximum of 50 unique reusable workflows in total across all nesting levels. These limits are rarely hit in practice, but they're worth knowing if you're building a platform layer with multiple tiers of shared workflows.

### Best Practices

**Protect your shared workflows repository like infrastructure.** Set up a `CODEOWNERS` file pointing to your platform or DevOps team, enable branch protection on `main`, and require PR reviews before anything merges. A broken change here can affect every pipeline in your organization simultaneously.

**Keep your reusable workflow inputs minimal and well-documented.** A workflow with a long, undocumented input list is harder to call correctly and harder to maintain. Describe each input clearly in the `description` field — it's the only documentation most callers will ever read.

**Test reusable workflows before promoting to a version tag.** Create a feature branch in `shared-workflows` and point a test caller at `@your-branch-name` before merging and tagging. This gives you a real end-to-end test without risking callers on `main` or a stable tag.

**Use floating major version tags alongside full semver tags.** Maintaining a `v1` tag that always points to the latest stable `v1.x.x` release gives callers a choice: pin tightly to `v1.2.3` for maximum stability, or follow `v1` to get non-breaking improvements automatically. The `git tag -fa v1` pattern keeps this manageable.

**Set up Dependabot in caller repositories.** Configuring the `github-actions` ecosystem in each caller's `dependabot.yml` means version bump PRs are opened automatically when you cut a new release — no repo silently falls behind on a stale pin.

---

## Reference Links

- [Reusing workflows — GitHub Docs](https://docs.github.com/en/actions/sharing-automations/reusing-workflows)
- [Workflow syntax for GitHub Actions — GitHub Docs](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Sharing actions and workflows from your private repository — GitHub Docs](https://docs.github.com/en/actions/creating-actions/sharing-actions-and-workflows-from-your-private-repository)
- [Creating a composite action — GitHub Docs](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)
- [Security hardening for GitHub Actions — GitHub Docs](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [Managing releases in a repository — GitHub Docs](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)
- [Configuring Dependabot version updates — GitHub Docs](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuring-dependabot-version-updates)
- [Using immutable releases and tags — GitHub Docs](https://docs.github.com/en/actions/creating-actions/releasing-and-maintaining-actions)

---

## Conclusion

Reusable workflows won't solve every problem in your CI/CD setup, but they solve the right one: the slow sprawl of copied, diverged pipeline YAML that becomes harder to maintain with every new repository. With a shared workflows repository, a clean `workflow_call` interface, and a versioning strategy your teams can reason about, you have the foundation for treating your pipelines with the same discipline you'd apply to application code.

The pattern we've built here — a shared repo, a parameterized reusable workflow, a versioned caller, and a set of guardrails around common failure modes — is the same pattern that scales from a handful of repositories to hundreds. Start small, get one team calling the shared workflow in production, and let the value speak for itself. The second team is always easier to onboard than the first.
