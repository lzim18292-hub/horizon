# Performance audit — 2026-11-14

## Scope and evidence

This is a source-and-results audit of the public benchmark reports already committed to this repository. It is **not** a benchmark of the current `performance-development` branch and it does not prove that the current branch has gained FPS.

The comparable historical pair is the Linux desktop set from 2026-11-04:

- Horizon: `benchmarks/perf/horizon-desktop-rd12.json` and `horizon-desktop-rd32.json`, build `nightly-2026-11-04-67dad0d`.
- Vanilla: `benchmarks/perf/vanilla-desktop-rd12.json` and `vanilla-desktop-rd32.json`, Minecraft 26.3 with Fabric Loader 0.19.5.

Both were recorded on the same reported host: Linux/NixOS, Intel Core i7-13700KF (24 threads), NVIDIA RTX 4080, 1280×720, seed 1234, simulation distance 12, VSync off, windowed rendering, three repeats, and the same four scenes. The reported one-minute load averages were high (about 28–30), so these measurements should be treated as a noisy, loaded-host snapshot rather than a clean laboratory baseline.

## Comparable average FPS

Percent change is `(Horizon / Vanilla - 1) × 100`. Positive values favour Horizon; negative values identify a measured regression.

| Render distance | Scene | Horizon FPS | Vanilla FPS | Change |
|---:|---|---:|---:|---:|
| 12 | Village | 2,958.6 | 2,161.5 | +36.9% |
| 12 | Forest | 2,333.6 | 1,841.6 | +26.7% |
| 12 | Ocean | 2,192.9 | 1,525.5 | +43.7% |
| 12 | Mountain | 1,412.1 | 1,769.1 | **−20.2%** |
| 32 | Village | 853.8 | 525.7 | +62.4% |
| 32 | Forest | 898.2 | 599.4 | +49.8% |
| 32 | Ocean | 735.2 | 1,175.0 | **−37.4%** |
| 32 | Mountain | 712.7 | 485.8 | +46.7% |

## Frame-time lows

The benchmark defines 1% and 0.1% lows from the slowest frame-time tails. These are included because average FPS alone hides stutter.

| Distance | Scene | Horizon 1% / 0.1% low | Vanilla 1% / 0.1% low |
|---:|---|---:|---:|
| 12 | Village | 368.5 / 91.0 | 162.6 / 27.8 |
| 12 | Forest | 336.6 / 160.7 | 255.1 / 92.9 |
| 12 | Ocean | 345.4 / 172.6 | 191.4 / 49.5 |
| 12 | Mountain | 182.5 / 88.5 | 195.9 / 52.2 |
| 32 | Village | 130.2 / 70.7 | 44.2 / 10.6 |
| 32 | Forest | 178.5 / 103.4 | 61.9 / 15.8 |
| 32 | Ocean | 134.7 / 62.1 | 119.8 / 24.2 |
| 32 | Mountain | 117.9 / 72.4 | 28.6 / 4.6 |

The two average-FPS regressions (mountain/RD12 and ocean/RD32) do not show the same regression in the slowest-frame tails. That suggests a throughput-versus-tail distinction worth profiling; it does **not** identify the cause.

## Startup and resident memory

| Distance | Metric | Horizon | Vanilla | Horizon change |
|---:|---|---:|---:|---:|
| 12 | Title startup | 66.7 s | 121.1 s | −44.9% |
| 12 | World startup | 74.8 s | 191.1 s | −60.9% |
| 12 | Steady RSS | 1,348 MB | 2,985 MB | −54.8% |
| 12 | Peak RSS | 1,546 MB | 3,108 MB | −50.3% |
| 32 | Title startup | 45.4 s | 112.7 s | −59.7% |
| 32 | World startup | 86.6 s | 235.0 s | −63.2% |
| 32 | Steady RSS | 1,811 MB | 3,291 MB | −45.0% |
| 32 | Peak RSS | 2,051 MB | 3,372 MB | −39.2% |

Client tick time was also lower for Horizon in all eight scene/distance pairs in these reports. This is useful CPU-side evidence, but it does not explain the two average-FPS regressions by itself.

## Comparisons deliberately excluded

