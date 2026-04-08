# How to Add SonarQube Cloud Analysis to Your GitLab CI Pipeline

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

A CI pipeline that builds and tests your code but never checks its quality is only doing half its job. Code quality issues tend to compound quietly over time, and by the time they're visible in production they're significantly more expensive to fix than they would have been at the pull request stage.

SonarQube Cloud brings code quality enforcement directly into your pipeline. Every push gets analyzed, the results surface where developers are already working, and you can optionally fail the pipeline when code doesn't meet your quality standards. This guide shows you how to set that up in GitLab CI from scratch — creating a `.gitlab-ci.yml` that runs the SonarScanner and gets your first analysis result into SonarQube Cloud.

By the end you'll have a working analysis pipeline ready to add to any GitLab repository in your organization.

---

## Prerequisites

Before starting, make sure you have the following in place:

**SonarQube Cloud organization** — You should already have a SonarQube Cloud organization created and linked to your GitLab group. If not, create one at [sonarcloud.io](https://sonarcloud.io) before proceeding.

**SonarQube Cloud project** — Your project should already be created in SonarQube Cloud and bound to the GitLab repository you'll be working with. When setting up the project, select **GitLab** as the DevOps platform.

**Project token** — Generate an analysis token in SonarQube Cloud: in your organization go to **Scoped Organization Tokens**. Create a token — you can choose to scope it to all your projects or just a selected group. You'll add this to GitLab as a CI/CD variable shortly.

**GitLab repository** — A GitLab repository you have at least Maintainer access to. No existing CI pipeline is required — we'll create one from scratch.

---

## Section 1: GitLab CI Fundamentals

GitLab CI works similarly to other CI platforms, with one key difference: there are no pre-built actions or tasks to drop into your pipeline. Instead, jobs run shell scripts inside Docker containers, which makes the pipeline explicit and transparent — you see exactly what runs, with no abstraction layer in between. This section covers the core concepts you need before we build the pipeline.

### The `.gitlab-ci.yml` File

GitLab CI uses a single `.gitlab-ci.yml` file at the root of your repository. This one file defines your entire pipeline — all stages, all jobs, all variables. GitLab picks it up automatically on every push.

### Stages and Jobs

A GitLab CI pipeline is organized into **stages**. Stages run sequentially — the next stage only starts when every job in the current stage has passed. Within a stage, jobs run in parallel.

You define your stages at the top of the file, then assign each job to a stage:

```yaml
stages:
  - test
  - build
  - deploy

unit-tests:
  stage: test
  script:
    - npm test

build-app:
  stage: build
  script:
    - npm run build
```

### Docker Images

GitLab CI jobs run inside Docker containers by default on GitLab-hosted runners. You specify the image at the global level or per job using the `image` keyword:

```yaml
image: node:20  # default for all jobs

sonarqube-analysis:
  image: sonarsource/sonar-scanner-cli:latest  # override for this job
  script:
    - sonar-scanner
```

### Variables

Variables in GitLab CI can be defined globally, per job, or set in the GitLab UI under **Settings → CI/CD → Variables** (for secrets and sensitive values). You reference them with the standard `$VARIABLE_NAME` syntax:

```yaml
variables:
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"  # global variable
  GIT_DEPTH: "0"

sonarqube-analysis:
  variables:
    SONAR_EXTRA_ARGS: "-Dsonar.verbose=true"  # job-level variable
  script:
    - sonar-scanner $SONAR_EXTRA_ARGS
```

GitLab also provides a set of **predefined variables** that are automatically injected into every job. A few you'll see in the SonarScanner configuration:

| Variable | What it contains |
|---|---|
| `CI_PROJECT_DIR` | Full path to the cloned repository |
| `CI_COMMIT_BRANCH` | Branch name (available in branch pipelines) |
| `CI_PIPELINE_SOURCE` | What triggered the pipeline (`push`, `merge_request_event`, etc.) |

SonarScanner automatically reads these GitLab predefined variables to determine the context of the analysis — branch name, pipeline trigger, and so on — you don't need to pass them explicitly.

### Runners

Runners are the agents that execute your jobs. GitLab.com provides shared runners you can use immediately without any setup, which is what we'll use in this guide. If your organization uses self-hosted runners, the pipeline configuration is identical — only the `tags` key in your job definition changes to target the right runner.

That's all the GitLab CI background you need. In the next section, we'll build the pipeline.

---

## Section 2: Building the Pipeline

This section walks through the three pieces you need to get SonarQube Cloud analysis running in GitLab CI: adding a CI/CD variable for your token, creating a `sonar-project.properties` file, and writing the `.gitlab-ci.yml`.

### Adding the Analysis Token as a CI/CD Variable

Your analysis token allows the SonarScanner to authenticate with SonarQube Cloud. It should never be committed to the repository — GitLab CI/CD variables are the right place to store it, as they are injected securely into pipeline jobs at runtime.

In your GitLab repository, go to **Settings → CI/CD → Variables** and click **Add variable**. Configure it as follows:

- **Key:** `SONAR_TOKEN`
- **Value:** the token you generated from Scoped Organization Tokens in SonarQube Cloud
- **Visibility:** Masked — this prevents the token from appearing in job logs
- **Protect variable:** enable this if you only want the token available on protected branches; leave it off if you want analysis to run on all branches

While you're here, add a second variable:

- **Key:** `SONAR_HOST_URL`
- **Value:** `https://sonarcloud.io`

This tells the scanner which SonarQube instance to send results to. Setting it as a CI/CD variable rather than hardcoding it in the pipeline file makes it easy to update across all jobs if needed.

<!-- SCREENSHOT PLACEHOLDER 4
     What to capture: GitLab repository Settings → CI/CD → Variables page, showing the
     SONAR_TOKEN variable added with the Masked flag enabled.
     Suggested caption: "Figure 4: Adding SONAR_TOKEN as a masked CI/CD variable in GitLab —
     Settings → CI/CD → Variables." -->

### Configuring the `sonar-project.properties` File

The `sonar-project.properties` file is a configuration file that the SonarScanner reads automatically when it runs. It tells the scanner which SonarQube Cloud project to send analysis results to, and lets you define project-level settings like which directories to scan and what to exclude. Keeping this configuration in a dedicated file rather than passing everything as command-line arguments keeps your `.gitlab-ci.yml` clean and lets you manage scanner settings independently of your pipeline definition.

Create a `sonar-project.properties` file at the root of your repository. This is where the scanner picks up your project identity:

```properties
sonar.projectKey=your_project_key
sonar.organization=your_organization_key
```

Both values are available in SonarQube Cloud under your project's **Information** page. The `projectKey` uniquely identifies your project within the organization — it's typically in the format `your-org_your-repo`.

You can optionally add properties like `sonar.projectName`, `sonar.sources`, and `sonar.exclusions` to fine-tune what gets analyzed — we'll cover those in the best practices section.

### Writing the `.gitlab-ci.yml`

Create a `.gitlab-ci.yml` file at the root of your repository with the following content:

```yaml
stages:
  - sonarqube-analysis

variables:
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  GIT_DEPTH: "0"

sonarqube-check:
  stage: sonarqube-analysis
  image: sonarsource/sonar-scanner-cli:latest
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  script:
    - sonar-scanner
  only:
    - main
    - merge_requests
```

Here's what each part does:

**`SONAR_USER_HOME`** — sets the scanner's cache directory to a path inside the project directory, which is where GitLab CI expects cached files to live.

**`GIT_DEPTH: "0"`** — disables GitLab's default shallow clone. The SonarScanner needs access to the full git history to correctly compute new code and blame data. Without this, analysis either fails or produces incomplete results.

**`image: sonarsource/sonar-scanner-cli:latest`** — uses the official Sonar Scanner CLI Docker image, which comes with the scanner pre-installed. No additional setup steps are needed.

**`cache`** — caches the scanner's working files between runs, which speeds up subsequent analyses significantly. The cache key is scoped to the job name so it doesn't conflict with other jobs.

**`script: sonar-scanner`** — runs the scanner. It picks up your `sonar-project.properties` automatically, and reads `SONAR_TOKEN` and `SONAR_HOST_URL` from the environment.

**`only`** — limits analysis to your main branch and merge requests. You can expand this list to include other long-lived branches if your branching strategy requires it.

### Running Your First Analysis

Commit both files to your repository:

```bash
git add sonar-project.properties .gitlab-ci.yml
git commit -m "Add SonarQube Cloud analysis pipeline"
git push origin main
```

GitLab will automatically detect the `.gitlab-ci.yml` and trigger the pipeline. Navigate to **CI/CD → Pipelines** in your repository to watch it run. Once the job completes, head to your SonarQube Cloud project — your first analysis results will be waiting there.

<!-- SCREENSHOT PLACEHOLDER 5
     What to capture: GitLab CI/CD Pipelines view showing the sonarqube-check job as passed,
     and/or the SonarQube Cloud project page showing the first analysis result.
     Suggested caption: "Figure 5: The sonarqube-check job passing in GitLab CI (left) and the
     resulting analysis in SonarQube Cloud (right)." -->

---

## Section 3: Quality Gate Enforcement

By default, the pipeline completes successfully even if the Quality Gate fails — the scanner sends results to SonarQube Cloud and exits cleanly regardless of the gate outcome. To block the pipeline when quality standards aren't met, enable the `sonar.qualitygate.wait` parameter, which tells the scanner to wait for the gate result before exiting.

### Enabling the Quality Gate Failure

Add the following property to your `sonar-project.properties` file:

```properties
sonar.qualitygate.wait=true
```

With this set, the SonarScanner will poll SonarQube Cloud after analysis completes until the Quality Gate result is available, then exit with a non-zero code if the gate fails. This causes the GitLab CI job to fail and blocks the pipeline from continuing to subsequent stages.

You can also control how long the scanner waits before giving up:

```properties
sonar.qualitygate.wait=true
sonar.qualitygate.timeout=300
```

`sonar.qualitygate.timeout` is in seconds and defaults to 300 (five minutes). On large codebases where analysis takes longer to process, you may need to increase this. If the timeout is reached before a result is returned, the job fails.

### Soft Enforcement with `allow_failure`

If you want the pipeline to surface quality gate failures without blocking downstream stages — useful when onboarding a brownfield project where the gate isn't passing yet — GitLab CI's `allow_failure` flag lets the job fail visibly without stopping the pipeline:

```yaml
sonarqube-check:
  stage: sonarqube-analysis
  image: sonarsource/sonar-scanner-cli:latest
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  script:
    - sonar-scanner
  allow_failure: true
  only:
    - main
    - merge_requests
```

With `allow_failure: true`, the job will show as a warning in the pipeline UI when the gate fails, but subsequent stages will still run. This is a practical way to get visibility into quality issues without immediately breaking the build — set a deadline with your team to reach a passing gate, then remove `allow_failure` once you're there.

<!-- SCREENSHOT PLACEHOLDER 6
     What to capture: GitLab CI/CD pipeline view showing the sonarqube-check job in a failed
     or warning state (allow_failure), with downstream stages still running.
     Suggested caption: "Figure 6: With allow_failure: true, a failing Quality Gate shows as a
     warning in the pipeline without blocking downstream stages." -->

---

## Section 4: Common Gotchas and Best Practices

### Common Gotchas

**`GIT_DEPTH: "0"` doesn't always fully unshallow the repository.** There is a known edge case in GitLab CI where setting `GIT_DEPTH: "0"` doesn't deliver a full git history if an earlier stage in the same pipeline already checked out a shallow clone. If you see warnings like "Missing blame information" or "Could not find ref" in your scanner output, add the following before the `sonar-scanner` call in your script:

```yaml
script:
  - git fetch --unshallow || true
  - sonar-scanner
```

The `|| true` prevents the step from failing on repositories that are already unshallowed.

**`sonar.qualitygate.wait=true` adds time to every pipeline run.** The scanner has to poll SonarQube Cloud until the gate result is ready, which can add anywhere from a few seconds to a few minutes depending on project size and server load. Factor this into your pipeline time budget, and increase `sonar.qualitygate.timeout` if you're seeing premature timeouts on larger codebases.

**A wrong `sonar.projectKey` creates a new project instead of updating the existing one.** If the project key in your `sonar-project.properties` doesn't exactly match the key in SonarQube Cloud, the scanner will silently create a new project rather than fail. Double-check the key against your project's **Information** page in SonarQube Cloud if results aren't appearing where you expect them.

**Analysis runs on all branches if you don't set `only` or `rules`.** Without branch filtering, the sonarqube-check job will trigger on every push to every branch, including short-lived feature branches. This isn't necessarily wrong, but it generates a lot of analysis noise and consumes runner minutes. The `only` block in the pipeline we built limits this to `main` and merge requests — adjust it to match your branching strategy.

### Best Practices

**Pin the scanner image version rather than using `:latest`.** The `sonarsource/sonar-scanner-cli:latest` image is convenient but means your pipeline can silently pick up a new scanner version on any run. For production pipelines, pin to a specific version tag (e.g., `sonarsource/sonar-scanner-cli:5`) to keep analysis reproducible and avoid unexpected behavior after scanner updates.

**Use `sonar.exclusions` to keep analysis focused.** Scanning generated files, dependency directories, and test fixtures adds noise to your results without improving the quality signal. A sensible starting point for most projects:

```properties
sonar.exclusions=**/node_modules/**,**/dist/**,**/build/**,**/*.test.*,**/coverage/**
```

Tune this list to match your project structure — the goal is to analyze the code your team writes, not the code your tools generate.

**Use `sonar.sources` to scope the scan to your source directory.** If your repository has a non-standard layout, explicitly set `sonar.sources` to point at the directory containing your application code rather than letting the scanner default to the repository root:

```properties
sonar.sources=src
```

**Store `SONAR_TOKEN` at the GitLab group level if you're rolling this out across multiple repositories.** Rather than adding the token as a project-level variable in every repository, define it once at the group level under **Group Settings → CI/CD → Variables**. All projects in the group inherit it automatically, and you only need to rotate it in one place.

**Set a team deadline for removing `allow_failure`.** If you introduce `allow_failure: true` to onboard a brownfield project, treat it as a temporary measure. Set a concrete date with your team to get the Quality Gate passing and remove the flag — otherwise it tends to stay in indefinitely and the gate loses its enforcement value.

---

## Conclusion

Adding SonarQube Cloud analysis to a GitLab CI pipeline is a straightforward process once the moving parts are clear: a token stored as a CI/CD variable, a `sonar-project.properties` file that identifies the project, and a minimal `.gitlab-ci.yml` job that runs the scanner. From there, Quality Gate enforcement is a single property away.

The pipeline you've built in this guide is a solid foundation. As your team's confidence grows, you can extend it — tightening the Quality Gate conditions, scoping exclusions more precisely, or rolling the same pattern out across every repository in your organization using group-level variables. The scanner configuration stays the same; only the project keys change.

Code quality tends to improve when it's measured consistently and automatically. Getting the pipeline in place is the first step — everything else follows from the data it produces.

---

## Reference Links

- [GitLab CI — SonarQube Cloud Docs](https://docs.sonarsource.com/sonarqube-cloud/advanced-setup/ci-based-analysis/gitlab-ci)
- [Getting started with GitLab — SonarQube Cloud Docs](https://docs.sonarsource.com/sonarqube-cloud/getting-started/gitlab)
- [CI/CD variables — GitLab Docs](https://docs.gitlab.com/ci/variables/)
- [CI/CD YAML syntax reference — GitLab Docs](https://docs.gitlab.com/ci/yaml/)
- [Predefined CI/CD variables — GitLab Docs](https://docs.gitlab.com/ci/variables/predefined_variables/)
- [sonar-scanner-cli Docker image — Docker Hub](https://hub.docker.com/r/sonarsource/sonar-scanner-cli)
 