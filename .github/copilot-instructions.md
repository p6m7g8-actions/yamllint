# Copilot Coding Agent Onboarding

This repository defines a **composite GitHub Action** that **builds and tests `p6df-` plugins**. The Action installs prerequisites, resolves dependent modules from `p6common`, and runs the project’s test entrypoint.

---

## 1) High‑Level Details

- **Type:** GitHub Action (composite).  
- **Primary entry point:** `action.yml`.  
- **Languages:** YAML + Bash/Zsh.  
- **Target runtime:** `ubuntu-latest` GitHub-hosted runners.  
- **What it does:**  
  1. Installs `zsh` and `time`.  
  2. Checks out `p6m7g8-dotfiles/p6common` into `./p6common`.  
  3. Resolves module dependencies for the current `p6df-<module>` repo.  
  4. Runs `p6common/bin/p6ctl test` under `zsh -e`.

**Repo size:** Small; action + workflows; no app build artifacts.

---

## 2) Build & Validation Instructions

> These steps reflect the exact sequence defined in `action.yml`. They are safe defaults for local repos and CI.

### Bootstrap
Always ensure a clean Ubuntu shell with `apt` available. Then:
```bash
sudo apt-get update && sudo apt-get install -y zsh time
```
Rationale: the Action installs these packages before executing tests.

### Build
There is no compile step. The Action is interpreted YAML + shell. If you add scripts, keep them POSIX-compatible and `zsh`-aware.

### Test (local reproduction)
Mimic the Action locally:
```bash
# From a checkout of a p6df-<module> repository:
git clone https://github.com/p6m7g8-dotfiles/p6common ./p6common

# Resolve module dependencies (same call path the Action uses)
zsh -c '. ./p6common/init.zsh; p6df::modules::p6common::gha::ModuleDeps "$(basename "$GITHUB_REPOSITORY" | cut -d- -f2)"'

# Run tests exactly as CI
P6_DFZ_SRC_P6M7G8_DOTFILES_DIR=. TERM=xterm zsh -e p6common/bin/p6ctl test
```
**Preconditions:** Internet access to fetch `p6common`; `zsh` installed.  
**Postconditions:** Exit code 0 indicates success; non‑zero means test failure.

### Run
The Action is invoked by consumer workflows with:
```yaml
- uses: p6m7g8-actions/p6df-build@<ref>
```
No user inputs are required by this Action.

### Lint
No linters are configured in this repo. Keep YAML valid (1.2) and shell scripts `shellcheck`-clean if added.

### Known pitfalls and mitigations
- **Missing `zsh`:** Always install via `apt` before running tests.  
- **Dependency resolution fails:** Ensure `p6common` is present at `./p6common` and that `. ./p6common/init.zsh` succeeds.  
- **Non‑interactive assumptions:** CI sets `TERM=xterm`. Preserve this when running locally.

---

## 3) Project Layout & CI Gates

### Key files
```
action.yml                   # Composite Action: install zsh, fetch p6common, resolve deps, run tests
.github/workflows/
  build.yml                  # "Build / build": required CI no-op check (PR + merge_group + manual)
  auto-queue.yml             # Enqueue PR to Merge Queue after successful Build
  auto-approve.yml           # Auto-approve for trusted authors/labels
  pr-labeler.yml             # Auto-label 'contribution/core' and then 'auto-merge'
  pull-request-lint.yml      # Enforce Conventional Commit PR titles
.vscode/settings.json        # Editor/YAML defaults
LICENSE
README.md
```

### What runs before merge
- **Build / build**: quick success check; must pass for Merge Queue.  
- **PR title lint**: requires one of the allowed types (e.g., `chore|ci|docs|feat|fix|major|perf|refactor|revert|style|test`).  
- **Labeler & Auto-approve**: labels and approvals applied based on author/labels.  
- **Auto-queue**: upon successful **Build**, adds the PR to the Merge Queue.

To replicate these checks:
- Keep the workflow **name** `Build` and job id `build`.  
- Ensure `build.yml` triggers include both `pull_request` and `merge_group`.  
- Enable Merge Queue and mark `Build / build (pull_request)` as a **required status check** on the protected branch.

### Action internals (source of truth)
The composite Action executes, in order:
1. `actions/checkout@v5` (repo code)  
2. `apt-get install time zsh`  
3. Second checkout: `p6m7g8-dotfiles/p6common` into `./p6common`  
4. Dependency resolution:
   ```bash
   zsh -c ". ./p6common/init.zsh; p6df::modules::p6common::gha::ModuleDeps $mod"
   ```
   where `mod` is derived from `GITHUB_REPOSITORY` name (`p6df-<module>`).  
5. Test execution:
   ```bash
   P6_DFZ_SRC_P6M7G8_DOTFILES_DIR=. TERM=xterm zsh -e p6common/bin/p6ctl test
   ```

### Hidden dependencies
- Relies on **functions sourced from `p6common`** (`p6df::modules::p6common::gha::ModuleDeps`).  
- Assumes a `p6df-<module>` naming pattern to compute `mod`.  
- Requires network access to fetch `p6common` repository at runtime.

---

## 4) Agent Rules of Engagement

- **Trust this document first.** Search only if these instructions are incomplete or demonstrably incorrect.  
- Preserve workflow names, triggers, and ordering that downstream jobs depend on.  
- Keep steps idempotent and fast; avoid adding long-lived processes.  
- Prefer `bash` for inline steps but ensure compatibility with `zsh` where the tests run.  
- When editing `action.yml`, maintain the install → checkout → deps → test sequence.
