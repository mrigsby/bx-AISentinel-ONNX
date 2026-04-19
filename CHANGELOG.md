# Changelog

All notable changes to `bx-AISentinel-ONNX` are documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [0.4.2-pre] - 2026-04-19

Real-world host-app fix. The first end-user test of v0.4.1-pre against a fresh demo install hit a silent path-resolution bug — the detector reported "Tier 1: active" in the UI, but `OnnxSession.ensureLoaded()` failed with `modelPath missing or file does not exist: model.onnx` because `_readColdBoxModuleSettings()` couldn't find the host's configuration.

### Fixed

- **`OnnxNerDetector._readColdBoxModuleSettings()` now accepts both config shapes**, top-level keys win:
  - **`modules.bx-aisentinel-onnx.<key>`** (top-level) — what the testing-path doc and demo help page show, and what the core `bx-AISentinel` module already uses for its own settings. This is the recommended form.
  - **`modules.bx-aisentinel-onnx.settings.<key>`** (nested under `.settings`) — the strict ColdBox convention some hosts default to.

  Prior version required the nested `.settings` wrapper exclusively, so hosts following our docs got an empty settings struct, fell back to ModelAssetManager's empty-string default for `modelPath`, and `_joinPath("", "model.onnx")` returned just `"model.onnx"` — which `fileExists()` resolved relative to the JVM cwd, found nothing, and silently returned `[]` from every `scan()` call. The misleading "active" UI status comes from the detector LOADING successfully (passing the contract gate); only the session load failed, and that's what `validateAssets().sessionLoaded` is for. Tests didn't catch this because the test harness injects settings directly via `arguments.settings`, bypassing the ColdBox-context lookup entirely.

  Detection of which shape to use looks for any of our known top-level keys (`modelName`, `modelPath`, `modelSource`, `tokenizerSource`, `modelChecksum`, `tokenizerChecksum`, `assetMode`, `eagerInit`, `downloadTimeoutMs`, `acceptUnverified`, `acceptUnverifiedLicense`, `confidenceFloor`, `id2label`, `labelToSentinel`); if any are present at the top level, the entry is treated as the settings struct directly. Otherwise falls back to the `.settings` sub-key.

### Compatibility

Pure additive fix — hosts using the documented top-level form now work; hosts using `.settings` still work.

## [0.4.1-pre] - 2026-04-19

Doc + identifier fix. Every previously-documented WireBox ID for this module was wrong — `OnnxNerDetector@bx-AISentinel-ONNX` does not resolve because WireBox's `Class@Module` parser uses `this.cfmapping`, not `this.name`, and the dash characters in `this.name` were not what controlled lookup. The README, demo help page, testing-path doc, and integration spec comments all said the dashed form. Empirically confirmed by a host attempting to wire the detector via the documented form, watching it fail, and watching the cfmapping form succeed. Fixed across all three repos.

### Changed

- **`this.cfmapping` renamed `bxAiSentinelOnnx` → `bxAISentinelONNX`** (matching `this.modelNamespace` for unity, and treating `AI` / `ONNX` as acronyms — what most readers expect). The new form is the canonical WireBox suffix going forward.
- **`this.modelNamespace` renamed `bxaisentinelonnx` → `bxAISentinelONNX`** to match the cfmapping casing exactly. Single identifier across both fields removes the prior 3-way casing split (`bxAiSentinelOnnx` / `bxaisentinelonnx` / `bx-AISentinel-ONNX`).
- **WireBox resolution ID is now `OnnxNerDetector@bxAISentinelONNX`.** Hosts using `boxlang.json → modules.bx-aisentinel.externalDetectors` must use this form. The dashed `bx-AISentinel-ONNX` form was never working — there are no hosts to migrate.
- **README, lib/README, CHANGELOG, in-module comments** all updated to use the correct cfmapping form.
- **Test specs updated** to import via `new bxAISentinelONNX.models.X()` (was `bxAiSentinelOnnx`). Test harness `Application.bx` mapping renamed accordingly.
- **`ModelAssetManager` exception types** renamed (`bxAiSentinelOnnx.MissingAssets` → `bxAISentinelONNX.MissingAssets`, etc.). Callers using `catch ( bxAiSentinelOnnx.X )` need to update — but again, no documented host did this; tests were the only callers.
- **DEV-NOTES `Cross-module resolution` entry** populated with the empirical lesson so the next sibling module (GLiNER, DJL) inherits it on day one.

### Compatibility

This is a breaking change for the documented WireBox ID surface, but the documented form was non-functional. Hosts that already had Tier 1 wired-up successfully were using a workaround for the dashed-form failure; check `boxlang.json` and update to `OnnxNerDetector@bxAISentinelONNX`. `this.name` (`bx-AISentinel-ONNX`) is unchanged — install paths, repo names, and box.json `name` / `slug` stay aligned with the GitHub repo.

## [0.4.0-pre] - 2026-04-19

