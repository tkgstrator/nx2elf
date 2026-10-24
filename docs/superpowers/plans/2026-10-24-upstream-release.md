# Upstream Sync and Binary Release Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Synchronize the fork with upstream and publish five verified native binary archives plus a checksum manifest from one least-privilege GitHub Release job.

**Architecture:** Start from upstream `master`, keep CMake as the packaging source of truth, and make CI supply explicit architecture-qualified package names. A five-entry matrix produces immutable workflow artifacts; a dependent tag-only job publishes those exact files together. Preserve the former fork branch by renaming it before promoting the tested synchronization branch.

**Tech Stack:** C++17, CMake/CPack, GitHub Actions, GitHub Releases

**Spec:** `docs/superpowers/specs/2026-10-24-upstream-release-design.md`

## Global Constraints

- Preserve fork commit `911dc71dcbfc25ad4e5c76fb3de7e8c62ccfbabc` at `legacy-master-2026-10-24`.
- Base the new history on upstream commit `a1760ba54e2aefb41444933f53b90e6e8b05c399`.
- Never force push or delete the existing tag, release, feature branch, or old history.
- Release exactly `nx2elf-Linux-x86_64.zip`, `nx2elf-Linux-arm64.zip`, `nx2elf-macOS-x86_64.zip`, `nx2elf-macOS-arm64.zip`, and `nx2elf-Windows-x86_64.zip`.
- Pin `actions/checkout` to `d23441a48e516b6c34aea4fa41551a30e30af803`, `actions/upload-artifact` to `cf430e030ddbb5b0abf93d22962f4752f3646cd9`, `actions/download-artifact` to `9000827ccba6bdab643e8b6fd33ac0654aef8333`, and `softprops/action-gh-release` to `efb35369e0ad2afab669f228072c1b0d510eae64`.
- Default to `contents: read`; only the tag-only release job may use `contents: write`.
- Keep all new commit identities equal to the freshly authenticated human GitHub account and use short English Conventional Commit messages.
- Do not add Nintendo binaries, generated ELFs, secrets, or unlicensed fixtures.
- Delegated models must be `gpt-6-sol` or lower.

## Review Focus

- A matrix package name that does not match the actual CPack output must fail before artifact upload; Task 2 checks for the exact path.
- A tag build with one failed platform must never create or update a release; Task 2 makes `release` depend on the entire matrix.
- Pull requests must never receive write permission; Task 2 checks job-level permissions and tag gating.
- Two architectures on the same OS must never overwrite each other; Tasks 1 and 2 assert distinct package names.
- The published tag, release, and five assets must all refer to the tested `master` commit; Task 4 verifies every remote object and asset.

---

### Task 1: Make CPack accept an architecture-qualified package name

**Files:**
- Modify: `CMakeLists.txt`

**Interfaces:**
- Consumes: `CPACK_PACKAGE_FILE_NAME` supplied as a CMake cache variable.
- Produces: one ZIP whose base name is the supplied value, or the existing `nx2elf-${CMAKE_SYSTEM_NAME}` default when no override is supplied.

- [ ] **Step 1: Record the current failure**

Run a static assertion that fails because `CMakeLists.txt` unconditionally overwrites `CPACK_PACKAGE_FILE_NAME`.

Expected: nonzero exit and a message identifying the unconditional assignment.

- [ ] **Step 2: Make the default conditional**

Wrap the existing assignment in `if(NOT DEFINED CPACK_PACKAGE_FILE_NAME)` so command-line `-DCPACK_PACKAGE_FILE_NAME=...` values survive configuration.

- [ ] **Step 3: Verify the packaging contract statically**

Run a script that asserts the guard, default value, ZIP generator, install target, and `include(CPack)` remain present.

Expected: exit 0.

- [ ] **Step 4: Commit**

Commit message: `build: support architecture-specific packages`

### Task 2: Build and publish five least-privilege release artifacts

**Files:**
- Modify: `.github/workflows/build.yml`

**Interfaces:**
- Consumes: Task 1's `CPACK_PACKAGE_FILE_NAME` override and the four pinned action revisions in Global Constraints.
- Produces: five workflow artifacts on branch/PR/tag/manual builds and one release containing those same five ZIPs plus `SHA256SUMS` whenever the workflow ref is a `v*` tag.

- [ ] **Step 1: Record the current workflow gaps**

Run static assertions that fail because the upstream workflow has only three OS entries, no architecture-qualified packages, no separate release job, and no explicit least-privilege boundary.

Expected: nonzero exit naming the missing contract.

- [ ] **Step 2: Define the five-entry build matrix**

