# Continuous Integration

This document describes the CI/CD pipeline, what each stage verifies, and how
to run the same checks locally.

## Pipeline overview

| Workflow (`.github/workflows/`) | Trigger | What it gates |
| --- | --- | --- |
| `pre-commit.yml` | every PR | Formatting/lint of the lines changed in the PR |
| `pre-commit-all.yml` | push to `main` | Formatting/lint of the whole tree |
| `ci.yml` | every PR; push to `main`, `claude/**`, `dev/**`; manual | Build + C++/lit tests + Python e2e tests + generated-source check, on torch `nightly` and `stable` |
| `ci_windows.yml` | push to `main`; manual | MSVC build still compiles |
| `bazelBuildAndTest.yml` | push to `main`; manual | Bazel build and tests |
| `buildRelease.yml`, `*SnapshotPackage.yml`, `gh-pages-releases.yml` | manual only | Python wheels and release publishing (upstream-only) |
| `RollPyTorch.yml`, `merge-rollpytorch.yml` | nightly / after roll | Automatic PyTorch version bump (upstream-only; skipped on forks) |

"CD" in this project means producing the Python wheels and snapshot releases.
Those workflows are deliberately disabled on forks; see
[Release workflows](#release-workflows).

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

Why both torch versions: *nightly* is what the repo pins and tests most deeply;
*stable* confirms the build and ONNX path still work for users on a released
torch. A change that passes only on nightly is a compatibility risk.

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

## Running the checks locally

```shell
# Lint/format (same hooks as CI)
pip install pre-commit && pre-commit run --all-files

# Build + unit tests + e2e (same scripts as CI; needs clang, lld, ccache, ninja)
git submodule update --init --recursive
./build_tools/ci/install_python_deps.sh nightly
./build_tools/ci/build_posix.sh
./build_tools/ci/test_posix.sh nightly
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