- The M4 Pro 1080p Horizon and Vanilla reports use different actual window heights (1080 versus 930) and different graphics backends (Metal versus OpenGL); they are not a fair FPS comparison.
- The Windows Ryzen 7 9800X3D / RTX 5080 reports use Vulkan for Horizon and OpenGL for Vanilla; they are useful platform snapshots but not a controlled renderer-to-renderer comparison.
- The older `benchmarks/graphics/{baseline,after}-*.json` experiment is historical (commits `27a32cc3` to `9006566b`, 2026-10-13). It documents an earlier large renderer improvement, but it is not evidence for the current branch.

## Architecture inspection

The audit inspected the existing renderer and streaming architecture, including:

- `crates/horizon-client-render/src/streaming.rs`: shared worker pool, persistent per-worker scratch, bounded generation plans, priority queue, non-blocking result mailbox, stage census and reusable per-update scratch.
- `crates/horizon-client-render/src/world.rs` and `chunk_map.rs`: dirty-section masks, membership revisions/deltas, neighbourhood counts and cached chunk membership.
- `crates/horizon-client-render/src/mesher.rs`: packed terrain vertices, reusable mesh buffers, face visibility and mesh generation.
- `crates/horizon-client-render/src/gfx.rs`: GPU culling, batched/indirect section drawing, vertex arenas, buffered uploads and bounded arena compaction.

These code paths already contain substantial optimizations. A broad rewrite, arbitrary worker-count increase or Rust-to-C++ port would be unjustified without profiling the current commit on a repeatable workload.

## Next optimization targets

1. **Mountain / RD12:** profile CPU frame phases and mesh preparation on the current commit; this is the measured average-FPS regression at medium distance.
2. **Ocean / RD32:** profile terrain/fluid draw, GPU time and CPU frame phases on the current commit; this is the measured average-FPS regression at high distance.
3. **Preserve frame pacing:** compare 1% and 0.1% lows before accepting any change, not only average FPS.
4. **Protect memory/startup gains:** keep RSS and startup in the acceptance report.
5. **Do not introduce C++ yet:** first isolate a CPU hotspot, benchmark an equivalent Rust implementation, then compare any FFI cost against a native C++ prototype.

## Benchmark input edge cases found during review

The final input review found one alias-expansion case beyond literal duplicate detection: selecting `stress` together with an individual member such as `stress_mobs` caused that stress scene to be selected twice. `validate_benchmark_args` now rejects that overlap before the run begins, and the regression cases cover both argument orders. The repository tests have not yet been executed in this session.

## Current branch verification status

The current branch work so far changes benchmark input validation and the direct-test entrypoint. The input-validation tests have been added, the direct `unittest.main()` guard is now after all test classes, and every resolution-derived report label is checked against the same length rule used by `--table`.

The fork initially inherited CI runner selectors intended for the canonical `horizon-rust/horizon` repository. Because PR #1 is opened inside this fork, the original expressions selected private `horizon-checks` and `horizon-light` runners that the fork does not own. This branch now gates those five normal CI selectors on `github.repository == 'horizon-rust/horizon'`, so forks use GitHub-hosted Ubuntu runners; `tests/test_ci_runners.py` includes a regression test for this routing.

Even after that correction, the GitHub API still reports zero Actions workflow runs for the branch and zero commit checks. The API connector does not expose repository Actions settings or workflow dispatch, so whether Actions are disabled or the PR run is awaiting approval cannot be established from this session. No Desktop Commander device is connected, and this session cannot clone GitHub directly (the execution environment has no DNS/network route to github.com). Therefore:

- Python unit tests: **not executed here**.
- `tools/pr_ready.py`: **not executed here**.
- CI runner selection: **patched in the fork and regression test added; GitHub Actions execution still not observed**.
- Rust formatting, Clippy, crate tests and game build: **not executed here**.
- Current-commit FPS/CPU/GPU/memory benchmark: **not executed here**.
- PR remains a draft; do not merge until the project checks pass.

A valid next performance result requires the current commit to run through the repository's documented self-check and, for FPS claims, the benchmark suite on a named hardware/backend with the report artifacts preserved.
