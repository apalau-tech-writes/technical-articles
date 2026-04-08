# How to integrate SonarQube Cloud with GitHub Actions for automated code quality

Code quality issues rarely surface at the right time. A vulnerability gets merged, a bug slips past review, and by the time anyone notices, it's already in production. The problem isn't that developers don't care — it's that manual code review doesn't scale, and feedback that arrives too late gets ignored.

SonarQube Cloud solves this by bringing automated static analysis directly into your development workflow. When integrated with GitHub Actions, it analyzes every push and every pull request automatically, surfacing bugs, vulnerabilities, and code smells before they ever reach your main branch. And with quality gates, you can configure your pipeline to actually *block* a merge when code doesn't meet your standards — not just report on it.

By the end of this tutorial, you'll have a working GitHub Actions workflow that:

- Triggers a SonarQube Cloud analysis on every push and pull request
- Enforces a quality gate that fails the workflow if issues are found
- Decorates pull requests with inline feedback directly in GitHub

This guide uses the [SonarQube Scan Action](https://github.com/marketplace/actions/official-sonarqube-scan), which works for any language or project type. If you're using Maven or Gradle, the same concepts apply — only the configuration differs.

---

## Why CI-based analysis over SonarQube Automatic Analysis

SonarQube Cloud offers two ways to analyze your code: SonarQube Automatic Analysis and CI-based analysis.

SonarQube Automatic Analysis is the zero-configuration option — SonarQube Cloud reads directly from your repository and starts analyzing without any pipeline setup. It's a great way to get a first look at your codebase, but it comes with real limitations. It only supports GitHub repositories, it doesn't support branch analysis beyond your default branch, and it has no support for code coverage, monorepos, or external rule engine reports. Critically, you also have no access to analysis logs when things go wrong.

CI-based analysis is the recommended approach for any team that's serious about code quality. By running the SonarQube Scan Action directly inside your GitHub Actions workflow, you get full control over when and how analysis runs, complete branch and pull request analysis, code coverage integration, and — most importantly — the ability to enforce a quality gate that actually stops a build.

If SonarQube Automatic Analysis is a read-only observer, CI-based analysis is an active gatekeeper.

---

## Prerequisites

Before setting up the integration, make sure you have the following in place:

**A SonarQube Cloud account and organization**
Sign up at [sonarcloud.io](https://sonarcloud.io) using your GitHub account. Once logged in, import your GitHub organization to create a SonarQube Cloud organization. A free plan is available for public repositories.

**A GitHub repository with Actions enabled**
Your repository needs to have GitHub Actions enabled. If you can see the **Actions** tab in your repo, you're good to go.

**A `SONAR_TOKEN` configured as a GitHub secret**
This token authenticates your GitHub Actions workflow with SonarQube Cloud. How you generate it depends on your plan:

- **Free plan** — Generate a Personal Access Token (PAT) by going to **My Account** > **Security** in SonarQube Cloud. Copy the token value immediately — you won't be able to retrieve it again after leaving the page.
- **Team plan** — It is recommended to use a Scoped Organization Token (SOT) instead of a PAT. You can generate one from your organization's administration settings.

Once you have the token, add it to your GitHub repository under **Settings** > **Secrets and variables** > **Actions** as a new secret named `SONAR_TOKEN`.

**Runner requirements**
If you're using GitHub-hosted runners, no additional setup is needed — all required utilities come pre-installed. If you're using self-hosted runners, make sure `unzip` and either `wget` or `curl` are installed and available in your `PATH`.

---

## Setting up the workflow file

With your `SONAR_TOKEN` secret in place, you're ready to configure the GitHub Actions workflow. This section walks through two files you'll need: the workflow YAML and the `sonar-project.properties` configuration file.

### The `sonar-project.properties` file

Create a `sonar-project.properties` file in the root of your repository. This file tells the SonarQube Scan Action how to identify and connect your project to SonarQube Cloud:

```properties
sonar.projectKey=your_project_key
sonar.organization=your_organization_key
```

You'll find both values in SonarQube Cloud under **Your Project** > **Project Information**. The in-product tutorial will also display the exact values pre-populated for your account.

> **Note:** `sonar.token` does not belong in this file. It is passed securely through the GitHub secret you configured earlier — never hardcode your token in a configuration file.

### The workflow YAML file

Create a `.github/workflows/build.yml` file in your repository with the following:

```yaml
name: SonarQube Cloud Analysis

on:
  push:
    branches:
      - main
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  sonarcloud:
    name: SonarQube Cloud
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: SonarQube Cloud Scan
        uses: SonarSource/sonarqube-scan-action@v4
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

A few things worth understanding in this configuration:

**`fetch-depth: 0`** — This is important. By default, GitHub Actions performs a shallow clone of your repository. SonarQube Cloud needs the full git history to accurately calculate metrics like new code and blame information. Setting `fetch-depth: 0` ensures a full clone is performed.

**`on.push.branches`** — Triggers the analysis whenever code is pushed to your `main` branch. You can add additional branches here if your workflow requires it.

**`on.pull_request`** — Triggers the analysis on pull request activity, including when a PR is opened, updated, or reopened. This is what enables PR decoration — inline feedback posted directly in your GitHub pull request.

**`SONAR_TOKEN`** — Passed as an environment variable from your GitHub secret. The action uses this to authenticate with SonarQube Cloud without exposing the token value anywhere in your codebase.

---

## Enforcing quality gates in your pipeline

With your workflow running, SonarQube Cloud will analyze your code on every push and pull request. But by default, the analysis runs and reports results without actually stopping your pipeline if issues are found. That's where quality gates come in.

### What is a quality gate?

A quality gate is a set of conditions applied to your analysis results that determines whether your code meets the minimum quality required to move forward. After each analysis, SonarQube Cloud evaluates the results against those conditions and returns one of two statuses: **Passed** or **Failed**.

Quality gates answer two specific questions depending on context:
- For your main branch: *"Can I release my code today?"*
- For a pull request: *"Can I merge this pull request?"*

Every new project in SonarQube Cloud is assigned the built-in **Sonar way** quality gate by default. The Sonar way is a well-balanced set of conditions designed to work for most projects out of the box — you don't need to customize it to get real value from it right away.

### Making the workflow fail on a quality gate failure

By default, the SonarQube Scan Action does not fail the GitHub Actions workflow even if the quality gate returns a **Failed** status. To enforce the quality gate — meaning the workflow stops and blocks a merge when code doesn't pass — add the `sonar.qualitygate.wait` parameter to your `sonar-project.properties` file:

```properties
sonar.projectKey=your_project_key
sonar.organization=your_organization_key
sonar.qualitygate.wait=true
```

With `sonar.qualitygate.wait=true`, the scanner will wait for SonarQube Cloud to process the analysis results and return the quality gate status. If the gate fails, the workflow exits with a non-zero status, which GitHub Actions treats as a failed step — blocking any branch protection rules you have configured.

You can optionally pair this with `sonar.qualitygate.timeout` to control how many seconds the scanner waits for the result before timing out. The default is 300 seconds.

```properties
sonar.qualitygate.wait=true
sonar.qualitygate.timeout=300
```

### A practical recommendation on where to enforce the gate

While it may be tempting to enforce the quality gate across all branches, this is generally not recommended for active development teams. Failing a developer's feature branch mid-work can interrupt their flow and create unnecessary friction before the code is even ready for review.

A better approach is to limit `sonar.qualitygate.wait=true` to your `main` branch workflow trigger only. You can do this by using a conditional step in your workflow:

```yaml
- name: SonarQube Cloud Scan
  uses: SonarSource/sonarqube-scan-action@v4
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_SCANNER_OPTS: >
      ${{ github.ref == 'refs/heads/main' && '-Dsonar.qualitygate.wait=true' || '' }}
```

This way, analysis still runs on all branches and pull requests — giving developers visibility into issues early — but only the `main` branch push will block the pipeline on a quality gate failure.

> **Note:** The quality gate is computed starting from your second analysis. If you run the workflow for the first time and see a **Not Computed** status in SonarQube Cloud, this is expected — run it once more and the gate will be evaluated normally.

---

## Pull request decoration and branch analysis

One of the most practical benefits of CI-based analysis is what happens automatically once your workflow is running — SonarQube Cloud decorates your pull requests with a quality summary directly inside GitHub, without any additional configuration on your part.

### What PR decoration looks like

When a pull request triggers the analysis workflow, SonarQube Cloud posts a summary comment on the PR in GitHub. This comment includes the quality gate status — Passed or Failed — along with a breakdown of any new issues introduced by that pull request. Depending on your setup, issues may also appear as inline annotations on the specific lines of code where they were detected.

This means developers get code quality feedback exactly where they're already working, without needing to navigate to the SonarQube Cloud interface to understand what needs to be fixed.

It's worth noting that PR analysis focuses only on issues introduced by the pull request itself — not pre-existing issues in the target branch. This keeps the feedback relevant and actionable for the developer making the change.

### A note on plan availability

Full pull request analysis — meaning analysis triggered on every push to a PR branch — is available on the **Team plan**. On the **Free plan**, pull request analysis runs only when the PR is merged into the main branch. If your team relies on catching issues before merge, the Team plan is where that workflow becomes fully operational.

### Branch analysis

Because your workflow is configured with `on.push.branches` and `on.pull_request`, SonarQube Cloud also tracks each branch separately. You can review the quality gate status and issues for any analyzed branch directly in SonarQube Cloud under **Your Project** > **Branches and Pull Requests**. This gives you a clear picture of code quality across your entire development workflow, not just on main.

---

## Common gotchas

Even with a straightforward setup, there are a few mistakes that come up repeatedly when teams first integrate SonarQube Cloud with GitHub Actions. Here's what to watch out for.

### Using the old `sonarcloud-github-action`

If you find older tutorials or Stack Overflow answers referencing `sonarcloud-github-action`, that is the legacy action and should no longer be used. It was Docker-based, only supported Linux runners, and is no longer maintained. The current action is `sonarqube-scan-action`, which is what this guide uses throughout. If you're migrating from an older setup, update your workflow to use `SonarSource/sonarqube-scan-action` and remove any Docker-related configuration.

### Forgetting `fetch-depth: 0`

As mentioned in the workflow setup section, GitHub Actions performs a shallow clone by default. If you omit `fetch-depth: 0` from your checkout step, SonarQube Cloud won't have access to the full git history it needs to accurately calculate new code metrics and blame information. The analysis will still run, but the results — particularly around new code detection — may be unreliable. Always include it.

```yaml
- name: Checkout code
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

### Missing utilities on self-hosted runners

If you're using self-hosted runners and the action fails immediately, check that `unzip` and either `wget` or `curl` are installed and available in your runner's `PATH`. GitHub-hosted runners include these by default, but self-hosted runners often don't — and the error message isn't always obvious about what's missing.

### The `v6` argument parsing change

If you're upgrading from an older version of `sonarqube-scan-action` to v6 or later, be aware that how arguments passed via the `args` input are parsed changed in v6. If your workflow passes scanner arguments this way and starts behaving unexpectedly after an upgrade, review the quoting and formatting of your `args` input against the v6 release notes.

### `SONAR_TOKEN` not being picked up

If your workflow runs but authentication fails, double-check two things: first, that the secret in GitHub is named exactly `SONAR_TOKEN` — casing matters. Second, that the secret is defined at the repository level under **Settings** > **Secrets and variables** > **Actions**, not at the environment level unless your workflow explicitly references that environment.

---

## Wrapping up

You now have a fully functional GitHub Actions workflow that integrates SonarQube Cloud into your development pipeline. Every push to your main branch and every pull request will trigger an automated analysis, your quality gate will block code that doesn't meet your standards, and your developers will see inline feedback directly in GitHub without ever leaving their workflow.

To recap what we covered:

- The difference between SonarQube Automatic Analysis and CI-based analysis, and why CI-based analysis gives you real control
- Setting up your `SONAR_TOKEN` secret and configuring the `sonar-project.properties` file
- Building a GitHub Actions workflow using the `sonarqube-scan-action`
- Enforcing quality gates selectively — blocking on `main` without disrupting active development branches
- Understanding how PR decoration works and what your plan tier includes

This is a solid foundation, but there's more you can build on top of it. A few natural next steps:

- **Customize your quality gate** — the built-in Sonar way works well for most projects, but as your team matures you may want to tighten or adjust conditions to match your specific standards. See [Managing quality gates](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-quality-gates) in the SonarQube Cloud docs.
- **Add code coverage** — SonarQube Cloud can display test coverage data alongside your analysis results, giving you a more complete picture of code health. See [Test coverage](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/test-coverage) for setup instructions.
- **Explore Connected Mode** — by linking SonarQube Cloud with SonarQube for IDE, developers can see the same quality rules and new code definitions from the server applied locally in their editor, catching issues even before they push. See [SonarQube for IDE](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/connected-mode).