Use `ubuntu-24.04`, `ubuntu-24.04-arm`, `macos-15-intel`, `macos-15`, and `windows-2025`, each with its exact package base name from Global Constraints. Keep `workflow_dispatch` available so an immutable release tag created through the REST API can be built explicitly when necessary.

- [ ] **Step 3: Configure, build, package, and check exact output**

Pass the matrix package name to CMake, build Release, run CPack, and fail unless `build/<package>.zip` exists and is nonempty.

- [ ] **Step 4: Upload each ZIP as a distinct workflow artifact**

Use the pinned upload action, the package base name as the artifact name, a seven-day retention period, and `if-no-files-found: error`.

- [ ] **Step 5: Add the single tag-only release job**

Require the complete build matrix, give this job only `contents: write`, download all `nx2elf-*` artifacts into `dist`, verify the exact five-file set, generate `SHA256SUMS`, then use the pinned release action with `fail_on_unmatched_files: true` and generated release notes.

- [ ] **Step 6: Verify workflow structure**

Run YAML parsing, action-pin checks, matrix/package assertions, permission assertions, dependency assertions, and `actionlint` when available.

Expected: all checks exit 0.

- [ ] **Step 7: Commit**

Commit message: `ci: publish cross-platform release binaries`

### Task 3: Document upstream capabilities and release assets

**Files:**
- Modify: `README.md`
- Create: `docs/superpowers/specs/2026-10-24-upstream-release-design.md`
- Create: `docs/superpowers/plans/2026-10-24-upstream-release.md`

**Interfaces:**
- Consumes: Task 2's exact five asset names and tag behavior.
- Produces: contributor and user documentation matching the implemented workflow.

- [ ] **Step 1: Record the stale documentation**

Run assertions that fail while the README claims 32-bit files are unsupported and omits the five release assets.

Expected: nonzero exit identifying both gaps.

- [ ] **Step 2: Update the README**

Document release downloads, all five exact asset names, SHA-256 verification commands, CMake source builds, `v*` tag publication, and current 32-bit/64-bit NSO support. Preserve NSO/NRO/MOD usage and genuine known limitations.

- [ ] **Step 3: Verify documentation against the workflow**

Assert that every workflow package name appears exactly in the README, the obsolete limitation is absent, and the build commands match CMake.

Expected: exit 0.

- [ ] **Step 4: Commit**

Commit message: `docs: document release binaries`

### Task 4: Validate, promote, and publish the synchronized branch

**Files:**
- No source file changes expected.

**Interfaces:**
- Consumes: the three tested implementation commits, fork administration permission, and the approved branch names.
- Produces: default `master` at the tested commit, preserved legacy history, tag `v0.2.0`, and a verified public release with five ZIP assets plus `SHA256SUMS`.

- [ ] **Step 1: Create and push the remote synchronization branch**

Create `upstream-sync-2026-10-24` at the upstream base, attach the three guarded commits, and push it without force.

Expected: remote branch target equals the local tested HEAD.

- [ ] **Step 2: Wait for integration evidence**

Require all five build jobs on `upstream-sync-2026-10-24` to complete successfully and inspect each package step and artifact record.

- [ ] **Step 3: Perform the reversible branch cutover**

Rename old `master` to `legacy-master-2026-10-24`. Poll until the API reports that branch at `911dc71dcbfc25ad4e5c76fb3de7e8c62ccfbabc`, the old branch name is available, and the legacy branch is the temporary default. Rename the validated synchronization branch to `master`, poll until it resolves to the tested HEAD, then set `master` as the default branch.

Expected: the legacy branch points to `911dc71dcbfc25ad4e5c76fb3de7e8c62ccfbabc`; default `master` points to the tested HEAD.

- [ ] **Step 4: Verify the new default branch**

Manually dispatch `.github/workflows/build.yml` on `master`, require its complete five-target run to succeed, confirm `master` is not behind upstream, and verify the local tree is clean.

- [ ] **Step 5: Publish `v0.2.0`**

Create the lightweight tag ref through the GitHub REST API at the verified `master` SHA without moving any existing tag. Wait for a matching tag-ref run; if ref creation does not emit the `push` workflow, manually dispatch `.github/workflows/build.yml` at `v0.2.0` exactly once. Require its matrix and release job to succeed, and never dispatch a duplicate while a matching run exists or has an uncertain outcome.

- [ ] **Step 6: Verify the public release**

Confirm the release is public and non-draft, the tag target equals `master`, all five expected ZIPs and `SHA256SUMS` exist once with nonzero sizes, each archive contains the platform executable, and every downloaded ZIP passes the published checksum manifest.

- [ ] **Step 7: Final repository audit**

Verify default branch, branches, tags, releases, workflow state, commit identities/messages, no unexpected repository changes, and no uncommitted local files.

Expected: every acceptance criterion from the design is satisfied.