First release with the full pipeline verified end-to-end against a real model. The `v0.3.x` line declared itself "functional with real assets" — but no one had actually run a real-model integration against it. This release is the product of doing that, which surfaced eight real bugs across `OnnxSession`, `OnnxNerDetector`, `EntityDecoder`, and `SupportedModels`. Every one is fixed; every fix is covered by a unit spec; the `real-model` gated spec asserts structural correctness of entity labels, values, and positions rather than just `arrayLen > 0`.

### Fixed

- **`OnnxSession.tokenize`** — DJL's `Encoding.getCharTokenSpans()` returns `null` `CharSpan` entries for special tokens (CLS, SEP, padding, added tokens) that don't correspond to source-text characters. The prior code called `.getStart()` unconditionally, causing an NPE on every real-model scan. Null entries now map to the `[0,0]` sentinel the decoder skips.
- **`OnnxSession.ensureLoaded`** — `HuggingFaceTokenizer.newInstance(String)` treats its String argument as a HuggingFace model identifier and tries to download it from `huggingface.co`. For local files you MUST pass a `java.nio.file.Path` to hit the disk-loading overload. Prior code always took the network path for any configured `tokenizerPath`, failing with `404` errors referencing a URL cobbled together from the local path string.
- **`OnnxSession.runInference`** — `OrtSession.run(Map<String, OnnxTensor>)` expects native `String` keys. A BoxLang struct literal `{ "k": v }` is keyed by `ortus.boxlang.runtime.scopes.Key`, so ONNX Runtime's internal iteration fails with `ClassCastException: ortus.boxlang.runtime.scopes.Key cannot be cast to java.lang.String`. Inputs are now built as a real `java.util.HashMap` with `javaCast("string", ...)` keys.
- **`OnnxNerDetector._normalizeLogits`** — ONNX NER models return logits with shape `[batch, seqLen, numLabels]`. Single-sentence inputs always have `batch=1`, but the dimension is still there. Prior code did not peel batch, so downstream argmax saw each row as `float[][]` and crashed with `Can't compare [float[]] against [float[]]`. Batch dimension is now peeled explicitly.
- **`EntityDecoder` state machine** — now supports pure-IO tagging (no B- prefix). Piiranha (verified via its `config.json`) uses `I-LABEL` exclusively for entities; prior BIO-strict decoder never opened a span and returned zero hits for any pure-IO model. The decoder now treats `I-LABEL` as opening a new span when no matching span is open, preserving existing BIO semantics (B- always opens, matching I- extends).
- **`EntityDecoder._flushHit` offset translation** — DJL/HuggingFace emits 0-based char offsets with exclusive end; CFML/BoxLang `mid(string, start, count)` is 1-based with inclusive count. Prior code passed DJL offsets straight through, producing hit values that were shifted left by one character (`"Alice "` instead of `"Alice"`, `"e Martinez"` instead of `" Martinez"`). Offsets now translate at the decoder boundary so consumers see consistent 1-based positions.
- **`EntityDecoder._flushHit` whitespace trimming** — SentencePiece-based tokenizers (mdeberta-v3, XLM-R, etc.) encode word boundaries as part of the token, so non-first-content tokens arrive with leading whitespace in the offset. Prior code kept the whitespace in the hit value, which would have produced `"lives at⟦TOKEN⟧"` with no separator when the sentinel redacts. Leading + trailing whitespace is now trimmed and start/end offsets adjust to match the trimmed bounds.
- **`EntityDecoder._argmax` confidence scoring** — ONNX outputs raw logits, not probabilities. A `confidenceFloor` of `0.5` compared against raw logits is a no-op (logits routinely sit at 5+ to 13+ for confident predictions), so prior versions effectively had no confidence filtering. `_argmax` now applies numerically-stable row-wise softmax and returns the winner's probability (0–1) as `confidence`.
- **`SupportedModels` Piiranha registry entry** — completely rewritten against the model's actual `config.json`. Prior entry assumed BIO tagging with 19 labels and `O` at index 0; reality is pure-IO with 18 labels and `O` at index 17. Every other label index was off — idx 17, which we thought was `B-ZIPCODE`, is actually `O`. Result was that every non-entity token the model correctly predicted as `O` got decoded as `PII_ADDRESS` with high confidence. Registry entry now matches the model card; `labelToSentinel` expanded to cover all 17 entity types the model recognizes (added `ACCOUNTNUM`, `BUILDINGNUM`, `CREDITCARDNUMBER`, `DRIVERLICENSENUM`, `IDCARDNUM`, `PASSWORD`, `TAXNUM`, `USERNAME`). `decoderStrategy` field changed from `"bio"` to `"io"`.

### Changed

