[English](./TESTING_STRATEGY.md) | [Español](./TESTING_STRATEGY.es.md) | [Русский](./TESTING_STRATEGY.ru.md)
***
# Official Testing Strategy & Verification Plan
## 1. Goals and Scope
**Goal:** To verify that the implemented automated versioning system and branch protection rules work according to the documented processes, correctly handle standard scenarios, and block erroneous actions.
**Scope of Testing:**
*   Configuration of branch protection via GitHub Rulesets.
*   Functionality of **local** linting for commits using Husky.
*   The end-to-end, two-phase workflow of `release-please.yml`.
*   Correctness of version generation, `CHANGELOG.md` updates, and release creation for a monorepo structure.
*   Advanced scenarios, including rollbacks and commit-to-file path validation.
## 2. Roles

*   **"Developer":** Performs actions in feature branches, creates PRs to `dev`.
*   **"Release Manager":** Performs PR merges, including promotions between `dev`, `stg`, and `main`.
## 3. Test Execution Protocol
### Part 0: Test Environment Prerequisites (Mandatory)

This initial setup must be completed before running any test cases to ensure the local environment is correctly configured.

| ID | Action | Commands to Execute | Expected Result |
| :--- | :--- | :--- | :--- |
| **0.1** | Install Dependencies & Git Hooks | In the root of a freshly cloned repository, run:<br>`npm install` | 1. The command completes successfully.<br>2. The `node_modules` and `.husky` directories are created in the project root. |

---
### Part A: Local and Server-Side Protection Verification
**Goal:** To ensure that local hooks and server-side Rulesets work in tandem to correctly block and permit actions.

| ID | Scenario | Actions | Expected Result |
| :--- | :--- | :--- | :--- |
| **A-1** | **Local** Commit Format Validation | **1. Negative Test (no scope):**<br> `git commit --allow-empty -m "test: this must fail"` <br><br> **2. Positive Test (with valid scope):**<br> `git commit --allow-empty -m "test(project): this must pass"` | 1. The `git commit` command **FAILS**. The console shows an error from `commitlint` (`scope may not be empty`). The commit is not created.<br><br> 2. The `git commit` command **PASSES** successfully. |
| **A-2** | Direct Push to `dev` is Blocked | `git checkout dev && git pull && git commit --allow-empty -m "test(dev): push rejected" && git push` | The `git push` command **FAILS** with an error from the server (`(protected branch hook declined)`). |
| **A-3** | Verification of Flexible Merge Methods on `dev` | Open a valid PR to `dev`. Open the merge menu. | In the merge menu, **ALL** options are **AVAILABLE**: "Create a merge commit", "Squash and merge", "Rebase and merge". |
| **A-4** | Enforce "Merge Commit" Strategy on `stg`/`main` | Open a PR from `dev` to `stg`. Open the merge menu. | The merge menu **ONLY** shows the "Create a merge commit" option. |
| **A-5** | Block Outdated Branch on **all** target branches | 1. Open a PR to `stg` (or `dev`).<br>2. Before merging, merge another change into the same target branch (`stg` or `dev`). | The first PR is now **BLOCKED**, and the "Update branch" button appears. |

---
### Part B: `release-please` Core Logic Verification
**Goal:** To verify the end-to-end, two-phase release automation cycle.

| ID | Scenario | Actions | Expected Result |
| :--- | :--- | :--- | :--- |
| **B-1** | Release PR Creation | Promote a `fix(payment): ...` commit to `main`. Ensure the commit modifies a file **inside** the `services/payment/` directory. | A **new Pull Request** titled `chore(release): prepare new release` is automatically created. It contains updates to `CHANGELOG.md`, `.release-please-manifest.json`, and the component's `version.txt`. |
| **B-2** | Release PR Auto-Update | 1. Do not merge the PR from B-1.<br>2. Promote another `feat(activity): ...` commit to `main`. | **No new PR is created.** The existing Release PR is automatically updated with a new commit from the bot, now including changes for the `activity` component. |
| **B-3** | Release Finalization | Merge the updated Release PR from B-2 into `main`. | 1. The `release-please` workflow runs. The `prepare-release-pr` job is **SKIPPED**. <br>2. The `finalize-release` job **RUNS and succeeds**. <br>3. New Git tags (e.g., `payment-vX.Y.Z`) and corresponding GitHub Releases are created. |
| **B-4** | "Noise-Cancelling" & Breaking Change | Promote a `feat(project)!: ...` commit to `main`, using `chore(release): ...` for the promotion merge messages. | 1. In the new Release PR, the `CHANGELOG.md` **does NOT contain** any `chore(release)` entries. <br> 2. The version for the `project` component is bumped by a **MAJOR** version (e.g., `1.x.x` -> `2.0.0`). |
| **B-5** | Correct `revert` Handling | 1. Promote a `feat(activity): ...` commit to `main`.<br>2. Immediately after, promote a `revert(activity): ...` commit to `main`. | In the resulting Release PR: <br>1. The version for the `activity` component **has NOT changed**. <br>2. The `CHANGELOG.md` contains entries for both the `feat` under "Features" and the `revert` under a new "Reverts" section. |

---
### Part C: Commit-to-File Path Validation Logic
**Goal:** To prove that the system is protected against "phantom" releases, where the commit `scope` does not match the modified files.

| ID | Scenario | Actions | Expected Result |
| :--- | :--- | :--- | :--- |
| **C-1** | "Phantom" commit for one component triggers a release for another | Promote a commit to `main` with the message `fix(payment): phantom commit`, where the code change is made to a file **outside** the `services/payment/` directory (e.g., in the project root). | 1. A Release PR **is created/updated**.<br>2. In this PR, the version is bumped only for the `project` component (matching the `.` path).<br>3. The version for the `payment` component is **NOT CHANGED**. `release-please` has correctly ignored the commit's `scope`. |