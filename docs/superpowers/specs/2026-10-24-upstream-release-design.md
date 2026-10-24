# Upstream Sync and Binary Release Design

**Status:** Approved on 2026-10-24

## Intent

Bring `tkgstrator/nx2elf` onto the current `shuffle2/nx2elf` history without losing the fork's existing history, then publish auditable native binaries through GitHub Releases.

## Current baseline

- The fork's `master` is `911dc71dcbfc25ad4e5c76fb3de7e8c62ccfbabc`.
- Upstream `master` is `a1760ba54e2aefb41444933f53b90e6e8b05c399`.
- The branches diverged at `11b4c20d9b96630202b46d8f349373e9752e755a`; the fork is 8 commits ahead and 16 behind.
- A trial merge conflicts in ten paths, including the core converter sources. Most fork changes have upstream equivalents; the remaining `Makefile` `-fpermissive` flag is not part of the CMake release build.
- Upstream already has CMake packaging and a three-OS workflow, but its release uploads run concurrently in matrix jobs and package names omit architecture.
- The fork's existing `v0.1` release and tag must remain untouched.

## Goals

- Preserve the old fork tip under `legacy-master-2026-10-24`.
- Make the new `master` descend directly from the current upstream `master`.
- Avoid a force push, history deletion, and manual conflict resolution that could combine stale fork code with current upstream code.
- Build and package five native targets:
  - Linux x86_64
  - Linux arm64
  - macOS x86_64
  - macOS arm64
  - Windows x86_64
- Publish architecture-qualified ZIP files and their checksum manifest from one release job only after every build succeeds.
- Create and verify a public `v0.2.0` release at the tested `master` commit.

## Non-goals

- Do not redesign converter behavior or import the obsolete Visual Studio project files removed upstream.
- Do not add copyrighted Nintendo binaries or conversion fixtures.
- Do not add or invent a license on behalf of upstream.
- Do not delete `feature/actions`, the legacy branch, the existing tag, or the existing release.

## Constraints

- The user approved the branch cutover and public `v0.2.0` publication and confirmed that the release may proceed on that basis.
- Preserve history by branch renames, not by force-updating or deleting the old `master` tip.
- Use only the user's freshly authenticated human GitHub identity for new commits.
- Use printable ASCII English Conventional Commit messages no longer than 128 bytes.
- Pin every third-party GitHub Action to a verified full commit SHA.
- Default workflow permissions are `contents: read`; only the release job receives `contents: write`.
- Use models no stronger than `gpt-6-sol` for delegated work and review.

## Approaches considered

### Preserve the old branch and cut over `master` (selected)

Create an upstream-based synchronization branch, validate it, rename the old `master` to `legacy-master-2026-10-24`, rename the validated branch to `master`, and restore it as the default branch. This preserves every old commit while giving the fork a clean upstream ancestry and low-friction future synchronization.

### Merge upstream into the divergent fork branch

This retains one connected branch history, but requires resolving ten conflicts across stale and current converter implementations. It also requires a merge commit outside the guarded commit path. The additional risk has no identified functional benefit.

### Copy the upstream tree into one ordinary fork commit

This avoids a merge commit, but GitHub would continue to report the fork as behind upstream and future synchronization would remain difficult. It does not meet the maintenance goal.

## Branch cutover

1. Create `upstream-sync-2026-10-24` in the fork at the verified upstream commit.
2. Add the packaging, workflow, and documentation commits on that branch.
3. Push the branch and require its complete five-target integration run to pass.
4. Rename the old default branch to `legacy-master-2026-10-24`, then poll until the rename is complete, its SHA is unchanged, and GitHub reports it as the temporary default branch.
5. Rename the validated synchronization branch to `master`, poll until its expected SHA resolves, and set it as the default branch.
6. Manually dispatch the workflow on the new `master`, then verify branch SHAs, default-branch state, fork comparison state, and the complete five-target run.

## Release architecture

Keep `.github/workflows/build.yml` as the single build and release workflow. It runs for pushes, pull requests, `v*` tags, and explicit manual dispatches. One matrix job configures, builds, packages, checks, and uploads a uniquely named ZIP per target. A separate release job runs only when the workflow ref is a `v*` tag, depends on the full matrix, downloads all ZIP artifacts, and publishes them together.

The release job must not build source again. This guarantees that the files attached to the release are the same artifacts accepted by the matrix. The expected assets are:

- `nx2elf-Linux-x86_64.zip`
- `nx2elf-Linux-arm64.zip`
- `nx2elf-macOS-x86_64.zip`
- `nx2elf-macOS-arm64.zip`
- `nx2elf-Windows-x86_64.zip`
- `SHA256SUMS`

`CMakeLists.txt` keeps its existing OS-only package name as the default for local users, while accepting an explicitly supplied `CPACK_PACKAGE_FILE_NAME` from CI.

## Documentation

Update the README to describe CMake builds, supported NSO/NRO/MOD inputs, the five release assets, checksum verification, and the tag-driven release process. Remove the stale claim that 32-bit files are unsupported because upstream added 32-bit NSO support.

## Verification

- Validate YAML syntax and action pins before pushing.
- Confirm the CMake package-name override is effective in every matrix job.
- Require all five jobs to configure, build, package, and find exactly one expected ZIP.
- Require artifact upload to fail when the expected ZIP is absent.
- Require the release job to verify exactly five nonempty ZIPs, generate `SHA256SUMS` from those accepted artifacts, and fail when the expected set does not match.
- After cutover, verify the remote default branch and exact `master` SHA, then manually dispatch and require a successful five-target workflow run on that ref.
- Create `v0.2.0`, wait for the tag workflow, and verify all five downloadable archives plus `SHA256SUMS`, nonzero sizes, archive contents, and checksums. If GitHub's REST ref creation does not emit the `push` workflow, manually dispatch the same workflow at the immutable `v0.2.0` ref exactly once.
- Compare the release tag target with the tested `master` commit.

