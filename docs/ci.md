# Continuous Integration

This document describes the CI/CD pipeline, what each stage verifies, and how
to run the same checks locally.

## Pipeline overview

| Workflow (`.github/workflows/`) | Trigger | What it gates |
| --- | --- | --- |
| `pre-commit.yml` | every PR | Formatting/lint of the lines changed in the PR |
| `pre-commit-all.yml` | push to `main` | Formatting/lint of the whole tree |
| `ci.yml` | every PR; push to `main`, `claude/**`, `dev/**`; manual | Build + C++/lit tests + Python e2e tests + generated-source check, on torch `nightly` and `stable`. See [nightly vs stable](#nightly-vs-stable-torch-versions) |
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
| `.github/workflows/ci.yml` | Hosted runner by default (`CI_LINUX_RUNNER` override); triggers on PRs to any branch and on pushes to `main`, `claude/**`, `dev/**`; installs clang+lld; frees disk; removed the ccache action whose save condition could never be true |
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

1. Merge the PR containing these changes.
2. **Settings > Actions > General**: allow actions, and enable workflows on the
   fork from the **Actions** tab.
3. Optional **Settings > Secrets and variables > Actions > Variables**:
   `CI_LINUX_RUNNER` (bigger runner label).
4. **Settings > Branches**: protect `main`/`dev`; require `pre-commit`,
   `Build and Test (Linux, torch-nightly, assertions)` and
   `(…, torch-stable, …)`.
5. Open a PR and check the Actions tab.

### Known issue: stale PyTorch pin

`pytorch-requirements.txt` pins a PyTorch nightly wheel that the PyTorch index
has since deleted, so the pinned `nightly` leg fails at "Install python deps"
until the pin is refreshed. Upstream refreshes it daily with `RollPyTorch.yml`,
which is skipped on forks. Fix by merging the current upstream
(`iree-org/torch-mlir`) `main` into your branch, or by rolling the pin by hand
(`pytorch-hash.txt`, `pytorch-requirements.txt`, `torchvision-requirements.txt`,
then `build_tools/update_torch_ods.sh` and `update_abstract_interp_lib.sh`).

## Nightly vs stable torch versions

"Nightly" is easy to misread. Here it is **not** a time of day, a schedule or
a release. It is one of two PyTorch versions that every CI run tests:

| Leg | PyTorch installed |
| --- | --- |
| torch **`nightly`** | The daily PyTorch dev build **pinned** in `pytorch-requirements.txt`. The repo's code generators and importers are written against this build. |
| torch **`stable`** | The latest released PyTorch. |

CI runs on every PR and on pushes to `main`, `claude/**` and `dev/**`; each run
starts both legs in parallel. There are no scheduled runs and no separate test
suites: the legs differ only in which PyTorch they install and therefore which
tests apply (next table). Require both in branch protection.

What each leg runs:

| Leg | Build + lit | ONNX e2e | FX-importer e2e (Linalg, StableHLO, TOSA) | Generated-source check |
| --- | --- | --- | --- | --- |
| `stable` | yes | yes | no | no |
| `nightly` | yes | yes | yes | yes |

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

Why both torch versions: *nightly* (pinned) is what the repo is developed
against and tested most deeply; *stable* confirms the build and ONNX path still
work for users on a released torch. A change that passes only on nightly is a
compatibility risk.

### The two kinds of test, concretely

**Unit tests (lit + FileCheck)** live under `test/` as `.mlir` files. Each has
a few lines of input IR and the IR expected after a pass. Nothing is executed.
Example (`test/Conversion/TorchToLinalg/elementwise.mlir`):

```
// RUN: torch-mlir-opt <%s -convert-torch-to-linalg ... | FileCheck %s
func.func @elementwise$unary(%arg0: !torch.vtensor<[],f32>) ... {
  %0 = torch.aten.tanh %arg0 ...      // input: a PyTorch tanh
}
// CHECK: linalg.generic ...          // expected: a linalg loop
// CHECK:   math.tanh ...             // applying math.tanh per element
```

Main folders: `test/Dialect/Torch` (op verification, folding, decompositions,
shape/dtype inference), `test/Conversion/TorchToLinalg`, `TorchToStablehlo`,
`TorchToTosa`, `TorchOnnxToTorch` (one folder per lowering), and
`test/Dialect/TorchConversion`. A failure means a pass now produces different
IR; it may be a real bug or an intended change whose expectation needs
updating.

**End-to-end (e2e) tests** check the *numbers*. Each is a small PyTorch module
plus an input, in `projects/pt1/python/torch_mlir_e2e_test/test_suite/*.py`
(about 1700 cases). Example (`elementwise.py`):

```python
class ElementwiseAddModule(torch.nn.Module):
    def forward(self, a, b):
        return a + b

@register_test_case(module_factory=lambda: ElementwiseAddModule())
def ElementwiseAddModule_basic(module, tu):
    module.forward(tu.rand(4), tu.rand())
```

For each test the runner (1) runs the module in plain PyTorch to get the
expected result, (2) compiles it through torch-mlir, (3) runs it on the
reference backend, (4) compares. A mismatch is a FAIL.

### XFAIL: expected failures

Torch-mlir does not support every PyTorch op, so some e2e tests fail today.
Rather than leave CI permanently red, known failures are listed per config in
`projects/pt1/e2e_testing/xfail_sets.py` (`ONNX_XFAIL_SET`,
`FX_IMPORTER_XFAIL_SET`, ...). The runner classifies every test:

| Result | On the xfail list? | Outcome | CI |
| --- | --- | --- | --- |
| passes | no | PASS | green |
| fails | no | **FAIL** (new bug) | red |
| fails | yes | XFAIL (known gap) | green |
| passes | yes | **XPASS** (list is stale) | red |

XPASS is deliberately an error: when you fix an op, the test starts passing and
CI forces you to delete its name from the list in the same PR, so the list
never rots and a later regression in that test is not silently ignored.
Practical rules:

- Fixed a lowering and CI reports XPASS: remove the name from the right set.
- Added a test that cannot pass yet: add it to the right set with a comment
  saying why.
- Lists are **per config** and tied to the PyTorch version they were written
  for, so rolling the pin can produce XPASS or FAIL entries that need updating.
- Tests that crash the process (segfault) cannot be XFAILed; they go in the
  matching `*_CRASHING_SET` and are not run at all.

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
   `(…, torch-stable, …)`.
3. Optionally set the `CI_LINUX_RUNNER` variable (see above).

## Script reference

### End-to-end testing

| Script / file | Role |
| --- | --- |
| `build_tools/ci/test_posix.sh <nightly\|stable>` | CI entry point for e2e tests. Sets `PYTHONPATH`, then runs `python3 -m e2e_testing.main --config=<cfg> -v` for `onnx` (all legs) and `fx_importer`, `fx_importer_stablehlo`, `fx_importer_tosa` (nightly legs) |
| `projects/pt1/e2e_testing/main.py` | The e2e runner. Picks the config (compile route), the xfail/crashing sets, filters (`-f`), and prints the PASS/FAIL/XFAIL/XPASS report |
| `projects/pt1/e2e_testing/xfail_sets.py` | Per-config lists of tests expected to fail or crash |
| `projects/pt1/python/torch_mlir_e2e_test/test_suite/*.py` | The ~1700 test cases themselves |
| `projects/pt1/python/torch_mlir_e2e_test/configs/*.py` | How each config compiles and runs a module (`onnx_backend.py`, `fx_importer_backend.py`, ...) |
| `projects/pt1/python/torch_mlir_e2e_test/reporting.py` | Compares results with the golden PyTorch output and classifies outcomes |

Unit (lit/FileCheck) tests are not a script: `build_posix.sh` builds the CMake
target `check-torch-mlir-all`, which runs `lit` over `test/`.

### Build and CI helpers

| Script | Role | Used by |
| --- | --- | --- |
| `build_tools/ci/install_python_deps.sh` | Installs the torch variant plus test requirements | `ci.yml` |
| `build_tools/ci/build_posix.sh` | CMake configure, build, and `check-torch-mlir-all` | `ci.yml` |
| `build_tools/ci/check_generated_sources.sh` | Regenerates and diffs generated files | `ci.yml` (nightly leg) |
| `build_tools/update_torch_ods.sh`, `update_abstract_interp_lib.sh` | Regenerate `GeneratedTorchOps.td` and `AbstractInterpLibrary.cpp`. Run these yourself after editing `torch_ods_gen.py` / `abstract_interp_lib_gen.py` | You; `check_generated_sources.sh` |
| `build_tools/python_deploy/build_windows_ci.sh` | Windows build | `ci_windows.yml` |
| `build_tools/build_standalone.sh`, `write_env_file.sh` | Local out-of-tree configure; `.env` for IDE/Python paths | Developers |
| `build_tools/e2eshark_build.sh` | Builds torch-mlir for the separate *e2eshark* suite | That project's CI, not this one |

### Release packaging ("nightly release" / snapshots)

These are unrelated to the `nightly` torch leg. They produce Python wheels and
GitHub releases, and are **not enabled** on this fork.

| Script / workflow | Role |
| --- | --- |
| `releaseSnapshotPackage.yml` | Tags `snapshot-<date>.<run>`, creates a draft GitHub release, then dispatches the build. Upstream's `cron` for it is commented out, so it only runs manually |
| `oneshotSnapshotPackage.yml` | Same, tagged `oneshot-<date>.<run>`, from a chosen ref |
| `buildRelease.yml` | Builds wheels for Linux, macOS and Windows and uploads them to the release |
| `build_tools/python_deploy/build_linux_packages.sh` | Builds manylinux wheels inside Docker (`build_tools/docker/Dockerfile`) |
| `build_tools/python_deploy/build_macos_packages.sh`, `install_macos_deps.sh`, `build_upload_m1_snapshot.sh` | macOS wheels |
| `build_tools/python_deploy/build_windows.ps1` | Windows wheels |
| `build_tools/build_python_wheels.sh` | Builds and smoke-tests wheels in a venv |
| `gh-pages-releases.yml` + `build_tools/scrape_releases.py` | Regenerates a pip-installable release index page from the GitHub releases API |
| `RollPyTorch.yml` | Daily: finds the newest PyTorch nightly, bumps the pin files, regenerates sources, opens a PR |

## Running the checks locally

```shell
# Lint/format (same hooks as CI)
pip install pre-commit && pre-commit run --all-files

# Build + unit tests + e2e (same scripts as CI; needs clang, lld, ccache, ninja)
git submodule update --init --recursive
./build_tools/ci/install_python_deps.sh nightly   # or: stable
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
