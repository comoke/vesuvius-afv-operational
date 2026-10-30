# AFV generation on a memory-limited CPU

These are source patches for [Villa](https://github.com/ScrollPrize/villa), based on [Qualzz's Automated Fiber Volume generator PR #1934](https://github.com/ScrollPrize/villa/pull/1934). That PR supplies the fiber model workflow. Our additions provide opt-in tiled CPU convolution and block checkpoint/restart, carry the Python generator onto Villa commit `8ff06f7620dcb663ed327028dbb8a0cf7e9a8198`, and add a separate schema-17 SQLite handoff for a standalone read-only file. The generator's default path is unchanged.

The repository contains two source-only packages:

| Package | Contents |
| --- | --- |
| [AFV source patch](packages/afv_current_8ff_source_only_candidate_v3.tar.gz) | Composite patch for exact Villa `8ff06f`, the separately identified tiling/restart delta against PR #1934, apply receipts, tested dependency versions and notices. |
| [Schema-17 handoff patch](packages/schema17_portability_source_v5.tar.gz) | Small importer patch and one-fiber synthetic test for publishing a closed SQLite main file that the tested native reader can open without WAL sidecars. Includes native build instructions and notices. |

Check both archives against [SHA256SUMS](SHA256SUMS) before extracting them. These V3/V5 packages update release documentation and metadata from the reviewed V2/V4 candidates. The patches, tests and license scopes are unchanged. Apply the AFV composite patch to a clean Villa checkout. The schema-17 demo copies the unmodified importer from that exact commit into its own test directory before patching it. The AFV archive's original tiling/restart delta is for inspection; do not apply it on top of its composite patch.

## What we measured

On one cached, source-pinned Scroll 3 development block, we ran the same installed candidate, model, CT block and settings under a sampled 8 GiB RSS guard. The untiled run crossed the guard at **8,979,775,488 bytes** and stopped without an AFV. With only CPU tiling enabled, it completed at **4,688,232,448 bytes** sampled peak and wrote an AFV with **67 fibers and 2,024 points**. The guard samples memory; these are not continuous hard limits or a comparison of two completed peaks.

In a separate four-block run, we stopped the process after two blocks were committed and before the third model call. Restart replayed the two completed blocks, computed the remaining two, and produced the same AFV tables as the uninterrupted run (**192 fibers, 4,756 points**). A newly built wheel at the exact Villa commit imported that result into a schema-17 store; native readback matched 576 geometry arrays, and a local VC3D test displayed the fibers over the matching task-local CT crop. These runs used one Mac and local development regions. They do not establish fiber identity, better unwrapping, ink, readable text, a speed gain or full-scroll behavior. The stopped untiled run has no output for a same-block geometry comparison.

## Apply the AFV patch

Use Python 3.14 for the documented setup. From a clean copy of this repository:

```sh
mkdir work
tar -xzf packages/afv_current_8ff_source_only_candidate_v3.tar.gz -C work
git clone --filter=blob:none https://github.com/ScrollPrize/villa.git work/villa
git -C work/villa checkout 8ff06f7620dcb663ed327028dbb8a0cf7e9a8198
cd work/villa
git apply --check ../afv_current_8ff_source_only_candidate_v3/AFV_MAIN8FF_CODE_ONLY.patch
git apply ../afv_current_8ff_source_only_candidate_v3/AFV_MAIN8FF_CODE_ONLY.patch
python3.14 -m venv ../afv-env
../afv-env/bin/python -m pip install ./vesuvius
../afv-env/bin/python -m pip check
../afv-env/bin/python -m vesuvius.afv_spline_generator --help
```

These commands check patch application, core package installation and CLI discovery **only**; `--help` is not an inference test. The package README and `CODE_ONLY_APPLY_RECEIPT.json` give the base-file and patch hashes. For a real model run, install the separate model stack as described in patched `vesuvius/docs/afv_spline_generator.md` (its `uv sync --extra models` route resolves monorepo-only native packages), and provide a compatible nnU-Net checkpoint and CT volume. `requirements.tested-macos-py314.txt` records versions observed on the tested Mac, **not** a cross-platform hash lock or a complete fresh-host installer. The accepted run used `--no-extend`. This source package omits the optional `gap_model.pt` asset, so it does not support an extension run or reproduce one. The asset-free source was checked for code-file parity and three synthetic default-path tests, but was not itself rebuilt and rerun on the real CT.

## Check the standalone schema-17 handoff

Extract the schema-17 package first and install it in its own Python 3.14 environment. Then follow `work/schema17_portability_candidate_v5/BUILD_NATIVE.md` to build the `vc_object_store` extension from the same Villa commit for that environment before running the demo:

```sh
cd ../..
tar -xzf packages/schema17_portability_source_v5.tar.gz -C work
python3.14 -m venv work/importer-env
work/importer-env/bin/python -m pip install ./work/schema17_portability_candidate_v5
work/importer-env/bin/python -m pip check
# Build the native extension per work/schema17_portability_candidate_v5/BUILD_NATIVE.md,
# using work/importer-env/bin/python as PYTHON, before running the demo.
work/importer-env/bin/vc-afv-standalone-demo \
  --native-pythonpath /path/to/native-build/python \
  --upstream-importer work/villa/vc_object_store/scripts/import_afv.py \
  --output-dir work/one-fiber-test
```

Replace `/path/to/native-build/python` with the native build's `python` directory, and use a new output directory. The synthetic test checks the patched single-file handoff and geometry; the original main-file-only read failure was observed on the tested Mac and may depend on SQLite behavior. The package README describes an optional flag for requiring that baseline failure. This test does not need the model or CT.

## Source and license boundaries

Qualzz's PR #1934 supplies the original generator and optional gap-model asset. Villa's Python source carries its MIT notice. The original additions in these two packages have a separate, scoped MIT grant in each `LICENSE-TASK-ADDITIONS.txt`; the notices do not claim ownership of upstream code or assets. VC3D is a separate GPLv3-or-later viewer and is not included. The packages contain no trained weights, CT, labels, AFV outputs, native binaries or dependency wheels. The tested full wheel included the optional gap-model file, but the accepted command disabled extension; that file's separate redistribution terms have not been established, so it is excluded here.