- **`OnnxNerDetector.validateAssets()` return shape** — `ok` no longer requires `session.loaded == true`. The prior behavior made `ok` structurally `false` for any fresh detector until something else triggered `ensureLoaded()` (sessions are lazy-init by default). New `ok` means "assets present + paths valid + label map populated" — i.e. "the detector CAN load on first scan." Two new fields surface the orthogonal state: `sessionLoaded` (current lifecycle state) and `lastScanError` (last error caught by `scan()`, empty when clean). `validateAssets()` itself has NO side effects — it never triggers a model load, so health endpoints calling it don't absorb 5-second cold starts. Hosts wanting forced load at construction pass `eagerInit: true`.
- **`OnnxNerDetector.scan()` error handling** — the `try/catch` still swallows exceptions (graceful degradation is the documented contract), but the caught error is now stashed on `lastScanError` and exposed via the new `getLastScanError()` method. Successful scans clear the cached error. Callers that previously couldn't distinguish "no PII detected" from "detector crashed silently" can now see exactly what went wrong. Log channel output is unchanged.
- **DJL JAR dependency set** — install instructions now document THREE required JARs in `lib/` instead of two. The `ai.djl.huggingface:tokenizers` extension JAR is not self-contained; it depends on `ai.djl:api` for the base `Tokenizer` interface. Without `api-*.jar`, the JVM fails with `NoClassDefFoundError: ai/djl/modality/nlp/preprocess/Tokenizer` at first tokenizer construction. `lib/README.md` and main README updated.

### Added

- **`OnnxNerDetector.getLastScanError()`** — public getter for the cached error state described above. Returns `""` when no scan has run or the most recent scan succeeded.
- **`EntityDecoderSpec`** — new coverage for IO-tagging (`I-LABEL` as opener + adjacent-same-label merge) and SentencePiece leading-whitespace trim. Existing BIO specs retained and updated to the new 0-based-exclusive offset convention + raw-logit values.
- **`OnnxNerDetectorSpec`** — new assertions on `validateAssets()` v0.4.0+ shape (`sessionLoaded`, `lastScanError`) and a "validateAssets has no side effects" spec that locks in the lazy-init contract. Existing full-pipeline mock spec updated to the new conventions.
- **`MockOnnxSession.runInference`** now wraps logits in a batch dimension so mock fixtures match real ONNX output shape. Spec authors still construct clean 2-D `[seqLen][numLabels]` arrays; the wrap is invisible.
- **DEV-NOTES discovery log** enriched with the full lessons from this debugging arc — tokenizer quirks, Java↔BoxLang interop gotchas, ONNX shape conventions, TestBox fixture patterns, and a chronological session summary. File is gitignored but persists for future maintainers and the eventual DJL sibling module.

### Removed

- **`OnnxNerDetector._diagnoseScan`** and its private helpers (`_firstN`, `_lastN`, `_allOffsetsZero`, `_argmaxRows`) — introduced temporarily during the v0.3.3→v0.4.0 debugging arc to surface intermediate pipeline state when `scan()`'s `try/catch` was swallowing exceptions. With the `lastScanError` stash in place, the diagnostic method is no longer needed. Removed to keep the module surface tight. If a future debug session needs similar introspection, re-add it as scoped temporary code rather than shipping it permanently.

### Migration notes for v0.3.x hosts

- Hosts that read `validateAssets().ok` to mean "detector is fully loaded and ready" should switch to checking `sessionLoaded` for that semantic. The new `ok` means assets + config are in order — a weaker but more useful guarantee for health checks.
- Hosts that encountered `scan() → []` with no way to distinguish "no matches" from "caught exception" should now inspect `getLastScanError()` or `validateAssets().lastScanError`.
- Hosts that downloaded only `onnxruntime-*.jar` + `tokenizers-*.jar` into `lib/` need to also place `api-*.jar` (matching the `tokenizers` version). Without it, the first scan fails with `NoClassDefFoundError` even when the registered paths look fine.

## [0.3.3-pre] - 2026-04-18

### Fixed

- **`OnnxNerDetector` now reads its own ColdBox module settings from `.boxlang.json → modules.bx-aisentinel-onnx.settings`.** Previously, `init()` only looked at `arguments.settings` (what the sentinel passed via `externalDetectorOptions`). The README and demo help page both documented the `modules.bx-aisentinel-onnx` config pattern, but nothing actually read from there — a fresh install following the docs produced a detector with no `modelPath` / `modelName`, `OnnxSession.ensureLoaded()` failed the `fileExists` check silently, and `scan()` returned `[]` with no explanation. Detector now pulls module settings as defaults and merges caller-supplied `arguments.settings` on top.
- **`OnnxSession.ensureLoaded()` now logs to the `bx-aisentinel-onnx` channel when the modelPath / tokenizerPath guard rejects a call.** Silent early-returns previously left operators with no signal — the detector would report `loaded` in one breath via the plugin seam and then return empty hits in the next. Log fires once per process per failure to avoid spamming on every `scan()` call.

### Known behavior unchanged

- The sentinel's `externalDetectorOptions[id]` still takes precedence over module settings — that's how tests + advanced hosts inject overrides. Only the default lookup path changed.

## [0.3.2-pre] - 2026-04-18

### Fixed

- **Model download URLs** in `SupportedModels.bx` and the README pointed at `urchade/gliner_multi_pii-v1`, which ships only PyTorch weights (no `onnx/model.onnx`, no `tokenizer.json`). HuggingFace returned `Entry not found` on every curl attempt. Fixed: URLs now point at `onnx-community/gliner_multi_pii-v1` which hosts the actual ONNX export (fp32, fp16, int8, q4, and several other quantized variants) plus the tokenizer. License + model-card links still reference the original author's repo (provenance unchanged).

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
