# template-repo-base

A language-agnostic GitHub repository template. It gives a new repo a working
baseline on day one — pinned tools (mise), generic pre-commit hooks, PR/merge CI
workflows, Renovate dependency updates, and Conventional Commits enforcement —
with no application code, so you add your own source and layer
project-specific tooling on top.

## Getting started

1. Create a repo from this template (**Use this template** on GitHub, or `gh
   repo create --template`).
2. Once per machine, set up the tools with
   [jay-withers/toolchain](https://github.com/jay-withers/toolchain)
   (`bash src/setup.sh`), which installs [mise](https://mise.jdx.dev).
3. In the repo, install its pinned tools and the git hooks:

   ```bash
   mise trust && make install   # make install runs `mise install` first
   ```

`mise.toml` pins every tool this repo's hooks and CI use. CI installs the same
versions (jay-withers/workflows v1.26.0+), so add a tool there rather than
installing it by hand.

## Commands

Run `make` (or `make help`) to list the available targets:

```bash
make install           # install mise tools and pre-commit hooks (run once after cloning)
make lint              # run all pre-commit hooks against every file
```

## Pre-commit hooks

Hooks live in `.pre-commit-config.yaml`, pinned by commit SHA with the tag as a
frozen comment. They're deliberately language-agnostic:

- `pre-commit/pre-commit-hooks` — large-file, case-conflict, merge-conflict,
  symlink, YAML, end-of-file, trailing-whitespace, line-ending and shebang
  checks, `forbid-new-submodules`, plus `no-commit-to-branch` (blocks direct
  commits to `main`). The end-of-file check leaves NuGet's `packages.lock.json`
  alone: NuGet rewrites it without a final newline on every restore, and a
  .NET repo's locked restore check (workflows' `dotnet.yml`) would fail on the
  hook's fix
- `gitleaks` — secret scanning
- `actionlint` — GitHub Actions workflow linting
- `shellcheck` — shell scripts
- `hadolint` — Dockerfiles, through Docker (`hadolint-docker`), so nothing to
  install; it only runs once the repo has a Dockerfile
- `commitlint` — [Conventional Commits](https://www.conventionalcommits.org/),
  at the `commit-msg` stage

As your repo gains a language, add its formatter/linter hooks here — don't
remove these. Renovate keeps the frozen hook revisions up to date.

## Commit messages

Commits must follow [Conventional Commits](https://www.conventionalcommits.org/),
enforced by commitlint at commit-msg time. The commit type drives the automatic
version bump on merge (see below). Examples:

```text
feat: add health check endpoint
fix: correct retry backoff
chore: bump dependency
```

## CI/CD

Workflows are prefixed `ci-` (pull-request checks) or `cd-` (post-merge delivery):

- **`.github/workflows/ci-lint.yml`** — runs all pre-commit hooks on PRs
  to `main`, by calling the shared reusable workflow
  `jay-withers/workflows/.github/workflows/pre-commit.yml`. Its status
  check reports as `pre-commit / Pre-commit`.
- **`.github/workflows/ci-codeql.yml`** — CodeQL code scanning of the
  workflows, on PRs, on `main` and weekly. Findings land in Security → Code
  scanning; the job doesn't fail on them.
- **`.github/workflows/cd-tag.yml`** — on every merge to `main`, creates a
  semver tag and matching GitHub release from the Conventional Commits since the
  last release (default bump: patch), via
  `jay-withers/workflows/.github/workflows/release.yml`.

Each pins its reusable workflow by commit SHA with the tag as a comment. Add your
own `ci-*` workflows (build, test, etc.) as you add code, and require their
checks in branch protection (below).

## Renovate

`renovate.json` extends the shared preset `github>jay-withers/renovate` (see
[jay-withers/renovate](https://github.com/jay-withers/renovate) for the
policy: Monday schedule, auto-merge once checks pass, a three-day wait on new
releases, and the `pre-commit` and `mise` managers that keep hook revisions
and `mise.toml` current), plus `autoApprove: true`. Its Docker, GitHub
Actions, Terraform and npm rules switch on by themselves as the repo gains
those ecosystems. `platformAutomerge` needs repo-level auto-merge to be
enabled — see [Configuring GitHub](#configuring-github-for-a-repo-created-from-this-template).

## Configuring GitHub for a repo created from this template

Settings that can't be templated as files — repo-level auto-merge, delete
branch on merge, and a branch-protection ruleset requiring status checks and
approving reviews — aren't bootstrapped by anything in this template anymore.
A repo just created from this template gets GitHub's defaults until it's
added to `var.repos` in
[jay-withers/github-repos](https://github.com/jay-withers/github-repos)'
`terraform/terraform.tfvars` and applied — that Terraform root module is now
the single source of truth for these settings across every jay-withers repo,
this template included.

## Structure

```text
mise.toml              # pinned tool versions, used locally and in CI
.vscode/
  extensions.json      # recommended VS Code extensions
.github/
  workflows/
    ci-lint.yml        # lints all files on PRs to main (reusable workflow)
    ci-codeql.yml      # CodeQL over the workflows
    cd-tag.yml         # auto-tags + releases on merge to main (semver, conventional commits)
.editorconfig          # baseline editor settings (aligned with pre-commit hooks)
.gitattributes         # git-level LF normalization
.pre-commit-config.yaml
commitlint.config.js   # commitlint (Conventional Commits) config
renovate.json          # automated dependency updates
CLAUDE.md              # guidance for Claude Code
LICENSE
Makefile
```
