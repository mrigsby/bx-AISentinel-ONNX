# Changelog

All notable changes to `bx-AISentinel-ONNX` are documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

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
