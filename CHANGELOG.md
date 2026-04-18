# Changelog

All notable changes to `bx-AISentinel-ONNX` are documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [0.3.1-pre] - 2026-04-18

Rolled back the `download-model` CommandBox task introduced in 0.3.0-pre — see rationale below.

### Removed

- `tasks/DownloadModel.bx` — CommandBox's CLI engine is Lucee, so the task needed to be a `.cfc`. Worse, even as a `.cfc` it couldn't call back into `ModelAssetManager` (which is a `.bx` file) because CommandBox's Lucee can't load BoxLang class files. Duplicating the download + checksum logic inline would have meant two code paths drifting apart over time.
- `scripts.download-model` from `box.json`.
- `tests/specs/unit/DownloadModelTaskSpec.bx`.

### Changed

- `README.md`: the "Place the model" section now documents `curl` as the primary install path (already the pattern used for JARs). The `assetMode: "auto-download"` flow remains fully functional for users who want the detector to fetch the files on first `scan()` — that path is unchanged and doesn't depend on a CommandBox task.
- "Supported models" section points directly at [`models/SupportedModels.bx`](models/SupportedModels.bx) instead of recommending a task-based listing.

### Rationale

The task was a "nice to have" for Docker image builds / CI warmup, but every use case it covered is already served by either `curl` (for explicit control) or the existing `assetMode: "auto-download"` flow (for hands-off first-run). Keeping a broken task in the tree while we wait for a concrete user was costing more than the feature's value. If demand materializes, a future release can reintroduce it as a self-contained `.cfc` that duplicates only what's strictly necessary.

## [0.3.0-pre] - 2026-04-18

Model asset management + model registry + download task + convenience API. The detector is now functional end-to-end when JARs and a model are in place — first version a host app can install and exercise against real text (subject to the usual asset caveats: JARs in `lib/`, model downloaded).

### Added

- `models/SupportedModels.bx` — authoritative registry of model metadata. Each entry declares source URLs, pinned SHA-256 checksums (when verified), BIO/span decoder strategy, `id2label` map, `labelToSentinel` map, license, and license-verification status. Ships with entries for `gliner-pii-v1` (Apache 2.0, checksum unverified) and `piiranha-pii-v1` (license + checksum unverified).
- `models/ModelAssetManager.bx` — resolves model / tokenizer paths, validates files on boot, and (in `auto-download` / `shared-cache` modes) fetches missing assets over HTTPS with SHA-256 verification. Safety rails refuse to auto-download unverified checksums or unverified licenses unless the operator explicitly opts in (`acceptUnverified`, `acceptUnverifiedLicense`). Checksum mismatches ALWAYS refuse the file — never silently proceeds with a wrong model. Uses streaming SHA-256 via `java.security.MessageDigest` to avoid loading multi-hundred-MB model files into heap.
- `tasks/DownloadModel.bx` — CommandBox task invoked via `box run-script download-model`. Subcommands: `run` (default) + `list` (enumerate registry). Prints structured status; accepts `modelName`, `modelPath`, `assetMode`, `acceptUnverified`, `acceptUnverifiedLicense`, `modelSource`, and `modelChecksum` as parameters. Rethrows the underlying exception on failure so scripted callers can detect issues.
- `box.json` gains `scripts.download-model`.
- `OnnxNerDetector` now:
  - Auto-constructs a `ModelAssetManager` from module settings; paths flow through it into `OnnxSession`.
  - Auto-populates `id2label` + `labelToSentinel` from `SupportedModels` when `modelName` resolves to a registered entry.
  - `validateAssets()` returns a richer struct aggregating asset-manager + session status.
  - Supports `eagerInit` to load at module boot instead of on first `scan()`.
