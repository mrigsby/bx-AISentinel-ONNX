# Changelog

All notable changes to `bx-AISentinel-ONNX` are documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

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
