# Continuous Integration

This document describes the CI/CD pipeline, what each stage verifies, and how
to run the same checks locally.

## Pipeline overview

| Workflow (`.github/workflows/`) | Trigger | What it gates |
| --- | --- | --- |
| `pre-commit.yml` | every PR | Formatting/lint of the lines changed in the PR |
| `pre-commit-all.yml` | push to `main` | Formatting/lint of the whole tree |
| `ci.yml` | every PR; push to `main`, `claude/**`, `dev/**`; **daily schedule**; manual | Build + C++/lit tests + Python e2e tests + generated-source check, on torch `nightly` and `stable` (scheduled/manual runs add a `nightly-latest` canary). See [CI tiers](#ci-tiers) |
| `ci_windows.yml` | push to `main`; manual | MSVC build still compiles |
| `bazelBuildAndTest.yml` | push to `main`; manual | Bazel build and tests |
| `buildRelease.yml`, `*SnapshotPackage.yml`, `gh-pages-releases.yml` | manual only | Python wheels and release publishing (upstream-only) |
| `RollPyTorch.yml`, `merge-rollpytorch.yml` | daily / after roll | Automatic PyTorch version bump (upstream-only; skipped on forks) |

"CD" in this project means producing the Python wheels and snapshot releases.
Those workflows are deliberately disabled on forks; see
[Release workflows](#release-workflows).

## Enabling the pipeline: files changed and not changed

### Files changed (the whole CI/CD change set)

| File | Change |
| --- | --- |
| `.github/workflows/ci.yml` | Hosted runner by default (`CI_LINUX_RUNNER` override); triggers on PRs to any branch and on pushes to `main`, `claude/**`, `dev/**`; daily `schedule`; `nightly-latest` canary leg on scheduled/manual runs; installs clang+lld; frees disk; removed the ccache action whose save condition could never be true |
| `build_tools/ci/install_python_deps.sh` | New `nightly-latest` case (newest PyTorch nightly, unpinned) |
| `build_tools/ci/test_posix.sh` | `nightly-latest` runs the same e2e suites as `nightly` |
| `docs/ci.md` | This document (new) |

### Files deliberately not changed

| File(s) | Why |
| --- | --- |
| `build_tools/ci/build_posix.sh`, `linux_default_toolchain.cmake`, `check_generated_sources.sh` | Already correct for any runner; changing them adds merge conflicts with upstream for no benefit |
| `.github/actions/setup-build/action.yml` | Reused as is (called with `cache-enabled: 'false'`) |
| `.github/workflows/pre-commit.yml`, `pre-commit-all.yml`, `.pre-commit-config.yaml` | Work unchanged on hosted runners |
| `ci_windows.yml`, `bazelBuildAndTest.yml` | Already use hosted runners; run on pushes to `main` only |
| `RollPyTorch.yml`, `merge-rollpytorch.yml`, `oneshotSnapshotPackage.yml`, `releaseSnapshotPackage.yml`, `gh-pages-releases.yml` | Upstream-only (secrets, runners); skipped on forks by their own `if:` guard |
| `buildRelease.yml` | Not touched; manual only and needs an `a100` runner, so do not run it on a fork |
| `pytorch-requirements.txt`, `pytorch-hash.txt`, `requirements.txt` | The PyTorch pin. Stale on this fork (see below); updating it is a separate decision |
| `projects/pt1/e2e_testing/xfail_sets.py` and all test sources | Test content is not part of enabling CI |

### Steps to turn it on

1. Merge the PR containing these changes (`ci.yml` must be on the repository's
   **default branch** for the schedule to fire).
2. **Settings > Actions > General**: allow actions, and enable workflows on the
   fork from the **Actions** tab.
3. Optional **Settings > Secrets and variables > Actions > Variables**:
   `CI_LINUX_RUNNER` (bigger runner label), `NIGHTLY_BRANCH` (default `dev`).
4. **Settings > Branches**: protect `main`/`dev`; require `pre-commit`,
   `Build and Test (Linux, torch-nightly, assertions)` and
   `(…, torch-stable, …)`. Do not require the `nightly-latest` leg.
5. Open a PR and check the Actions tab.

### Known issue: stale PyTorch pin

`pytorch-requirements.txt` pins a PyTorch nightly wheel that the PyTorch index
has since deleted, so the pinned `nightly` leg fails at "Install python deps"
until the pin is refreshed. Upstream refreshes it daily with `RollPyTorch.yml`,
which is skipped on forks. Fix by merging the current upstream
(`iree-org/torch-mlir`) `main` into your branch, or by rolling the pin by hand
(`pytorch-hash.txt`, `pytorch-requirements.txt`, `torchvision-requirements.txt`,
then `build_tools/update_torch_ods.sh` and `update_abstract_interp_lib.sh`).

## CI tiers

"Nightly" is overloaded in this project, so be precise:

| Term | Meaning |
| --- | --- |
| torch **`nightly`** | A *PyTorch version*: the daily PyTorch dev build **pinned** in `pytorch-requirements.txt`. A job leg, not a time of day. |
| torch **`stable`** | A *PyTorch version*: the latest released PyTorch. |
| torch **`nightly-latest`** | A *PyTorch version*: the **newest** PyTorch nightly, unpinned. A canary. |
| **Nightly run** | A *schedule*: `ci.yml` firing from `cron` at 02:17 UTC. Not a release. |

| Tier | When | What runs | Blocking? |
| --- | --- | --- | --- |
| **1. PR / push gate** | every PR; push to `main`, `claude/**`, `dev/**` | `pre-commit`, then `ci.yml` legs `nightly` and `stable` | Yes: require them in branch protection |
| **2. Scheduled nightly run** | daily, on the tip of `NIGHTLY_BRANCH` (default `dev`) | Legs `nightly`, `stable` **and** `nightly-latest`; every leg runs to completion (no fail-fast) | No: it reports, nobody is waiting on it |

What differs between the tiers is *what is checked and against which code*, not
the test programs. Both tiers run the same scripts and the same e2e test
suites.

- **Tier 1 answers:** "does this change break anything?" It tests the PR's
  code against two PyTorch versions.
- **Tier 2 answers:** "is the branch still healthy, and is PyTorch about to
  break us?" It re-tests the branch with no new commit (catching expired
  dependencies, flaky tests and drift), and the `nightly-latest` leg tests
  against tomorrow's PyTorch. A red `nightly-latest` with green `nightly` means
  the next roll of the pin will need code changes; fix it on your schedule.
- `nightly-latest` runs everything `nightly` does except the generated-source
  check, which only makes sense against the pinned build.

Pieces of one run, for reference:

| Leg | Build + lit | ONNX e2e | FX-importer e2e (Linalg, StableHLO, TOSA) | Generated-source check |
| --- | --- | --- | --- | --- |
| `stable` | yes | yes | no | no |
| `nightly` | yes | yes | yes | yes |
| `nightly-latest` (tier 2 only) | yes | yes | yes | no |

### ONNX e2e vs FX-importer e2e

Both are end to end (import, lower, run on the reference backend, compare with
eager PyTorch). They differ in the front door:

- `--config=onnx` exports each test model to **ONNX** and re-imports it through
  the torch-onnx-to-torch path. It tests the ONNX importer and its conversions.
- `--config=fx_importer*` imports the PyTorch **FX graph** (`torch.export`)
  directly, then lowers to Linalg, StableHLO or TOSA. It tests the FX importer
  and each lowering path.

A bug in the ONNX conversion fails only the first; a bug in a Linalg lowering
typically fails both, and a StableHLO- or TOSA-only bug fails only that config.

### Requirements for the schedule to run

- GitHub only fires `schedule` from the **default branch's** copy of `ci.yml`.
  If your default branch is `main` and you develop on `dev`, merge this
  workflow into `main`; `ci.yml` then checks out `dev` (set the repository
  variable `NIGHTLY_BRANCH` to use another branch).
- On forks, scheduled workflows are off until you enable Actions under the
  **Actions** tab. GitHub also pauses schedules after 60 days without repo
  activity.
- Scheduled failures notify the person who last changed the cron line. Watch
  the Actions tab or set up notifications for failed workflows.

## What is tested, and why

`ci.yml` runs `build_tools/ci/*.sh` inside a matrix of
`torch-version: [nightly, stable]`. The stages, in order:

1. **Install deps** (`install_python_deps.sh`). Installs the pinned nightly
   torch (`requirements.txt`) or the latest stable torch, plus
   `test-requirements.txt`. *Catches:* broken or unsatisfiable dependency pins.
2. **Build** (`build_posix.sh`). Builds LLVM/MLIR with torch-mlir as an
   external project, with assertions on and `-Werror`
   (`TORCH_MLIR_ENABLE_WERROR_FLAG=ON`), including the Python bindings and the
   PyTorch extensions. *Catches:* compile errors, new warnings, and
   LLVM-assertion failures that a release build would hide.
3. **Unit tests** (`check-torch-mlir-all`, also in `build_posix.sh`). The
   `lit`/FileCheck tests under `test/`: dialect verifiers, op parsing/printing,
   canonicalizers, folders, every conversion pass
   (Torch -> Linalg/TOSA/StableHLO), decompositions, shape/dtype refinement.
   *Catches:* a change to a pass or lowering that alters the produced IR.
4. **ONNX e2e tests** (`--config=onnx`). Imports ONNX models/ops, lowers them
   through torch-mlir and runs them on the reference backend, comparing against
   the original result numerically. *Catches:* wrong ONNX -> Torch conversions
   (semantics, not just IR shape).
5. **FxImporter e2e tests** (nightly only; configs `fx_importer`,
   `fx_importer_stablehlo`, `fx_importer_tosa`). Exports PyTorch programs with
   `torch.export`, imports via the FX importer, lowers to
   Linalg / StableHLO / TOSA, executes, and compares to eager PyTorch.
   *Catches:* importer regressions and numerical bugs in each lowering path.
   Expected failures are tracked in `projects/pt1/e2e_testing/xfail_sets.py`;
   when you fix a lowering, update that list in the same change.
6. **Generated-source check** (nightly only, `check_generated_sources.sh`).
   Regenerates `GeneratedTorchOps.td` and `AbstractInterpLibrary.cpp` and fails
   if the checked-in copies differ. *Catches:* edits to
   `torch_ods_gen.py` / `abstract_interp_lib_gen.py` without regenerating, and
   drift against the pinned PyTorch.

Why several torch versions: *nightly* (pinned) is what the repo is developed
against and tested most deeply; *stable* confirms the build and ONNX path still
work for users on a released torch; *nightly-latest* (scheduled only) gives
early warning of upstream PyTorch changes. A change that passes only on
nightly is a compatibility risk.

### Things the pipeline does not cover (decide if you need them)

- Release builds (`-DCMAKE_BUILD_TYPE=Release` is used, but with assertions
  on). Wheel building is only exercised by the release workflows.
- macOS and GPU/CUDA backends.
- Sanitizers (ASan/UBSan) and a code-coverage report.
- The Lazy Tensor Core path (`TORCH_MLIR_ENABLE_LTC=OFF`; its e2e run is
  commented out in `test_posix.sh` because it is failing upstream).

Adding any of these is a new job in `ci.yml`; none are required to start.

## Runners and cost

`ci.yml` defaults to the GitHub-hosted `ubuntu-24.04` runner so it works on a
fork. Upstream's `torch-mlir-cpubuilder-*` runners only exist in the upstream
organisation, so jobs targeting them never start on a fork.

- A cold LLVM+MLIR build on a 4-core hosted runner is slow (expect well over an
  hour). `ccache` is stored with `actions/cache` (keyed per torch version), so
  subsequent runs recompile only what changed.
- For faster feedback, create a larger runner or self-hosted runner and set the
  repository **variable** `CI_LINUX_RUNNER` to its label. No file edit is
  needed.
- The job timeout is 350 minutes (hosted runners cap at 360).
- Concurrent runs for the same PR/commit are cancelled automatically.

## Repository settings to configure (one time, in the GitHub UI)

These cannot be set from files in the repo:

1. **Settings > Actions > General**: allow Actions, and allow
   "Read and write permissions" only if you later add publishing jobs.
2. **Settings > Branches > Branch protection** for `main` (and the shared
   development branch): require pull requests and require these status checks:
   `pre-commit` and `Build and Test (Linux, torch-nightly, assertions)` /
   `(…, torch-stable, …)`. Do **not** require the `torch-nightly-latest` leg:
   it only runs on the schedule.
3. Optionally set the `CI_LINUX_RUNNER` variable (see above) and
   `NIGHTLY_BRANCH` (branch the scheduled run tests; default `dev`).

## Running the checks locally

```shell
# Lint/format (same hooks as CI)
pip install pre-commit && pre-commit run --all-files

# Build + unit tests + e2e (same scripts as CI; needs clang, lld, ccache, ninja)
git submodule update --init --recursive
./build_tools/ci/install_python_deps.sh nightly   # or: stable | nightly-latest
./build_tools/ci/build_posix.sh
./build_tools/ci/test_posix.sh nightly            # same argument as above
./build_tools/ci/check_generated_sources.sh
```

To run a single e2e test: `python -m e2e_testing.main --config=fx_importer -f <TestName>`
with `PYTHONPATH` set as in `build_tools/ci/test_posix.sh`.

## Release workflows

`oneshotSnapshotPackage.yml`, `releaseSnapshotPackage.yml`,
`gh-pages-releases.yml`, `RollPyTorch.yml` and `merge-rollpytorch.yml` need
upstream secrets (e.g. `WORKFLOW_INVOCATION_TOKEN`) and are skipped on forks via
`if: github.repository == 'llvm/torch-mlir'`. `buildRelease.yml` has no such
guard, but it is `workflow_dispatch` only and targets an `a100` self-hosted
runner, so do not trigger it on a fork. If you later want your
own wheels, add a dedicated workflow that runs `build_tools/python_deploy/`
scripts and uploads to your own release or package index.