- Convenience API on `OnnxNerDetector` (usable outside the sentinel):
  - `extractEntities( text, types = [] )` — filter hits by Sentinel label.
  - `containsPii( text )` — fast boolean short-circuiting on first hit.
  - `redactInline( text, replacement = "[REDACTED]" )` — inline scrub (lossy; not reversible like the sentinel's token round-trip).
- Tests:
  - `tests/specs/fixtures/TestableAssetManager.bx` — `ModelAssetManager` subclass that replaces `_downloadFile` with a controllable write / fail / no-op action. Used by the auto-download orchestration specs.
  - `tests/specs/unit/ModelAssetManagerSpec.bx` — 14 specs covering path resolution, validation (no assets / directory missing / partial / checksum match + mismatch), manual-mode ensureAssets, auto-download safety rails (unverified model, unverified license, unknown model), and auto-download orchestration (happy path, failure, post-download validation failure).
  - `tests/specs/unit/DownloadModelTaskSpec.bx` — 4 specs covering task instantiation, `list()` enumerating the registry, `run()` happy path (assets already present), and `run()` error path (manual mode + missing files).
  - `tests/specs/unit/OnnxNerDetectorSpec.bx` expanded — `validateAssets` new shape, SupportedModels auto-population (Piiranha end-to-end via mock session), full convenience-API coverage, label-gated `real-model` integration spec.

### Changed

- `OnnxNerDetector.validateAssets()` return shape changed from `{ loaded, modelPath, tokenizerPath, loadError }` (session-only) to `{ ok, assetManager: { ... }, session: { ... } }`. Host apps surfacing this struct will need to update their readers.
- Default `assetMode` remains `"manual"` (no surprise boot-time downloads). Host apps opt into `"auto-download"` or `"shared-cache"` explicitly.
- Module settings now include `acceptUnverified`, `acceptUnverifiedLicense`, `modelSource`, `tokenizerSource`, `modelChecksum`, `tokenizerChecksum` — all with safe defaults.

### Known limitations

- Registry entries ship with `verified: false` (no pinned SHA-256) because the maintainer has not personally downloaded + hashed them. Use `manual` mode or set `acceptUnverified: true` until the maintainer pins checksums.
- `piiranha-pii-v1` ships with `licenseStatus: "unverified"`. The auto-download gate refuses it by default. Resolve with the iiiorg maintainers before flipping the status.
- `gliner-pii-v1` uses a span-prediction output format, not BIO. The current `EntityDecoder` handles BIO correctly but would need a GLiNER-specific adapter for full entity recovery. A note captures this in `DEV-NOTES/discovery-log.md`. Piiranha's BIO output decodes correctly as-is.
- No JAR asset manager yet. `lib/` population is still manual (three documented paths). A `JarAssetManager` mirroring `ModelAssetManager` is a candidate for a future release.
- Integration spec `real-model` skips silently when assets aren't present. Run with JARs + model in place via `box testbox run labels=real-model` to exercise the real pipeline.

## [0.2.0-pre] - 2026-04-18

Real ONNX Runtime + HuggingFace tokenizer wiring. `scan()` now drives the full tokenize → inference → decode pipeline when JARs and a model are present; degrades gracefully to empty when they aren't. Model asset management (download / validate / `SupportedModels.bx`) still lands in Phase 4.

### Added

- [`cbjavaloader`](https://forgebox.io/view/cbjavaloader) as a runtime dependency. Declared in `box.json` (`box install` pulls it) and in `ModuleConfig.this.dependencies` (ColdBox loads it before us).
- `ModuleConfig.onLoad()` — resolves `loader@cbjavaloader` via WireBox, calls `appendPaths( modulePath/lib )`, and logs the outcome to the `bx-aisentinel-onnx` LogBox channel. Missing `lib/` is logged and skipped (no crash).
- `models/OnnxSession.bx` — thread-safe singleton holding one `OrtEnvironment` + `OrtSession` + HuggingFace tokenizer. Lazy `ensureLoaded()` with cached `loadError`; fast `isReady()`; `tokenize(text)` returns `{ inputIds, attentionMask, offsetMapping }`; `runInference(ids, mask)` returns raw logits; `getStatus()` for health checks.
- `models/EntityDecoder.bx` — pure-BoxLang BIO-tag decoder. Argmax per token, B-LABEL opens / I-LABEL extends / O closes, skips sub-word tokens with `[0,0]` offsets, drops below-confidence-floor spans, drops unmapped entities, configurable category / priority / source / `confidenceFloor`.
- `models/OnnxNerDetector.bx` rewired to drive the full pipeline: `ensureLoaded` → `tokenize` → `runInference` → `decode`. Accepts session / decoder / id2label / labelToSentinel via constructor for testability. Gracefully returns `[]` when session isn't ready or label map isn't configured. `validateAssets()` surfaces the session's status struct.
- Default `labelToSentinel` covering common NER entity types: `PER` / `LOC` / `EMAIL` / `PHONE` / `SSN` / `CREDIT_CARD` / `DOB` / `PASSPORT_NUMBER` / `MEDICAL_RECORD_NUMBER`. Phase 4 replaces this with model-specific maps in `SupportedModels.bx`.
- `lib/` directory (gitignored except for `lib/README.md`) with placement instructions for the ONNX Runtime + DJL HuggingFace tokenizer JARs. Three documented install paths: CommandBox Maven install, manual download, or shared system directory.
- `tests/specs/fixtures/MockOnnxSession.bx` — stand-in for `OnnxSession` that lets specs drive the full scan pipeline without JARs or a real model.
- `tests/specs/unit/EntityDecoderSpec.bx` — 13 specs covering empty input, all-O sequences, single-token spans, multi-token merging, O-break semantics, mismatched I-LABEL handling, consecutive B- spans, sub-word `[0,0]` skipping, unmapped-entity dropping, confidence-floor filtering, multi-hyphen entity names (`B-CREDIT-CARD`), end-of-sequence span flushing, and decoder-config propagation.
- `tests/specs/unit/OnnxNerDetectorSpec.bx` expanded — contract surface, no-assets graceful-empty behavior, `validateAssets` diagnostics, full pipeline via `MockOnnxSession`, and the empty-id2label-returns-empty guard.

### Changed

- `OnnxNerDetector` no longer returns a hardcoded empty array. Behavior is conditional on session + label-map readiness.

### Known limitations

- Still pre-functional for real text: Phase 4 delivers `ModelAssetManager`, `SupportedModels.bx`, real GLiNER PII v1 integration, and the `box run-script download-model` task.
- Java dep management is manual. `JarAssetManager` (parallel to `ModelAssetManager`) may be added in Phase 4 to automate `lib/` population.
- No `ColdFusion` fallback. The module targets BoxLang 1.8.0+ because cbjavaloader's native BoxLang class-loading path requires it.

## [0.1.0-pre] - 2026-04-18

Initial scaffolding. Module resolves and satisfies the [`IDetector@1.0.0`](https://github.com/mrigsby/bx-AISentinel/blob/main/models/detectors/CONTRACT.md) contract from `bx-AISentinel` but returns empty results. Real ONNX Runtime wiring and GLiNER PII integration land in subsequent phases.

### Added

- `box.json` + `ModuleConfig.bx` declaring the module to ColdBox, with `cfmapping = "bxAiSentinelOnnx"` and explicit `this.name = "bx-AISentinel-ONNX"` so the WireBox ID is `OnnxNerDetector@bx-AISentinel-ONNX` regardless of install-folder casing.
- Minimal `models/OnnxNerDetector.bx` — duck-types the `IDetector@1.0.0` contract (exposes `scan`, `getPriority`, `getSourceName`, `getContractVersion`). Returns an empty hit array. Enough to prove the sentinel's external-detector resolution path works cross-module.
- `tests/` harness on port 8083 with a smoke spec verifying the detector loads and satisfies the contract.
- `DEV-NOTES/` (gitignored) with `discovery-log.md` seeded to capture findings that will apply when the DJL sibling module is eventually built.
- Module settings defaults for `ModelAssetManager` (`modelPath`, `modelName`, `modelSource`, `modelChecksum`, `assetMode`, `eagerInit`, `downloadTimeoutMs`) — placeholders consumed in Phase 4.

### Known limitations

- No ONNX Runtime integration yet. `scan()` always returns `[]`.
- No model assets. `ModelAssetManager` class, download/validate logic, `box run-script download-model` task, and real entity decoding all land in Phase 4.
- Java dependency wiring for the ONNX Runtime Java binding is deferred — the Phase 3/4 work decides whether to use a module-local `lib/` folder, CommandBox Maven resolution, or a host-app requirement pattern.
