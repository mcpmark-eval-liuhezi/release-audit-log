# Release Audit: GitHub CLI v2.63.0

**Audited:** release `v2.63.0` of [`cli/cli`](https://github.com/cli/cli) (the GitHub CLI)
**Tracking issue:** #1
**Purpose:** document how the release was built and tested, with the audit trail living in GitHub.

---

## 1. Release notes — v2.63.0 ("GitHub CLI 2.63.0")

- **Release:** https://github.com/cli/cli/releases/tag/v2.63.0 (release ID 187910174)
- **Tag:** `v2.63.0` → target commit `1fe14c956d057e6abda50a76d8820a6e668ba70a`
- **Created / published:** 2024-11-27 20:57 UTC / 2024-11-27 21:10 UTC
- **Full changelog:** https://github.com/cli/cli/compare/v2.62.0...v2.63.0

### What shipped (7 merged PRs)

| Change | PR | Author |
|---|---|---|
| Support bare repo creation | [#9905](https://github.com/cli/cli/pull/9905) | @williammartin |
| Refactor the `getAttestations` functions | [#9892](https://github.com/cli/cli/pull/9892) | @malancas |
| Added a section on manual verification of the releases | [#9936](https://github.com/cli/cli/pull/9936) | @kommendorkapten |
| Adding option to return `baseRefOid` in `pr view` | [#9938](https://github.com/cli/cli/pull/9938) | @daliusd |
| Update verification results printing | [#9937](https://github.com/cli/cli/pull/9937) | @malancas |
| Fix some multiline command documentation to use `heredoc` strings | [#9948](https://github.com/cli/cli/pull/9948) | @BagToad |
| Print friendly error when `release create` fails due to missing `workflow` OAuth scope | [#9791](https://github.com/cli/cli/pull/9791) | @BagToad |

### Notable

- **Security:** a security vulnerability was identified in the GitHub CLI that could **leak authentication tokens when cloning repositories containing git submodules hosted outside of GitHub.com and ghe.com**. Details: [GHSA-jwcm-9g39-pmcw](https://github.com/cli/cli/security/advisories/GHSA-jwcm-9g39-pmcw).
- **New contributor:** @daliusd made their first contribution (#9938).
- **Release assets:** binaries for all supported platforms (e.g. `gh_2.63.0_linux_386.deb`, `gh_2.63.0_linux_386.rpm`, `gh_2.63.0_linux_386.tar.gz`, checksums file `gh_2.63.0_checksums.txt`, etc.), all uploaded by `github-actions[bot]` — i.e. the release artifacts themselves are produced and attached by CI automation.

---

## 2. CI — the main build-and-test workflow

**Workflow:** ["Unit and Integration Tests"](https://github.com/cli/cli/blob/trunk/.github/workflows/go.yml) — `.github/workflows/go.yml`, workflow ID `25016`.

**Triggers:** `push` to the default branch (`trunk`) and all pull requests. `permissions: contents: read`.

### Jobs

Both jobs run as a matrix over `ubuntu-latest`, `windows-latest`, `macos-latest` with `fail-fast: false` (so one OS failing doesn't cancel the others):

**`build`**
1. Check out code (`actions/checkout`, pinned by SHA)
2. Set up Go (version taken from `go.mod`)
3. `go mod download`
4. Run unit and integration tests: `go test -race -tags=integration ./...` (race detector enabled)
5. `go build -v ./cmd/gh`
6. *(Ubuntu only)* build with debug overrides: `go build -v -tags=ghdebug ./cmd/gh`

**`integration-tests`** (uses the job-level `GITHUB_TOKEN` as `GH_TOKEN`)
1. Check out code
2. Set up Go
3. `make` — builds the executable
4. Run the attestation command-set integration tests: `./test/integration/attestation-cmd/run-all-tests.sh "${{ matrix.os }}"`

Notably, this workflow uploads **no artifacts** and produces **no release binaries** — it is purely a test/build verification gate. Release binaries come from the separate `Deployment` workflow / release automation.

---

## 3. Run examined

No recent run of this workflow on `trunk` was failing (the latest several runs — #22973, #22982, #22985, #22987, #22990 — were all `success`), so per the audit brief the **most recent completed run** was dissected:

- **Run:** [#22990](https://github.com/cli/cli/actions/runs/39623013081) (ID `39623013081`), attempt 1
- **Workflow:** Unit and Integration Tests, event `push`, branch `trunk`
- **Commit:** `7948e8056e45ac3c729b829e0eafe2d0797a95c6` — "Merge pull request #14782 from cli/babakks/skill-gh-expiring-tokens"
- **Conclusion: `success`**

### Jobs that ran (6, all green)

| Job | ID | Duration | Result |
|---|---|---|---|
| `build (ubuntu-latest)` | 119301061971 | ~5m27s | success |
| `build (windows-latest)` | 119301061530 | ~11m34s | success |
| `build (macos-latest)` | 119301061554 | ~7m07s | success |
| `integration-tests (ubuntu-latest)` | 119301061179 | ~1m26s | success |
| `integration-tests (windows-latest)` | 119301061453 | ~2m14s | success |
| `integration-tests (macos-latest)` | 119301061547 | ~1m27s | success |

### Main build job — what the logs show (`build (ubuntu-latest)`, job 119301061971)

All steps completed successfully, in order:

1. **Set up job** → success
2. **Check out code** → success
3. **Set up Go** → success
4. **Download dependencies** (`go mod download`) → success
5. **Run unit and integration tests** (`go test -race -tags=integration ./...`) → success (~4m23s). The log is a long stream of package compile/test lines covering the entire `github.com/cli/cli/v2/...` tree (`pkg/cmd/project/...`, `pkg/cmd/release/...`, `pkg/cmd/repo/...`, `pkg/cmd/ruleset/...`, `pkg/cmd/run/...`, `pkg/cmd/search/...`, `pkg/cmd/secret/...`, `pkg/cmd/workflow/...`, `internal/skills/...`, `internal/barista/...`, plus third-party deps like `github.com/twitchtv/twirp`, `golang.org/x/crypto/...`, `github.com/joho/godotenv`, etc.). **No `FAIL`, no test failures, no race-detector reports.**
6. **Build** (`go build -v ./cmd/gh`) → success (~42s)
7. **Build with debug overrides** (`go build -v -tags=ghdebug ./cmd/gh`) → success (this step runs only on Ubuntu; it was *skipped* in the Windows and macOS matrix legs, as expected per the workflow's `if:` condition)

### Artifacts

- **None.** The run uploaded **zero artifacts** (verified via the run's artifact listing: `total_count: 0`). There are no artifact download links to provide — this is expected, since `go.yml` contains no `actions/upload-artifact` steps.

---

## 4. Conclusions

1. **v2.63.0 content:** a modest feature release — bare repo creation, `baseRefOid` in `pr view`, attestation refactors and verification-output improvements, docs fixes (heredoc examples), and a friendlier OAuth-scope error — with a **security advisory** attached (token leak via external submodules, GHSA-jwcm-9g39-pmcw).
2. **CI posture at the time of audit:** the main build-and-test workflow (`go.yml`) runs a full unit + integration test suite with the race detector across three OSes, plus a dedicated attestation integration suite. The examined run (#22990) was fully green across all 6 jobs.
3. **No failure forensics were possible** (nothing to dissect — the branch is green), and **no artifacts are retained** by this workflow, so verification relies on logs only.

---

## 5. Items for the team to double-check

- **Security advisory applicability:** confirm whether any team workflows clone repos with submodules hosted outside GitHub.com/ghe.com (GHSA-jwcm-9g39-pmcw) — if yes, plan the upgrade past v2.63.0 as a priority.
- **This audit examined CI as of the current `trunk`**, not the exact `trunk` state at tag `v2.63.0` (Nov 2024) — the workflow definition has evolved since. If you need the *exact* CI that validated v2.63.0, the `go.yml` at tag `v2.63.0` should be pulled and the corresponding run around 2024-11-27 located.
- **No artifacts are kept** by the test workflow — consider adding test-report or coverage artifacts if the audit needs reproducible evidence beyond logs.
- The release binaries/checksums themselves are attached by `github-actions[bot]`; verify them per the release's manual-verification guidance added in #9936.

---

*Audit performed via the GitHub API on behalf of @HeziLiu; recorded in the `release-audit-log` repository.*
